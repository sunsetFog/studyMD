# 34 · RabbitMQ 与 Spring AMQP

## 1. 核心模型

```
Producer → Exchange → (Binding/RoutingKey) → Queue → Consumer
```

| 组件 | 说明 |
|---|---|
| Exchange | 交换机，接收生产者消息并按规则路由 |
| Queue | 队列，存储消息 |
| Binding | 交换机与队列的绑定关系 |
| RoutingKey | 路由键 |
| VHost | 虚拟主机，逻辑隔离 |
| Channel | 复用 TCP 连接的轻量通道 |

### 四种交换机

| 类型 | 规则 | 场景 |
|---|---|---|
| Direct | RoutingKey 完全匹配 | 点对点 |
| Topic | 通配符 `*` 一个词、`#` 零或多个 | **最常用**，`order.#` |
| Fanout | 广播到所有绑定队列，忽略 RoutingKey | 广播通知 |
| Headers | 按 header 匹配（少用） | |

## 2. 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    connection-timeout: 5s
    # 生产者
    publisher-confirm-type: correlated    # ★ 开启发送确认（异步回调）
    publisher-returns: true               # 路由失败退回
    template:
      mandatory: true                     # 配合 returns
      retry:
        enabled: true
        max-attempts: 3
        initial-interval: 1s
    # 消费者
    listener:
      simple:
        acknowledge-mode: manual          # ★ 手动 ACK
        prefetch: 50                      # ★ 每个消费者未确认上限（限流）
        concurrency: 3                    # 最小消费者数
        max-concurrency: 10
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1s
          multiplier: 2
          max-interval: 10s
```

## 3. 声明队列/交换机/绑定

```java
@Configuration
public class RabbitConfig {
    public static final String EX_ORDER = "ex.order";
    public static final String Q_CREATE = "q.order.create";
    public static final String Q_DLX = "q.order.dlx";

    @Bean public TopicExchange orderExchange() { return new TopicExchange(EX_ORDER, true, false); }
    @Bean public Queue createQueue() {
        return QueueBuilder.durable(Q_CREATE)
                .withArgument("x-dead-letter-exchange", EX_ORDER)
                .withArgument("x-dead-letter-routing-key", "order.dead")
                .withArgument("x-message-ttl", 60000)
                .build();
    }
    @Bean public Binding bindCreate() { return BindingBuilder.bind(createQueue()).to(orderExchange()).with("order.create"); }

    // 死信队列
    @Bean public Queue dlxQueue() { return QueueBuilder.durable(Q_DLX).build(); }
    @Bean public Binding bindDlx() { return BindingBuilder.bind(dlxQueue()).to(orderExchange()).with("order.dead"); }
}
```

> 声明是**幂等**的：已存在且参数不一致会报错（注意改参数要先删）。

## 4. 发送消息

```java
@Service
@RequiredArgsConstructor
public class OrderProducer {
    private final RabbitTemplate rabbit;

    public void send(OrderCreatedEvent e) {
        CorrelationData cd = new CorrelationData(e.eventId());    // ★ 唯一 ID，用于确认
        rabbit.convertAndSend(EX_ORDER, "order.create", e, m -> {
            m.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);  // 持久化
            m.getMessageProperties().setExpiration("60000");
            return m;
        }, cd);
    }
}
```

### 发送确认（Confirm / Return）

```java
@Component
public class RabbitConfirm implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnsCallback {
    @PostConstruct
    public void init(RabbitTemplate t) {
        t.setConfirmCallback(this);
        t.setReturnsCallback(this);
    }
    @Override public void confirm(CorrelationData cd, boolean ack, String cause) {
        if (!ack) { log.error("发送失败 id={} cause={}", cd.getId(), cause); /* 落库/重发 */ }
    }
    @Override public void returnedMessage(ReturnedMessage m) {
        log.error("路由失败 msg={} replyText={}", m.getMessage(), m.getReplyText());
    }
}
```

## 5. 消费消息

```java
@Component
@Slf4j
public class OrderConsumer {

