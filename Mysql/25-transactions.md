# 25 - 事务

## 1. ACID 与实现

| 特性 | 实现 |
|---|---|
| 原子性 Atomicity | undo log 回滚（见 33） |
| 一致性 Consistency | 前三者 + 应用层约束共同保证 |
| 隔离性 Isolation | MVCC + 锁（见 26/27） |
| 持久性 Durability | redo log（WAL） |

## 2. 基本用法

```sql
SET autocommit = 0;    -- 关闭自动提交（默认开启）
START TRANSACTION;
  UPDATE account SET balance = balance - 100 WHERE id = 1 AND balance >= 100;
  -- 检查 ROW_COUNT()
  UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;    -- 或 ROLLBACK;

-- 保存点：部分回滚
SAVEPOINT sp1;
  UPDATE ...;
ROLLBACK TO sp1;
COMMIT;
```

SpringBoot 侧（`@Transactional`）的行为与失效场景见 SpringBoot 系列 14 篇（自调用、异常类型、传播行为）。

## 3. 隔离级别与并发问题

| 级别 | 脏读 | 不可重复读 | 幻读 |
|---|---|---|---|
| READ UNCOMMITTED | 有 | 有 | 有 |
| READ COMMITTED (RC) | 无 | 有 | 有 |
| REPEATABLE READ (RR，默认) | 无 | 无 | 通过临键锁基本杜绝（见 28） |
| SERIALIZABLE | 无 | 无 | 无（性能极差） |

```sql
SELECT @@transaction_isolation;
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

并发问题定义：
- 脏读：读到别的事务未提交数据
- 不可重复读：同一事务内两次读同行结果不同（他人 UPDATE 提交）
- 幻读：同一事务内两次范围查询行数不同（他人 INSERT 提交）

## 4. 事务最佳实践

1. **能短则短**：长事务 → undo 膨胀、锁持有久、主从延迟、回滚代价大
2. 事务内禁止：HTTP/RPC 调用、发消息（应 AFTER_COMMIT，见 SpringBoot 18）、sleep
3. 查询事务 vs 写事务分开：只读操作开 `START TRANSACTION READ ONLY`（优化器可优化）或干脆不开事务
4. 大批量写 → 分批事务（见 06）

## 5. 查看与杀事务

```sql
-- 8.0 新视图（替代 information_schema.innodb_trx）
SELECT * FROM information_schema.innodb_trx\G
-- trx_started 看事务已运行多久
KILL <processlist_id>;

SELECT * FROM performance_schema.data_locks;          -- 当前锁（8.0）
SELECT * FROM performance_schema.data_lock_waits;
```

长事务危害链条：undo 无法清理（purge 滞后）→ 历史版本链变长 → 查询越来越慢 → 磁盘上涨。

## 6. 分布式事务衔接

单库事务之外：TCC/SAGA/本地消息表/事务消息——见 SpringBoot 系列（25 RabbitMQ）与 Seata。MySQL 侧职责：保证本地事务 + 提供幂等表（唯一键）。

## 常见坑

1. 事务里调外部接口，连接被占满（雪崩）
2. GUI 客户端开着未提交事务，锁全表行，业务报锁等待超见 27
3. `SET autocommit=0` 后忘 COMMIT，长事务挂着
4. 以为 RR 下完全无幻读（当前读会看到新行，见 28）

## Checklist

- [ ] 事务内无外部 IO
- [ ] 能查 innodb_trx 定位长事务
- [ ] 金额变动在事务中且条件更新
