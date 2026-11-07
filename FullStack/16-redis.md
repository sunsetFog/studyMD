# 16 - Redis 考点

## 1. 数据结构与场景（必考⭐）

| 类型 | 底层 | 场景 |
|---|---|---|
| String | SDS 动态字符串 | 缓存、计数器（INCR 原子）、分布式锁 |
| Hash | ziplist/listpack → hashtable | 对象缓存（用户信息部分更新） |
| List | quicklist（双向链表+ziplist） | 消息队列（LPUSH/BRPOP）、时间线 |
| Set | intset/hashtable | 去重、共同好友（SINTER）、抽奖 |
| ZSet | listpack → skiplist+dict | **排行榜**（ZINCRBY/ZREVRANGE）、延迟队列（score=时间戳） |

- **跳表 vs 红黑树**（ZSet 为什么用跳表）：实现简单、范围查询顺链、按 rank 查询方便
- SDS vs C 字符串：O(1) 长度、二进制安全、预分配减少 realloc

## 2. 缓存三问（必考⭐⭐）

| 问题 | 定义 | 解法 |
|---|---|---|
| 缓存穿透 | 查不存在的数据，绕过缓存打库 | 空值缓存（短 TTL）、布隆过滤器、参数校验 |
| 缓存击穿 | 热点 key 过期瞬间并发打库 | 互斥锁重建、**逻辑过期**（不删 key）、热点永不过期 |
| 缓存雪崩 | 大量 key 同时过期 / Redis 宕机 | TTL 加随机抖动、多级缓存、集群高可用、限流降级 |

## 3. 缓存一致性问题（必考⭐⭐）

```
为什么先更新库再删缓存（Cache Aside）？
- 先删缓存后更新库：并发下旧值回填 → 长期脏
- 先更新库后删缓存：A 更新库 → B 读旧缓存 → A 删缓存 → 下次回填新值（窗口小，可接受）
```

- 删失败怎么办：重试队列（MQ）/ 订阅 binlog（Canal）异步删
- 延迟双删：更新库 → 删缓存 → sleep → 再删（应对读写分离延迟）
- 强一致场景不上缓存（或用分布式锁串行化），缓存本质是最终一致

## 4. 持久化

| 方式 | 原理 | 优缺点 |
|---|---|---|
| RDB | fork 子进程全量快照（COW 写时复制） | 恢复快、丢最后一段数据 |
| AOF | 追加写命令（appendfsync everysec） | 丢 1 秒、文件大、有重写（bgrewriteaof） |
| 混合（4.0+ 默认） | RDB 头 + AOF 增量 | 兼顾 |

fork 瞬间开销与页表复制；大实例 fork 抖动。

## 5. 分布式锁（必考⭐⭐）

```java
// Redisson 标准用法
RLock lock = redisson.getLock("order:" + orderId);
if (lock.tryLock(3, 30, TimeUnit.SECONDS)) {   // waitTime/leaseTime
    try { /* 业务 */ } finally { lock.unlock(); }
}
```

- SET NX PX + 唯一 value（Lua 保证原子判断+删除）——面试要求能手写
- 锁续期：看门狗（Redisson watchdog，leaseTime=-1 默认 30s 自动续）
- 主从切换锁丢失：RedLock 争议（大多用 Redisson + 业务幂等兜底即可）
- 对比 ZooKeeper 锁（CP，临时顺序节点）：Redis AP 高性能，ZK 可靠性高

## 6. 高可用与集群

- 主从复制：全量（RDB）+ 增量（repl_backlog）
- 哨兵 Sentinel：监控 + 自动故障转移 + 通知（客户端经哨兵发现主）
- Cluster 集群：**16384 slot**（CRC16(key)%16384）、gossip 协议、hash tag 保证多 key 同槽（MGET 限制）
- 大 key/热 key：拆分（hash 分桶）、本地缓存、监控 `redis-cli --bigkeys`

## 7. 其他高频

- 单线程为什么快：内存操作 + IO 多路复用（epoll）+ 无锁无上下文切换；6.0 多线程仅处理网络 IO
- 过期删除：惰性 + 定期抽样；内存淘汰：`allkeys-lru`（缓存常用）、`volatile-lru`
- Redis vs Memcached：数据结构丰富/持久化/集群 vs 纯 KV 多线程

## Checklist

- [ ] 缓存三问各给两个解法
- [ ] 缓存一致性选 Cache Aside + 理由
- [ ] 能手写 SET NX PX + Lua 解锁逻辑
- [ ] ZSet 能落地排行榜与延迟队列
