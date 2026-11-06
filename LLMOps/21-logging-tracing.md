# 21 - 日志规范与全链路追踪落库

> 20 章定义了结构，本篇讲落地：trace 记录哪些字段、日志怎么打、数据存哪里、
> 怎么在不爆存储的前提下把成本和质量算清楚。落库字段设计直接决定 22 章看板能不能做出来。

---

## 1. Trace 落库字段规范

### 1.1 LLM 调用 span 的必备字段（计量与归因的最小全集）

```json
{
  "trace_id": "t_20261105_8812_001",
  "session_id": "s_8812",
  "user_id": "u_1024",
  "tenant_id": "org_7",
  "feature": "support-assistant",          // 功能场景（多场景产品的归因维度）
  "experiment": null,                      // 17 章灰度实验标签
  "prompt_key": "support-ticket-classifier",
  "prompt_version": 7,
  "prompt_snapshot_sha": "a3f9c2...",      // 渲染后快照的哈希 + 全文引用
  "model_tier": "mini",                    // 04 网关 tier
  "model_id": "model-x-2026-03-01",        // 锁定的快照版本（18 章）
  "provider": "provider-a",
  "params": {"temperature": 0, "max_tokens": 800},
  "usage": {
    "input_tokens": 1210,
    "output_tokens": 384,
    "cached_input_tokens": 896,            // 08 章前缀缓存命中部分
    "reasoning_tokens": 0                  // 思维链模型的隐藏 token
  },
  "cost_cny": 0.0027,                      // 按计价表折算（见 1.3）
  "ttft_ms": 410,
  "total_ms": 1980,
  "status": "ok",                          // ok / error / timeout / degraded
  "degradation_level": null,               // 11 章降级级别（若走了降级）
  "retries": 0,
  "parse_ok": true,                        // 结构化输出解析是否成功
  "finish_reason": "stop"
}
```

### 1.2 字段清单的取舍原则

| 必须有 | 理由 | 可省略 |
|---|---|---|
| prompt_key@version | 成本/质量按 prompt 归因 | 完整 prompt 全文（存对象存储，span 存引用） |
| usage 全字段 | 06 章计量以此为准（API usage 优先） | 每次调用的候选 chunk 全文（存 chunk id + score） |
| tier + model_id | 18 章版本锁定的对账依据 | 输出全文（采样存储，见 1.4） |
| TTFT + total | 09 章延迟归因 | 网关内部中间状态 |

### 1.3 成本落库：落「金额」还是只落「token」？

推荐**两层都落**：

- span 落 `usage`（事实，永不因调价失效）；
- 汇总层按当日计价表折算 `cost`（调价后可重算，事实不改）。

计价表本身配置化（05 章热更），供应商调价当日生效：

```yaml
# pricing.yaml（provider 变更历史化，带生效区间）
provider-a:
  model-x-2026-03-01:
    - effective_from: 2026-03-01
      input_per_1k: 0.002   # 单位：元
      output_per_1k: 0.008
```

对账红线：**自己折算的成本与供应商账单差异 >2% 必须排查**（常见原因：漏计 cached/reasoning token、批量 API 折扣没算）。

### 1.4 大 payload 的分级存储

prompt 全文、输出全文、召回 chunk 全文都很占空间。分级策略：

| 数据 | 保留策略 |
|---|---|
| 结构化字段（usage/延迟/版本/状态） | 全量，保留 ≥180 天（月度/季度成本报表需要） |
| prompt 渲染快照 | 全量 30 天 → 采样 10% 保留 90 天 → badcase 关联的永久保留 |
| 输出全文 | 采样 5-10% + 所有低分/点踩会话强制保留 |
| 召回 chunk 详情 | 30 天，仅保留 id 与分数 |

badcase 永久保留机制：用户点踩/负反馈触发时，把该 trace 的 payload 打上 `retain=permanent` 标记——**评估集（25 章）的原料就来自这里**。

---

## 2. 日志规范

### 2.1 结构化日志（不要打印多行文本）

```
// ✅ 单行 JSON，字段与 trace 对齐
{"ts":"2026-11-05T10:23:01Z","level":"WARN","trace_id":"t_001",
 "event":"llm.retry","attempt":2,"reason":"rate_limit","backoff_ms":2000}

// ❌ 叙事式日志：无法聚合检索
"Retry #2 for chat request due to rate limit, waiting 2s..."
```

### 2.2 AI 专属事件清单（按事件名检索）

