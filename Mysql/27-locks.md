# 27 - 锁体系

## 1. 锁分类全景

```
按粒度：  表锁、行锁（InnoDB 行锁加在索引记录上！）
按模式：  共享锁 S（读锁）、排他锁 X（写锁）
按算法：  Record Lock（记录锁）、Gap Lock（间隙锁）、Next-Key Lock（记录+间隙）
意向锁：  IS/IX（表级，协调表锁与行锁）
其他：    元数据锁 MDL、AUTO-INC 锁、插入意向锁
```

## 2. 行锁 = 索引锁

行锁加在**索引项**上：

```sql
-- user_id 无索引
UPDATE orders SET status=1 WHERE user_id=1;
-- 全表扫描 = 锁全部行（近似锁表）！
```

给连接条件列建索引是缩小锁范围的根本手段。

## 3. 三种行锁算法（RR 下）

| 锁 | 范围 | 触发 |
|---|---|---|
| Record Lock | 单条索引记录 | 等值命中唯一键/主键 |
| Gap Lock | 开区间间隙，防插入 | RR 下范围查询/等值未命中 |
| Next-Key Lock | 记录 + 前间隙 (左开右闭) | RR 下普通索引等值/范围 |

- RC 级别：只有 Record Lock，无 Gap → 死锁少但幻读存在
- RR 下等值命中**唯一键**时退化为 Record Lock（间隙锁没有必要）
- 等值**未命中**：加 Gap Lock

```sql
-- RR 下实验
SELECT * FROM orders WHERE id = 5 FOR UPDATE;      -- 唯一命中：只锁 id=5
SELECT * FROM orders WHERE id > 5 AND id < 10 FOR UPDATE;  -- Next-Key：锁住 (5,10] 相关间隙
INSERT INTO orders(id) VALUES(7);                  -- 被阻塞
```

## 4. 悲观锁与乐观锁

```sql
-- 悲观锁（当前读）
SELECT * FROM account WHERE id=1 FOR UPDATE;    -- 8.0 可 NOWAIT / SKIP LOCKED
-- NOWAIT：拿不到立即报错；SKIP LOCKED：跳过被锁行（秒杀队列场景）

-- 乐观锁（无锁，version 校验）
UPDATE account SET balance=?, version=version+1 WHERE id=? AND version=?;
-- affected=0 → 重试或失败
```

选型：冲突高 + 临界区短 → 悲观；冲突低 → 乐观。

## 5. 表级锁：MDL

- 查询/CRUD 自动加 MDL 读锁；DDL 加 MDL 写锁
- **长事务阻塞 DDL，DDL 等待又阻塞后续所有该表查询**（雪崩级故障）
- 应对：DDL 前 kill 长事务、用 `lock_wait_timeout` 短超时重试、大表走 pt-osc（见 05）

## 6. 死锁

```sql
SHOW ENGINE INNODB STATUS\G        -- LATEST DETECTED DEADLOCK 部分
SELECT * FROM performance_schema.data_lock_waits;
SELECT @@innodb_deadlock_detect;   -- 默认开启，检测后回滚代价小的事务
```

- 两事务以不同顺序加锁（A 锁 1 等 2，B 锁 2 等 1）→ 检测器选一个牺牲者（1213 错误），应用需重试
- Gap Lock 交叉是 RR 下死锁高发根因
- 预防：**统一加锁顺序**（按 id 升序）、事务短、锁范围靠索引缩小、RC 降级

## 7. 热点行问题

```sql
-- 秒杀同一商品行，UPDATE 串行化瓶颈
-- 方案：热点拆行（库存拆 N 段）、Redis 预扣、队列串行（见 SpringBoot 系列）
UPDATE stock SET num = num - 1 WHERE id = 1;   -- 行锁排队
```

## 常见坑

1. 条件列无索引 → 行锁变全表锁
2. 长事务 + DDL 互相等待引发全表雪崩
3. RR 下批量更新顺序不一致导致死锁 → 统一 ORDER BY id
4. 锁等待超时 1205 与死锁 1213 混淆：前者是等待超时（默认 50s，`innodb_lock_wait_timeout`），后者是死循环检测

## Checklist

- [ ] UPDATE/DELETE 条件列有索引
- [ ] 并发事务加锁顺序统一
- [ ] 应用层对 1213 做重试
- [ ] DDL 前检查长事务
