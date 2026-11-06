# 39 - 速查卡

## 1. 建表模板

```sql
CREATE TABLE t (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted TINYINT NOT NULL DEFAULT 0,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='';
```

## 2. 常用诊断命令

```sql
SHOW PROCESSLIST;
SHOW ENGINE INNODB STATUS\G
SHOW VARIABLES LIKE '%timeout%';
SELECT * FROM information_schema.innodb_trx\G
SELECT * FROM performance_schema.data_lock_waits;
EXPLAIN ANALYZE SELECT ...;
ANALYZE TABLE t;
```

## 3. 数字经验值

| 项 | 值 |
|---|---|
| 页大小 | 16KB |
| B+ 树 3 层 | ≈ 2000 万行（行 1KB） |
| 单表分表阈值 | 5000 万行 / 2GB（视增长） |
| buffer pool | 物理内存 50%~70% |
| 双 1 | flush_log_at_trx_commit=1 + sync_binlog=1 |
| 慢查询阈值 | 1s（线上可 0.5） |
| 锁等待超时 | innodb_lock_wait_timeout=50s |
| 批量删除 | LIMIT 500~1000/批 + sleep |
| 连接池 | cores*2+磁盘数（远小于 max_connections） |
| cte_max_recursion_depth | 1000 |
| group_concat_max_len | 默认 1024（需调大） |

## 4. 症状 → 篇目反查

| 症状 | 去看 |
|---|---|
| SQL 突然变慢 | 24 失效 / 23 执行计划 / 31 统计过期 |
| 越翻页越慢 | 30 深分页 |
| 死锁报错 1213 | 27 死锁排查 |
| 锁等待超时 1205 | 27/28 原子更新替代 |
| 主从延迟高 | 35 并行复制/大事务 |
| 写入高峰抖动 | 31 redo 太小 checkpoint |
| DDL 卡死全表 | 27 MDL / 05 pt-osc |
| 金额对账差分钱 | 16 DECIMAL |
| emoji 存不进去 | 20 utf8mb4 |
| NOT IN 返回空 | 10 NULL 陷阱 |
| count 越来越慢 | 30 汇总表 |
| 连接报 Too many connections | 03 连接池/31 参数 |
| 误删数据 | 34 PITR/延迟从库 |
| 数据库被占满连接 | 25 长事务查 innodb_trx |

## 5. 必背 SQL 片段

```sql
-- 组内 Top-N
SELECT * FROM (
  SELECT t.*, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) rn FROM t
) x WHERE rn <= 3;

-- 树查询
WITH RECURSIVE sub AS (
  SELECT id, parent_id, 1 lv FROM dept WHERE id = ?
  UNION ALL
  SELECT d.id, d.parent_id, s.lv+1 FROM dept d JOIN sub s ON d.parent_id = s.id WHERE s.lv < 10
) SELECT * FROM sub;

-- 原子扣减
UPDATE stock SET num = num - 1 WHERE sku_id = ? AND num > 0;

-- 延迟关联分页
SELECT t.* FROM t JOIN (SELECT id FROM t ORDER BY id LIMIT ?, ?) i USING (id);

-- 未命中的反连接
SELECT u.* FROM user u
LEFT JOIN orders o ON o.user_id = u.id WHERE o.id IS NULL;
```

## 6. 参数速查（my.cnf 生产模板精简）

```ini
character_set_server = utf8mb4
innodb_buffer_pool_size = 物理内存 60%
innodb_redo_log_capacity = 2G
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
binlog_format = ROW
binlog_expire_logs_seconds = 604800
gtid_mode = ON
enforce_gtid_consistency = ON
slow_query_log = ON
long_query_time = 1
max_connections = 500
```

## 7. 学习闭环建议

- 动手实验：两个会话做 MVCC/锁实验（26/28）胜过背书
- 用 EXPLAIN ANALYZE 养成习惯，所有上线 SQL 过一遍
- 与仓库内 SpringBoot 系列 09/13/14（数据源/Redis/事务）配合实践
