# 32 - 大表优化与分库分表

## 1. 什么时候分表

信号（满足其一）：
- 单表 > 5000 万行且仍高速增长
- B+ 树 4 层以上、buffer pool 命不中热数据
- DDL 无法承受（见 05）
- 单实例 CPU/IO 持续 70%+

先确认慢是数据量导致：很多"大表问题"其实是缺索引/烂 SQL（见 24/30），**先优化后拆分**。

## 2. 非拆分手段（优先级更高）

1. **冷热分离**：历史数据搬到归档表/归档库（日志类最常见）

```sql
-- 按时间归档分批搬
INSERT INTO orders_archive SELECT * FROM orders WHERE created_at < '2025-01-01' LIMIT 1000;
DELETE FROM orders WHERE created_at < '2025-01-01' LIMIT 1000;
```

2. **中间表/汇总表**：统计不再扫大表（见 18/30）
3. **分区表**（同一库内物理切分）：

```sql
CREATE TABLE log (
  id BIGINT, created_at DATETIME NOT NULL, ...
) PARTITION BY RANGE (TO_DAYS(created_at)) (
  PARTITION p202601 VALUES LESS THAN (TO_DAYS('2026-02-01')),
  PARTITION p202602 VALUES LESS THAN (TO_DAYS('2026-03-01')),
  PARTITION pmax VALUES LESS THAN MAXVALUE
);
-- 查询带分区键才裁剪（partition pruning）；不支持外键、8.0 InnoDB 支持的分区函数有限
ALTER TABLE log DROP PARTITION p202601;    -- 秒删历史
```

分区适合时间序列日志类；核心交易表一般直接分库分表。

## 3. 分库分表策略

### 拆分键（sharding key）选择

| 业务 | 拆分键 | 理由 |
|---|---|---|
| 订单（C 端） | user_id | "我的订单"高频；运营查询另建异构索引 |
| 订单（平台端） | order_no | 按单号查映射表得 user_id |
| 消息 | 会话 ID | 会话内时序 |
| 通用 | snowflake id hash | 无业务键时 |

原则：**80% 高频查询能直接定位到分片**；跨片查询走异构（ES/宽表/二级索引服务）。

### 拆分算法

- hash 取模 `user_id % 1024`：分布均匀，扩容要迁移 → 用一致性哈希或**预留倍数**（1024 逻辑片映射 16 物理库）
- range 按时间：写入热点最新片，但归档方便
- 查表法（mapping 表）：灵活，多一次查询

## 4. 分片后的问题清单

| 问题 | 方案 |
|---|---|
| 全局唯一 ID | 雪花/号段（见 19） |
| 跨片 JOIN | 应用层组装/异构宽表 |
| 跨片分页排序 | 各片 topN 归并（深度分页难看，产品上规避） |
| 分布式事务 | 本地消息表/SAGA（见 SpringBoot 系列） |
| 二级索引 | ES/全局索引表 |
| 扩容 | 双写迁移 + 灰度切流 + 对账 |

## 5. 中间件

- **ShardingSphere-JDBC**：应用内代理，无中间层延迟，Java 生态主流
- ShardingSphere-Proxy / MyCat：独立代理，语言无关
- 云方案：TDSQL/DRDS/PolarDB-X

## 6. 迁移步骤（双写方案）

1. 新库建表，存量同步（DTS/全量导入）
2. 开双写（旧为准），校验不一致
3. 读切新库（灰度比例递增）
4. 停写旧库，观察，下线

## 常见坑

1. 分片键选了 created_at，C 端查询全扫所有分片
2. 分页在分片上深翻页结果错乱
3. 唯一约束跨片失效（uk 只在片内有效）→ 全局 ID 含分片位或映射表
4. 迁移时只读切流不校验，脏数据上线
5. 分区表查询不带分区键扫全部分区

## Checklist

- [ ] 先确认是 SQL 问题还是规模问题
- [ ] 分片键覆盖 80% 查询
- [ ] 全局 ID 方案先于拆分确定
- [ ] 迁移有双写+对账+灰度
