# 13 - CTE 与递归查询

## 1. CTE（WITH 子句，8.0）

```sql
WITH
  active_users AS (
    SELECT id, username FROM user WHERE status = 1
  ),
  big_orders AS (
    SELECT user_id, SUM(amount) total FROM orders
    WHERE amount > 1000 GROUP BY user_id
  )
SELECT u.username, b.total
FROM active_users u JOIN big_orders b ON b.user_id = u.id;
```

价值：
- 替代多层嵌套子查询，自顶向下可读
- 同一查询内可复用（引用多次只写一次）
- 比派生表更利于优化器做半连接改写

## 2. 递归 CTE：查树形结构

经典场景：组织架构、分类树、评论楼层。

```sql
-- 建表示例
CREATE TABLE dept (
  id BIGINT PRIMARY KEY,
  name VARCHAR(64) NOT NULL,
  parent_id BIGINT NULL          -- NULL 为根
);

-- 向下查：某部门及所有子孙
WITH RECURSIVE sub AS (
  SELECT id, name, parent_id, 1 AS level FROM dept WHERE id = 100   -- 锚点
  UNION ALL
  SELECT d.id, d.name, d.parent_id, s.level + 1
  FROM dept d JOIN sub s ON d.parent_id = s.id                      -- 递归部分
)
SELECT * FROM sub;

-- 向上查：祖先链
WITH RECURSIVE up AS (
  SELECT id, name, parent_id FROM dept WHERE id = 205
  UNION ALL
  SELECT d.id, d.name, d.parent_id FROM dept d JOIN up ON up.parent_id = d.id
)
SELECT * FROM up;
```

## 3. 递归安全阀

坏数据（环）会让递归无限循环 → 限制深度：

```sql
WITH RECURSIVE sub AS (
  SELECT id, name, parent_id, 1 AS level FROM dept WHERE id = 100
  UNION ALL
  SELECT d.id, d.name, d.parent_id, s.level + 1
  FROM dept d JOIN sub s ON d.parent_id = s.id
  WHERE s.level < 10                    -- 深度熔断
)
SELECT * FROM sub;

-- 全局深度限制参数
SET SESSION cte_max_recursion_depth = 1000;   -- 默认 1000
```

## 4. 其他递归应用

```sql
-- 生成连续日期序列（补齐报表空档）
WITH RECURSIVE dates AS (
  SELECT DATE '2026-01-01' AS d
  UNION ALL
  SELECT d + INTERVAL 1 DAY FROM dates WHERE d < '2026-01-31'
)
SELECT dates.d, COALESCE(o.cnt, 0) FROM dates
LEFT JOIN (SELECT DATE(created_at) d, COUNT(*) cnt FROM orders GROUP BY d) o
  ON o.d = dates.d;
```

## 5. 树形存储方案对比

| 方案 | 查子树 | 写入 | 适用 |
|---|---|---|---|
| 邻接表（parent_id）+ 递归 CTE | 一条递归 SQL | 简单 | 8.0 首选，层级有限 |
| 路径枚举（path 列 '1/5/12/'） | LIKE 'prefix%' 前缀索引 | 改名要重写子树 | 层级浅 |
| 左右值编码（nested set） | 极快 | 慢且复杂 | 读多写少的分类树 |
| 闭包表（ancestor/descendant 全量存） | 快 | 维护行多 | 复杂查询需求 |

## 常见坑

1. 递归无熔断遇环数据跑满 cte_max_recursion_depth 报错
2. `UNION ALL` vs `UNION`：递归中 UNION 会去重（慢），一般用 UNION ALL
3. 递归结果行数爆炸（宽组织树）→ LIMIT 提前截断
4. 5.7 不支持 CTE → 升级或应用层组装/路径方案

## Checklist

- [ ] 递归 CTE 都有深度限制
- [ ] 树形查询优先递归 CTE + parent_id
- [ ] 日期补齐用递归序列 + LEFT JOIN
