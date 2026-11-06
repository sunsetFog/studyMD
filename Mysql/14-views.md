# 14 - 视图

## 1. 普通视图

存储的是**查询定义**（不存数据），每次查询实时展开。

```sql
CREATE VIEW v_active_user AS
SELECT id, username, phone, created_at
FROM user
WHERE status = 1 AND deleted = 0;

SELECT * FROM v_active_user WHERE id = 1;

ALTER VIEW v_active_user AS SELECT ...;
DROP VIEW IF EXISTS v_active_user;
SHOW CREATE VIEW v_active_user\G
```

## 2. 合并 vs 物化（MERGE/TEMPTABLE）

- 简单视图（无聚合/去重/限制）：优化器 **MERGE**——视图 SQL 直接展开进外层，与手写等价，可走索引
- 复杂视图（GROUP BY/DISTINCT/LIMIT）：**TEMPTABLE**——结果物化到临时表再查询，外层条件无法下推，性能差

```sql
EXPLAIN SELECT * FROM v_x WHERE id=1;   -- 看 select_type 是否 DERIVED
```

> MySQL 没有原生物化视图（不像 PG/Oracle）。替代方案：汇总表 + 定时任务刷新，或 ClickHouse/数仓处理分析场景。

## 3. 可更新性

- 简单视图（MERGE 型、无聚合/DISTINCT/GROUP BY）可 INSERT/UPDATE/DELETE，作用于基表
- 含聚合的视图不可更新；含 `WITH CHECK OPTION` 会校验修改后仍满足视图条件

```sql
CREATE VIEW v_cn_user AS SELECT * FROM user WHERE country = 'CN'
WITH CHECK OPTION;    -- 通过视图 UPDATE country='US' 会被拒绝
```

## 4. 使用边界

| 适合 | 不适合 |
|---|---|
| 权限裁剪（GRANT 视图而隐藏敏感列） | 高频复杂统计（TEMPVIEW 物化慢） |
| 稳定对外接口（底层表重构不改视图） | 性能要求高的核心链路 |
| 简化重复 JOIN 片段 | 当作"存储"用 |

## 5. 与应用集成

- MyBatis/JPA 可直接映射视图为只读实体
- JPA 写视图需要可更新视图，不建议——ORM 层减少对视图的写操作

## 常见坑

1. 复杂视图套复杂视图层层物化，性能雪崩 → 控制嵌套层级 ≤1
2. 视图隐藏了 `SELECT *` 的真实列集，底层加列后应用映射崩 → 视图也显式列名
3. 用视图做权限但用户能直接查表 → 权限只授视图，不给基表
4. 以为视图缓存数据（物化）→ 每次都是实时计算

## Checklist

- [ ] 视图仅用于权限裁剪/接口稳定，不用于性能
- [ ] 知道 MERGE/TEMPTABLE 差异
- [ ] 需要物化效果时用汇总表方案
