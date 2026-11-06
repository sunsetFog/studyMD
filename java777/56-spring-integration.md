# 56 · Spring Integration（企业集成模式）

## 1. 定位

Spring Integration 是 **EIP（Enterprise Integration Patterns）** 的实现：以**消息驱动**的方式把异构系统（文件、MQ、HTTP、DB、邮件）连接起来。

> 与 Spring Cloud Stream 区别：Stream 是**消息中间件抽象**（面向流式数据）；Integration 是**系统间集成编排**（面向流程），功能更全（文件、FTP、邮件、TCP…）。

## 2. 三大核心概念

```
Message（消息）= Payload（载荷） + Headers（头）
Channel（通道）  消息流转的管道
Endpoint（端点） 通道两端的行为（过滤器/转换器/路由器/适配器…）
```

## 3. 依赖与启用

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-integration</artifactId>
</dependency>
<!-- 扩展：spring-integration-file / -ftp / -mail / -jdbc / -mqtt / -http -->
```

```java
@Configuration
@EnableIntegration
public class IntegrationConfig { }
```

## 4. 通道类型

| 通道 | 特点 |
|---|---|
| `DirectChannel` | 默认，**同一线程**直接调用（事务在同一线程内） |
| `QueueChannel` | 队列，**异步**，需 Poller 拉取 |
| `PublishSubscribeChannel` | 广播到多个订阅者 |
| `ExecutorChannel` | 线程池异步 |
| `PriorityChannel` | 按优先级 |
| `RendezvousChannel` | 阻塞直到接收 |

> 选择要点：`DirectChannel` 保证事务上下文；`QueueChannel` 会**丢失事务与安全上下文**（跨线程）。

## 5. Java DSL 示例

```java
@Configuration
@EnableIntegration
public class FileFlowConfig {

    @Bean
    public IntegrationFlow fileToDbFlow() {
        return IntegrationFlow
            .from(Files.inboundAdapter(new File("/data/in"))
                        .patternFilter("*.csv")
                        .preventDuplicates(true),
                  e -> e.poller(Pollers.fixedDelay(5000)))
            .transform(File.class, File::getName)
            .filter(String.class, n -> n.endsWith(".csv"))
            .handle(m -> service.importFile(m.getPayload()))
            .get();
    }

    // HTTP → 转换 → MQ
    @Bean
    public IntegrationFlow httpToMqFlow() {
        return IntegrationFlow
            .from(Http.inboundGateway("/ingest")
                      .requestMapping(m -> m.methods(HttpMethod.POST)))
            .transform(Transformers.toJson())
            .handle(Amqp.outboundAdapter(rabbitTemplate).routingKey("ingest"))
            .get();
    }
}
```

## 6. 常用端点（EIP）

| EIP | 实现 |
|---|---|
| Transformer（转换器） | `.transform(...)` |
| Filter（过滤器） | `.filter(...)` |
| Router（路由器） | `.route(...)` |
| Splitter（拆分器） | `.split(...)` |
| Aggregator（聚合器） | `.aggregate(...)` |
| Service Activator（服务激活器） | `.handle(...)` |
| Channel Adapter（通道适配器） | inbound/outbound |
| Gateway（网关） | 应用与集成流的桥接 |
| Bridge（桥） | 连接两个通道 |
| Delayer | 延迟 |
| Enricher（充实器） | 补充数据 |
| Idempotent Receiver | 幂等接收 |

```java
// 拆分 + 并行处理 + 聚合
@Bean
public IntegrationFlow splitAggregate() {
    return IntegrationFlow.from("input")
            .split()                                   // List → 单条
            .channel(c -> c.executor(taskExecutor()))   // 并行
            .transform(this::processOne)
            .aggregate()                                // 聚合回 List
            .handle(System.out::println)
            .get();
}
```

## 7. Messaging Gateway（应用调用集成流）

```java
@MessagingGateway(defaultRequestChannel = "input")
public interface OrderGateway {
    void publish(@Payload OrderEvent event, @Header("type") String type);
}
// 注入使用：orderGateway.publish(e, "CREATED");
```

## 8. 错误处理

```java
@Bean
public IntegrationFlow errorHandling() {
    return IntegrationFlow.from("errorChannel")
            .handle(m -> log.error("集成流异常", m.getPayload()));
}
```
- 每个流可配 `errorChannel`。
- 支持重试：`RequestHandlerRetryAdvice` / `.handle((GenericHandler<?> ..., e -> e.advice(retryAdvice()))`。

## 9. 幂等与去重

```java
@Bean
public IntegrationFlow idempotent() {
    return IntegrationFlow.from("input")
            .filter(new MetadataStoreSelector(new RedisMetadataStore(redisConnectionFactory)))
            .handle(...)
            .get();
}
```

## 10. 典型场景

| 场景 | 做法 |
|---|---|
| 文件导入 | `Files.inboundAdapter` + Poller + 转换器 + DB 写入 |
| FTP/SFTP 同步 | `Sftp.inboundAdapter` |
| 邮件通知 | `Mail.outboundAdapter` |
| DB 轮询变更 | `JdbcPollingChannelAdapter` |
| MQTT/IoT | `MqttPahoMessageDrivenChannelAdapter` |
| 协议桥接 | HTTP ↔ MQ ↔ TCP |
| 批处理编排 | Splitter + Aggregator |

## 11. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 事务丢失 | 异步通道导致 | 用 `DirectChannel` 保证同线程 |
| 消息丢失 | QueueChannel 无持久化 | 用持久化通道（`MessageStore`）或外部 MQ |
| Poller 配置错 | 不触发/过于频繁 | 配 `fixedDelay/fixedRate/cron` |
| 幂等未做 | 重复处理 | `MetadataStoreSelector` |
| 文件重复处理 | 重复导入 | `preventDuplicates(true)` |
| 错误无感知 | 静默失败 | 配 errorChannel 与告警 |
| DSL 复杂难维护 | 代码难读 | 拆分流，用 `@ServiceActivator` 注解式 |

## 12. 面试速答

- **Spring Integration 是什么**：EIP（企业集成模式）的 Spring 实现，通过 Message/Channel/Endpoint 把异构系统消息化集成。
- **通道类型**：Direct（同步、保事务）、Queue（异步、需 Poller）、PublishSubscribe（广播）、Executor（线程池）。
- **与 Spring Batch 区别**：Batch 面向**大批量数据处理**（事务、重启、分片）；Integration 面向**系统间实时集成与流程编排**。
- **EIP 常见模式**：转换、过滤、路由、拆分、聚合、服务激活、幂等接收、死信通道。
