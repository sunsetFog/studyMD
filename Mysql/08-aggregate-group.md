# 08 - 聚合与分组

## 1. 聚合函数

```sql
SELECT COUNT(*), COUNT(phone), SUM(balance), AVG(age), MIN(age), MAX(age) FROM user;

COUNT(*)  -- 统计行数（含 NULL），最快语义
COUNT(col)     -- 不统计该列为 NULL 的行
COUNT(DISTINCT col)  -- 去重计数
```

- AVG/SUM 忽略 NULL；COUNT(col) 也忽略 NULL
- 空集上 SUM = NULL（不是 0），`IFNULL(SUM(x), 0)` 兜底
- 金额求和用 DECIMAL 精确；AVG 对 DECIMAL 结果是 DECIMAL，对 DOUBLE 有精度误差

## 2. GROUP BY

```sql
SELECT dept_id, COUNT(*) AS cnt, AVG(salary) AS avg_salary
FROM emp
WHERE status = 1
GROUP BY dept_id
HAVING cnt > 10        -- 对分组结果过滤
ORDER BY cnt DESC;
```

执行顺序：WHERE（过滤行）→ GROUP BY（分组）→ HAVING（过滤组）→ ORDER BY。

## 3. ONLY_FULL_GROUP_BY（8.0 默认开启）

规则：SELECT 出现的非聚合列必须都在 GROUP BY 里。违反报 `ERROR 1055`。

```sql
-- 报错：name 不在 GROUP BY 中
SELECT dept_id, name, COUNT(*) FROM emp GROUP BY dept_id;

-- 正确写法 1：把 name 聚合起来
SELECT dept_id, MAX(name) AS name, COUNT(*) FROM emp GROUP BY dept_id;

-- 正确写法 2：ANY_VALUE（明确告诉优化器"任选一个"）
SELECT dept_id, ANY_VALUE(name), COUNT(*) FROM emp GROUP BY dept_id;

-- 正确写法 3：窗口函数取每组第一行（见 12）
```

不要用关闭 ONLY_FULL_GROUP_BY 来"修复"——非确定语义的结果随时变。

## 4. 分组技巧

```sql
-- 按表达式分组（如按月统计）
SELECT DATE_FORMAT(created_at, '%Y-%m') AS month, COUNT(*) FROM orders
GROUP BY month ORDER BY month;

-- GROUP_CONCAT：组内拼接（默认截断 1024 字符！）
SET SESSION group_concat_max_len = 102400;
SELECT dept_id, GROUP_CONCAT(name ORDER BY name SEPARATOR ',') FROM emp GROUP BY dept_id;

-- WITH ROLLUP：小计/总计行
SELECT dept_id, COUNT(*) FROM emp GROUP BY dept_id WITH ROLLUP;
```

## 5. 分组性能

- `GROUP BY col` 且 col 有索引 → 可走索引避免临时表
- 无索引时触发 **Using temporary; Using filesort**（见 23）
- 大结果集分组优先在索引上做：`(dept_id, status)` 联合索引让 WHERE+GROUP 一次完成

## 常见坑

1. ERROR 1055 直接关 sql_mode → 隐患，用上面三种正解
2. GROUP_CONCAT 截断 → 调 group_concat_max_len
3. COUNT(col) 与 COUNT(*) 混用统计口径不一致 → 统一 COUNT(*)
4. 按天分组用 `DATE(created_at)` → 函数包裹列后索引失效（见 24），可用 `created_at >= ? AND < ?` 范围代替或生成列索引

## Checklist

- [ ] 能默写 WHERE→GROUP BY→HAVING→ORDER BY 顺序
- [ ] 遇到 1055 用 ANY_VALUE/聚合/窗口函数修正而非关 mode
- [ ] 统计口径统一 COUNT(*)
