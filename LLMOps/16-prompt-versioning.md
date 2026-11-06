# 16 - Prompt 版本化：Prompt 也是代码

> 核心论点：Prompt 是你系统中**变更最频繁、影响面最大、却最没有工程保障**的资产。
> 一次「顺手改两个字」可能导致质量暴跌或成本翻倍。本篇把 Prompt 纳入和代码同等的工程管理。

---

## 1. 为什么 Prompt 必须版本化

| 没有版本化的典型事故 | 根因 |
|---|---|
| 线上质量突然下降，没人知道为什么 | 上周有人改了 system prompt 里的一个词 |
| 想回滚旧版本，找不到旧版内容 | Prompt 散落在代码注释、Confluence、某人电脑里 |
| 同一功能在不同环境表现不一致 | dev 用 A 版本、prod 用 B 版本 |
| 评估集跑出的结果无法复现 | 不知道当时用的是哪个 prompt 版本 |
| 多人协作互相覆盖 | 没有唯一可信源（Single Source of Truth） |

**原则：Prompt 的变更历史 = 系统行为的变更历史。** 没有版本化，可观测性（20 章）和评估（25 章）都失去意义。

---

## 2. 存储结构：Prompt 放在哪

### 2.1 三种方案对比

| 方案 | 做法 | 优点 | 缺点 | 适用 |
|---|---|---|---|---|
| **A. 代码仓库** | prompt 作为模板文件（`prompts/*.yaml`）进 Git | 有版本史、可 review、零额外设施 | 改动需走发版流程，无法热更 | 团队小、变更少 |
| **B. 数据库/配置中心** | prompt 存 DB 或 Nacos/Apollo，带版本表 | 热更新、可灰度、可回滚 | 需要自己建变更管理与 review 机制 | 团队大、高频调优 |
| **C. Prompt 平台** | Langfuse Prompt Management、LangSmith、PromptLayer 等 | 自带版本、灰度、标注、与 trace 打通 | 供应商锁定、成本、私有化部署问题 | 快速起步 |

**推荐落地：B 方案为主，Git 为备份。** DB 中的每条 prompt 记录都对应 Git 中的一个 yaml 文件（内容一致），DB 是运行时唯一可信源，Git 承担 review 与审计。

### 2.2 推荐的存储结构（DB 表设计）

```sql
CREATE TABLE prompt_template (
  id            BIGINT PRIMARY KEY,
  prompt_key    VARCHAR(128) NOT NULL,   -- 逻辑名，如 "客服工单分类"
  version       INT NOT NULL,            -- 递增版本号
  status        VARCHAR(16) NOT NULL,    -- draft / testing / active / deprecated
  messages      JSON NOT NULL,           -- 完整消息数组（含 system/user 结构）
  model_tier    VARCHAR(64),             -- 推荐/绑定的模型档位（配合 04 网关 tier 路由）
  params        JSON,                    -- 默认 temperature / max_tokens
  variables     JSON,                    -- 变量声明 [{"name":"ticket","required":true}]
  created_by    VARCHAR(64),
  created_at    DATETIME,
  activated_at  DATETIME,                -- 何时转为 active
  UNIQUE KEY uk_key_version (prompt_key, version)
);
```

要点：

- **prompt_key 是稳定标识**，代码中永远只引用 key，不引用具体内容。
- **version 单调递增，永不覆盖修改**。要改就加新版本，旧版本永远可回滚。
- `status` 控制生命周期，同一 key 只有一个 `active` 版本（灰度例外，见 17 章）。

### 2.3 模板文件格式（Git 侧）

```yaml
# prompts/support-ticket-classifier.yaml
key: support-ticket-classifier
version: 7
status: testing
model_tier: mini
params:
  temperature: 0
  max_tokens: 200
variables:
  - name: ticket_title
    required: true
  - name: ticket_body
    required: true
messages:
  - role: system
    content: |
      你是工单分类器。只输出 JSON：{"category": "...", "priority": "P0|P1|P2|P3"}
      分类只允许：billing / technical / account / other
      P0 判定标准：影响多用户的服务不可用、资金安全。
  - role: user
    content: |
      标题：{{ticket_title}}
      正文：{{ticket_body}}
changelog: |
  v7: P0 判定标准增加"资金安全"（修复把退款咨询误判 P0 的问题）
  v6: 初始结构化输出 JSON 化
```

`changelog` 是强制的：**每次变更必须写清动机**，否则三个月后没人敢动这个文件。

---

## 3. 变量与渲染：模板引擎的选择

### 3.1 渲染规则

```python
import re

def render(template: str, variables: dict) -> str:
    # 1. 严格模式：未提供的变量直接报错，而不是渲染成空字符串
    placeholders = set(re.findall(r"\{\{(\w+)\}\}", template))
    missing = placeholders - set(variables.keys())
    if missing:
        raise ValueError(f"缺少变量: {missing} (prompt_key=v)")
    # 2. 不做隐式 str() 转换：None 渲染进 prompt 是常见事故源
    for k, v in variables.items():
        if v is None:
            raise ValueError(f"变量 {k} 为 None")
        template = template.replace("{{" + k + "}}", str(v))
    return template
```

### 3.2 变量设计原则

| 原则 | 反例 | 正例 |
|---|---|---|
| 用户内容永远作为变量，不拼进模板代码 | `f"总结这段话：{user_input}"` 写死在代码里 | 模板 `{{content}}`，代码传入 |
| 传入前清洗：剥 control 字符、截断超长 | 直接透传 50KB 粘贴内容 | 按 prompt 用途设上限并截断 |
| 结构化数据用定界符包裹 | 直接把用户输入夹在指令中间 | `<ticket>...</ticket>` 包裹，防注入 |
| 禁止变量内容里出现指令性文本时无隔离 | — | 见 27 章注入防护 |

