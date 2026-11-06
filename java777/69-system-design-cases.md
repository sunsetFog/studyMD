# 69 · 系统设计案例

## 案例一：秒杀系统

### 特点
瞬时超高并发（10w+ QPS）、读多写少、商品有限、防超卖、防刷。

### 分层架构

```
用户 → CDN/静态页 → 网关(限流/风控) → 秒杀服务(Redis 预扣) → MQ 异步下单 → 订单服务 → DB
```

### 关键技术

| 层 | 措施 |
|---|---|
| 前端 | 按钮置灰、倒计时、防重复提交、验证码/答题 |
| 接入层 | 限流（网关令牌桶）、黑名单、风控（设备指纹、IP） |
| 缓存 | 商品详情/库存预热到 Redis；本地缓存热点 |
| 库存 | **Redis 原子预扣** `DECR`，扣到 0 即售罄 |
| 异步 | 扣减成功后发 MQ，订单服务**串行消费**落库 |
| 数据库 | 乐观锁 `update ... set stock=stock-1 where stock>0`；行锁分段 |
| 防超卖 | Redis 预扣 + DB 乐观锁双保险 + 幂等 |
| 防刷 | 一人一单（Redis SETNX）、验证码、限购 |

```java
// Redis 预扣库存（Lua 保证原子）
String lua = """
    local stock = tonumber(redis.call('get', KEYS[1]))
    if stock == nil or stock <= 0 then return -1 end
    redis.call('decr', KEYS[1])
    return stock - 1
    """;
Long r = redis.execute(new DefaultRedisScript<>(lua, Long.class), List.of("seckill:stock:" + skuId));
if (r == null || r < 0) return "已售罄";
// 一人一单
if (!Boolean.TRUE.equals(redis.opsForValue().setIfAbsent("seckill:user:"+skuId+":"+uid, "1", Duration.ofHours(2))))
    return "请勿重复购买";
// 发 MQ 异步创建订单
```

### 注意
- 库存分桶（把 1000 件分到 10 个 key）降低热点。
- 售罄标记放本地缓存，避免每次打 Redis。
- 订单异步化，前端轮询/推送结果。
- 隔离：秒杀服务独立部署、独立 DB、独立 Redis，避免拖垮主站。

## 案例二：短链服务

### 需求
长 URL → 短码（如 `t.cn/Ab3xK9`），访问短码 301/302 跳转。

### 设计

| 环节 | 方案 |
|---|---|
| 生成短码 | 雪花 ID → **Base62** 编码（6~8 位）；或哈希（有冲突需处理） |
| 存储 | MySQL（`short_code` 唯一索引）或 KV（Redis + 持久化） |
| 跳转 | 302（可统计，但多一跳）/ 301（浏览器缓存，少一跳） |
| 缓存 | 热点短链缓存到 Redis，命中率极高 |
| 统计 | 访问计数异步累加（Redis INCR + 定时落库） |
| 过期 | TTL + 定时清理 |
| 防滥用 | 限流、内容审核、黑名单 |

```java
// Base62
private static final String ALPHABET = "0-9a-zA-Z...";
public String encode(long id) {
    StringBuilder sb = new StringBuilder();
    while (id > 0) { sb.append(ALPHABET.charAt((int)(id % 62))); id /= 62; }
    return sb.reverse().toString();
}
// 缓存穿透：不存在的短码缓存空值
```

## 案例三：IM / 消息推送

### 架构

```
客户端 ←WebSocket→ 接入层(Netty/Spring WS) ←MQ→ 业务逻辑服务
                        ↓
                  连接管理服务（userId → 节点/会话）
                        ↓
              Redis（在线状态、路由表）+ 消息存储（MySQL/Cassandra）
```

| 要点 | 方案 |
|---|---|
| 连接 | WebSocket 长连接，心跳保活（30~60s） |
| 路由 | Redis 存 `userId → nodeId`，跨节点通过 MQ 转发 |
| 消息可靠 | 消息 ID + 序号（seq）；离线消息入库；ACK 确认；重传 |
| 已读/未读 | 每会话记录 `last_read_seq` |
| 群聊 | 读扩散（写一条，各人拉）vs 写扩散（每人写一条） |
| 削峰 | MQ 削峰，异步推送 |
| 存储 | 消息按会话分表/分库；冷热分离 |
| 未读数 | Redis 计数，异步落库 |

> 小团队直接用云信/融云/环信；自研只在强定制场景。

## 案例四：Feed 流（朋友圈/动态）

| 模式 | 说明 |
|---|---|
| **推（写扩散）** | 发动态时写到所有粉丝的收件箱；读极快，大 V 成本高 |
| **拉（读扩散）** | 读时聚合关注者的动态；写简单，读慢 |
| **推拉结合** | 大 V 用拉，普通用户用推（主流方案） |

- 收件箱用 Redis ZSet（按时间排序）。
- 分页用游标（score = 时间戳）。
- 计数（点赞/评论）用 Redis，异步落库。

## 案例五：分布式定时任务

| 场景 | 方案 |
|---|---|
| 单实例执行 | 分布式锁（Redis/ZK）或 ShedLock |
| 分片执行 | ElasticJob / PowerJob（按数据分片并行） |
| 可视化调度 | XXL-Job / PowerJob（调度中心 + 执行器） |
| 简单 | `@Scheduled` + ShedLock |

## 通用设计套路

```
1 估算容量：QPS、存储量、带宽
2 画架构图：接入层 → 服务层 → 数据层
3 定存储：SQL/NoSQL/缓存/对象存储
4 解决核心难点：一致性、高并发、高可用
5 补非功能：监控、限流、降级、灰度、安全
6 找瓶颈并优化
```

| 万能手段 | 说明 |
|---|---|
| 缓存 | 空间换时间，注意一致性 |
| 异步 | MQ 削峰解耦 |
| 批量 | 减少网络与 IO 次数 |
| 分片 | 水平扩展 |
| 限流降级熔断 | 保护系统 |
| 冗余 | 多副本、多机房 |
| CDN | 静态资源就近 |

## 面试速答

- **秒杀核心**：层层拦截（前端→网关→缓存）、Redis 原子预扣、MQ 异步落库、DB 乐观锁、一人一单幂等、独立部署隔离。
- **短链核心**：发号器（雪花）+ Base62、KV 存储、302 跳转、缓存、防穿透。
- **IM 核心**：长连接接入层、路由表（Redis）、消息序号与 ACK、离线消息、MQ 削峰。
- **Feed 流**：推拉结合（大 V 拉、普通推）、ZSet 收件箱、游标分页。
