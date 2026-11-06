# 32 · 监控：Actuator + Prometheus + Grafana

## 一、Actuator 端点

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env,loggers
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true              # K8s liveness/readiness
  health:
    redis: { enabled: true }
    db: { enabled: true }
  metrics:
    tags:
      application: demo-app        # 全局标签：Grafana 按应用筛选
```

常用端点：

| 端点 | 用途 |
|---|---|
| /health | 组件健康（DB/Redis/MQ）聚合 |
| /metrics | JVM、HTTP、连接池等指标列表 |
| /prometheus | Prometheus 抓取格式 |
| /env | 环境配置（注意敏感项脱敏） |
| /loggers | **运行时动态调整日志级别** |
| /threaddump, /heapdump | 线程/堆诊断 |

**安全**：Actuator 绝不能裸奔公网——Security 配置单独鉴权、或仅内网/管理端口暴露（`management.server.port=9090`）。

## 二、自定义健康检查与指标

```java
@Component
public class ThirdPartyHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean ok = pingThirdService();
        return ok ? Health.up().build() : Health.down().withDetail("timeout", "3000ms").build();
    }
}

// 业务指标
CountedCounterService {
    private final Counter orderCounter;

    public OrderMetrics(MeterRegistry registry) {
        this.orderCounter = registry.counter("business.order.created", "type", "normal");
    }

    public void recordOrder() { orderCounter.increment(); }

    // 计时器
    Timer timer = registry.timer("business.pay.duration");
    timer.recordCallable(() -> payService.pay(order));
}
```

## 三、Prometheus + Grafana 标准链路

```
SpringBoot (/actuator/prometheus)
   → Prometheus 定时抓取
   → Grafana 展示大盘 + 告警规则
   → Alertmanager → 钉钉/飞书/短信
```

常用内置指标（Grafana JVM 模板：dashboard id 4701）：
- `jvm_memory_used_bytes` / GC：`jvm_gc_pause_seconds`
- `http_server_requests_seconds_count{uri,method,status}`：QPS 与错误率
- `hikaricp_connections_active` / `hikaricp_connections_pending`：连接池
- `process_cpu_usage`

**核心告警规则示例**（PromQL）：
```promql
# 错误率 > 5% 持续 5 分钟
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count[5m])) > 0.05

# P99 延迟 > 2s
histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m])) > 2
```

## 四、可观测性三支柱

| 支柱 | 工具 | 用途 |
|---|---|---|
| Metrics | Micrometer → Prometheus | 趋势/容量/告警 |
| Logging | Logback → ELK/Loki | 事件详情，traceId 串联 |
| Tracing | Micrometer Tracing + Zipkin/OTLP | 跨服务调用链路 |

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
```

配合 23 篇 MDC traceId，一次请求全链路（日志+指标+trace）可关联。

## 五、日常运维监控清单

| 检查项 | 工具 |
|---|---|
| 接口错误率/延迟 | Prometheus 大盘 |
| JVM 内存/GC | jstat / Grafana / heapdump 分析 |
| 连接池饱和 | hikaricp 指标，pending > 0 即告警 |
| 线程池排队 | 自定义 gauge |
| 慢 SQL | MySQL slow log / Druid 监控 |
| 定时任务执行 | 自定义指标 + 日志告警 |

## 六、压测基线（上线前）

```bash
# wrk / JMeter / k6
wrk -t8 -c200 -d60s --latency https://demo.example.com/api/v1/users
```

关注：QPS、P95/P99、错误率、GC 频率、连接池水位。压测结论写入上线评审。

## 常见坑

- **坑 1**：Actuator 全端点暴露公网 → /env 泄露密码、/heapdump 泄露内存数据
- **坑 2**：health 依赖外部服务挂了导致 liveness 失败被反复重启 → 就绪探针用 readiness，别把重依赖放 liveness
- **坑 3**：指标无 application 实例标签 → 多实例混淆
- **坑 4**：监控打点本身慢（同步 reporter）→ Prometheus pull 模式无此问题；push 型注意异步

## Checklist

- [ ] Actuator 最小暴露 + 独立鉴权/端口
- [ ] probes 开启，K8s 就绪探针配置
- [ ] Prometheus + Grafana 大盘与告警规则
- [ ] 业务关键指标自定义打点
- [ ] 上线前压测基线
