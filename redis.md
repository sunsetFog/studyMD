# Redis 知识点总结

## 1. Redis 简介

Redis（Remote Dictionary Server）是一个开源的、基于内存的高性能键值（Key-Value）数据库，支持丰富的数据结构，常用于缓存、消息队列、计数器、排行榜等场景。

特点：
- 数据存在内存中，读写极快（单机可达 10 万+ QPS）
- 支持持久化（RDB / AOF）
- 支持主从复制、哨兵、集群
- 单线程处理命令（6.0 后网络 I/O 多线程，命令执行仍单线程），避免锁竞争

## 2. 五种基本数据类型

| 类型 | 结构 | 应用场景 |
|------|------|----------|
| String | 字符串 | 缓存对象、计数 |
| Hash | 哈希表 | 存储对象、购物车 |
| List | 链表 | 消息队列、最新列表 |
| Set | 无序集合 | 去重、共同好友 |
| ZSet | 有序集合 | 排行榜、延迟队列 |

## 3. String 常用命令

String 是最基础的类型，value 可以是字符串、数字、二进制数据，单个 value 最大 512MB。

```
SET key value         # 设置值（存在则覆盖）
GET key               # 获取值，不存在返回 nil
SET key value EX 10   # 设置值并指定过期时间 10 秒
INCR key              # 自增
DECR key              # 自减
APPEND key value      # 追加
STRLEN key            # 长度
MSET k1 v1 k2 v2      # 批量设置
MGET k1 k2            # 批量获取
```

示例：

```
SET user:1:name "zhangsan"
GET user:1:name

SET count 0
INCR count          # 返回 1
INCR count          # 返回 2

SETNX lock "1"      # 不存在才设置，常用于分布式锁
```

## 4. Hash 常用命令

Hash 用于存储对象，一个 key 下包含多个字段值对，适合保存结构化数据（如用户信息、购物车）。

```
HSET key field value  # 设置单个字段
HGET key field        # 获取单个字段值
HMSET key f1 v1 f2 v2 # 批量设置多个字段
HMGET key f1 f2       # 批量获取多个字段值
HGETALL key           # 获取所有字段和值
HDEL key field        # 删除字段
HLEN key              # 字段数量
HEXISTS key field     # 判断字段是否存在
HINCRBY key field n   # 字段值自增 n
```

示例：

```
HSET user:1 name zhangsan age 20
HGET user:1 name          # "zhangsan"
HINCRBY user:1 age 1      # 21
HGETALL user:1            # name, zhangsan, age, 21
```

## 5. List 常用命令

List 是一个双向链表，可以两端操作，支持队列和栈。

```
LPUSH key value      # 左侧插入
RPUSH key value      # 右侧插入
LPOP key             # 左侧弹出
RPOP key             # 右侧弹出
LRANGE key 0 -1      # 查看所有元素
LLEN key             # 长度
LINDEX key index     # 按下标取值
```

示例：

```
RPUSH queue a b c
LRANGE queue 0 -1    # a, b, c
LPOP queue           # a
RPOP queue           # c

LPUSH stack x y      # 先进后出
LPOP stack           # y
```

## 6. Set 常用命令

无序、元素唯一的集合。

```
SADD key member       # 添加成员
SREM key member       # 删除成员
SMEMBERS key         # 查看所有成员
SISMEMBER key member # 是否存在
SCARD key            # 元素数量
SINTER k1 k2         # 交集
SUNION k1 k2         # 并集
SDIFF k1 k2          # 差集（k1 有 k2 没有）
```

示例：

```
SADD friends:a 1 2 3
SADD friends:b 2 3 4
SINTER friends:a friends:b   # 2, 3（共同好友）
SDIFF friends:a friends:b    # 1
```

## 7. ZSet 有序集合常用命令

每个成员带一个 score，按 score 排序。

