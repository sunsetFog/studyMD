# 06 - 增删改（DML）

## 1. INSERT

```sql
-- 基础
INSERT INTO user(username, phone) VALUES ('alice', '13800000001');

-- 批量（一次多值，网络往返与解析开销小）
INSERT INTO user(username, phone) VALUES
  ('bob','13800000002'), ('carl','13800000003');

-- 幂等插入：冲突则跳过/更新
INSERT INTO user(username, phone) VALUES ('bob','13800000002')
  ON DUPLICATE KEY UPDATE phone = VALUES(phone);     -- 8.0.20+ 建议 alias 写法：
-- INSERT INTO ... VALUES (...) AS new ON DUPLICATE KEY UPDATE phone = new.phone;

-- 冲突忽略
INSERT IGNORE INTO user(username) VALUES ('bob');    -- 唯一键冲突静默跳过（其他错误也会被吞，慎用）

-- 从查询结果插入（数据迁移常用）
INSERT INTO user_bak SELECT * FROM user WHERE created_at < '2020-01-01';

-- REPLACE INTO：冲突时先 DELETE 再 INSERT（主键变、其他列丢失）——语义危险，慎用
```

## 2. UPDATE

```sql
UPDATE user SET status = 0 WHERE id = 1;

-- 危险动作必须先 SELECT 验证 WHERE
SELECT id FROM user WHERE balance > 10000;
UPDATE user SET status = 0 WHERE balance > 10000;

-- 自引用更新
UPDATE account SET balance = balance - 100 WHERE id = 1 AND balance >= 100;  -- 受影响行数=0 表示余额不足（配合事务）
```

`sql_safe_updates=1` 模式下禁止无 WHERE 的 UPDATE/DELETE，生产推荐开启（连接参数或 CLI）。

## 3. DELETE vs TRUNCATE vs DROP

| | DELETE | TRUNCATE | DROP |
|---|---|---|---|
| 类型 | DML | DDL | DDL |
| WHERE | 支持 | 不支持 | 不支持 |
| 回滚 | 可回滚（undo） | 不可 | 不可 |
| 触发器 | 触发 | 不触发 | 不触发 |
| 自增值 | 保留 | 重置为 1 | 表删除 |
| 速度 | 慢（逐行+日志） | 快 | 快 |
| 记入 binlog | 记（ROW 记行事件） | 记 | 记 |

清空大表一律 TRUNCATE；删除部分数据要**分批**（见下）。

## 4. 大批量删除/更新的正确姿势

一次性 DELETE 500 万行 → 长事务、锁大量行、主从延迟（见 35）、undo 膨胀。正确做法：

```sql
-- 分批删除，循环直到 affected=0
DELETE FROM log WHERE created_at < '2025-01-01' LIMIT 1000;
-- 每批之间 sleep，主从延迟高则降低批次/频率
```

```java
// 应用侧模板
while (true) {
    int n = jdbc.update("DELETE FROM log WHERE created_at < ? LIMIT 1000", ts);
    if (n == 0) break;
    Thread.sleep(50);
}
```

注意：`DELETE ... LIMIT` 无 ORDER BY 时删除哪批不确定，配合索引列条件保证走索引且顺序稳定。

## 5. 影响行数语义

- `affectedRows`：UPDATE 值没变化的行**不计入**（可用连接参数 `useAffectedRows=true` 改变 JDBC 语义）
- `INSERT ON DUPLICATE KEY UPDATE`：插入=1，更新=2，无变化=0——判断真实效果时注意

## 常见坑

1. UPDATE/DELETE 忘写 WHERE（GUI 里手滑）→ 开 sql_safe_updates、事务包裹后确认再 COMMIT
2. 大事务删除导致主从延迟暴增、磁盘 undo 膨胀 → 分批
3. `INSERT IGNORE` 把字段超长截断等错误也吞掉 → 用 `INSERT ... ON DUPLICATE KEY UPDATE` 更可控
4. REPLACE INTO 导致 id 变化和关联数据断链 → 除非全量重建，否则用 upsert
5. 自增主键用尽 BIGINT 上限（千万级写入持续多年）→ 提前规划分布式 ID（见 19）

## Checklist

- [ ] 危险 UPDATE/DELETE 前先 SELECT 同 WHERE 条件
- [ ] 生产开启 sql_safe_updates
- [ ] 批量删除分批执行并控制主从延迟
- [ ] 金额扣减用 `balance = balance - x WHERE balance >= x` 原子语义
