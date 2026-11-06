# 12 · 重试与超时：失败处理的基本功

## 1. AI 调用的失败类型

| 类型 | 表现 | 是否该重试 |
|---|---|---|
| 网络抖动 | 连接失败、TLS 错误 | ✅ 重试 |
| 供应商瞬时 5xx | 500/502/503 | ✅ 有限重试 |
| 限流 429 | rate limit | ⚠️ 按 Retry-After 退避重试 |
| 超时 | TTFT 或总时长超限 | ⚠️ 可重试一次，注意幂等 |
| 请求被拒 400 | 参数错误、内容违规 | ❌ 不重试（改了才会好） |
| 输出格式不合法 | JSON 解析失败 | ⚠️ 重试 1 次 / 自动修复 |
| 内容安全拦截 | 输出被拒答 | ❌ 不重试（重试大概率同样结果） |
| 上下文超限 | 超出窗口 | ❌ 不重试（需裁剪后重试） |

**原则：只重试「瞬时故障」，不重试「确定性失败」。**

## 2. 重试策略设计

```
指数退避 + 抖动（Exponential Backoff + Jitter）：

delay_n = base × 2^n + random(0, base)
示例：base=1s → 1s, 2s, 4s, 8s...（加随机抖动防止重试风暴）

限制：
① 最大重试次数：2-3 次（AI 场景不宜多，成本随重试累加）
② 总超时上限：重试总时长不超过用户可接受等待
③ 只重试幂等请求：流式已输出部分内容后不盲目重试
④ 遵守 Retry-After：429 响应头给了等待时间就照做
```

### 重试与计费的关系（AI 特有）

```
每次重试都是新的一次计费调用！
① 重试必须计入该请求的成本（trace 里合并）
② 输出已经生成了 50% 后断流 → 重试 = 重新全额计费
   → 长输出场景可考虑「断点续传」或接受部分结果
③ 重试风暴防护：全站重试要在网关层做全局并发限制
```

## 3. 超时设计（分层超时）

```
分层：
① 连接超时：5s（连不上=供应商挂了，快速失败）
② TTFT 超时：交互场景 5-10s（没首字说明有问题）
③ 总超时：按场景延迟预算 × 1.5
④ 网关层总兜底：防止应用层漏配

关键：超时后必须主动取消上游请求！
  否则供应商继续生成 → 继续计费 → 白烧钱
```

```python
# 网关层取消上游示例（流式场景）
async def call_model(req):
    try:
        async with timeout(total_budget):
            async for chunk in provider.stream(req):
                yield chunk
    except asyncio.TimeoutError:
        await provider.cancel(req.request_id)   # ★ 取消上游
        raise TimeoutError("model timeout, request cancelled")
```

## 4. 输出格式失败的专用处理

结构化输出（JSON）的失败处理链：

```
模型输出 → JSON 解析
  ├─ 成功 → schema 校验 → 通过
  ├─ 解析失败（包裹了 ```json 或尾随文本）
  │    → 自动修复：剥离 markdown 包裹、截取首个 {...}
  ├─ 修复失败
  │    → 重试 1 次（prompt 附加"只输出 JSON"强化）
  ├─ 仍失败
  │    → 降级：schema 校验宽松版 / 换模型 / 兜底模板
  └─ 全程记录失败样本 → 进 bad case 库（见 Product/36）
```

## 5. 幂等与重试安全

```
① 生成类请求天然幂等风险低（多生成一份没有副作用）
② 有副作用的调用（Agent 写数据库、发消息）必须带 idempotency_key：
   网关按 key 去重，同 key 重试返回首次结果
③ 计费按 idempotency_key 合并，防止重复计费
```

## 6. Java/Spring 实现参考

```java
// Resilience4j：java777/44 的能力直接复用
Retry retry = Retry.of("llm-call", RetryConfig.custom()
    .maxAttempts(3)                          // 含首次
    .waitDuration(Duration.ofSeconds(1))
    .enableExponentialBackoff()
    .retryOnException(e -> isTransient(e))   // 只重试瞬时故障
    .failAfterDuration(Duration.ofSeconds(30))
    .build());

TimeLimiter limiter = TimeLimiter.of("llm", TimeLimiterConfig.custom()
    .timeoutDuration(Duration.ofSeconds(30)).build());
```

## 7. 常见坑

| 坑 | 现象 | 解法 |
|---|---|---|
| 重试一切错误 | 400/内容违规反复重试烧钱 | 失败分类，只重试瞬时故障 |
| 无抖动的固定间隔 | 高峰期重试同步扎堆 | Exponential + Jitter |
| 超时不取消上游 | 超时后供应商继续计费 | 主动 cancel |
| 重试成本没记账 | 单请求成本统计偏低 | trace 合并重试费用 |
| 429 无退避 | 越限流越猛打 | 尊重 Retry-After |
| JSON 失败直接报错 | 本可自动修复 | 修复链（剥离/截取/强化重试） |
| 有副作用的调用重试 | 数据重复写入 | idempotency_key |
| 重试次数太多 | 用户等太久且成本×3 | 上限 2-3 次 + 总时长上限 |

## 8. Checklist

- [ ] 失败分类表明确（哪些重试/哪些不重试/哪些要修复）
- [ ] 指数退避 + 抖动，重试 ≤ 3 次且有总时长上限
- [ ] 超时分层（连接/TTFT/总时长），超时后取消上游
- [ ] 流式/长输出场景的重试策略单独设计
- [ ] JSON 输出有自动修复链
- [ ] 重试成本合并计入 trace
- [ ] 有副作用的调用有幂等键
- [ ] 429 遵守 Retry-After
