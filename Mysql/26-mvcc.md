# 26 - MVCC（多版本并发控制）

## 1. 目标

读写不互斥：读不阻塞写、写不阻塞读，靠保存数据的历史版本实现。

## 2. 三要素

### (1) 隐藏列

每行 InnoDB 记录额外包含：
- `DB_TRX_ID`：最后修改该行的事务 ID
- `DB_ROLL_PTR`：回滚指针，指向 undo log 中的历史版本

### (2) undo log 版本链

```
当前行 (trx_id=100) → undo → (trx_id=80) → undo → (trx_id=60) → ...
```

每次 UPDATE 在 undo 中保留旧版本，通过 ROLL_PTR 串成链表。

### (3) ReadView（读视图）

事务做**一致性读**（快照读）时生成，包含四个关键值：

```
m_ids          ：生成时活跃（未提交）事务 ID 列表
min_trx_id     ：活跃事务最小 ID
max_trx_id     ：系统将分配的下一个事务 ID（不是最大活跃）
creator_trx_id ：当前事务自己的 ID
```

## 3. 可见性判断规则（背下来）

对版本链上某个版本（trx_id = t）：

```
1. t == creator_trx_id              → 可见（自己改的）
2. t < min_trx_id                   → 可见（生成前已提交）
3. t >= max_trx_id                  → 不可见（生成后才开启）
4. min_trx_id <= t < max_trx_id：
     t ∈ m_ids（未提交）            → 不可见
     t ∉ m_ids（已提交）            → 可见
不可见则沿 ROLL_PTR 找上一版本，重复判断。
```

## 4. RC 与 RR 的 ReadView 时机（核心差异）

- **RC**：每次 SELECT 都生成新 ReadView → 能读到别人最新提交 → 不可重复读
- **RR**：事务**第一次** SELECT 时生成，之后复用 → 整个事务看到同一快照 → 可重复读

这一条解释了 25 中并发问题表的所有行为。

## 5. 快照读 vs 当前读

```
快照读（MVCC）：普通 SELECT
当前读（读最新版并加锁）：
  SELECT ... FOR UPDATE / FOR SHARE
  UPDATE / DELETE / INSERT（内部当前读）
```

RR 下幻读为什么"基本没有"：快照读靠 MVCC 避开；当前读靠间隙锁/临键锁阻止插入（见 27/28）。

## 6. 与 undo/purge 的关系

- 已提交事务的历史版本不能立即删：可能有别的 RR 事务还在用旧 ReadView → 等 purge 线程清理
- 长事务 = 保留完整版本链 = undo 膨胀（见 25）

## 7. 实验验证（面试复现）

```sql
-- 会话 A (RR)
START TRANSACTION;
SELECT * FROM t WHERE id=1;     -- balance=100
-- 会话 B：UPDATE t SET balance=200 WHERE id=1; COMMIT;
SELECT * FROM t WHERE id=1;     -- 仍是 100（ReadView 复用）
SELECT * FROM t WHERE id=1 FOR UPDATE;  -- 200（当前读）
```

## 常见坑

1. RR 长事务 + 高频更新 → 版本链超长，快照读变慢
2. "MVCC 下没有锁"——错误，UPDATE/DELETE 是当前读加锁
3. RC 下乐观锁 version 字段不生效的错觉：RC 每次新 ReadView，需 FOR UPDATE 或原子条件更新
4. 只依赖 MVCC 做库存扣减 → 用原子 UPDATE（当前读），快照读数据是旧的

## Checklist

- [ ] 能背 ReadView 四元组与可见性四规则
- [ ] 区分快照读/当前读
- [ ] 知道 RR 复用 ReadView 是可重复读的根源
