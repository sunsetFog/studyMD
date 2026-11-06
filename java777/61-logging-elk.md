# 61 · 日志规范与 ELK/Loki

## 1. 日志分级与使用

| 级别 | 含义 | 生产 |
|---|---|---|
| ERROR | 需要人工介入的错误 | 必开，**要告警** |
| WARN | 可自愈/预期外但不致命 | 开 |
| INFO | 关键业务节点、启动信息 | 开 |
| DEBUG | 调试细节 | **关** |
| TRACE | 最细（含框架内部） | 关 |

原则：
- 每条日志要有 **traceId**（见 [60](60-distributed-tracing.md)）。
- 记录**关键 ID**：userId、orderId、skuId。
- 异常必须打印堆栈：`log.error("xxx失败 id={}", id, e)`。
- **禁止**打印密码、token、身份证、银行卡（必须脱敏）。
- 用占位符 `{}`，避免字符串拼接开销。
- 慢操作单独 WARN（超阈值）。

## 2. 结构化日志（JSON）

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

```xml
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
        <customFields>{"app":"${APP_NAME}","env":"${ENV}"}</customFields>
    </encoder>
</appender>
```

输出：
```json
{"@timestamp":"2024-01-01T10:00:00.000+08:00","level":"INFO","thread":"http-nio-8080-exec-1",
 "logger":"c.e.OrderService","message":"创建订单","traceId":"a1b2c3","app":"order-service"}
```

> 容器化部署推荐**直接输出到 stdout**（Docker 采集），不落文件。

## 3. 采集方案

| 方案 | 组成 | 适用 |
|---|---|---|
| **ELK** | Elasticsearch + Logstash + Kibana | 功能全，资源占用大 |
| **EFK** | Elasticsearch + **Fluentd/Fluent Bit** + Kibana | K8s 常用 |
| **Loki** | Loki + Promtail + Grafana | **轻量**，按标签索引，成本低 |
| 云日志服务 | 阿里云 SLS、腾讯云 CLS | 免运维 |

### Filebeat（轻量采集器）

```yaml
filebeat.inputs:
  - type: filestream
    paths: ["/data/logs/*.log"]
    fields: {app: order-service, env: prod}
    multiline:                        # 合并多行堆栈
      pattern: '^\d{4}-\d{2}-\d{2}'
      negate: true
      match: after
output.elasticsearch:
  hosts: ["http://es:9200"]
  index: "app-%{[fields.app]}-%{+yyyy.MM.dd}"
```

### Logstash Grok（解析非结构化日志）

```
filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:ts} %{LOGLEVEL:level} \[%{DATA:thread}\]\[%{DATA:traceId}\] %{JAVACLASS:logger} - %{GREEDYDATA:msg}" }
  }
  date { match => ["ts", "yyyy-MM-dd HH:mm:ss.SSS"] }
}
```

## 4. Loki（Grafana 生态）

```yaml
# promtail-config.yml
clients:
  - url: http://loki:3100/loki/api/v1/push
scrape_configs:
  - job_name: app
    static_configs:
      - targets: [localhost]
        labels:
          __path__: /data/logs/*.log
          app: order-service
    pipeline_stages:
      - multiline:
          firstline: '^\d{4}-\d{2}-\d{2}'
      - regex:
          expression: 'traceId=(?P<traceId>[a-f0-9]+)'
```

查询（LogQL）：
```logql
{app="order-service"} |= "ERROR" != "健康检查" | json | traceId="a1b2c3"
{app="order-service"} |~ "耗时.*ms"
sum(rate({app="order-service"} |= "ERROR" [5m]))
```

> Loki 只索引**标签**，日志体压缩存储，成本远低于 ES。

## 5. K8s 日志采集

- 应用输出到 stdout/stderr。
- DaemonSet 方式部署 Fluent Bit / Promtail，读取 `/var/log/containers/*.log`。
- 日志带上 Pod 标签（namespace、pod、container、app）。

## 6. 日志保留与成本

| 项 | 建议 |
|---|---|
| 保留期 | 7~30 天（合规要求另说） |
| 索引 | 按天分索引，定期删除（ILM/Curator） |
| 采样 | 超高 QPS 的 DEBUG/INFO 可采样 |
| 冷数据 | 转对象存储 |
| 大字段 | 不要打完整请求体（或截断） |

## 7. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 日志打满磁盘 | 服务挂 | 滚动策略 + `totalSizeCap` |
| 多行堆栈被拆散 | 难读 | multiline 配置 |
| 中文乱码 | 乱码 | 统一 UTF-8 |
| 时间戳时区错 | 差 8 小时 | 配 `Asia/Shanghai` 或容器时区 |
| ES 分片过多 | 性能下降 | 控制索引数量 |
| 采集端丢日志 | 日志缺失 | 队列缓冲 + 背压 |
| 日志量爆炸 | 费用高 | 降级别、采样、缩短保留期 |
| 敏感信息入日志 | 合规风险 | 脱敏 + 审计 |

## 8. 面试速答

- **ELK 与 Loki 区别**：ELK 全文索引、检索能力强但资源占用大；Loki 只索引标签、日志体压缩，成本低，与 Grafana 无缝集成。
- **容器化日志怎么采**：应用输出到 stdout，节点上跑 Fluent Bit/Promtail DaemonSet 采集。
- **日志为什么要结构化（JSON）**：便于检索、聚合、告警，无需 Grok 解析。
- **如何降低日志成本**：控制级别、采样、缩短保留期、大字段截断、Loki 替代 ES。
