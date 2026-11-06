# 32 · 分布式 ID 生成

## 1. 需求

- **全局唯一**（多库多服务不冲突）
- **趋势递增**（有利于 B+ 树索引，避免页分裂）
- 高性能、高可用
- 尽量短（URL/展示友好）
- 最好含时间信息（可排序、可反解）

## 2. 方案对比

| 方案 | 长度 | 递增 | 性能 | 依赖 | 缺点 |
|---|---|---|---|---|---|
| 数据库自增 | 短 | ✅ | 中 | DB | 单库瓶颈、易被猜 |
| UUID | 36/32 | ❌ | 高 | 无 | 无序（索引碎片）、太长 |
| 雪花算法 | 64bit(19位) | 趋势✅ | 极高 | 无 | 时钟回拨、需配 workerId |
| 号段（Leaf-segment） | 短 | ✅ | 高 | DB | 号段耗尽需取、可能不连续 |
| Redis INCR | 短 | ✅ | 高 | Redis | 持久化丢失风险 |
| MongoDB ObjectId | 24 | 趋势✅ | 高 | Mongo | 需 Mongo |
| 百度 UidGenerator / 美团 Leaf | — | ✅ | 高 | DB/ZK | 需部署 |

## 3. UUID

```java
String id = UUID.randomUUID().toString();              // 36 位带横线
String short = UUID.randomUUID().toString().replace("-", "");   // 32 位
```

- 优点：本地生成、无协调。
- 缺点：**完全无序**，作为 MySQL 主键会导致 B+ 树频繁页分裂、索引膨胀、性能下降。
- 改进：**UUID v7**（2024 新标准，前 48 位是时间戳，趋势递增）——JDK 目前无内置实现，可自实现或用 `com.fasterxml.uuid:java-uuid-generator`。
- 有序替代：MySQL 的 `UUID_TO_BIN(uuid(), 1)`（交换时间位）+ `BIN_TO_UUID`。

## 4. 雪花算法（Snowflake）

```
 0 | 0000000000 0000000000 0000000000 0000000000 0 | 00000 | 00000 | 000000000000
 ↑          41 位时间戳（毫秒，相对 epoch）          ↑ 5位   ↑ 5位    12 位序列号
符号位(固定0)                                       数据中心  机器ID   同毫秒内自增
```

- 每毫秒可生成 `2^12 = 4096` 个 ID。
- 41 位时间戳可用约 69 年（相对自定义 epoch，如 2024-01-01）。
- 结果：**long**（19 位以内），趋势递增。

```java
public class Snowflake {
    private static final long EPOCH = 1704067200000L;      // 2024-01-01
    private final long workerId, dataCenterId;
    private long sequence = 0L, lastTimestamp = -1L;

    public synchronized long nextId() {
        long ts = System.currentTimeMillis();
        if (ts < lastTimestamp) throw new ClockMovedBackException(lastTimestamp - ts);
        if (ts == lastTimestamp) {
            sequence = (sequence + 1) & 0xFFF;             // 4095
            if (sequence == 0) ts = tilNextMillis(lastTimestamp);   // 等下一毫秒
        } else sequence = 0L;
        lastTimestamp = ts;
        return ((ts - EPOCH) << 22)
             | (dataCenterId << 17)
             | (workerId << 12)
             | sequence;
    }
}
```

### 关键问题

| 问题 | 说明 | 解决 |
|---|---|---|
| **时钟回拨** | 服务器时间倒退 → ID 重复 | ① 抛异常拒绝服务；② 等待追平；③ 借用未来时间（UidGenerator）；④ 用上次时间戳继续（可能重复） |
| **workerId 分配** | 手动配置易冲突 | 用 ZK/Redis/DB 注册分配；K8s 用 StatefulSet 序号；或用机器 IP 哈希取模 |
| 前端精度丢失 | JS Number 最大安全整数 `2^53-1`，19 位 ID 会被截断 | **返回前端用 String**（Jackson 全局序列化 long→String 或用 `@JsonSerialize(using=ToStringSerializer.class)`） |

