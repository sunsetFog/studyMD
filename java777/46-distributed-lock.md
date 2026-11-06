# 46 · 分布式锁：Redis / ZK / 数据库

## 1. 为什么需要

单机 `synchronized`/`ReentrantLock` 只锁当前 JVM；多实例部署时无效，需跨进程的**分布式锁**。

典型场景：防止超卖、定时任务单实例执行、幂等防重复提交、缓存重建防击穿。

## 2. 三种实现对比

| 方案 | 实现 | 优点 | 缺点 |
|---|---|---|---|
| Redis | `SET NX EX` / Redisson | 性能高、简单 | 主从切换可能丢锁，需续期 |
| Zookeeper | 临时顺序节点 | **强一致**、无超时问题 | 性能较低、需维护 ZK |
| 数据库 | 唯一索引 / `for update` | 简单、可靠 | 性能差、占连接 |

## 3. Redis 锁（主流）

### 手写要点

```java
// 加锁（原子）
Boolean ok = redis.opsForValue().setIfAbsent(key, token, Duration.ofSeconds(30));
// 解锁（Lua 保证原子：只能解自己的锁）
String script = "if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end";
```

四个必要条件：
1. **加锁原子**：`SET key val NX EX 30`（不能用 setnx + expire 两步）。
2. **value 唯一**：UUID/UUID+threadId，释放时校验。
3. **释放原子**：Lua 脚本（get + del 非原子会误删）。
4. **超时释放**：必须有过期时间，防死锁。

### Redisson（推荐）

```java
RLock lock = redissonClient.getLock("order:" + orderId);
boolean acquired = lock.tryLock(10, 30, TimeUnit.SECONDS);   // 等待 10s，租约 30s
if (!acquired) throw new BizException("操作频繁，请稍后重试");
try {
    // 业务逻辑（不能超过租约）
} finally {
    if (lock.isHeldByCurrentThread()) lock.unlock();
}
```

| 能力 | 说明 |
|---|---|
| **看门狗** | 不指定 leaseTime 时默认 30s，每 10s 自动续期，业务未完成不会过期 |
| 可重入 | hash 结构记录线程 ID + 重入次数 |
| 公平锁 | `getFairLock` |
| 读写锁 | `getReadWriteLock` |
| 联锁 | `getMultiLock`（多 Redis 实例，即 RedLock） |
| 信号量 | `getSemaphore` |

> RedLock 争议：Martin Kleppmann 与 antirez 有著名论战；官方（Redis）也不再强烈推荐。生产通常**单实例/哨兵/集群 + 业务幂等兜底**即可。

### 主从切换丢锁问题

```
客户端 A 在主库加锁 → 主库宕机，锁未同步到从库 → 从库升主 → 客户端 B 也能加锁 → 锁失效
```
缓解：
- 业务层加**幂等/状态机**（最终防线，必做）。
- 对强一致要求高的场景用 ZK 或 DB 锁。
- 使用 RedLock（多实例多数派，成本高且有争议）。

## 4. Zookeeper 锁

### 原理

```
1 在 /lock 下创建临时顺序节点：/lock/seq-00000001
2 获取所有子节点，判断自己是否最小
3 是 → 获得锁；否 → 监听前一个节点的删除事件（避免羊群效应）
4 业务完成 → 删除自己的节点（或 session 断开自动删除）
```

```java
// Curator（封装好了）
InterProcessMutex lock = new InterProcessMutex(client, "/locks/order/" + orderId);
if (lock.acquire(10, TimeUnit.SECONDS)) {
    try { doBiz(); } finally { lock.release(); }
}
```

- 优点：**临时节点**随 session 断开自动删除（无死锁）；顺序节点 + watch 前驱，公平锁，无惊群。
- 缺点：需要 ZK 集群；性能不如 Redis（写要过半确认）。

## 5. 数据库锁

### 悲观锁（行锁）

```sql
SELECT * FROM t_order WHERE id=? FOR UPDATE;   -- 必须在事务内，走索引
```
- 可靠但性能差，占连接，容易死锁。

### 乐观锁（推荐）

```sql
UPDATE t_order SET stock = stock - 1, version = version + 1
WHERE id = ? AND stock > 0 AND version = ?;
```
- 影响行数 = 0 说明冲突，重试或失败。
- 优点：无锁、并发高；缺点：冲突多时重试成本高。

### 唯一索引防重

```sql
INSERT INTO t_lock(biz_key) VALUES (?)   -- 唯一键冲突即抢锁失败
```

## 6. 选型建议

| 场景 | 推荐 |
|---|---|
| 高频、允许极小概率失效 | **Redis（Redisson）** |
| 强一致、低频（如资金） | ZK / DB 悲观锁 |
| 简单防重 | DB 唯一索引 |
| 更新类 | 乐观锁（版本号） |
| 定时任务单实例 | Redis 锁 / ShedLock |
| 已有 K8s | 可用 Lease 资源 |

> **最重要的原则**：锁是「尽力而为」的，最终防线永远是**业务幂等**（唯一索引、状态机）。

## 7. 锁粒度设计

- 锁到具体资源：`order:123`、`sku:456`，而非全局 `order`。
- 库存扣减：可按 `skuId` 分段（如 10 个段），降低冲突。
- 不要在锁内做远程调用（耗时超过租约会失效）。
- 锁内逻辑要短。

## 8. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 未设过期时间 | 死锁 | 必须用 `NX EX` |
| 释放他人锁 | 并发问题 | value 唯一 + Lua 校验 |
| 未在 finally 释放 | 泄漏 | `finally` + 判 `isHeldByCurrentThread` |
| 业务超时 > 租约 | 锁失效 | 看门狗续期 / 评估耗时 |
| 主从切换 | 双写 | 幂等兜底 / 换 ZK |
| 锁粒度太粗 | 性能差 | 细粒度 |
| 锁内远程调用 | 超时失效 | 移到锁外 |
| GC STW | 租约过期 | 增大租约 / 看门狗 |
| 数据库锁不走索引 | 锁表 | 确保索引命中 |

## 9. 面试速答

- **Redis 分布式锁要点**：`SET NX EX` 原子加锁、唯一 value、Lua 原子解锁、看门狗自动续期、finally 释放。
- **Redisson 看门狗**：不指定 leaseTime 时默认 30s 租约，每 10s 续期；线程挂掉后续期停止，30s 后自动释放。
- **ZK 锁原理**：临时顺序节点 + 监听前驱节点；临时节点随 session 断开自动删除，天然防死锁。
- **主从切换丢锁怎么办**：接受它，用业务幂等兜底；强一致场景改 ZK/DB 锁。
- **乐观锁 vs 悲观锁**：冲突少用乐观（版本号），冲突多用悲观（`for update`）。
