# 36 · RocketMQ

## 1. 核心模型

```
Producer → Topic → Queue(多个) → ConsumerGroup
NameServer（路由发现，无状态）  Broker（存储，主从）  Producer/Consumer
```

| 概念 | 说明 |
|---|---|
| NameServer | 轻量级注册中心（无状态，集群间不通信），管理 Topic 路由 |
| Broker | 消息存储与转发，Master/Slave |
| Topic | 一类消息的集合 |
| Tag | **二级分类**，用于消息过滤（如 `TagA`） |
| Message Queue | Topic 下的物理分区（读写队列） |
| Producer Group / Consumer Group | 生产者组 / 消费者组 |
| Offset | 消费进度（集群模式存 Broker，广播模式存本地） |

> 与 Kafka 的区别：RocketMQ 有 **NameServer**（Kafka 用 ZK/KRaft）；支持 **Tag 过滤、事务消息、延时消息、消息重试、顺序消息**。

## 2. 依赖与配置

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.3.1</version>
</dependency>
```

```yaml
rocketmq:
  name-server: localhost:9876
  producer:
    group: order-producer-group
    send-message-timeout: 3000
    retry-times-when-send-failed: 2
    retry-times-when-send-async-failed: 2
    max-message-size: 4194304
  consumer:
    group: order-consumer-group
    # 由注解指定
```

## 3. 三种发送方式

```java
@RestController
@RequiredArgsConstructor
public class ProducerDemo {
    private final RocketMQTemplate rocketMQ;

    // ① 同步（可靠，最常用）
    public void sync() {
        SendResult r = rocketMQ.syncSend("order-topic:create",   // topic:tag
                MessageBuilder.withPayload(new OrderCreatedEvent(1L)).build());
        log.info("status={} msgId={}", r.getSendStatus(), r.getMsgId());
    }

    // ② 异步（高吞吐）
    public void async() {
        rocketMQ.asyncSend("order-topic", payload, new SendCallback() {
            public void onSuccess(SendResult r) { log.info("ok"); }
            public void onException(Throwable e) { log.error("fail", e); }
        });
    }

    // ③ 单向（最快，可能丢）
    public void oneway() { rocketMQ.sendOneWay("log-topic", payload); }

    // 顺序发送：按 shardingKey hash 到同一队列
    public void order() {
        rocketMQ.syncSendOrderly("order-topic", payload, String.valueOf(orderId));
    }

    // 延时消息
    public void delay() {
        Message<OrderEvent> msg = MessageBuilder.withPayload(e).build();
        rocketMQ.syncSend("order-topic:timeout", msg, 3000,
                3);   // ★ 第 5 个参数是 delayLevel（3 = 10s）
    }
}
```

### 延时级别（固定，不支持任意时间）

`1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h`
对应 level `1~18`。任意时间需自行用定时扫表或时间轮。

## 4. 消费

```java
@Service
@Slf4j
@RocketMQMessageListener(
        topic = "order-topic",
        consumerGroup = "order-consumer-group",
        selectorExpression = "create || paid",      // ★ Tag 过滤，* 表示全部
        consumeMode = ConsumeMode.CONCURRENTLY,     // ORDERLY 为顺序消费
        messageModel = MessageModel.CLUSTERING,     // BROADCASTING 广播
        maxReconsumeTimes = 3
)
public class OrderConsumer implements RocketMQListener<OrderEvent> {
    @Override
    public void onMessage(OrderEvent e) {
        // 抛异常即触发重试；重试达到 maxReconsumeTimes 后进入死信队列
        handle(e);
    }
}
```

顺序消费需 `consumeMode = ORDERLY` + `syncSendOrderly`，且失败时**不会跳到下一批**（会阻塞重试）。

## 5. 消息类型

| 类型 | 说明 |
|---|---|
| 普通消息 | 默认 |
| 顺序消息 | 全局/分区顺序（`ORDERLY` + hashKey） |
| 延时消息 | 18 个固定级别 |
| **事务消息** | 半消息 + 本地事务 + 回查（见 [38](38-transactional-message.md)） |
| 批量消息 | 一次发送多条（总大小 < 4MB，同一 topic） |
| 过滤消息 | Tag（服务端）或 SQL92 属性过滤（需开启 `enablePropertyFilter`） |

## 6. 事务消息（重点）

```
① 发送半消息（Half Message，对消费者不可见）
② Broker 返回 OK → 执行本地事务
③ 本地事务成功 → Commit（可见）；失败 → Rollback（丢弃）
④ 若 Broker 长时间未收到二次确认 → 回查本地事务状态（默认 1 分钟后，最多 15 次）
```

```java
// 生产者
rocketMQ.sendMessageInTransaction("tx-topic", MessageBuilder.withPayload(e).build(), arg);

