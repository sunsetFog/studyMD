# 27 · Redis 典型场景：穿透/击穿/雪崩、分布式锁、限流

## 1. 缓存读写策略

### Cache Aside（旁路缓存，最常用）

```
读：命中缓存返回 → 未命中读 DB → 写入缓存 → 返回
写：更新 DB → 删除缓存
```

> **为什么是删缓存而不是更新缓存**：更新缓存代价高（可能计算复杂且没人读），且有并发写导致脏数据风险。

### 其他策略

| 策略 | 说明 | 适用 |
|---|---|---|
| Read/Write Through | 应用只跟缓存交互，缓存负责写 DB | 少用 |
| Write Behind（异步回写） | 先写缓存，异步批量落 DB | 高吞吐、可容忍丢失 |

### 一致性问题与缓解

| 方案 | 说明 |
|---|---|
| 延迟双删 | 删缓存 → 更新 DB → sleep 500ms → 再删 |
| 事务提交后删 | `@TransactionalEventListener(AFTER_COMMIT)` |
| MQ 重试删 | 删除失败进 MQ 重试 |
| binlog 订阅（Canal） | 异步保证最终一致 |
| 设置较短 TTL | 兜底 |

> 强一致场景不要靠缓存，直接读 DB 或用分布式锁。

## 2. 缓存穿透（查不存在的 key）

**现象**：恶意请求 id=-1 / 不存在的用户，缓存永远不命中，全部打 DB。

**方案**：
```java
// ① 缓存空值（简单）
User u = userMapper.selectById(id);
redis.opsForValue().set(key, u == null ? NULL_PLACEHOLDER : u, Duration.ofMinutes(2));  // 空值短 TTL

// ② 布隆过滤器（推荐，前置拦截）
RBloomFilter<Long> bf = redisson.getBloomFilter("userBF");
bf.tryInit(1_000_000L, 0.01);          // 预期 100 万，误判率 1%
bf.add(id);
if (!bf.contains(id)) return null;     // 一定不存在，直接返回

// ③ 参数校验 + 限流（id <= 0 直接拒绝）
```

> 布隆过滤器：不存在则**一定**不存在（无漏判）；存在则**可能**存在（有误判）。不支持删除（用 Counting Bloom / Cuckoo Filter）。

## 3. 缓存击穿（热点 key 过期）

**现象**：某个极热 key 过期瞬间，成千上万请求同时打到 DB。

```java
// 方案1：互斥锁（强一致，有等待）
public Product get(Long id) {
    String key = "p:" + id;
    Product p = (Product) redis.opsForValue().get(key);
    if (p != null) return p;
    RLock lock = redisson.getLock("rebuild:" + id);
    try {
        if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
            p = (Product) redis.opsForValue().get(key);     // 双重检查
            if (p != null) return p;
            p = mapper.selectById(id);
            redis.opsForValue().set(key, p, Duration.ofMinutes(30));
            return p;
        } else {
            Thread.sleep(50);
            return get(id);                                  // 或返回降级数据
        }
    } finally { if (lock.isHeldByCurrentThread()) lock.unlock(); }
}

// 方案2：逻辑过期（无等待，高可用）
// 缓存中存 {data, expireAt}，物理 TTL 设很长；发现逻辑过期则拿锁异步重建，先返回旧数据

// 方案3：热点 key 永不过期 + 后台定时刷新
```

## 4. 缓存雪崩（大量 key 同时失效）

**现象**：同一时刻大批 key 过期，或 Redis 集群宕机，请求全部打 DB。

方案：
1. **TTL 加随机值**：`baseTTL + Random.nextInt(300)` 秒。
2. **多级缓存**：本地 Caffeine + Redis。
3. **Redis 高可用**：哨兵/集群、持久化、限流降级。
4. **熔断降级**：Redis 不可用时走本地缓存或直接限流。
5. **预热**：启动时加载热点数据。

## 5. 分布式锁

### 演进

```java
// v1：错误 —— setnx + expire 非原子
if (setnx(k,1)) { expire(k,30); ... }     // setnx 后宕机 → 死锁

// v2：正确 —— 原子 set NX EX
redis.opsForValue().setIfAbsent(k, uuid, Duration.ofSeconds(30));

// v3：错误 —— 释放时未校验持有者
del(k);                                    // 可能删掉别人的锁（业务超时导致）

// v4：正确 —— Lua 校验后删除
if get(k) == uuid then return del(k) end

// v5：正确 —— 自动续期（看门狗）
RLock lock = redisson.getLock(k);
lock.tryLock(10, 30, TimeUnit.SECONDS);    // 默认 watchdog 30s，每 10s 续期
```

```java
RLock lock = redisson.getLock("order:" + orderId);
boolean ok = lock.tryLock(5, 30, TimeUnit.SECONDS);
if (!ok) throw new BizException("操作过于频繁");
try { doBiz(); }
finally { lock.unlock(); }                 // ★ 必须 finally，且加锁与解锁在同一线程
```

### Redisson 看门狗

- 不传 `leaseTime` 时启用：默认 30s，每 10s（1/3）自动续期。
- 线程持有锁期间定期续期，业务未完成时不会过期。
- 线程挂掉 → 续期停止 → 30s 后自动释放。

### 可重入 / 公平锁 / 联锁