    @RabbitListener(queues = Q_CREATE, concurrency = "3-10")
    public void handle(Message message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
        try {
            OrderCreatedEvent e = JsonUtil.fromBytes(message.getBody(), OrderCreatedEvent.class);
            doBiz(e);
            channel.basicAck(tag, false);          // ★ 手动确认
        } catch (Exception ex) {
            log.error("消费失败", ex);
            // 重试次数超限后拒绝（进入死信）
            channel.basicNack(tag, false, false);  // requeue=false → 死信/丢弃
        }
    }
}
```

自动 ACK 模式：方法正常返回即 ACK，抛异常则根据 `retry` 配置重试或拒绝。

## 6. 可靠性三要素

| 环节 | 风险 | 方案 |
|---|---|---|
| 生产者 → Broker | 网络丢消息 | **Publisher Confirm** + 失败重试 + 落库（消息表） |
| Broker 存储 | 宕机丢消息 | **持久化**（交换机/队列/消息）+ 镜像队列/Quorum Queue |
| Broker → 消费者 | 消费失败/崩溃 | **手动 ACK** + 重试 + 死信 + 幂等 |

### 消息丢失排查清单

1. 交换机、队列、消息三者都持久化了吗？
2. `publisher-confirm-type: correlated` 开了吗？
3. 消费者手动 ACK 了吗？
4. 自动 ACK 模式下异常会无限重投吗？

## 7. 消费者幂等

MQ 只保证 **at least once**，重复消费不可避免。

```java
// 幂等表（消息 ID 唯一索引）
if (!Boolean.TRUE.equals(redis.opsForValue().setIfAbsent("msg:" + msgId, "1", Duration.ofDays(7)))) {
    channel.basicAck(tag, false);     // 已处理，直接确认
    return;
}
doBiz();
```

其他幂等手段：数据库唯一索引、乐观锁版本号、状态机（`status` 前置判断）。

## 8. 死信队列（DLX）

消息变死信的三种情况：
1. 被 `basicNack`/`basicReject` 且 `requeue=false`
2. 消息 TTL 过期
3. 队列达到最大长度

用途：**延迟消息**（TTL + DLX）、失败隔离、人工补偿。

### 延迟消息

```java
// 方案1：TTL + DLX（简单，但队列内消息 FIFO，不同 TTL 会互相阻塞）
// 方案2：rabbitmq_delayed_message_exchange 插件（推荐）
rabbit.convertAndSend(EX_DELAYED, "key", msg, m -> {
    m.getMessageProperties().setHeader("x-delay", 30_000);   // 延迟 30s
    return m;
});
// 交换机类型：x-delayed-message
```

## 9. 顺序性

RabbitMQ 保证**单队列内的顺序**（FIFO），但：
- 多个消费者并发 → 乱序。
- 重试/死信 → 乱序。

保证顺序：
- 单队列 + **单消费者**（concurrency=1）+ prefetch=1（吞吐最低）。
- 或业务上用**状态机/版本号**容忍乱序（推荐）。
- 或按业务键 hash 到不同队列，每个队列单消费者。

## 10. 消息堆积处理

| 现象 | 处理 |
|---|---|
| 消费慢 | 增加消费者（加实例/concurrency）、批量消费 |
| 消费者故障 | 修复后加速消费 |
| 峰值堆积 | 临时扩容消费者 + 降级 |
| 长期堆积 | 检查是否有死循环/异常频繁重试 |

> 紧急方案：新建临时队列，把堆积消息快速转存，再用 10 倍消费者慢慢处理。

## 11. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 自动 ACK 丢消息 | 消费一半崩溃 | 改手动 ACK |
| 异常无限重试 | 队列被毒丸消息堵死 | 重试次数限制 + 死信 |
| `prefetch` 不设 | 消费者内存爆 | 设 50~200 |
| 消息重复消费 | 数据重复 | 幂等 |
| 消息顺序错乱 | 状态回退 | 单消费者或状态机 |
| 连接断开 | 消费者停止 | `CachingConnectionFactory` 自动重连；监听 `ListenerContainerConsumerFailedEvent` |
| 队列参数改了 | 启动报 `PRECONDITION_FAILED` | 删除重建或改代码 |
| 事务消息性能 | 很慢 | 用 Confirm 代替 tx |

## 12. 面试速答

- **如何保证消息不丢**：生产者 Confirm + 持久化（交换机/队列/消息）+ 消费者手动 ACK。
- **如何保证不重复消费**：MQ 只保证至少一次，业务层做幂等（唯一 ID 表/Redis/乐观锁/状态机）。
- **消息堆积怎么办**：先定位（消费慢/故障/峰值），扩容消费者、批量消费、临时队列转存。
- **延迟消息怎么实现**：TTL+DLX 或 `x-delayed-message` 插件。
- **四种交换机**：Direct（精确）、Topic（通配，最常用）、Fanout（广播）、Headers。
