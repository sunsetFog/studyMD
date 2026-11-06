# 26 · Kafka 集成

## 一、与 RabbitMQ 选型

| 维度 | Kafka | RabbitMQ |
|---|---|---|
| 吞吐 | 极高（百万级/s） | 中（万级/s） |
| 消息回溯 | 可（offset 重放） | 否 |
| 顺序 | 分区内有序 | 队列内有序 |
| 语义 | at-least-once / exactly-once | at-least-once |
| 适用 | 日志流/大数据/事件溯源/削峰 | 业务消息/延迟队列/复杂路由 |

## 二、集成配置

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
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                        # 0/1/all：all 最可靠
      retries: 3
      properties:
        enable.idempotence: true       # 生产者幂等，防重发
    consumer:
      group-id: order-service
      auto-offset-reset: earliest      # 无 offset 时从头读
      enable-auto-commit: false        # 手动提交
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: com.example.demo.message   # 安全：限制反序列化包
    listener:
      ack-mode: manual_immediate       # 处理完立即提交
```

## 三、生产

```java
@Component
@RequiredArgsConstructor
public class EventProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public void send(String topic, String key, Object event) {
        // key 相同 → 同一分区 → 分区内有序（如同一订单的事件按 orderId 路由）
        kafkaTemplate.send(topic, key, event)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("kafka 发送失败 topic={} key={}", topic, key, ex);
                        // 落本地消息表补偿
                    }
                });
    }
}
```

## 四、消费

```java
@Component
@Slf4j
public class OrderEventListener {

    @KafkaListener(topics = "order-events", groupId = "order-service")
    public void onEvent(ConsumerRecord<String, OrderEvent> record, Acknowledgment ack) {
        log.info("partition={}, offset={}, key={}",
                record.partition(), record.offset(), record.key());
        try {
            handle(record.value());
            ack.acknowledge();
        } catch (Exception e) {
            log.error("消费失败 offset={}", record.offset(), e);
            // 不 ack → 重启后从该 offset 重放；或发到死信 topic
        }
    }

    // 监听指定分区/并发
    @KafkaListener(topics = "order-events", concurrency = "3")
    public void concurrent(ConsumerRecord<String, OrderEvent> record) { }
}
```

## 五、顺序保证

```
同一 key 的消息进同一分区（默认分区器按 key hash）
→ 分区内消费天然有序
→ 消费端 concurrency 最多设为分区数，且同一 key 的处理不能并发乱序
```

需要严格有序的流（如订单状态流转）：**按 orderId 作 key**，其余场景追求吞吐可无 key 轮询。

## 六、重平衡与消费组

- 同一 group 内消费者数 ≤ 分区数，多余的闲着
- rebalance 期间会停止消费 → 避免频繁扩缩容、max.poll.interval.ms 内完成处理

```yaml
      properties:
        max.poll.interval.ms: 300000      # 单批最大处理时间
        session.timeout.ms: 30000
```

## 七、exactly-once（简述）

- 生产端：幂等 producer（enable.idempotence）+ acks=all
- 消费端：`isolation.level=read_committed` 读事务消息
- 端到端：事务 API（`kafkaTemplate.executeInTransaction`）或消费处理 + 手动提交 offset 的「先处理后提交」
- **业务层面幂等仍是底线**，exactly-once 不能代替幂等设计

## 八、典型应用场景

| 场景 | 设计要点 |
|---|---|
| 日志采集 | Fire-and-forget + acks=1，吞吐优先 |
| 订单事件流 | key=orderId 保序，消费者幂等 |
| 削峰 | 生产快消费慢，利用积压缓冲 |
| CDC/事件溯源 | 用 binlog/领域事件填充 topic，可回放重建状态 |

## 常见坑

- **坑 1**：auto-commit=true + 处理失败 → 消息丢失；统一手动提交
- **坑 2**：concurrency > 分区数 → 多余线程空转
- **坑 3**：key 为 null 且分区扩容 → 同一数据顺序被打乱
- **坑 4**：JsonDeserializer 未配置 trusted packages → 反序列化失败（Boot 3 默认全信任有风险，要显式限制）
- **坑 5**：单条消息处理超时导致 rebalance → 调 max.poll.interval.ms / 减小 batch

## Checklist

- [ ] acks=all + 幂等 producer + 手动提交
- [ ] 需要有序的流按业务 key 路由
- [ ] 消费者幂等，失败有重放/死信路径
- [ ] trusted.packages 显式配置
