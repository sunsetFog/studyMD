# 21 · Actuator：健康检查、端点与自定义指标

## 1. 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  server:
    port: 9091                 # ★ 管理端口与业务端口分离（安全最佳实践）
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,loggers,beans,conditions,threaddump,heapdump
        exclude: shutdown
    enabled-by-default: false  # 只开放显式列出的
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true          # K8s 存活/就绪探针
    shutdown:
      enabled: false
  health:
    diskspace:
      enabled: true
    redis:
      enabled: true
  info:
    env:
      enabled: true
```

## 2. 内置端点

| 端点 | 作用 | 敏感 |
|---|---|---|
| `/actuator/health` | 健康检查（DB、Redis、MQ 等） | 低 |
| `/actuator/health/liveness` | K8s 存活探针 | 低 |
| `/actuator/health/readiness` | K8s 就绪探针 | 低 |
| `/actuator/info` | 应用信息 | 低 |
| `/actuator/metrics` | 指标列表 | 中 |
| `/actuator/env` | 环境变量（**有泄露风险**） | 高 |
| `/actuator/beans` | 所有 Bean | 中 |
| `/actuator/conditions` | 自动配置匹配报告 | 中 |
| `/actuator/loggers` | 动态改日志级别 | 高 |
| `/actuator/threaddump` | 线程栈 | 高 |
| `/actuator/heapdump` | 堆转储（**很大**） | 高 |
| `/actuator/mappings` | 所有接口映射 | 中 |
| `/actuator/scheduledtasks` | 定时任务 | 中 |
| `/actuator/shutdown` | 优雅停机（默认禁用） | 高 |

> **生产铁律**：`/actuator/**` 绝不能暴露公网；用独立端口 + 内网 + 鉴权。

## 3. 健康检查原理

```java
public interface HealthIndicator {
    Health health();
}
```

`CompositeHealthContributor` 聚合所有 Indicator，返回 `UP / DOWN / OUT_OF_SERVICE / UNKNOWN`。

状态聚合顺序：`DOWN > OUT_OF_SERVICE > UP > UNKNOWN`（最差的胜出）。

### 自定义健康检查

```java
@Component
public class SmsHealthIndicator implements HealthIndicator {
    private final SmsClient client;
    @Override public Health health() {
        try {
            boolean ok = client.ping();
            return ok ? Health.up().withDetail("channel", "aliyun").build()
                      : Health.down().withDetail("reason", "ping failed").build();
        } catch (Exception e) {
            return Health.down(e).build();       // 异常详情（show-details 时才展示）
        }
    }
}
```

响应式用 `ReactiveHealthIndicator`。

## 4. 优雅停机

```yaml
server:
  shutdown: graceful        # 等待请求处理完再关闭
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

```java
@Component
public class GracefulShutdown implements ApplicationListener<ContextClosedEvent> {
    @PreDestroy public void onExit() {
        // 注销注册中心、等待线程池任务、关闭连接池
    }
}
```

> K8s 中 `terminationGracePeriodSeconds` 必须大于 shutdown 超时。

## 5. 自定义 info

```yaml
info:
  app:
    name: @project.name@
    version: @project.version@
  build:
    time: @maven.build.timestamp@
```
```java
@Component
public class GitInfoContributor implements InfoContributor {
    public void contribute(Info.Builder b) {
        b.withDetail("git", Map.of("commit", BuildInfo.GIT_COMMIT));
    }
}
```

## 6. 自定义端点

```java
@Component
@Endpoint(id = "features")
public class FeatureEndpoint {
    @ReadOperation
    public Map<String, Boolean> features() { return FeatureToggle.current(); }

    @WriteOperation
    public void toggle(@Selector String name, boolean enabled) { FeatureToggle.set(name, enabled); }
}
// 访问 /actuator/features、/actuator/features/{name}
```

Web 专用：`@WebEndpoint`、`@RestControllerEndpoint`（返回 JSON）。

## 7. Micrometer 指标（基础）

```java
@Autowired MeterRegistry registry;

Counter c = Counter.builder("order.created").tag("channel", "app").register(registry);
c.increment();

registry.timer("api.cost", "uri", "/api/orders").record(120, TimeUnit.MILLISECONDS);

// Gauge（瞬时值，需持有引用）
AtomicInteger queueSize = new AtomicInteger();
Gauge.builder("queue.size", queueSize, AtomicInteger::get).register(registry);

// 函数式计时
Timer.Sample sample = Timer.start(registry);
// ...
sample.stop(registry.timer("my.timer"));
```

指标命名规范：`.` 分隔、小写、以动词/名词结尾；tag 用于维度（不要塞进名字）。

Boot 自动提供的指标：`jvm.*`、`system.*`、`http.server.requests`、`tomcat.*`、`hikaricp.*`、`logback.events`、`process.*`。

## 8. 常见指标与含义

| 指标 | 说明 | 告警建议 |
|---|---|---|
| `http.server.requests` | 请求量、耗时、状态码 | P99 > 1s、5xx 率 > 1% |
| `jvm.memory.used` | 各区内存 | 老年代持续上升 → 内存泄漏 |
| `jvm.gc.pause` | GC 停顿 | 单次 > 1s |
| `hikaricp.connections.pending` | 等待连接数 | > 0 说明池不够 |
| `tomcat.threads.busy` | 繁忙线程 | 接近 max 说明线程池瓶颈 |
| `system.cpu.usage` | CPU | > 80% |

## 9. 与 Prometheus 集成

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```
暴露 `/actuator/prometheus`。详见 [62](62-prometheus-grafana.md)。

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 端点 404 | 访问不到 | 默认只暴露 `health`/`info`，需 `exposure.include` |
| `/actuator` 无内容 | HATEOAS 未引入 | 加 `spring-boot-starter-hateoas` |
| 健康检查拖慢应用 | 频繁探测 DB | 设置缓存：`management.endpoint.health.cache.time-to-live` |
| 就绪探针 DOWN 导致 K8s 摘流量 | 启动即失败 | 用 `readiness` 而非 `health` 做流量探针 |
| heapdump 打爆磁盘 | 磁盘满 | 不暴露或限制目录 |
| 敏感信息泄露 | `/env` 暴露密码 | 不暴露 env/beans/heapdump 到外网 |
| 优雅停机不生效 | kill 后立即中断 | `server.shutdown=graceful` + 用 SIGTERM |

## 11. 面试速答

- **Actuator 是什么**：Boot 提供的生产级监控模块，通过 HTTP/JMX 端点暴露健康检查、指标、环境、线程等信息。
- **liveness 与 readiness 区别**：liveness 判断进程是否存活（不健康则重启）；readiness 判断是否可接流量（不健康则摘除）。
- **优雅停机**：`server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase`，配合容器 SIGTERM 与优雅等待。