```
ZADD key score member         # 添加成员并指定分数
ZRANGE key start stop          # 按分数升序
ZREVRANGE key start stop       # 按分数降序
ZSCORE key member              # 查分数
ZRANK key member               # 升序排名
ZREVRANK key member            # 降序排名
ZREM key member               # 删除成员
ZINCRBY key n member           # 分数增加 n
ZRANGEBYSCORE key min max      # 按分数范围取
```

示例：

```
ZADD rank 100 tom 90 jerry 95 lucy
ZREVRANGE rank 0 -1            # tom, lucy, jerry
ZINCRBY rank 20 jerry          # jerry 变 110
ZREVRANK rank jerry            # 0
```

## 8. 键的通用命令

这些命令针对 key 本身操作，与 value 类型无关，任何数据类型的 key 都适用。

```
DEL key               # 删除
EXISTS key            # 是否存在
EXPIRE key seconds    # 设置过期时间
TTL key               # 查看剩余存活时间（-1 永不过期，-2 不存在）
PERSIST key           # 移除过期时间
TYPE key              # 查看类型
RENAME k1 k2          # 重命名
KEYS pattern          # 模糊匹配（生产慎用，用 SCAN）
SCAN cursor           # 渐进式遍历
```

示例：

```
SET temp hello
EXPIRE temp 30        # 30 秒后过期
TTL temp              # 剩余秒数

SCAN 0 MATCH user:* COUNT 100
```

## 9. 发布订阅 Pub/Sub

消息的发布与订阅，不是可靠队列，订阅者离线会丢消息。

```
SUBSCRIBE channel     # 订阅
PUBLISH channel msg   # 发布
UNSUBSCRIBE channel   # 取消订阅
```

示例：

```
# 客户端 A
SUBSCRIBE news

# 客户端 B
PUBLISH news "hello"
```

## 10. 事务

Redis 事务通过 MULTI/EXEC 实现，不具备原子性回滚（某条命令出错不影响其他命令执行）。

```
MULTI            # 开启事务
SET a 1
INCR a
EXEC             # 提交执行
DISCARD          # 取消事务
WATCH key        # 乐观锁，监控 key
```

示例：

```
MULTI
SET k 1
INCR k
EXEC             # 依次执行，返回 [OK, 2]
```

## 11. 持久化

- RDB：按时间快照保存数据，文件小、恢复快，但可能丢失最近一次快照后的数据。
- AOF：记录每条写命令，数据更完整，文件大、恢复慢。

```
# redis.conf
save 900 1            # 900 秒内至少 1 次修改触发 RDB
appendonly yes        # 开启 AOF
appendfsync everysec  # 每秒刷盘一次
```

## 12. 主从复制

主节点负责写，从节点复制主节点数据并提供读，实现读写分离和数据备份。

```
# 从节点配置（redis.conf）
replicaof <master-ip> <master-port>
# 或命令行
REPLICAOF 192.168.1.10 6379
```

## 13. 哨兵模式 Sentinel

监控主节点，主节点宕机后自动从从节点中选举新主节点，实现高可用。

```
# sentinel.conf
sentinel monitor mymaster 127.0.0.1 6379 2   # 2 个哨兵判定主观下线
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
```

## 14. 集群 Cluster

将数据分片存储到多个节点，每个节点只保存一部分数据（16384 个槽位），解决单机容量和性能瓶颈。

```
# 启动集群（创建 6 节点，3 主 3 从）
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 \
  127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
  --cluster-replicas 1

# 查看集群信息
redis-cli -c -p 7000 cluster info
```

## 15. 缓存穿透、击穿、雪崩

- 穿透：查询不存在的数据，绕过缓存直接打 DB。解决：缓存空值、布隆过滤器。
- 击穿：热点 key 过期瞬间大量请求打 DB。解决：热点 key 不过期、加互斥锁。
- 雪崩：大量 key 同时过期或 Redis 宕机。解决：过期时间加随机值、集群高可用。

