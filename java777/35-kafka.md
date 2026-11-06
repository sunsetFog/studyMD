# 35 · Kafka 与 Spring Kafka

## 1. 核心概念

| 概念 | 说明 |
|---|---|
| Topic | 逻辑主题 |
| Partition | 分区，**并行与顺序的基本单位**（分区内有序） |
| Offset | 分区内消息递增序号 |
| Consumer Group | 组内一个分区只被一个消费者消费 |
| Broker | 服务节点 |
| Replica | 副本（Leader/Follower） |
| ISR | 与 Leader 保持同步的副本集合 |

```
Producer → Topic(3 partitions): [P0][P1][P2]
Group A: C1→P0, C2→P1, C3→P2     ← 分摊消费，offset 独立
Group B: C1→P0,P1,P2              ← 不同组 = 广播
```

> **分区数决定最大并行度**，消费者数 > 分区数时多余者空闲。

## 2. 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      acks: all                    # 0 / 1 / all
      retries: 3
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      properties:
        linger.ms: 10
        batch.size: 32768
        compression.type: snappy
        enable.idempotence: true   # ★ 幂等生产者
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      enable-auto-commit: false    # ★ 手动提交
      max-poll-records: 500
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: com.example.*
        max.poll.interval.ms: 300000
    listener:
      ack-mode: manual
      concurrency: 3               # 建议 = 分区数
```

## 3. 发送

```java
CompletableFuture<SendResult<String, Object>> f = kafka.send(topic, key, payload);
f.whenComplete((r, ex) -> {
    if (ex != null) log.error("发送失败", ex);
    else log.debug("partition={} offset={}", r.getRecordMetadata().partition(), r.getRecordMetadata().offset());
});

// 指定分区/头
kafka.send(MessageBuilder.withPayload(p)
        .setHeader(KafkaHeaders.TOPIC, "t")
        .setHeader(KafkaHeaders.KEY, key)
        .build());
```

**分区策略**：指定分区 → 直接用；有 key → `hash(key) % 分区数`（同 key 进同分区，保序）；无 key → 轮询/粘性。

## 4. 消费

```java
@KafkaListener(topics = "order-events", groupId = "order-service", concurrency = "3")
public void listen(ConsumerRecord<String, String> record, Acknowledgment ack) {
    try { handle(record.value()); ack.acknowledge(); }   // ★ 手动提交
    catch (Exception e) { log.error("消费失败 offset={}", record.offset(), e); throw e; }
}
```

| ack-mode | 说明 |
|---|---|
| `RECORD` | 每条提交 |
| `BATCH`（默认） | 每批提交 |
| `MANUAL` | 手动 ack（批量） |
| `MANUAL_IMMEDIATE` | 手动 ack（立即） |

> 先处理后提交 → 可能**重复消费**（需幂等）；先提交后处理 → 可能**丢消息**。推荐前者 + 幂等。

## 5. 顺序性

只保证**分区内有序**。保证业务顺序：
1. 用业务键（orderId）作 key → 同分区。
2. 单分区 + 单消费者（吞吐最低）。
3. 业务层用**版本号/状态机**容忍乱序（推荐）。
4. 失败重试会破坏顺序 → 用 `max.in.flight.requests.per.connection=1` 或改用重试队列。

## 6. 幂等与事务

```yaml
producer:
  properties:
    enable.idempotence: true       # 单分区幂等（PID + SequenceNumber），防重试产生重复
    transactional.id: order-tx-1   # 跨分区事务（精确一次）
```

```java
// 事务：消费-处理-生产（consume-transform-produce）精确一次
@KafkaListener(topics = "in")
public void listen(String in) {
    kafka.executeInTransaction(kt -> {
        kt.send("out", transform(in));
        return null;
    });
}
```

> 幂等生产者只解决**单会话单分区**；事务（`transactional.id`）解决跨分区与 EOS，但有性能开销。

## 7. 重试与死信

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> factory(
        ConsumerFactory<String, String> cf, KafkaTemplate<String, String> template) {
    var f = new ConcurrentKafkaListenerContainerFactory<String, String>();
    f.setConsumerFactory(cf);
    f.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
    f.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(template),      // 重试耗尽 → 死信 topic
            new FixedBackOff(1000L, 3)));                      // 1s 间隔，重试 3 次
    return f;
}
// 死信 topic 命名：原topic + ".DLT"
```

- 阻塞重试（同一分区会卡住后续消息）vs 重试队列（`.RETRY` topic + 延迟）。
- 非阻塞重试：SeekToCurrentErrorHandler 已演进为 `DefaultErrorHandler`。

## 8. 消息堆积

| 现象 | 处理 |
|---|---|
| 消费慢 | 加分区 + 加消费者；批量消费；异步化业务处理 |
| 消费阻塞 | 检查异常/死循环/外部依赖超时 |
| 峰值 | 临时扩容消费者（加实例） |
| rebalance 频繁 | 调大 `max.poll.interval.ms`、减小 `max-poll-records` |

> 加分区会改变 key 的分区映射，**破坏顺序性**，需谨慎。

监控指标：`consumer_lag`（`records-lag-max`）是最重要的告警项。

## 9. 常用命令

```bash
kafka-topics.sh --create --topic t --partitions 3 --replication-factor 2 --bootstrap-server localhost:9092
kafka-topics.sh --describe --topic t --bootstrap-server localhost:9092
kafka-console-consumer.sh --topic t --from-beginning --bootstrap-server localhost:9092
kafka-consumer-groups.sh --describe --group g --bootstrap-server localhost:9092     # 看 LAG
kafka-consumer-groups.sh --reset-offsets --group g --topic t --to-earliest --execute
```

## 10. 与 RabbitMQ 选型

| 维度 | Kafka | RabbitMQ |
|---|---|---|
| 吞吐 | **百万级** | 万~十万级 |
| 延迟 | 毫秒~（批量换吞吐） | **微秒~毫秒** |
| 消息确认 | 位移提交（批量） | 单条 ACK |
| 顺序 | 分区内有序 | 队列有序 |
| 功能 | 简单（无延迟/优先级） | 丰富（DLX、优先级、延迟插件） |
| 场景 | **日志、流处理、大数据、事件溯源** | 业务异步解耦、任务队列 |

## 11. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 重复消费 | 数据重复 | 幂等（唯一表/Redis/乐观锁） |
| 顺序错乱 | 状态回退 | 同 key 同分区 + 版本号 |
| `rebalance` 风暴 | 消费暂停 | 调超时参数、避免长时间处理 |
| 自动提交丢消息 | 处理失败但已提交 | 手动提交 |
| 反序列化失败 | `ClassCastException` | 配 `spring.json.trusted.packages` 或用 String + 手动解析 |
| `max.poll.interval` 超时 | 消费者被踢出 | 减小 `max-poll-records` 或调大超时 |
| 分区不足 | 并行上不去 | 提前规划分区数 |
| 磁盘写满 | Broker 宕机 | 保留策略（`retention.ms`） |

## 12. 面试速答

- **Kafka 为什么快**：顺序磁盘 I/O、零拷贝（sendfile）、批量压缩、页缓存、分区并行。
- **如何保证不丢**：`acks=all` + `min.insync.replicas>=2` + 手动提交 offset + 副本机制。
- **如何保证不重复**：幂等生产者（`enable.idempotence`）+ 业务幂等；EOS 用事务。
- **分区与消费者关系**：一个分区只能被组内一个消费者消费；消费者数 > 分区数则闲置。
- **零拷贝**：数据从磁盘经内核直接发到网卡，跳过用户态拷贝（Kafka 用 `sendfile`）。