@RocketMQTransactionListener
class TxListener implements RocketMQLocalTransactionListener {
    @Override public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try { createOrder(); return RocketMQLocalTransactionState.COMMIT; }
        catch (Exception ex) { return RocketMQLocalTransactionState.ROLLBACK; }
    }
    @Override public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        return orderExists(msg) ? COMMIT : ROLLBACK;    // ★ 回查必须实现，需查本地事务状态
    }
}
```

## 7. 消费重试与死信

- 消费失败 → 进入 **重试队列** `%RETRY%consumerGroup`，延时级别递增（10s → 30s → 1m…）。
- 超过 `maxReconsumeTimes`（默认 16）→ 进入 **死信队列** `%DLQ%consumerGroup`。
- 死信消息需人工处理或另外监听 DLQ 进行补偿。

## 8. 消息存储

- **CommitLog**：所有 topic 消息顺序写入同一个文件（顺序 I/O，快）。
- **ConsumeQueue**：逻辑队列索引（定长 20 字节：offset + size + tagHashCode）。
- **IndexFile**：按 key 的哈希索引。
- 刷盘：同步刷盘（可靠、慢）/ 异步刷盘（默认，快）。
- 主从：同步复制 / 异步复制。

## 9. 堆积与排查

```bash
mqadmin consumerProgress -n localhost:9876 -g order-consumer-group    # 看延迟
mqadmin topicStatus -n localhost:9876 -t order-topic
mqadmin updateTopic -n localhost:9876 -t order-topic -c DefaultCluster -w 16   # 扩队列
```

| 现象 | 处理 |
|---|---|
| 消费能力不足 | 加队列 + 加消费者实例（**队列数 ≥ 消费者数**） |
| 消费阻塞 | 查异常、外部依赖 |
| 顺序消息卡住 | 单条失败会阻塞整个队列，需快速失败 |
| 历史消息堆积 | 跳过堆积位点（谨慎）或扩容追平 |

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| Tag 写错 | 消费不到 | `selectorExpression` 与发送 tag 一致；`*` 默认全部 |
| 顺序消息不生效 | 乱序 | 必须 `syncSendOrderly` + `ORDERLY` |
| 事务消息不回查 | 卡在半消息 | 实现 `checkLocalTransaction`，且本地事务要留可查记录 |
| 延时不准 | 只能选固定级别 | 用 level 或自行实现 |
| 消费者数量 > 队列数 | 部分消费者空闲 | 增加读写队列数 |
| 消息体过大 | 发送失败 | 拆分或只发 ID |
| `connect to nameserver failed` | 连不上 | 检查 9876 与 broker 注册 IP（`brokerIP1`） |
| 重复消费 | 数据重复 | 幂等 |

## 11. 面试速答

- **RocketMQ 与 Kafka 区别**：RMQ 有 NameServer（轻量）、支持事务消息/延时消息/Tag 过滤/消息重试/DLQ；Kafka 吞吐更高、生态偏向流处理。
- **事务消息原理**：半消息（对消费者不可见）→ 本地事务 → Commit/Rollback → 超时回查。
- **顺序消息**：`syncSendOrderly`（同 hashKey 进同队列）+ 消费者 `ORDERLY` 模式 + 单线程消费该队列，失败会阻塞重试。
- **存储为什么快**：所有消息顺序写 CommitLog + ConsumeQueue 索引 + 页缓存 + mmap。
