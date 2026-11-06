# 04 · 模型网关：多模型的统一入口

## 1. 为什么必须有网关

```
没有网关：每个服务直连各供应商 SDK
  → 日志散乱、限流各自为政、换模型要改 N 处、无法统计成本

有网关：所有模型调用走唯一入口
  → 统一路由/重试/熔断/缓存/日志/计量/降级
```

> 网关是 LLMOps 的地基。哪怕只是一个函数，也要先抽象出来。

## 2. 网关的核心职责

| 职责 | 说明 | 篇目 |
|---|---|---|
| **协议统一** | 把各家 API 统一成内部标准接口 | 本文 |
| **路由** | 按场景/成本/质量把请求分到不同模型 | 本文 |
| **重试与熔断** | 统一的失败处理 | 12-13 |
| **计量计费** | 记录 token 与成本 | 06 |
| **日志与 trace** | 全量请求记录 | 21 |
| **缓存** | 请求级缓存 | 08 |
| **限流** | 按用户/场景/全局 | 10 |
| **降级** | 主模型失败自动切换 | 11 |

## 3. 统一接口设计（协议抽象）

```
内部统一请求：
{
  "scene": "summary",            // 场景标识
  "model_tier": "balanced",      // 意图级别：cheap/balanced/premium
  "messages": [...],
  "params": { "temperature": 0.2, "max_tokens": 1000 },
  "response_format": { "type": "json", "schema": {...} },
  "meta": { "user_id": "...", "trace_id": "..." }
}

内部统一响应：
{
  "content": "...",
  "usage": { "input_tokens": 1500, "output_tokens": 400 },
  "model": "gpt-x@2026-01",     // 实际使用的模型与版本
  "latency": { "first_token_ms": 800, "total_ms": 4200 },
  "cost": 0.032,
  "trace_id": "..."
}
```

**关键**：业务代码只写 `model_tier`，具体用哪家模型由网关配置决定 → 换模型零代码改动。

## 4. 路由策略

| 策略 | 逻辑 | 适用 |
|---|---|---|
| **按场景路由** | scene → 固定模型映射表 | 最简单，首选 |
| **按层级路由** | cheap/balanced/premium 三档 | 平衡成本 |
| **按意图分类** | 先用小模型分类，再分派 | 成本敏感 |
| **按长度/复杂度** | 输入 token 数决定 | 粗糙但有效 |
| **灰度路由** | 按 hash 把 X% 流量导到新模型 | 发布用 |
| **降级路由** | 主模型熔断时切备用 | 11 篇 |

### 场景映射表示例（配置化，不写死在代码）

```yaml
routes:
  summary:      { tier: cheap,     model: mini-latest }
  chat:         { tier: balanced,  model: mid-latest }
  long-writing: { tier: premium,   model: flagship }
  classify:     { tier: cheap,     model: mini-latest }
fallbacks:
  premium:  [flagship-a, flagship-b, degraded-template]
  balanced: [mid-a, mini-latest]
```

## 5. 自建 vs 托管网关

| 方案 | 代表 | 优点 | 缺点 |
|---|---|---|---|
| 自建薄网关 | 一个服务/函数 | 完全可控、无外部依赖 | 要自己实现全部功能 |
| 开源自托管 | LiteLLM、One API | 开箱即用多供应商 | 定制受限 |
| 商业平台 | OpenRouter、Portkey、云厂商 | 全功能、有 SLA | 成本、数据出境问题 |

**推荐**：小团队从 LiteLLM/One API 起步 + 自己加计量落库；有特殊合规（数据不出境）时自建。

### Java 栈示例（Spring AI 网关思路）

```java
// 统一入口接口：业务代码只面向此接口
public interface ChatGateway {
    ChatResult chat(ChatRequest req);   // 内部完成 路由/重试/熔断/日志/计量
}

// 网关实现骨架
ChatResult chat(ChatRequest req) {
    Route r = routeResolver.resolve(req.scene(), req.tier());   // 配置化路由
    try {
        ChatResult res = retry.execute(() -> invoke(r.model(), req)); // 重试
        meter.record(res.usage(), res.model(), req.meta());           // 计量
        traceLog.save(req, res);                                      // trace
        return res;
    } catch (ModelUnavailableException e) {
        return fallback(r, req);                                      // 降级
    }
}
```

## 6. 流式请求的网关处理

```
流式的特殊性：
① 网关要转发 SSE 流，同时累计 usage（token 计量在流结束时才完整）
② 首 token 延迟是独立指标，必须单独记录
③ 流中断（客户端断开）也要计费并记录
④ 流式失败的重试要慎重（可能已输出部分内容）
```

## 7. 供应商隔离与容灾

```
隔离原则：
① 账号隔离：不同业务用不同 API Key，避免限流互相影响
② 配额隔离：高优业务预留配额
③ 多供应商：核心场景至少 2 家可替换
④ 版本锁定：关键场景锁定模型版本（防静默更新），如 gpt-x@2026-01
   （不锁定 = 供应商更新后你的产品行为可能一夜变化）
```

## 8. 常见坑

| 坑 | 现象 | 解法 |
|---|---|---|
| 业务代码直接 import 供应商 SDK | 换模型 = 全局重构 | 统一接口 + tier 路由 |
| 不锁定模型版本 | 供应商更新后质量突变 | 版本号写进配置，升级走流程 |
| 流式请求漏计量 | 成本统计偏低 | 流结束时读 usage 字段 |
| 重试放大故障 | 供应商限流时重试风暴 | 统一退避 + 熔断（12-13） |
| 网关单点 | 网关挂了全站停摆 | 网关本身也要高可用/降级策略 |
| API Key 进代码仓库 | 泄漏被刷 | 密钥管理（见 05、32） |
| 路由表写死在代码 | 调整要发版 | 配置中心 + 热更新 |

## 9. Checklist

- [ ] 所有模型调用经由统一入口
- [ ] 业务代码只引用内部抽象，不 import 供应商 SDK
- [ ] 路由表配置化，支持热更新
- [ ] 关键场景锁定了模型版本
- [ ] 网关记录：usage、实际模型、延迟、成本、trace_id
- [ ] 流式请求正确计量（含中断场景）
- [ ] 核心场景有 ≥ 2 家可替换供应商
- [ ] API Key 不在代码仓库中（见 05）
