# 17 - Prompt 的 CI/CD：灰度与一键回滚

> 16 章解决了「prompt 存在哪」，本篇解决「prompt 怎么安全上线」：
> 变更自动化验证 → 灰度放量 → 秒级回滚。目标是让改 prompt 的风险不高于改后端代码。

---

## 1. Prompt CI：提交时自动验证

### 1.1 流水线阶段

```
PR 提交 prompt 变更
  ├─ 1. 静态检查
  ├─ 2. 渲染快照 diff（人工确认意图）
  ├─ 3. 结构化输出 schema 校验
  ├─ 4. 冒烟集快速跑（~50 case，<3 分钟）
  └─ 5. 变更分级标签检查（重大变更需 2 人 approve）
合并后（nightly / 全量）
  └─ 6. 全量 evals（25 章），报告附在 PR
```

### 1.2 静态检查项（便宜但能拦住大部分事故）

```yaml
# ci/prompt-lint.yaml
checks:
  - id: variables-declared        # 模板中的 {{var}} 都在 variables 里声明
  - id: no-hardcoded-user-data    # 模板中不含真实用户样例数据（脱敏遗漏）
  - id: output-schema-present     # 声明了结构化输出的必须有 schema
  - id: forbidden-phrases         # 检查已知的坑词（如「请勿提及」类双关指令）
  - id: token-budget              # 渲染后最长样例不超过 max_tokens 预算的 80%
  - id: changelog-updated         # changelog 有新增条目
  - id: fewshot-count             # fewshot 片段示例数 ≤ 5
```

### 1.3 渲染快照 diff：让 review 看到「模型实际看到什么」

review prompt 源文件 diff 不够，**必须 diff 渲染后的结果**：

```python
# CI 步骤：对固定样例集渲染新旧两版，输出 diff
SAMPLES = load_samples("ci/samples/ticket-classifier/")  # 10 个典型输入

for name, sample_vars in SAMPLES:
    old = render(prompt_v6, sample_vars)
    new = render(prompt_v7, sample_vars)
    emit_diff(name, old, new)   # 写入 PR 评论，reviewer 逐条确认
```

价值：能提前发现「变量位置移动导致指令被用户内容打断」「占位符被吃掉」这类源文件 diff 看不出的问题。

### 1.4 冒烟集 vs 全量集

| | 冒烟集 | 全量 evals |
|---|---|---|
| 规模 | 30-80 case | 数百~数千 |
| 构成 | 历史重大 badcase + 代表性场景 | 分层抽样全场景 + 边界 case |
| 时机 | 每次 PR（分钟级） | 每夜 + 发布前 |
| 阈值 | 关键指标不许倒退 | 全指标达标才可发布 |

冒烟集的第一批 case 从哪来：**上线后第一个月的线上 badcase，逐条转化为测试用例**。之后每周增补。

---

## 2. Prompt CD：灰度放量

### 2.1 状态机与放量节奏

```
draft ──evals通过──> testing ──灰度──> active（旧版置 deprecated）
                        │
                        └──指标不达标──> 回滚/终止
```

灰度不是拍脑袋切比例，按**暴露面从小到大**推进：

| 阶段 | 流量 | 观察时长 | 通过标准 |
|---|---|---|---|
| 内部试用 | 团队成员账号 | 1-2 天 | 无 crash、输出格式正确 |
| 金丝雀 | 1-5% 真实流量 | ≥1 天（覆盖流量高峰） | 质量指标 ≥ 旧版 - ε；成本/延迟无恶化 |
| 半量 | 50% | ≥2 天 | 同上，且按用户维度无差评率上升 |
| 全量 | 100% | 持续监控 1 周 | 稳定 |

**按用户维度分流**（同一用户固定在某一版本），不要按请求随机分——否则同一用户体验前后不一致，且 trace 无法按用户聚合对比。

### 2.2 灰度路由实现

配合 04 章模型网关或独立的 prompt 服务：

```python
def resolve_prompt(prompt_key: str, user_id: str) -> PromptVersion:
    exp = experiment_repo.get_active(prompt_key)   # e.g. {testing: v7, active: v6, ratio: 5}
    if exp is None:
        return get_active_version(prompt_key)
    bucket = stable_hash(f"{prompt_key}:{user_id}") % 100
    if bucket < exp.ratio:
        return get_version(prompt_key, exp.testing_version)   # 打上实验标签
    return get_active_version(prompt_key)
```

关键点：

- `stable_hash`：同一用户每次解析结果一致（可用 MurmurHash/MD5 取模）。
- trace 中记录 `prompt_key@version + experiment_id`，22 章看板才能做**分版本对比**。
- 比例调整是配置热更（05 章）操作：发现异常 5% → 0% 秒级止血。

### 2.3 灰度期看什么

不只是「没报错就行」，要对比新旧版本的：

