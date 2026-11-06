# 09 - 多表连接 JOIN

## 1. JOIN 家族

```sql
-- 内连接：两边都匹配
SELECT u.username, o.amount
FROM user u JOIN orders o ON o.user_id = u.id;

-- 左连接：左表全保留，右表缺失补 NULL
SELECT u.username, o.amount
FROM user u LEFT JOIN orders o ON o.user_id = u.id;

-- 右连接：等价于交换 LEFT JOIN，可读性差，规范上少用
-- 交叉连接：笛卡尔积，几乎不用
CROSS JOIN / FROM a, b        -- 逗号写法已废弃
```

## 2. JOIN 结果可视化

```
A LEFT JOIN B ON cond     = A 全部 + B 匹配部分（无匹配 NULL）
A LEFT JOIN B ... WHERE B.col IS NULL   = A 独有部分（反连接）
A INNER JOIN B            = 交集
```

经典反连接：

```sql
-- 没下过单的用户
SELECT u.* FROM user u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;
-- 与 NOT EXISTS 等价（见 10），NOT EXISTS 语义更清晰
```

## 3. ON vs WHERE 的差异（LEFT JOIN 必考）

```sql
-- 1) 条件放 ON：左表仍全保留，只是右表匹配更少
SELECT u.id, o.amount FROM user u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 1;   -- 每个用户都出现，没有效订单则 amount NULL

-- 2) 条件放 WHERE 作用于右表：LEFT JOIN 退化成 INNER JOIN
SELECT u.id, o.amount FROM user u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 1;                                        -- 没订单的用户被过滤掉！
```

规则：**驱动表过滤放 WHERE，被驱动表过滤放 ON（LEFT JOIN 时）**。

## 4. 执行原理（理解 JOIN 性能）

- **驱动表（外层）**：先扫描的表。策略 `JOIN` 顺序由优化器基于成本决定
- **Simple Nested-Loop**：驱动表每行扫全被驱动表——极慢，8.0 不用
- **Index Nested-Loop（INLJ）**：被驱动表连接列有索引，走索引查找——最常见，这是「被驱动表连接列必须有索引」的根据
- **Block Nested-Loop（BNLJ）**：无索引时用 join buffer 分块，8.0.18 起被 Hash Join 取代
- **Hash Join（8.0.18+）**：小表建哈希表，大表探测；等值连接无索引时大幅提速

```sql
EXPLAIN FORMAT=TREE SELECT ... ;  -- 观察是否 hash join
```

## 5. JOIN 优化要点

1. 小表驱动大表（优化器通常自动选；`STRAIGHT_JOIN` 可强制顺序，慎用）
2. 被驱动表连接列建索引：`orders(user_id)` 必须有
3. JOIN 的列**类型/字符集必须一致**（隐式转换致索引失效，见 24）
4. 避免 JOIN 后大量行膨胀（一对多 SUM 场景先聚合再连）
5. 三表以上 JOIN 或多维度统计 → 考虑在应用层组装或落宽表/数仓

## 6. JOIN vs 子查询 vs 应用层组装

| 场景 | 推荐 |
|---|---|
| 单条 SQL 简单关联取数 | JOIN |
| 过滤存在性 | EXISTS / JOIN（见 10） |
| 微服务架构下跨库 | 应用层两次查询组装（数据不在一起） |
| 报表多维 | 宽表/数仓 |

## 常见坑

1. LEFT JOIN 后直接 COUNT(*) 得到比预期多的行（一对多重复计数）→ 先 GROUP BY 聚合再 JOIN，或 COUNT(DISTINCT u.id)
2. 连接列类型不同（int vs varchar）隐式转换全表扫
3. 被驱动表连接列忘建索引
4. LEFT JOIN + WHERE 右表条件误删行 → 移到 ON

## Checklist

- [ ] 被驱动表连接列有索引、类型一致
- [ ] LEFT JOIN 明确 ON/WHERE 条件放法
- [ ] 一对多统计先聚合再连接