### 3.3 渲染结果必须可追溯

**评估复现的前提：记录「模板 key + version + 变量值 + 渲染后全文」。** 21 章的 trace 落库会用到。变量值可能含敏感信息，落库前走 05 章的脱敏。

---

## 4. 变更管理流程

### 4.1 变更分级

| 级别 | 典型场景 | 流程要求 |
|---|---|---|
| 轻微 | 错别字、格式微调，不影响语义 | 1 人 review，直接生效，跑冒烟集 |
| 普通 | 调整指令措辞、增删输出字段 | 1 人 review + 全量 evals 通过 |
| 重大 | 换输出格式（影响下游解析）、换模型档位 | 2 人 review + evals + 灰度发布（17 章） |

### 4.2 标准变更流程（方案 B）

```
1. 在 Git 中修改 yaml，提 PR（diff 一目了然）
2. CI 自动渲染若干样例，diff 渲染结果（见 17 章）
3. review 通过后合并 → 流水线把新版本写入 DB（status=draft）
4. 对 draft 版本跑离线 evals（25 章），达标才置 testing
5. testing 版本灰度放量 → 数据达标置 active，旧版本自动置 deprecated
6. deprecated 版本保留至少 90 天，随时可一键切回
```

### 4.3 谁可以改 Prompt

- **产品/运营可以直接起草**（这是 prompt 相比代码的优势：非工程师也能调优），但**生效必须走评审**。
- 建议在 DB 层实现「起草-评审-生效」双人机制，避免「今天心情好改一下」。
- 每次 active 版本切换都产生审计事件（谁、何时、从哪版到哪版、evals 结果链接）。

---

## 5. Prompt 与代码的版本对齐

一个容易被忽视的问题：**代码依赖 prompt 的输出结构，prompt 改格式 = 接口变更。**

```
prompt v6 输出: {"category": "...", "priority": "..."}
prompt v7 输出: {"category": "...", "priority": "...", "tags": [...]}
                        ↑ 新增字段 OK          ↑ 删除字段 = breaking change
```

规则：

1. **输出结构变更视为 breaking change**，必须与解析代码同版本发布，或在解析层做兼容（新旧字段都认）。
2. 解析层对未知字段宽容、对缺失字段报错（fail-fast 而非静默兜底）。
3. prompt 的 `version` 与应用发布单关联：发布记录里写清「本版本依赖 prompt_key@v7」。
4. **回滚应用代码时必须同时回滚 prompt**——两者单独回滚是最常见的翻车点。用 05 章配置热更把 prompt 回滚做成秒级操作。

---

## 6. 结构化 Prompt 的组织策略

当 prompt 超过几百行，需要拆分组装而非一整块：

```python
# 组装式 prompt：模块化片段 + 运行时拼接
segments = {
    "base":      load_segment("support/base"),          # 角色+总规则，很少变
    "policy":    load_segment("support/policy@2026-10"),# 业务政策，随版本更新
    "fewshot":   load_segment("support/fewshot/v3"),    # 示例，独立演进
    "dynamic":   render(ticket_context),                 # 本次工单上下文
}
messages = assemble(segments, order=["base", "policy", "fewshot", "dynamic"])
```

好处：

- **政策类内容**（如退款规则）单独版本化，运营改政策不用动核心 prompt。
- **few-shot 示例独立演进**：线上发现 badcase 后，把修正示例加入 fewshot，是最高频的迭代手段。
- 与 08 章前缀缓存配合：**把稳定的片段放前面**（base/policy/fewshot 不变 → 命中供应商前缀缓存省钱），动态内容放后面。

注意：片段各自版本化后，**最终生效的 prompt = 片段版本的组合**。trace 里记录组合快照（或片段 key@version 列表），否则无法复现。

---

## 7. 常见坑

| 坑 | 后果 | 对策 |
|---|---|---|
| Prompt 写死在代码字符串里 | 改一个词要发版，没人走流程 | 全部外置为模板（key 引用） |
| 版本覆盖式更新（update 而非 insert） | 旧版本丢失，无法回滚与归因 | append-only，永不覆盖 |
| 变量缺失渲染成空字符串 | prompt 语义悄悄变化，模型胡说 | 严格模式渲染，缺失即报错 |
| 改了 prompt 忘了改下游解析 | 大量解析失败或字段静默丢失 | 输出结构变更 = breaking change 流程 |
| 只记 prompt 不记变量值 | badcase 无法复现 | trace 记录渲染快照 |
| 多个服务各自复制 prompt 副本 | 副本漂移，行为不一致 | 唯一可信源 + 客户端拉取缓存 |
| fewshot 示例越堆越多 | 成本涨、旧示例与新规则矛盾 | 定期清理，保持 ≤5 个高代表性示例 |
|「临时改一下，明天改回来」 | 临时版成了永久版，无人知晓 | 一切变更走版本流程，没有临时 |

---

## 8. Checklist

- [ ] 所有 prompt 通过 `prompt_key` 引用，代码中零硬编码 prompt 文本
- [ ] DB 存储为唯一运行时可信源，append-only 版本表，Git 同步备份
- [ ] 每次变更强制填写 changelog（动机 + 期望效果）
- [ ] 渲染严格模式：缺变量 / None 直接报错
- [ ] 用户内容一律经变量传入，并用定界符包裹
- [ ] trace 记录 key + version + 变量值（脱敏）+ 渲染快照
- [ ] 输出结构变更按 breaking change 走评审，解析层兼容处理
- [ ] 应用回滚与 prompt 回滚绑定为同一动作
- [ ] 长 prompt 片段化，稳定片段前置以命中前缀缓存
- [ ] 变更审计事件完整（谁/何时/vN→vN+1/evals 链接）
