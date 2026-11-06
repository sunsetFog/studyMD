# 10 - 子查询

## 1. 分类

```sql
-- 标量子查询（返回一个值）
SELECT (SELECT MAX(amount) FROM orders) AS max_amount;

-- 列子查询（IN）
SELECT * FROM user WHERE id IN (SELECT user_id FROM orders WHERE amount > 1000);

-- 行子查询
SELECT * FROM emp WHERE (dept_id, salary) = (SELECT dept_id, MAX(salary) FROM emp GROUP BY dept_id LIMIT 1);

-- 派生表（FROM 子查询，8.0 可带 LIMIT）
SELECT t.dept_id, t.cnt FROM
  (SELECT dept_id, COUNT(*) cnt FROM emp GROUP BY dept_id) t WHERE t.cnt > 10;
```

## 2. IN vs EXISTS（经典面试题）

- `IN (子查询)`：子查询结果物化后与外表匹配；8.0 中优化器可半连接（semi-join）改写
- `EXISTS (子查询)`：外表逐行探测子查询是否命中

**选择原则：小表驱动大表**——但 8.0 优化器大多会自动改写，一般不用手选。仍需注意：

```sql
-- NOT IN 遇 NULL：子查询结果含 NULL 时整个 NOT IN 返回空！
SELECT * FROM user WHERE id NOT IN (SELECT user_id FROM orders);  -- user_id 有 NULL 则全空

-- 正解：NOT EXISTS
SELECT * FROM user u WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

`SELECT 1` 与 `SELECT *` 在 EXISTS 里性能相同（只判断存在性）。

## 3. 相关子查询的性能陷阱

```sql
-- 外表每行都执行一次子查询（相关）
SELECT u.username,
  (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM user u;
```

- 8.0 对 SELECT 列表中的相关标量子查询无法完全优化，大结果集时慢
- 改写为 JOIN + 聚合，或窗口函数（见 12）：

```sql
SELECT u.username, COUNT(o.id) AS order_count
FROM user u LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```

## 4. 派生表（子查询物化）的优化

8.0 会尝试将派生表**合并**进外层查询（derived_merge）或物化成临时表：
- 合并失败条件：派生表含 GROUP BY/DISTINCT/LIMIT/聚合
- 含 LIMIT 的派生表会物化，`EXPLAIN` 中 select_type=DERIVED

## 5. 子查询改写决策表

| 原写法 | 问题 | 改写 |
|---|---|---|
| NOT IN 子查询 | NULL 陷阱 | NOT EXISTS |
| SELECT 列表相关标量子查询 | 逐行执行 | JOIN+GROUP 或窗口函数 |
| FROM 派生表大结果 | 物化临时表 | 上提条件到外层/直接 JOIN |
| 多层嵌套子查询 | 不可读+难优化 | CTE（见 13） |

## 常见坑

1. NOT IN 遇 NULL 返回空集（最隐蔽的逻辑 bug 之一）
2. 派生表里 LIMIT 与外层 JOIN 组合结果不确定（没有稳定排序）
3. 相关子查询在大表上逐行执行退化
4. 子查询里 ORDER BY 无 LIMIT 会被优化器直接丢弃（SQL 语义如此）

## Checklist

- [ ] 一律用 NOT EXISTS 替代 NOT IN 子查询
- [ ] 相关标量子查询在主流程大结果集中改写
- [ ] 嵌套超过 2 层改用 CTE