```java
lock.lock();  lock.lock();     // 可重入（hash 结构记录线程 + 重入次数）
redisson.getFairLock("f");     // 公平锁
redisson.getMultiLock(l1, l2, l3);   // RedLock（多实例，官方已不推荐）
```

### 注意事项

| 问题 | 说明 |
|---|---|
| 主从切换丢锁 | 主写锁未同步即宕机 → 从升主后锁丢失 | 
| 解决方案 | 用 RedLock（争议大）或改用 DB/ZK 锁、或业务层幂等兜底 |
| 锁粒度 | 锁到具体资源（`order:123`）而非全局 |
| 锁超时 | 必须设 leaseTime；GC STW 可能导致误释放 |

### 与数据库锁对比

| 方案 | 优点 | 缺点 |
|---|---|---|
| Redis 锁 | 性能高 | 不可靠（主从切换）、需续期 |
| DB 悲观锁 `for update` | 可靠、简单 | 性能差、占连接 |
| DB 乐观锁 `@Version` | 无锁、并发高 | 冲突需重试 |
| ZK 锁 | 强一致 | 性能较低、需维护 ZK |

## 6. 限流

### 固定窗口计数器（Redis）

```java
String key = "rl:" + userId + ":" + (System.currentTimeMillis() / 1000);
Long n = redis.opsForValue().increment(key);
if (n == 1) redis.expire(key, 1, TimeUnit.SECONDS);
if (n > 100) throw new BizException("请求过于频繁");
```
缺点：临界突刺（窗口切换瞬间可打 2 倍）。

### 滑动窗口（ZSet）

```java
long now = System.currentTimeMillis();
redis.opsForZSet().add(key, now + "", now);
redis.opsForZSet().removeRangeByScore(key, 0, now - 60_000);
Long cnt = redis.opsForZSet().zCard(key);
redis.expire(key, 60, TimeUnit.SECONDS);
```

### 令牌桶（Lua / Redisson）

```java
RRateLimiter limiter = redisson.getRateLimiter("api:limit");
limiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.SECONDS);   // 100 QPS
if (!limiter.tryAcquire()) throw new BizException("限流");
```

### 漏桶 vs 令牌桶

| 算法 | 特点 |
|---|---|
| 漏桶 | 恒定速率出水，**平滑**，无法应对突发 |
| 令牌桶 | 恒定速率生成令牌，**允许突发**（桶内有令牌即可） |

## 7. 延迟队列 / 延时任务

```java
// 方案1：ZSet（score = 执行时间戳）
redis.opsForZSet().add("delay", taskId, executeAt);
Set<ZSetOperations.TypedTuple<Object>> ready =
        redis.opsForZSet().rangeByScoreWithScores("delay", 0, System.currentTimeMillis());
// 取出后 ZREM 保证原子（用 Lua）

// 方案2：Redisson RDelayedQueue
RQueue<String> q = redisson.getQueue("target");
RDelayedQueue<String> dq = redisson.getDelayedQueue(q);
dq.offer("task", 30, TimeUnit.SECONDS);

// 方案3：Redis Stream / Key 过期回调（不可靠，不推荐生产）
```

## 8. 计数器与排行榜

```java
// UV 去重：HyperLogLog（误差 0.81%，内存固定 12KB）
redis.opsForHyperLogLog().add("uv:20240101", userId);
redis.opsForHyperLogLog().size("uv:20240101");

// 签到/活跃位图：Bitmap（极省内存）
redis.opsForValue().setBit("sign:1:202401", dayOfMonth, true);
redis.opsForValue().bitCount("sign:1:202401");

// 排行榜：ZSet
redis.opsForZSet().reverseRangeWithScores("rank", 0, 9);
redis.opsForZSet().reverseRank("rank", "user1");     // 名次
```

## 9. 分布式 Session

```xml
<dependency><groupId>org.springframework.session</groupId>
           <artifactId>spring-session-data-redis</artifactId></dependency>
```
```yaml
spring:
  session:
    store-type: redis
    timeout: 30m
    redis:
      namespace: myapp:session
```

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 大 key | 阻塞 Redis | 拆分、Scan 分批删、避免整表缓存 |
| 热 key | 分片不均 | 本地缓存 + 多副本 |
| `KEYS` | 阻塞 | `SCAN` |
| 缓存与 DB 不一致 | 脏读 | 删缓存 + 事务后删 + 短 TTL |
| 锁未释放 | 死锁 | `finally` + 超时 + 看门狗 |
| 锁误删 | 并发问题 | value 唯一 + Lua 校验 |
| 缓存全量重建 | 雪崩 | 预热 + 随机 TTL |
| Redis 阻塞 | 慢命令（Lua 大循环、大 key 删除） | 用 `UNLINK` 替代 `DEL` |

## 11. 面试速答

- **穿透/击穿/雪崩区别**：穿透是查不存在的 key；击穿是单个热点 key 失效；雪崩是大量 key 同时失效或 Redis 不可用。
- **分布式锁要点**：原子加锁（`SET NX EX`）、唯一 value + Lua 校验释放、自动续期（看门狗）、`finally` 释放、锁粒度要细。
- **限流算法**：固定窗口（简单有突刺）、滑动窗口（精确）、令牌桶（允许突发）、漏桶（匀速）。