```java
// 全局解决 JS 精度
@Bean
public Jackson2ObjectMapperBuilderCustomizer customizer() {
    return b -> b.serializerByType(Long.class, ToStringSerializer.instance)
                 .serializerByType(Long.TYPE, ToStringSerializer.instance);
}
```

## 5. 号段模式（Leaf-segment）

```
DB 表：biz_tag | max_id | step | update_time
应用一次取 [max_id, max_id+step) 号段到内存，用完再取
```

```sql
CREATE TABLE leaf_alloc (
  biz_tag VARCHAR(64) PRIMARY KEY,
  max_id BIGINT NOT NULL DEFAULT 1,
  step INT NOT NULL DEFAULT 1000,
  update_time TIMESTAMP
);
UPDATE leaf_alloc SET max_id = max_id + step WHERE biz_tag = 'order';  -- 一次取段
```

- 优点：ID 短、递增、DB 压力极小（每 1000 个 ID 才访问一次 DB）。
- 缺点：号段用尽时取号会阻塞（可用**双 buffer 预取**解决）；ID 不连续（无所谓）。
- 美团 Leaf 实现了双 buffer + 动态调整 step。

## 6. Redis INCR

```java
Long id = redis.opsForValue().increment("id:order");      // 原子自增
// 按天重置：id:order:20240101
```

- 优点：简单、递增、性能好。
- 缺点：Redis 持久化（RDB）可能丢数据导致 ID 重复 → 可用 `INCRBY` 批量 + AOF，或接受小概率回退（加时间戳前缀规避）。

## 7. 数据库自增（多主）

```sql
-- 2 台 DB，步长 2，起始不同
-- DB1: auto_increment_increment=2, auto_increment_offset=1  → 1,3,5...
-- DB2: auto_increment_increment=2, auto_increment_offset=2  → 2,4,6...
```
扩容麻烦（改步长需停机），不推荐新项目。

## 8. 业界实现

| 实现 | 特点 |
|---|---|
| 美团 Leaf | 号段 + Snowflake 双模式，Web 控制台 |
| 百度 UidGenerator | 基于 Snowflake，解决时钟回拨（借用未来时间），RingBuffer 高性能 |
| 滴滴 Tinyid | 号段模式，HTTP/Client 双接入 |
| ShardingSphere | 内置 `SNOWFLAKE`、`UUID` 键生成器 |

## 9. 业务上的 ID 设计

| 类型 | 建议 |
|---|---|
| 数据库主键 | 雪花/号段 bigint |
| 订单号（对外） | 雪花 + 业务前缀 + 用户 ID 后几位（便于分库定位）：`2024010112345678` |
| 防刷/邀请码 | 短码（Base62） |
| 追踪 ID | UUID / traceId |
| 幂等号 | 业务唯一键（如 `orderNo`） |

> 订单号常见做法：**时间(yyyyMMddHHmmss) + 用户ID后4位 + 随机/序列**，可读且能反查。

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| JS 精度丢失 | 前端 ID 变成 `...000` | 序列化为字符串 |
| 时钟回拨 | ID 重复 / 抛异常 | NTP 校时 + 回拨保护策略 |
| workerId 冲突 | ID 重复 | 统一分配（ZK/Redis/K8s 序号） |
| 号段模式 ID 不连续 | 业务方以为出错 | 无需连续，趋势递增即可 |
| UUID 主键性能 | 插入越来越慢 | 改有序 ID 或 `UUID_TO_BIN(...,1)` |
| 单毫秒超 4096 | 并发极高 | 增大序列号位或阻塞等待 |

## 11. 面试速答

- **分布式 ID 方案**：UUID（简单但无序）、数据库号段（递增、短）、Snowflake（高性能、趋势递增）、Redis INCR。
- **Snowflake 结构**：1 符号位 + 41 时间戳 + 10 机器位 + 12 序列号，每毫秒 4096 个。
- **时钟回拨怎么办**：拒绝服务/等待追平/借用未来时间，生产要监控并告警。
- **为什么不用 UUID 做主键**：随机写入导致 B+ 树页分裂、索引膨胀、缓存命中率低，性能显著下降。
