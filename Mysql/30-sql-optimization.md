# 30 - SQL 优化实战

## 1. 深分页优化

```sql
-- 慢：LIMIT 1000000, 20 扫描 100 万+行
SELECT * FROM orders ORDER BY id LIMIT 1000000, 20;

-- 优化 1：延迟关联（先用覆盖索引取 id）
SELECT o.* FROM orders o
JOIN (SELECT id FROM orders ORDER BY id LIMIT 1000000, 20) t ON o.id = t.id;

-- 优化 2：游标分页（推荐，翻页稳定且快）
SELECT * FROM orders WHERE id > #{lastId} ORDER BY id LIMIT 20;   -- 记住上一页最后一个 id
```

游标分页要求排序键单调，且只能"下一页"，产品需跳页时用延迟关联。

## 2. count 优化

```sql
SELECT COUNT(*) FROM t;                        -- InnoDB 全表扫
-- 为什么慢：MVCC 无全局计数（见 04）
-- 加 WHERE 更慢（逐行判断）
```

优化路径：
1. **近似值**：`SHOW TABLE STATUS` / information_schema（运维看板够用）
2. **汇总表**（见 18）或 Redis 计数器（业务计数）
3. count(*) 比 count(1)/count(col) 都好（官方已优化，无差异传言过时）
4. 多表 count 减少条件列参与判断，如 `count(*) WHERE deleted=0` → 部分场景可用索引（deleted 在索引里）

## 3. JOIN 优化

1. 小表驱动大表（优化器自动，必要时 STRAIGHT_JOIN）
2. 被驱动表连接列有索引 → INLJ
3. JOIN 键类型/字符集一致（见 24）
4. 先过滤再 JOIN（把 WHERE 下推）
5. 一对多统计先聚合子查询再 JOIN（见 09）
6. 8.0 无索引等值 JOIN 已有 Hash Join 兜底

## 4. 批量操作

```sql
-- 批量 INSERT 分批（单语句别超几 MB，max_allowed_packet）
INSERT INTO t(a,b) VALUES (?,?),(?,?)...;    -- 500~1000 行/批

-- upsert 批量
INSERT INTO t(id,v) VALUES ... ON DUPLICATE KEY UPDATE v=VALUES(v);
```

应用层：MyBatis foreach 批量、JDBC `rewriteBatchedStatements=true`（MySQL 驱动，速度差 10 倍）。

## 5. 常见改写套路

```sql
-- 1. 子查询改 JOIN（相关标量）
-- 2. OR 改 UNION
SELECT ... WHERE a=1 UNION SELECT ... WHERE b=2;

-- 3. 强制小结果优先：SELECT 明确列
-- 4. LIMIT 1 明确语义（优化器找到即停）
SELECT * FROM t WHERE uk='x' LIMIT 1;

-- 5. 索引列避免函数：范围改写（见 24）
-- 6. INSERT 大事务拆小
-- 7. 用覆盖索引减少 IO
```

## 6. 设计层面的 SQL 性能

1. 高频聚合 → 汇总表/缓存（见 18）
2. 列表页只查主表，详情才 JOIN（接口拆分）
3. 搜索类 LIKE '%x%' → ES
4. 复杂统计 → 数仓
5. 读写分离分担读压力（见 35）

## 7. 优化优先级

```
架构/业务优化（缓存、拆分、ES）> 索引优化 > SQL 改写 > 参数调优
```

改写 SQL 收益有上限，"不查"永远比"查得快"强。

## 常见坑

1. rewriteBatchedStatements 没开，批量像逐条
2. 深分页用 OFFSET 无脑上翻
3. count(1) 传说的性能差异浪费时间
4. 所有优化都压在 DB 上，忘了缓存层（见 SpringBoot 24）

## Checklist

- [ ] 分页用游标/延迟关联
- [ ] 批量写入 rewriteBatchedStatements=true
- [ ] 统计需求落汇总表/缓存
- [ ] 核心接口的 SQL 都过 EXPLAIN
