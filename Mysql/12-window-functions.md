# 12 - 窗口函数（8.0）

## 1. 是什么

窗口函数在**不合并行**的前提下对一组行做计算（区别于 GROUP BY 合并行）。8.0 起可用，彻底改变"每组取前 N"这类查询的写法。

执行位置：SELECT 阶段（WHERE 之后），所以 **WHERE/HAVING 不能引用窗口结果**，需包一层外查询。

## 2. 排名函数

```sql
SELECT name, dept_id, salary,
  ROW_NUMBER() OVER w AS rn,       -- 连续序号 1,2,3
  RANK()       OVER w AS rnk,      -- 并列同号，跳号 1,1,3
  DENSE_RANK() OVER w AS drnk      -- 并列同号不跳 1,1,2
FROM emp
WINDOW w AS (PARTITION BY dept_id ORDER BY salary DESC);
```

经典题：**每组薪资最高的前 3 名**

```sql
SELECT * FROM (
  SELECT e.*, ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
  FROM emp e
) t WHERE rn <= 3;
```

## 3. 偏移函数

```sql
LAG(salary, 1)   OVER (ORDER BY month)  -- 上一行
LEAD(salary, 1)  OVER (ORDER BY month)  -- 下一行
FIRST_VALUE(salary) OVER w
LAST_VALUE(salary)  OVER w    -- 注意默认窗口是"到当前行"，要 ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```

场景：环比 `amount / LAG(amount) OVER (ORDER BY month) - 1`。

## 4. 聚合函数做窗口

```sql
SUM(amount) OVER (ORDER BY day)              -- 累计值（Running Total）
SUM(amount) OVER (PARTITION BY dept)         -- 组内总和（每行都带）
AVG(amount) OVER (ORDER BY day ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)  -- 3日移动平均
COUNT(*) OVER (PARTITION BY user_id)         -- 每行的组内计数
```

## 5. 框架子句

```sql
ROWS BETWEEN <start> AND <end>
  UNBOUNDED PRECEDING | n PRECEDING | CURRENT ROW | n FOLLOWING | UNBOUNDED FOLLOWING
RANGE BETWEEN ...   -- 按值范围（默认 ORDER BY 时为 RANGE UNBOUNDED..CURRENT ROW）
```

记忆：默认帧 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`——这就是 LAST_VALUE 坑的来源。

## 6. 窗口 vs GROUP BY 选型

| 需求 | 用什么 |
|---|---|
| 按组汇总一行结果 | GROUP BY |
| 每行保留明细且带组统计 | 窗口函数 |
| 组内 Top-N | 窗口 ROW_NUMBER |
| 环比/同比 | LAG/LEAD |
| 累计/滑动平均 | SUM/AVG OVER |

## 7. 性能

- 窗口排序可能触发 filesort，`PARTITION BY col ORDER BY ...` 有匹配索引（col, ...）可减负
- 大结果集多窗口 → 先 WHERE 缩小数据
- 8.0 优化器不合并多个相同窗口定义，用命名 WINDOW 复用

## 常见坑

1. WHERE 里写窗口函数报错 → 外层查询过滤
2. LAST_VALUE 取到当前行而非组末 → 显式指定 ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
3. ROW_NUMBER 无唯一排序键导致并列行序号不稳定 → ORDER BY salary DESC, id
4. MySQL 5.7 无窗口函数 → 用变量模拟或升级

## Checklist

- [ ] 会写组内 Top-N 模板
- [ ] 理解默认窗口帧
- [ ] 窗口排序带唯一次序键
