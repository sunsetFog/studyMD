# 38 · 事务消息与最终一致性

## 1. 问题：本地事务与消息发送的原子性

```java
@Transactional
public void createOrder() {
    orderMapper.insert(order);      // ① DB 写入
    mq.send(new OrderCreatedEvent()); // ② 发消息
}
```

- ①成功 ②失败 → 下游不知道订单已创建。
- ②成功 ①回滚 → 下游收到不存在的订单。
- 分布式事务不可用（MQ 不在本地事务内）。

## 2. 方案对比

| 方案 | 一致性 | 复杂度 | 适用 |
|---|---|---|---|
| 事务消息（RocketMQ） | 最终一致 | 低（框架支持） | 有 RMQ 时首选 |
| 本地消息表 | 最终一致 | 中 | 通用 |
| 事务提交后发送 | 弱（可能丢） | 极低 | 可容忍丢失 |
| 2PC/XA | 强一致 | 高、性能差 | 极少用 |
| Seata | 最终/强 | 高 | 见 47 |

## 3. 方案一：RocketMQ 事务消息

```
① 发半消息（Half，消费者不可见）
② 执行本地事务
③ COMMIT（可见）/ ROLLBACK（丢弃）
④ 网络断开/超时 → Broker 回查本地事务状态
```

```java
rocketMQ.sendMessageInTransaction("tx-order", MessageBuilder.withPayload(e).build(), orderId);

@RocketMQTransactionListener
class TxListener implements RocketMQLocalTransactionListener {
    @Override public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try { orderService.createInTx((Long) arg); return COMMIT; }
        catch (Exception e) { return ROLLBACK; }
    }
    @Override public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        Long orderId = extract(msg);
        return orderMapper.exists(orderId) ? COMMIT : ROLLBACK;    // ★ 回查要能判定
    }
}
```

> 关键：本地事务必须留下**可查询的状态**（如订单记录），回查才能判断。

## 4. 方案二：本地消息表（通用，最常用）

```
① 本地事务：写业务数据 + 写消息表（同一 DB，原子）
② 定时任务/线程扫描消息表未发送记录 → 发 MQ
③ 发送成功 → 标记已发送；失败 → 重试（带次数上限）
④ 下游消费做幂等
```

```sql
CREATE TABLE t_message (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  biz_key VARCHAR(64) NOT NULL COMMENT '业务唯一键',
  topic VARCHAR(128) NOT NULL,
  payload JSON NOT NULL,
  status TINYINT NOT NULL DEFAULT 0 COMMENT '0待发送 1已发送 2失败',
  retry_count INT DEFAULT 0,
  next_retry_time DATETIME,
  create_time DATETIME,
  UNIQUE KEY uk_biz (biz_key, topic)
);
```

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderDTO dto) {
    orderMapper.insert(order);                                  // 业务
    messageMapper.insert(new Message(orderNo, "order-created", payload));  // 消息，同库事务
}

@Scheduled(fixedDelay = 5000)
@SchedulerLock(name = "msgSend")            // 分布式锁防多实例重复
public void sendPending() {
    List<Message> list = messageMapper.selectPending(100);
    for (Message m : list) {
        try {
            mq.send(m.getTopic(), m.getPayload());
            messageMapper.markSent(m.getId());
        } catch (Exception e) {
            messageMapper.incrRetry(m.getId());      // 超过 N 次标记失败并告警
        }
    }
}
```

- 优点：无侵入 MQ、可靠、可追溯、可对账。
- 缺点：多一次 DB 写入；需要定时任务。
- 优化：用 `TransactionSynchronization#afterCommit` 立即尝试发送（失败再由定时任务兜底）。

## 5. 方案三：事务提交后发送（轻量）

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderDTO dto) {
    Long id = orderMapper.insert(dto);
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override public void afterCommit() {
            try { mq.send(new OrderCreatedEvent(id)); }
            catch (Exception e) { messageMapper.insertPending(id); }   // ★ 失败兜底落表
        }
    });
}
```

> 注意：`afterCommit` 后发送失败仍会丢消息，**必须兜底**（落表重试）。单独使用只适合可容忍丢失的场景（如通知类）。

## 6. 消费端：必须幂等

```java
// 幂等表
if (!Boolean.TRUE.equals(redis.opsForValue()
        .setIfAbsent("consumed:" + msgId, "1", Duration.ofDays(7)))) {
    return;                       // 已消费
}
// 或 DB 唯一索引
// 或状态机：UPDATE ... SET status=2 WHERE id=? AND status=1
```

| 幂等手段 | 适用 |
|---|---|
| 唯一键（消息 ID） | 通用 |
| 状态机 / CAS 更新 | 状态流转 |
| 乐观锁 `@Version` | 更新 |
| Redis SETNX | 高频、允许过期 |
| 去重表 | 金融级别 |

## 7. 消息乱序与补偿

- 顺序问题：用业务状态机（只接受合法状态迁移）+ 版本号。
- 补偿：定时任务扫描「长时间未完成」的业务，主动查询上游或对账。
- 对账：定期比对上下游数据（如订单 vs 库存流水），发现差异告警/自动修复。

## 8. 一致性等级速查

| 等级 | 手段 |
|---|---|
| 强一致 | 2PC / XA / Seata AT（性能差） |
| 最终一致（推荐） | 事务消息 / 本地消息表 + 幂等 + 重试 + 对账 |
| 弱一致 | 先发消息后提交事务（有风险，仅通知类） |
| 读修复 | 查询时校验并补偿 |

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 事务消息回查未实现 | 半消息永久挂起 | 必须实现 `checkLocalTransaction` |
| 回查查不到状态 | 误回滚 | 本地事务先落可查记录 |
| 消息表无限增长 | 表变大 | 定期归档已发送记录 |
| 定时任务多实例重复 | 重复发送 | 分布式锁 / ShedLock |
| 消费未幂等 | 数据重复 | 幂等表/状态机 |
| 只发一次不重试 | 丢消息 | 必须重试 + 死信 + 告警 |
| 大事务里发消息 | 连接占用 | afterCommit 后发 |
| 消息乱序 | 状态回退 | 状态机 + 版本号 |

## 10. 面试速答

- **如何保证本地事务与消息一致**：RocketMQ 事务消息（半消息 + 回查）或本地消息表（业务与消息同库事务 + 定时重投）。
- **本地消息表流程**：本地事务写业务 + 消息表 → 定时扫描发送 → 成功标记/失败重试 → 下游幂等。
- **`afterCommit` 够不够**：不够，发送仍可能失败，必须落表兜底重试。
- **消费端一定幂等**：MQ 只能保证 at least once，重复不可避免。
