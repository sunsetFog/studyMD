# 37 · Spring Cloud Stream（Binder 抽象）

## 1. 定位

Spring Cloud Stream 是**消息中间件的统一抽象层**：应用代码只面向 `Binder` 编程，切换 RabbitMQ/Kafka/RocketMQ 只改依赖和配置。

```
应用 → Binding（输入/输出通道） → Binder（RabbitMQ / Kafka / RocketMQ）
```

| 概念 | 说明 |
|---|---|
| Binder | 与具体 MQ 的适配实现 |
| Binding | 通道与外部 MQ 目的地（topic/queue）的桥接 |
| Destination | 目标（topic / exchange） |
| Group | 消费组（同一组内竞争消费） |
| Partition | 分区支持 |

> 现状：Spring Cloud Stream 3.x 主推**函数式编程模型**，`@EnableBinding` / `@StreamListener` 已废弃。
> 轻量项目其实直接用 `spring-kafka` / `spring-rabbit` 更简单；Stream 适合需要**多 MQ 可切换**的中台场景。

## 2. 依赖（以 Kafka 为例）

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-kafka</artifactId>
</dependency>
<!-- 或 spring-cloud-stream-binder-rabbit -->
```

## 3. 函数式模型（推荐）

```java
@Configuration
public class StreamConfig {

    // ① 消费者：Function
    @Bean
    public Consumer<OrderCreatedEvent> orderCreated() {
        return e -> { log.info("收到 {}", e); service.handle(e); };
    }

    // ② 生产者：Supplier（定时/轮询产生消息）
    @Bean
    public Supplier<Heartbeat> heartbeat() {
        return () -> new Heartbeat(System.currentTimeMillis());
    }

    // ③ 消费者+生产者：Function（处理后再发）
    @Bean
    public Function<OrderEvent, PayEvent> orderToPay() {
        return e -> new PayEvent(e.orderId(), e.amount());
    }
}
```

```yaml
spring:
  cloud:
    function:
      definition: orderCreated;heartbeat;orderToPay   # ★ 必须声明才会创建 binding
    stream:
      bindings:
        # 约定：{函数名}-in-0 / {函数名}-out-0
        orderCreated-in-0:
          destination: order-events       # topic
          group: order-service            # 消费组（无 group 则为临时匿名）
        heartbeat-out-0:
          destination: heartbeat
        orderToPay-in-0:
          destination: order-events
          group: transform-g
        orderToPay-out-0:
          destination: pay-events
      kafka:
        binder:
          brokers: localhost:9092
        bindings:
          orderCreated-in-0:
            consumer:
              concurrency: 3
              autoCommitOffset: false
```

> 多个函数时必须配置 `spring.cloud.function.definition`，否则只会自动创建一个。

## 4. 发送（StreamBridge）

```java
@Service
@RequiredArgsConstructor
public class EventPublisher {
    private final StreamBridge bridge;

    public void publish(OrderCreatedEvent e) {
        boolean ok = bridge.send("orderCreated-out-0", e);
        if (!ok) log.error("发送失败");
    }

    // 动态目的地
    public void publishTo(String topic, Object payload) {
        bridge.send(topic, MessageBuilder.withPayload(payload)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build());
    }
}
```

## 5. 常用配置

```yaml
spring:
  cloud:
    stream:
      default-binder: kafka                 # 多 binder 时指定默认
      bindings:
        xxx-in-0:
          destination: topic
          group: g1
          consumer:
            concurrency: 3
            max-attempts: 3                # 重试次数
            back-off-initial-interval: 1000
            back-off-multiplier: 2
          content-type: application/json
        xxx-out-0:
          destination: topic
          producer:
            partition-count: 3
            partition-key-expression: headers['partitionKey']
```

## 6. 错误处理与 DLQ

```java
@Bean
public Consumer<Message<String>> orderCreated() {
    return msg -> { throw new RuntimeException("boom"); };
}

// 全局错误处理
@Bean
public Consumer<ErrorMessage> errorChannelHandler() {
    return em -> log.error("处理失败 payload={}", em.getPayload(), em.getPayload().getPayload());
}
```
```java
@Configuration
public class ErrorConfig {
    @Bean
    @ServiceActivator(inputChannel = "order-events.order-service.errors")
    public MessageHandler handler() {
        return m -> { /* 记录/补偿 */ };
    }
}
```

Kafka binder 支持 DLQ：
```yaml
spring.cloud.stream.kafka.bindings.orderCreated-in-0.consumer:
  enableDlq: true
  dlqName: order-events.DLT
  autoCommitOnError: true
```

重试耗尽后发到 DLQ；也可配合 `RetryTemplate`。

## 7. 多 Binder

```xml
<!-- 同时引入 kafka 与 rabbit binder -->
```
```yaml
spring:
  cloud:
    stream:
      binders:
        kafka1:
          type: kafka
          environment:
            spring.kafka.bootstrap-servers: localhost:9092
        rabbit1:
          type: rabbit
          environment:
            spring.rabbitmq.host: localhost
      bindings:
        xxx-in-0:
          destination: t1
          binder: kafka1
        yyy-out-0:
          destination: t2
          binder: rabbit1
```

## 8. 分区（Partitioning）

```yaml
spring.cloud.stream.bindings.output.producer:
  partition-key-expression: payload.userId      # 或 headers['key']
  partition-count: 3
spring.cloud.stream.bindings.input.consumer:
  partitioned: true
  instance-index: 0
  instance-count: 3
```
> 保证同一 key 进同一分区，配合消费者实例索引实现顺序处理。

## 9. 与原生客户端对比

| 维度 | Spring Cloud Stream | spring-kafka / amqp |
|---|---|---|
| 抽象 | 高，可换 MQ | 低，绑定具体 MQ |
| 灵活度 | 低（难以用 MQ 特性如 Tag、事务消息） | 高 |
| 学习成本 | 中 | 低 |
| 推荐场景 | 多 MQ 适配、中台 | 大多数业务项目 |

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 函数不生效 | 无 binding | 配 `spring.cloud.function.definition` |
| 消费者收不到 | topic/group 错 | 核对 `destination` 与 `group` |
| 多函数冲突 | 只创建一个 | 显式声明 definition |
| 序列化失败 | 反序列化错误 | 配 `content-type` 与 `use-native-encoding` |
| `@EnableBinding` 找不到 | 版本问题 | 3.x 已移除，用函数式 |
| 重试无效 | 异常未抛 | 必须抛异常才触发重试 |
| 消费者组未设 | 广播重复消费 | 设 `group`（Kafka）；Rabbit 会创建匿名队列 |

## 11. 面试速答

- **Spring Cloud Stream 解决了什么**：屏蔽不同 MQ 的 API 差异，通过 Binder 抽象让应用代码与具体消息中间件解耦。
- **函数式模型**：`Consumer`（消费）、`Supplier`（生产）、`Function`（转换），通过 `{函数名}-in-0` / `-out-0` 约定绑定。
- **如何发送**：`StreamBridge.send(bindingName, payload)`。
- **什么时候不用它**：需要使用 MQ 特有功能（RocketMQ 事务消息/Tag、Rabbit 优先级）时，直接用原生客户端更合适。
