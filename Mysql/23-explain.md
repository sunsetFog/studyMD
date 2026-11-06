# 23 - 执行计划 EXPLAIN

## 1. 基本用法

```sql
EXPLAIN SELECT ...;                  -- 传统格式
EXPLAIN FORMAT=TREE SELECT ...;      -- 树形（8.0，看 hash join/窗口）
EXPLAIN ANALYZE SELECT ...;          -- 实际执行并报真实耗时/行数（8.0.18+）
SHOW WARNINGS;                       -- EXPLAIN 后看优化器改写后的 SQL
```

## 2. id / select_type / table

- **id**：相同 id 从上到下执行；不同 id 大的先执行（子查询）
- **select_type**：SIMPLE（无子查询）、PRIMARY、SUBQUERY、DERIVED（派生表物化）、UNION RESULT

## 3. type：访问类型等级（核心）

从好到坏：

| type | 含义 | 场景 |
|---|---|---|
| system/const | 最多 1 行 | 主键/唯一键等值 |
| eq_ref | 被驱动表 1 行 | JOIN 用主键/唯一键 |
| ref | 被驱动表多行 | 普通索引等值 |
| range | 索引范围 | BETWEEN/IN/> |
| index | 全索引树扫描 | 扫整棵二级索引（比 ALL 快但不快） |
| ALL | 全表扫描 | 无索引/失效 |

**生产红线：核心表查询 type 不得低于 range。** index/ALL 都要优化。

## 4. key 相关列

```sql
possible_keys  -- 候选索引
key            -- 实际用的（NULL = 没用索引）
key_len        -- 用了索引的多少字节（判断联合索引用了几列）
rows           -- 预估扫描行数（越小越好）
filtered       -- 条件过滤比例估算
```

key_len 估算：TINYINT=1、INT=4、BIGINT=8、DECIMAL(12,2)=7、CHAR(n)=4n(utf8mb4)、VARCHAR(n)=4n+2；可 NULL 再 +1。用于验证联合索引用到第几列。

## 5. Extra（信息密度最高）

| Extra | 含义 | 应对 |
|---|---|---|
| Using index | 覆盖索引 | 好事 |
| Using where | 服务层过滤 | 正常 |
| Using index condition | ICP（索引条件下推） | 好事 |
| Using temporary | 临时表（GROUP BY/DISTINCT 无索引） | 优化 |
| Using filesort | 额外排序 | 给排序列建索引 |
| Using join buffer (hash join) | JOIN 无索引 | 给被驱动表连接列建索引 |
| Select tables optimized away | 聚合被直接优化（MIN/MAX） | 理想 |

## 6. EXPLAIN ANALYZE 实战

```sql
mysql> EXPLAIN ANALYZE SELECT o.id FROM orders o WHERE o.user_id=1 ORDER BY o.created_at DESC LIMIT 10;
-> Limit: 10 row(s)
  -> Index lookup on o using idx_user_created (cost=...) (actual rows=10 loops=1)
```

`cost=` 优化器估算，`actual rows=` 真实行数——两者差异大说明统计信息过期：

```sql
ANALYZE TABLE orders;   -- 重新收集统计信息
```

## 7. 分析顺序模板

1. type 是否 ≥ range？key 是否为 NULL（失效，见 24）？
2. rows 是否与数据量级匹配？
3. Extra 有没有 temporary/filesort？
4. JOIN 顺序是否小表驱动？被驱动表 key 有没有命中？
5. 需要真实数据时上 EXPLAIN ANALYZE

## 常见坑

1. 只看 key 非 NULL 就以为没问题——rows 巨大 + ALL 才是真问题
2. possible_keys 有但 key NULL：数据量太小优化器认为全表更快（正常）
3. `Using temporary; Using filesort` 组合出现
4. 统计信息过期导致选错索引 → ANALYZE TABLE

## Checklist

- [ ] 上线前核心 SQL 过 EXPLAIN
- [ ] 能按 id/type/key/Extra 顺序读计划
- [ ] 疑难 SQL 用 EXPLAIN ANALYZE 看真实成本
