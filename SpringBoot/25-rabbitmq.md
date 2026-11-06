# 25 · RabbitMQ 集成与可靠投递

## 一、集成

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
    publisher-confirm-type: correlated   # 发送确认
    publisher-returns: true              # 路由失败回调
    listener:
      simple:
        acknowledge-mode: manual         # 手动 ack
        retry:
          enabled: true
          max-attempts: 3
```

## 二、声明队列/交换机/绑定

```java
@Configuration
public class RabbitConfig {

    public static final String ORDER_EXCHANGE = "order.exchange";
    public static final String ORDER_QUEUE = "order.created.queue";
    public static final String DEAD_QUEUE = "order.dead.queue";

    @Bean
    public TopicExchange orderExchange() {
        return ExchangeBuilder.topicExchange(ORDER_EXCHANGE).durable(true).build();
    }

    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable(ORDER_QUEUE)
                .withArgument("x-dead-letter-exchange", "")     // 死信：消费失败进入
                .withArgument("x-dead-letter-routing-key", DEAD_QUEUE)
                .withArgument("x-message-ttl", 60000)
                .build();
    }

    @Bean
    public Binding orderBinding() {
        return BindingBuilder.bind(orderQueue()).to(orderExchange()).with("order.created");
    }
}
```

## 三、发送

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderMessageSender {

    private final RabbitTemplate rabbitTemplate;

    public void sendOrderCreated(OrderCreatedMessage msg) {
        CorrelationData cd = new CorrelationData(UUID.randomUUID().toString());
        rabbitTemplate.convertAndSend("order.exchange", "order.created", msg, m -> {
            m.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);  // 持久化
            m.getMessageProperties().setHeader("x-idempotent-id", cd.getId());
            return m;
        }, cd);
    }

    // 确认回调（broker 收到）
    @PostConstruct
    public void init() {
        rabbitTemplate.setConfirmCallback((cd, ack, cause) -> {
            if (!ack) {
                log.error("消息未确认 id={}, cause={}", cd != null ? cd.getId() : null, cause);
                // 记录表 → 补偿重发
            }
        });
        rabbitTemplate.setReturnsCallback(returned ->
                log.error("消息路由失败: {}", returned.getMessage()));
    }
}
```

## 四、消费

```java
@Component
@Slf4j
public class OrderMessageConsumer {

    @RabbitListener(queues = "order.created.queue")
    public void handle(OrderCreatedMessage msg, Channel channel,
                       @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
        try {
            // 幂等校验（消息 ID 表去重）
            if (idempotentService.isProcessed(msg.getMessageId())) {
                channel.basicAck(tag, false);
                return;
            }
            orderService.process(msg);
            channel.basicAck(tag, false);
        } catch (BusinessException e) {
            log.warn("业务异常，进入重试: {}", e.getMessage());
            channel.basicNack(tag, false, false);   // 不重回队列 → 死信队列
        } catch (Exception e) {
            log.error("消费异常", e);
            channel.basicNack(tag, false, false);
        }
    }
}
```

## 五、可靠性三问（面试高频）

| 环节 | 丢失风险 | 保障 |
|---|---|---|
| 生产者 → broker | 网络失败 | publisher confirm + 本地消息表重发 |
| broker 存储 | 未持久化宕机 | exchange/queue/message 三层持久化（镜像/仲裁队列） |
| broker → 消费者 | 自动 ack 后处理失败 | 手动 ack：处理完再 ack |

**本地消息表方案**（最终一致标准实现）：

```
业务事务内写入 message 表（同库同事务，保证原子性）
→ 定时任务扫描未发送/未确认消息 → 重发 → 确认后标记完成
```

## 六、典型模式

| 模式 | 用法 |
|---|---|
| 简单队列 | 一生产一消费 |
| 工作队列 | 多消费者，prefetch 控制分发 |
| 路由/主题 | 按 routing-key 分发（最常用） |
| 延迟队列 | TTL + 死信 或 rabbitmq-delayed-message-exchange 插件（订单超时关闭） |
| 死信队列 | 失败消息隔离排查 + 补偿 |

## 常见坑

- **坑 1**：自动 ack + 异常 → 消息丢失；统一 manual ack
- **坑 2**：basicNack(true) 无限重回队列 → 死循环；改 false 进死信
- **坑 3**：消息重复消费 → 消费者必须幂等（唯一索引/去重表/状态机）
- **坑 4**：发送对象无法反序列化 → 生产消费统一 Jackson 转换器：
  `rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter())`
- **坑 5**：prefetch 未设置 → 消息倾斜；`listener.simple.prefetch=10-50`

## Checklist

- [ ] 生产者 confirm + 持久化，消费者手动 ack
- [ ] 消费逻辑幂等
- [ ] 死信队列就位，失败消息可排查补偿
- [ ] JSON 消息转换器统一