| 维度 | 指标 | 说明 |
|---|---|---|
| 质量 | evals 代理指标 / 点赞点踩率 / 人工抽检 | 新版必须 ≥ 旧版 - 可接受噪声带 |
| 行为 | 拒答率、格式解析失败率、重试率 | 新 prompt 常见的隐性退化点 |
| 成本 | 平均输入/输出 token | 措辞变化可能导致输出变长 20% |
| 延迟 | TTFT、总时长 P50/P95 | 同上 |

---

## 3. 一键回滚

### 3.1 回滚的三种速度

| 手段 | 速度 | 适用 |
|---|---|---|
| 切回 active 版本指针 | 秒级 | 灰度中发现明确退化 |
| 发布新版（revert 提交走完整流程） | 小时级 | 已全量、需要留痕审计 |
| 紧急禁用该功能（L4 兜底，11 章） | 秒级 | prompt 问题导致安全事故/合规风险 |

**秒级回滚是设计出来的，不是临场发挥的**：active 指针必须支持热更切换，所有实例无本地持久缓存（或缓存 TTL ≤30s）。

### 3.2 回滚操作与验证

```bash
# 1. 切换（幂等操作，重复执行安全）
prompt-cli rollback support-ticket-classifier --to-version 6 --reason "INC-1234 解析失败率 8%"

# 2. 自动化验证回滚生效（不要靠肉眼）
prompt-cli verify support-ticket-classifier --expect-active 6 --max-cache-age 30s

# 3. 观察恢复
# 22 章看板确认解析失败率回落，trace 中新流量已带 v6 标签
```

### 3.3 回滚也要留痕

- 回滚触发一次版本变更事件（v7 → deprecated，原因 + 关联事故单）。
- **回滚的 prompt 版本要标记为「禁再发布」**，除非明确修复原因后重新以新版本号提交——防止有人把失败版本原样再发一遍。

---

## 4. Prompt A/B 实验（进阶）

灰度天然就是 A/B 框架，可以直接复用来做 prompt 实验：

| 要素 | 要求 |
|---|---|
| 假设 | 「在 fewshot 中加入 X 类示例，工单分类准确率提升 ≥3%」 |
| 分流 | 用户维度稳定分流（2.2 节），对照组 = 当前 active |
| 样本量 | 流量小时可用序贯检验，或延长观察期至覆盖周末效应 |
| 主指标 | 预先定义好（如解析成功率 + 人工抽检准确率），避免事后挑好看的 |
| 卫星指标 | 成本、延迟、拒答率——质量提升但成本翻倍可能不可接受 |
| 结束 | 达到结论即结束：胜出版转 active，失败版关闭实验并记录结论 |

注意：**prompt 的输出是概率性的**，同一输入多次运行结果不同。对比实验时两边都要控制温度（如 temp=0 或固定 seed），否则差异可能只是采样噪声。

---

## 5. 常见坑

| 坑 | 后果 | 对策 |
|---|---|---|
| 本地缓存 prompt 且 TTL 过长 | 回滚后部分实例仍用旧版，行为不一致 | 缓存 TTL ≤30s 或配置推送强制失效 |
| 按请求随机分流 | 用户视角体验跳变，数据无法按用户分析 | 用户维度稳定哈希分流 |
| 灰度只看错误率不看质量 | prompt 变「能跑但变笨」没人发现 | 灰度期带人工抽检 + 点赞点踩对比 |
| 冒烟集从不更新 | 新类型 badcase 反复出现拦不住 | 每周把线上 badcase 转入冒烟集 |
| 只在 dev 环境测 | dev 用小模型/低流量，掩盖 prod 问题 | 灰度直接在 prod 小流量验证 |
| 实验不下结论 | 半量灰度跑三个月，白烧钱也白冒风险 | 实验必须设结束条件与结论记录 |
| 温度不固定的对比实验 | 结论被采样噪声污染 | 实验中统一 temp/seed |
| 回滚无留痕 | 下次有人把同一失败版本再发一遍 | 回滚事件 + 禁再发布标记 |

---

## 6. Checklist

- [ ] PR 级 CI：静态检查 + 渲染 diff + schema 校验 + 冒烟集
- [ ] 冒烟集来自真实 badcase，每周增补
- [ ] 全量 evals 在发布前必跑，报告归档可追溯
- [ ] 灰度按 内部→金丝雀→半量→全量 推进，每阶段有通过标准与最短观察期
- [ ] 分流为用户维度稳定哈希，trace 带版本与实验标签
- [ ] 看板支持分版本对比（质量/成本/延迟三维度）
- [ ] active 指针热更可切换，秒级回滚 + 自动验证回滚生效
- [ ] 回滚留痕并标记禁再发布
- [ ] A/B 实验有预设假设、主指标与结束条件
- [ ] 所有版本切换/回滚操作有审计事件
