# 60 · 分布式链路追踪

## 1. 为什么需要

一次请求跨多个服务，出问题时不知道卡在哪一跳。链路追踪提供：

- **traceId**：贯穿整条请求链
- **spanId**：每个服务/组件内的一个片段
- **parentId**：父子关系
- **耗时、状态、tag（SQL、URL、错误）**
- **拓扑图**：服务依赖关系

## 2. 核心概念

```
Trace（一次完整调用链，一个 traceId）
 └── Span（最小工作单元，有 spanId + parentId）
      ├── 注解/事件：cs（客户端发送）、sr（服务端接收）、ss（服务端发送）、cr（客户端接收）
      └── Tag：http.method、db.statement、error
```

```
Gateway(Span A) → Order(Span B, parent=A) → User(Span C, parent=B)
                                         → Inventory(Span D, parent=B)
```

## 3. 传播协议

| 协议 | 头 |
|---|---|
| **W3C Trace Context**（标准） | `traceparent: 00-<traceId>-<spanId>-01` |
| B3（Zipkin） | `X-B3-TraceId`、`X-B3-SpanId`、`X-B3-Sampled` |
| Jaeger | `uber-trace-id` |

> Boot 3 默认使用 **W3C**；可通过 `management.tracing.propagation.type=b3,w3c` 兼容。

## 4. Spring Cloud Sleuth → Micrometer Tracing

Boot 3 起 Sleuth 停止维护，官方方案是 **Micrometer Tracing**（`micrometer-tracing-bridge-brave` / `-otel`）+ **OpenTelemetry**。

### 依赖

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
<!-- 或用 -bridge-brave + zipkin-reporter -->
```

### 配置

```yaml
management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0          # 采样率（生产建议 0.1）
    propagation:
      type: w3c,b3
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"   # ★ 日志带 traceId
```

> 配置后 MDC 会自动填充 `traceId`/`spanId`，日志格式里用 `%X{traceId}` 输出（见 [13](13-boot-logging.md)）。

## 5. 自动埋点范围

Micrometer Tracing + Spring 自动为以下生成 span：

- HTTP 请求（MVC/WebFlux 服务端）
- RestTemplate、WebClient、OpenFeign、RestClient
- Spring Cloud Gateway
- JMS、RabbitMQ、Kafka（消息收发）
- R2DBC / JDBC（部分）
- `@Scheduled`、异步方法

手动埋点：

```java
@Autowired Tracer tracer;

public void biz() {
    Span span = tracer.nextSpan().name("custom-op").start();
    try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
        span.tag("orderId", String.valueOf(orderId));
        span.event("开始处理");
        doWork();
    } catch (Exception e) {
        span.error(e);
        throw e;
    } finally {
        span.end();              // ★ 必须结束
    }
}
```

或用注解：
```java
@Observed(name = "order.create", contextualName = "createOrder",
          lowCardinalityKeyValues = {"type", "normal"})
public void create() {}
// 需注册 ObservedAspect Bean
```

## 6. 采样策略

| 策略 | 说明 |
|---|---|
| 概率采样 | `probability: 0.1`（10%） |
| 速率限制 | 每秒最多 N 条 |
| 头部优先 | 上游已采样则全链路采样 |
| 自适应 | 根据负载调整 |

> 生产一般 1%~10%；**错误请求应 100% 采样**（可自定义 Sampler）。

## 7. 后端选型

| 后端 | 特点 |
|---|---|
| **Zipkin** | 轻量、部署简单 |
| **Jaeger** | CNCF 毕业，功能全 |
| **SkyWalking** | 国产，**无侵入**（Java Agent）、APM 全栈（指标+拓扑+告警） |
| Elastic APM | 与 ELK 集成 |
| Tempo (Grafana) | 与 Grafana 生态 |
| 商业 | 阿里云 ARMS、听云、Datadog |

### SkyWalking（国内主流）

```bash
# 无侵入接入
java -javaagent:/path/skywalking-agent.jar \
     -DSW_AGENT_NAME=order-service \
     -DSW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800 \
     -jar app.jar
```

优点：不用改代码、不用加依赖；缺点：与 Spring 生态的集成不如原生方案细（但够用）。

## 8. 排查实战

```
1 拿到 traceId（前端响应头 X-Trace-Id 或日志）
2 在 Zipkin/Jaeger 搜索 traceId
3 看火焰图：哪一跳耗时最长、哪个 span 报错
4 点开 span 看 tag：SQL 语句、HTTP URL、错误堆栈
5 结合日志（traceId 过滤）与指标定位根因
```

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| traceId 断链 | 链路不完整 | 检查异步线程（需 TaskDecorator）、MQ（需 header 注入）、Gateway 是否透传 |
| 日志无 traceId | `%X{traceId}` 为空 | 配 logging pattern + 引入 tracing 依赖 |
| 异步丢失 | 异步 span 不关联 | 用 `Tracer` 手动或 `TaskDecorator` 传 MDC |
| 采样率 0 | 无数据 | 检查 `sampling.probability` |
| 上报失败 | 后端无数据 | 检查 endpoint、网络、服务是否可达 |
| 性能开销 | 采样率 100% 拖慢 | 降到 1%~10% |
| 高基数 tag | 后端撑不住 | 不要放 userId/orderId |

## 10. 面试速答

- **链路追踪三要素**：traceId（整链唯一）、spanId（单段）、parentId（父子关系）。
- **Boot 3 方案**：Sleuth 已废弃，改用 Micrometer Tracing（Brave/OTel bridge）+ Zipkin/OTLP 导出。
- **如何保证 traceId 不断链**：HTTP 通过 header 传播（W3C `traceparent`）；异步线程池通过 `TaskDecorator` 传 MDC；MQ 通过消息头注入/提取。
- **采样为什么必要**：全量上报成本高、影响性能；生产一般 1%~10%，异常请求提高采样。
- **SkyWalking vs Zipkin**：SkyWalking 是 Java Agent 无侵入、带 APM（指标、拓扑、告警）；Zipkin 轻量、专注链路，需 SDK 埋点。