```
# 缓存空值示例
GET key
if nil:
    GET db
    SET key "null" EX 60   # 空值也缓存，避免穿透
```

## 16. 过期策略

Redis 采用惰性删除 + 定期删除：
- 惰性删除：访问 key 时判断是否过期，过期则删除。
- 定期删除：每隔一段时间随机抽一批 key 检查删除。

## 17. 内存淘汰策略

内存满时按策略淘汰 key：

```
# redis.conf
maxmemory 100mb
maxmemory-policy allkeys-lru   # 常用：对全部 key 做 LRU 淘汰
```

常见策略：
- noeviction：不淘汰，写入报错
- allkeys-lru / volatile-lru
- allkeys-lfu / volatile-lfu
- allkeys-random / volatile-random

## 18. 分布式锁

用 SET NX EX 实现带过期时间的分布式锁。

```
SET lock:order unique_id NX EX 30   # 加锁，30 秒过期
# 释放锁：比对 unique_id 后删除，防止误删（用 Lua 保证原子性）
```

Lua 释放锁示例：

```
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

## 19. 位图 Bitmap

用 bit 存储信息，节省空间，适合签到、在线状态。

```
SETBIT key offset value   # 设置某位 0/1
GETBIT key offset         # 读取某位
BITCOUNT key              # 统计 1 的数量
BITOP AND dest k1 k2      # 位运算
```

示例：

```
SETBIT sign:user1:2026 0 1   # 第 0 天签到
SETBIT sign:user1:2026 1 1
BITCOUNT sign:user1:2026     # 2（签到两天）
```

## 20. HyperLogLog

用于基数统计（去重计数），占用内存固定约 12KB，有约 0.81% 误差，适合统计 UV。

```
PFADD key element
PFCOUNT key            # 估算基数
PFMERGE dest k1 k2     # 合并
```

示例：

```
PFADD uv 192.168.1.1 192.168.1.2 192.168.1.1
PFCOUNT uv             # 2（重复 IP 只算一次）
```

## 21. GEO 地理位置

存储经纬度，支持计算距离和查找附近的人。

```
GEOADD key lng lat member
GEODIST key m1 m2 km         # 两点距离
GEORADIUS key lng lat r km   # 半径内查找
GEOPOS key member            # 查询坐标
```

示例：

```
GEOADD city 116.40 39.90 beijing 121.47 31.23 shanghai
GEODIST city beijing shanghai km   # 约 1067 km
```

## 22. Stream 消息队列

Redis 5.0 引入的流数据结构，支持消费组，可作为轻量消息队列。

```
XADD key id field value      # 添加消息（id 可用 * 自动生成）
XLEN key                     # 长度
XRANGE key - +               # 查看消息
XGROUP CREATE key group 0    # 创建消费组
XREADGROUP GROUP group consumer COUNT 1 STREAMS key >
XACK key group id            # 确认消息
```

示例：

```
XADD mq * msg hello
XADD mq * msg world
XRANGE mq - +               # 查看两条消息
```

## 23. Lua 脚本

把多条命令封装成一个脚本原子执行，减少网络往返。

```
EVAL script numkeys key [key ...] arg [arg ...]
```

示例：

```
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 name zhangsan
GET name                    # zhangsan
```

## 24. Pipeline 管道

客户端一次性批量发送多条命令，减少 RTT（往返时间），非原子操作。

```
# redis-cli 管道示例
echo -e "SET a 1\nSET b 2\nINCR a" | redis-cli --pipe
```

## 25. 布隆过滤器 Bloom Filter

用于快速判断元素是否可能存在，可有效解决缓存穿透。Redis 需通过 Bloom 模块或 Redisson 等客户端使用。

```
# 使用 Redisson 示例（伪代码）
BF.ADD key item          # 添加
BF.EXISTS key item       # 判断是否存在（可能存在，绝对不存在的判定是准确的）
```

特点：判断不存在一定准确，判断存在可能有误判，能极大节省空间。
