# 59 · Micrometer 指标埋点

## 1. 定位

Micrometer 是**指标门面**（类似 SLF4J 于日志），支持对接 Prometheus、Datadog、InfluxDB、New Relic、Graphite、JMX 等。

Spring Boot Actuator 的 metrics 就基于它。

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

> 从 Spring Boot 3 起，**Sleuth 已被 Micrometer Tracing 取代**，指标仍然是 Micrometer（`micrometer-core`）。

## 2. 核心概念

```
MeterRegistry（注册表）  →  Meter（指标）
Meter 类型：Counter / Gauge / Timer / DistributionSummary / LongTaskTimer
维度：Tag（key=value）
命名：dot 分隔，小写（order.created）
```

## 3. 五种 Meter

```java
@Autowired MeterRegistry registry;

// ① Counter：只增不减（请求数、错误数）
Counter counter = Counter.builder("order.created")
        .description("订单创建数")
        .tag("channel", "app")       // 维度
        .register(registry);
counter.increment();
counter.increment(5);

// ② Gauge：瞬时值（队列长度、连接数）—— 需要持有引用
AtomicInteger queueSize = new AtomicInteger();
Gauge.builder("queue.size", queueSize, AtomicInteger::get).register(registry);
// 或
registry.gauge("cache.size", cache, c -> c.size());

// ③ Timer：耗时 + 次数（最常用）
Timer timer = Timer.builder("api.cost")
        .description("接口耗时")
        .publishPercentiles(0.5, 0.95, 0.99)      // 计算分位数
        .publishPercentileHistogram()              // 供 Prometheus 计算 quantile
        .sla(Duration.ofMillis(100), Duration.ofSeconds(1))   // 分级桶
        .register(registry);

timer.record(120, TimeUnit.MILLISECONDS);
timer.record(() -> doSomething());                  // 自动计时
Timer.Sample sample = Timer.start(registry);
// ...
sample.stop(timer);

// ④ DistributionSummary：分布（请求体大小、订单金额）
DistributionSummary summary = DistributionSummary.builder("order.amount")
        .baseUnit("CNY").publishPercentiles(0.95).register(registry);
summary.record(199.9);

// ⑤ LongTaskTimer：正在执行的长任务
LongTaskTimer lt = LongTaskTimer.builder("job.running").register(registry);
lt.record(() -> longRunningJob());
```

## 4. 注解式（AOP）

```java
@Configuration
public class MetricsConfig {
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) { return new TimedAspect(registry); }
    @Bean
    public CountedAspect countedAspect(MeterRegistry registry) { return new CountedAspect(registry); }
}

@Service
public class OrderService {
    @Timed(value = "order.create", percentiles = {0.95, 0.99})   // 自动计时
    @Counted(value = "order.create.count", recordFailuresOnly = true)
    public Long create(OrderDTO dto) { ... }
}
```

## 5. Boot 自动提供的指标

| 前缀 | 内容 |
|---|---|
| `jvm.memory.used/committed/max` | 堆/非堆各区 |
| `jvm.gc.pause`、`jvm.gc.memory.promoted` | GC |
| `jvm.threads.live/daemon/peak` | 线程 |
| `jvm.classes.loaded` | 类加载 |
| `process.cpu.usage`、`process.start.time` | 进程 |
| `system.cpu.usage`、`system.cpu.count` | 系统 |
| `http.server.requests` | **HTTP 请求**（uri/status/method/outcome） |
| `tomcat.threads.busy/current`、`tomcat.sessions.*` | Tomcat |
| `hikaricp.connections.*` | 连接池 |
| `spring.data.repository.invocations` | Repository 调用 |
| `logback.events` | 日志事件数 |
| `cache.gets` | 缓存命中 |
| `rabbitmq.*` / `kafka.*` / `redis.*` | 中间件 |

## 6. 常用配置

```yaml
management:
  metrics:
    tags:                          # 全局 tag（应用/环境维度）
      application: ${spring.application.name}
      env: ${spring.profiles.active}
    enable:
      jvm: true
      process: true
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.95, 0.99
      slo:
        http.server.requests: 100ms, 300ms, 1s      # 分级桶
    web:
      server:
        request:
          autotime:
            enabled: true
  endpoint:
    metrics:
      enabled: true
```

查询：
```bash
curl localhost:8080/actuator/metrics                 # 指标列表
curl localhost:8080/actuator/metrics/http.server.requests?tag=uri:/api/orders
curl -H "Accept: text/plain" localhost:8080/actuator/prometheus
```

## 7. 自定义健康检查与指标绑定

```java
@Component
public class BizMetrics {
    public BizMetrics(MeterRegistry registry, OrderRepository repo) {
        Gauge.builder("order.pending.count", repo, r -> r.countByStatus(OrderStatus.PENDING))
             .description("待处理订单数")
             .register(registry);
    }
}
```

## 8. 命名与 Tag 规范

| 项 | 规范 |
|---|---|
| 命名 | 小写 + `.` 分隔：`http.server.requests`、`order.created` |
| 单位 | 时间在名字里体现或用 `baseUnit`（建议毫秒） |
| Tag | 维度用 tag，**不要**拼进名字（`order_created_app` ❌） |
| 基数 | **避免高基数 tag**（userId、orderId、URL 含路径参数）→ 会把 Prometheus 打爆 |
| 数量 | 每个指标的 tag 组合数控制在万级以内 |

> 高基数是 Prometheus 最常见的性能杀手。URI 要用模板（`/api/orders/{id}`），Boot 已默认处理。

## 9. Prometheus 数据模型对应

| Micrometer | Prometheus |
|---|---|
| Counter | `counter`（`_total` 后缀） |
| Gauge | `gauge` |
| Timer | `summary` / `histogram`（`_bucket`、`_count`、`_sum`） |
| DistributionSummary | `summary` |

Prometheus 用 histogram 的 `_bucket` 计算分位数：`histogram_quantile(0.95, rate(x_bucket[5m]))`。

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 高基数 tag | Prometheus 内存暴涨 | 移除 userId 等维度 |
| Gauge 引用被 GC | 值变 0/NaN | 持有强引用 |
| Timer 不用 histogram | 无法算分位数 | `publishPercentileHistogram` |
| 指标名含大写 | 部分后端不支持 | 统一小写 |
| 每次请求 new Meter | 内存泄漏 | 用 `builder().register()`（内部缓存）或用 `registry.counter()` |
| `/actuator/prometheus` 404 | 未暴露 | `exposure.include` |
| 指标太多 | 拉取超时 | 精简指标与 tag |

## 11. 面试速答

- **Micrometer 是什么**：指标门面，统一 API 对接多种监控系统；Actuator 的 metrics 基于此。
- **五种 Meter**：Counter（累加）、Gauge（瞬时值）、Timer（耗时+次数）、DistributionSummary（分布）、LongTaskTimer（长任务）。
- **Tag 与名字的区别**：名字描述「测什么」，tag 描述「哪个维度」；高基数 tag（如用户 ID）要避免。
- **如何算 P99**：Timer 开启 `publishPercentileHistogram`，Prometheus 侧用 `histogram_quantile`。
- **Micrometer 与 Dropwizard Metrics**：前者是新一代、支持维度 tag 与多后端，Spring Boot 2+ 默认使用。
