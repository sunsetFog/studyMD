# 62 · Prometheus + Grafana 监控告警

## 1. 架构

```
应用 /actuator/prometheus  ←── Prometheus 拉取(scrape) ──→ TSDB 存储
                                                              ↓
                                              Grafana 展示 + Alertmanager 告警
```

| 组件 | 作用 |
|---|---|
| Prometheus | 时序数据库 + 拉取 + 告警规则 |
| Exporter | 第三方系统指标暴露（node、mysql、redis、kafka） |
| Grafana | 可视化看板 |
| Alertmanager | 告警去重、分组、路由、静默 |
| Pushgateway | 短任务推送（不推荐常规使用） |

## 2. 应用侧暴露指标

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
  endpoint:
    health:
      probes: {enabled: true}
  metrics:
    tags:
      application: ${spring.application.name}
```

验证：`curl localhost:8080/actuator/prometheus`

## 3. Prometheus 配置

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-apps'
    metrics_path: '/actuator/prometheus'
    # K8s 自动发现
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysqld-exporter:9104']
```

## 4. PromQL 常用查询

```promql
# QPS（每秒请求数）
sum(rate(http_server_requests_seconds_count{application="order"}[5m]))

# 错误率
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count[5m]))

# P99 延迟
histogram_quantile(0.99,
  sum by (le) (rate(http_server_requests_seconds_bucket{uri!~"/actuator.*"}[5m])))

# JVM 堆使用率
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}

# GC 停顿
rate(jvm_gc_pause_seconds_sum[5m]) / rate(jvm_gc_pause_seconds_count[5m])

# CPU
process_cpu_usage

# 线程池繁忙度
tomcat_threads_busy_threads / tomcat_threads_config_max_threads

# 连接池等待
hikaricp_connections_pending

# 过去 5 分钟新增错误
increase(logback_events_total{level="error"}[5m])
```

## 5. 告警规则

```yaml
# alert-rules.yml
groups:
  - name: app
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          / sum(rate(http_server_requests_seconds_count[5m])) > 0.05
        for: 5m
        labels: {severity: critical}
        annotations:
          summary: "5xx 错误率超过 5%"
          description: "当前值 {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum by (le) (rate(http_server_requests_seconds_bucket[5m]))) > 1
        for: 10m
        labels: {severity: warning}
        annotations: {summary: "P99 延迟 > 1s"}

      - alert: HeapUsageHigh
        expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.85
        for: 10m
        labels: {severity: warning}

      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels: {severity: critical}
```

## 6. Alertmanager

```yaml
route:
  group_by: ['alertname', 'application']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'webhook'
  routes:
    - matchers: [severity="critical"]
      receiver: 'phone'
receivers:
  - name: 'webhook'
    webhook_configs:
      - url: 'http://alert-dingtalk:8060/webhook'
  - name: 'phone'
    webhook_configs:
      - url: 'http://alert-phone:8060/webhook'
inhibit_rules:
  - source_matchers: [severity="critical"]
    target_matchers: [severity="warning"]
    equal: ['application']
```

> 告警原则：**可行动**（收到就知道该干什么）、**不噪音**（分级、聚合、抑制）、**有 Runbook**。

## 7. Grafana 看板

推荐导入官方 JVM 看板（ID 4701、12900）或自研，关键面板：

| 面板 | 指标 |
|---|---|
| QPS | `rate(http_server_requests_seconds_count[5m])` |
| 响应时间 P50/P95/P99 | `histogram_quantile` |
| 错误率 | 5xx / total |
| JVM 内存 | heap used/max |
| GC | 次数、耗时 |
| 线程 | live/busy |
| 连接池 | active/pending |
| 慢接口 Top10 | `topk(10, ...)` |

RED 黄金指标：**R**ate（请求量）、**E**rrors（错误率）、**D**uration（耗时）；再加饱和度（Saturation）。

## 8. 生产要点

| 项 | 建议 |
|---|---|
| 抓取间隔 | 15~30s（太密成本高） |
| 保留期 | 15 天 ~ 数月（或远端存储 Thanos/Mimir） |
| 高可用 | 多副本 + Thanos |
| 标签基数 | 严控（高基数会打爆 TSDB） |
| 联邦 | 大规模用 Prometheus 联邦或 Thanos |
| 监控自身 | 监控 Prometheus 与 Alertmanager |

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 高基数标签 | TSDB 内存暴涨 | 移除 userId/URL 参数维度 |
| 抓取超时 | 目标 DOWN | 精简指标或调大 `scrape_timeout` |
| 告警风暴 | 刷屏 | 分组、抑制、合理 `for` 时长 |
| P99 不准 | 与真实不符 | 桶（bucket）设置合理；别用 avg |
| 指标缺失 | 面板空 | actuator 暴露 + job 匹配 |
| 时间不同步 | 数据错乱 | NTP 同步 |
| 拉模式无法采集短任务 | 数据缺失 | Pushgateway（慎用） |

## 10. 面试速答

- **Prometheus 特点**：拉模型（pull）、多维数据模型（metric + label）、PromQL、本地 TSDB、自带告警。
- **四种指标类型**：Counter（累加）、Gauge（瞬时）、Histogram（分桶）、Summary（分位数）。
- **P99 怎么算**：`histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m]))`，注意用 `_bucket` 而非 `_sum/_count` 的均值。
- **为什么要 Alertmanager**：告警去重、分组、抑制、静默、多通道路由。
- **RED/USE**：RED 面向请求（Rate/Errors/Duration）；USE 面向资源（Utilization/Saturation/Errors）。
