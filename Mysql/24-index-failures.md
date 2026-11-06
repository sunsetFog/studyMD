# 24 - 索引失效场景全清单

## 1. 函数/运算包裹列

```sql
-- 失效
WHERE YEAR(created_at) = 2026;
WHERE id + 1 = 10;
-- 正确：范围改写 / 常量化
WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';
WHERE id = 9;
-- 无法改写时用函数索引（8.0.13+）
```

## 2. 隐式类型转换

```sql
-- phone 是 VARCHAR
WHERE phone = 13800000001;        -- 字符串列与数字比较：列被转换 → 全表扫
WHERE phone = '13800000001';      -- 正确
-- user_id 是 BIGINT，传字符串不失效（数字列与字符串比较：常量被转换）
```

规则：**字符串列必须用字符串比较**。字符集/collation 不一致的 JOIN 列同理（见 20）。

## 3. 左模糊 / 全模糊

```sql
WHERE name LIKE '%张';    -- 失效
WHERE name LIKE '张%';    -- 可用（前缀）
WHERE name LIKE '%张%';   -- 失效（生产用 ES/倒排）
```

## 4. OR 连接非索引列

```sql
WHERE indexed_col = 1 OR unindexed_col = 2;   -- 整体退化为全表
-- 改写：UNION 两段，或给另一列也建索引
```

## 5. 最左前缀破坏

```sql
KEY idx(a, b)
WHERE b = 1;              -- 用不上
WHERE a = 1 AND b = 1;    -- 全用上
```

## 6. 负向条件

```sql
WHERE status != 1;        -- 可能走全表（优化器视选择性而定，非绝对失效）
WHERE status NOT IN (1,2);
-- 建议：正写 WHERE status IN (0,3,...) 或重构枚举语义
```

> 注意：`!=`/`NOT IN` 不是必然失效，range 优化器基于成本决定；但选择性差的负条件通常全表。

## 7. 其他场景

```sql
-- IS NULL / IS NOT NULL：可走索引（不是失效！常见误区），同样看选择性
-- 联合索引中间范围列截断：
KEY idx(a,b,c); WHERE a=1 AND b>10 AND c=3;  -- c 不能用于定位
-- 排序方向混用（8.0 前无降序索引）：
ORDER BY a ASC, b DESC;   -- (a,b) 索引失效，需 (a ASC, b DESC)
-- 大数据量 IN 优化器放弃：
WHERE id IN (...1万个值...)  -- 解析与优化成本爆炸，改 JOIN/临时表/分批
```

## 8. 优化器"主动"不走索引（成本决策）

索引选择性差（如性别列）、预估回表行数太多超过全表 20%~30% 时，优化器选全表——**这是合理行为**。手段：

```sql
ANALYZE TABLE t;                 -- 更新统计
FORCE INDEX(idx_x);              -- 强制（临时排查用，勿长期固化）
SELECT * FROM t IGNORE INDEX(idx_x) ...;
```

## 9. 排查流程

1. EXPLAIN 看 key=NULL 与 type
2. SHOW INDEX FROM t 确认索引存在
3. 逐条对照本清单（函数→类型→模糊→OR→最左→负向）
4. SHOW WARNINGS 看改写后的真实条件
5. 都没问题则查统计信息 / 数据倾斜

## 常见坑

1. `WHERE DATE(created_at)=CURDATE()` 日常报表越跑越慢 → 范围改写
2. MyBatis 传参类型不匹配（字符串列传 Long）触发隐式转换
3. "IS NULL 不走索引"的讹传
4. FORCE INDEX 上线后再加列导致索引名变化忘记清理

## Checklist

- [ ] 不对索引列做函数/运算
- [ ] 字符串列恒用字符串字面量比较
- [ ] 生产 SQL 变更跑 EXPLAIN 审查
- [ ] 疑难时先 ANALYZE TABLE 再 FORCE INDEX
