# 29 - 慢查询定位

## 1. 开启慢日志

```ini
# my.cnf
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1          # 阈值 1 秒（线上可 0.5）
log_queries_not_using_indexes = ON
log_throttle_queries_not_using_indexes = 60   # 每分钟最多记 60 条未用索引的
```

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;
SELECT @@slow_query_log, @@long_query_time;
```

## 2. 分析三板斧

**第一板斧：pt-query-digest 聚合分析**

```bash
pt-query-digest /var/log/mysql/slow.log > slow-report.txt
# 输出：按总耗时排序的指纹 SQL、QPS、平均/95% 耗时、rows examine/send
```

重点看：总耗时占比最高（优化它收益最大）、Rows examine 远大于 Rows sent（索引问题）。

**第二板斧：EXPLAIN 单条**

见 23。重点 type/key/rows/Extra。

**第三板斧：EXPLAIN ANALYZE / SHOW PROFILE（8.0 前 profile）**

```sql
EXPLAIN ANALYZE SELECT ...;    -- 真实耗时分布
```

## 3. 顺手的实时观察

```sql
SHOW PROCESSLIST;                    -- 当前在跑什么（Time 大的注意）
SELECT * FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 10;  -- sys schema 慢 SQL 视图
SELECT * FROM sys.innodb_lock_waits;
SELECT * FROM sys.schema_table_statistics ORDER BY io_read_requests DESC LIMIT 10;
```

sys schema 是 8.0 排查利器，本质是 performance_schema 的友好封装。

## 4. 常见慢因与速配解法

| 慢因 | 特征 | 解法 |
|---|---|---|
| 无索引/索引失效 | type=ALL/rows 大 | 见 24 |
| 深分页 | LIMIT 大 offset | 见 30 |
| 回表过多 | rows 大 + 无覆盖索引 | 联合索引覆盖 |
| filesort/临时表 | Extra | 排序/分组索引 |
| 锁等待 | 耗时波动大、sys.innodb_lock_waits 有记录 | 见 27 |
| 统计信息过期 | EXPLAIN 估算离谱 | ANALYZE TABLE |
| 单表过大 | 全部指标都退化 | 见 32 |
| 返回列/行过多 | rows sent >> needed | 业务裁剪 |

## 5. 线上慢查询应急流程

1. `SHOW PROCESSLIST` 找出耗时 SQL，确认是否该 kill
2. 慢日志 + pt-query-digest 找 TOP 指纹
3. EXPLAIN ANALYZE 定位
4. 加索引/改写 SQL → 验证 → 回归
5. 无法快速修复的：限流、缓存、降级（见 SpringBoot 系列）

## 常见坑

1. long_query_time 设 10 秒——太迟钝，慢查询早已堆积
2. 只优化单次最慢而非总耗时最高的 SQL
3. 生产 log_queries_not_using_indexes 全开导致日志爆炸
4. 慢日志盘满 / 与数据同盘互相拖累

## Checklist

- [ ] 慢日志常开，阈值 ≤ 1s
- [ ] 会用 pt-query-digest 出报告
- [ ] 有 EXPLAIN→ANALYZE→改写→验证的固定流程