| 事件 | 级别 | 关键字段 |
|---|---|---|
| `llm.request` / `llm.response` | INFO | trace_id、tier、usage |
| `llm.retry` | WARN | attempt、reason |
| `llm.degradation` | WARN | level（11 章降级级别）、trigger |
| `llm.parse_fail` | ERROR | schema 错误详情、修复链动作（12 章 JSON 修复链） |
| `llm.refusal` | INFO | 拒答类型（模型自身拒答 vs 护栏拦截，27 章） |
| `gateway.fallback` | WARN | provider 切换（04 章） |
| `kb.sync_result` | WARN | 19 章同步报告摘要 |
| `budget.breach` | ERROR | 10 章预算熔断动作 |

事件名是跨服务检索的统一语言，写进团队规范，禁止各服务自造词。

### 2.3 日志与 trace 的关系

- 日志必须带 `trace_id`，任何日志可通过 trace_id 反查上下文（MDC/OTel baggage 自动注入）。
- **判定原则：能进 trace 结构化字段的，不要重复打日志**；日志只承载「过程事件」（重试、切换、拦截），trace 承载「结果事实」。

---

## 3. 技术实现（Spring / Java 参考）

```java
// LLM 调用封装处统一埋点（配合 04 章模型网关）
Observation obs = Observation.createNotStarted("llm.generate", registry)
    .lowCardinalityKeyValue("feature", feature)        // 低基数：进索引
    .lowCardinalityKeyValue("prompt_key_version", key + "@v" + version)
    .lowCardinalityKeyValue("model_tier", tier)
    .highCardinalityKeyValue("trace_id", traceId)      // 高基数：只进日志/trace 存储
    .start();
try {
    LlmResult r = gateway.invoke(req);
    recordUsage(traceId, r.usage(), pricing);          // 落计量表
    return r;
} finally {
    obs.stop();
}
```

要点：

- **低/高基数分离**：`prompt_key@version`、`tier`、`feature` 是低基数可进指标维度；`trace_id`、`user_id` 是高基数，只做检索键，不要做 Prometheus label（会爆基数）。
- 异步调用（15 章队列 worker）必须手工传递 trace context（OTel context propagation），否则异步链路断裂。
- 敏感字段（用户输入）在进入日志/trace 前统一走 05 章脱敏管道。

---

## 4. 存储与查询架构

```
应用 → OTel Collector / Kafka → ┬→ trace 存储（Tempo/Jaeger，短保留）
                                ├→ 计量明细表（ClickHouse/StarRocks，核心资产）
                                └→ 指标聚合（Prometheus/Micrometer → 看板 22 章）
```

| 存储 | 内容 | 保留 |
|---|---|---|
| 计量明细（最重要） | 1.1 字段全集，逐请求 | ≥180 天，冷热分层 |
| trace 存储 | 完整调用链 | 7-30 天 |
| 日志 | 事件日志 | 7-30 天（ERROR 可延长） |
| 对象存储 | prompt 快照、采样输出全文 | 按分级策略 |

**计量明细表是成本与质量分析的唯一事实源**，其余都是辅助。如果只能建一个存储，建它。

---

## 5. 常见坑

| 坑 | 后果 | 对策 |
|---|---|---|
| 只落 token 不落金额，调价后无法追溯 | 历史成本报表作废 | usage（事实）+ 当日计价折算双层落库 |
| 计量以本地估算为准 | 与供应商账单对不上，预算熔断失真 | 以 API 返回 usage 为准 |
| 漏计 cached/reasoning token | 成本低估、对账差异 | usage 全字段拆解 |
| trace_id/user_id 塞进 Prometheus label | 时间序列爆炸，监控系统 OOM | 低/高基数分离 |
| 异步任务丢 trace 上下文 | 队列链路无法追踪 | 手工 context 传递 |
| 负面反馈数据没有保留策略 | 评估集永远缺原料 | badcase 永久保留标记 |
| 日志打印完整用户输入 | 合规风险 + 存储爆炸 | 脱敏 + 采样 + 引用而非全文 |
| 自家成本与账单差异从不核对 | 计费 bug 潜伏数月 | 每日/每周自动对账，差异 >2% 告警 |

---

## 6. Checklist

- [ ] LLM span 字段全集落库（1.1 清单），计量明细表独立存储保留 ≥180 天
- [ ] usage 以 API 返回为准，cached/reasoning token 单列
- [ ] 成本双层落库：token 事实 + 当日计价折算，计价表热更且历史化
- [ ] 每日与供应商账单对账，差异 >2% 自动告警
- [ ] 大 payload 分级存储：快照 30 天→采样，badcase 永久保留
- [ ] 统一事件名规范（llm.retry/degradation/parse_fail/refusal...），全服务对齐
- [ ] 日志结构化单行 JSON，携带 trace_id，用户内容脱敏
- [ ] 异步链路手工传递 trace context
- [ ] 低/高基数分离，监控指标维度只用低基数
- [ ] 负反馈自动触发 trace 保留标记，打通评估集原料通道
