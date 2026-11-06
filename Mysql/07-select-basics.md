# 07 - 查询基础

## 1. SELECT 逻辑执行顺序（理解一切的基础）

```
FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT(含窗口函数) → DISTINCT → ORDER BY → LIMIT
```

由此推导的关键规则：
- **WHERE 里不能用 SELECT 的别名**（SELECT 在其后执行）
- **ORDER BY 可以用别名**（SELECT 在其前执行）
- **HAVING 可以用聚合别名**
- 窗口函数在 WHERE 之后执行，所以 WHERE 过滤不了窗口结果，要套一层子查询（见 12）

## 2. 基础语法

```sql
SELECT id, username FROM user WHERE status = 1 AND balance > 100 ORDER BY created_at DESC LIMIT 10;

SELECT DISTINCT dept_id FROM emp;                  -- 去重
SELECT * FROM user LIMIT 100000, 20;               -- 深分页性能炸弹（见 30）
SELECT id FROM user LIMIT 100000, 20;              -- 深分页优化：先查 id
```

## 3. WHERE 条件运算符

```sql
WHERE age BETWEEN 18 AND 30          -- 闭区间
WHERE dept_id IN (1,2,3)
WHERE name LIKE '张%'                 -- 前缀匹配可走索引；'%张' 不走（见 24）
WHERE deleted = 0 AND status IN (1,2)
WHERE created_at >= '2026-01-01' AND created_at < '2026-02-01'   -- 时间范围标准写法
WHERE email IS NULL                  -- NULL 判断必须 IS NULL，= NULL 恒为 UNKNOWN
WHERE NOT (a = 1)                    -- NOT 会弱化索引利用，尽量正写
```

**NULL 三值逻辑**：任何与 NULL 的比较结果都是 UNKNOWN（不是 TRUE 也不是 FALSE），WHERE 只保留 TRUE。这是「字段 NOT NULL DEFAULT」铁律的根源。

## 4. ORDER BY 排序

```sql
ORDER BY created_at DESC, id DESC;   -- 多列排序
ORDER BY FIELD(status, 3, 1, 2);     -- 自定义顺序
```

- 排序在内存/磁盘可能触发 **filesort**（见 23 Extra 字段）
- 分页排序字段无唯一值时结果不稳定 → 补充唯一列做次序键：`ORDER BY created_at DESC, id DESC`

## 5. LIMIT 分页

```sql
LIMIT 20             -- 前 20 条
LIMIT 20, 10         -- 跳过 20 条取 10 条（offset, size）
LIMIT 10 OFFSET 20   -- 同上，可读性更好
```

分页止步建议：业务上限制最深页码（如 Google 只给 1000 条），技术优化见 30。

## 6. SELECT * 的代价

- 无法覆盖索引（见 22），多回表 IO
- 大字段（TEXT/BLOB）被无谓拉取
- 表结构变更后 ORM 映射隐式问题
→ 明确列出所需列。

## 常见坑

1. `WHERE name = 'a' OR name IS NULL` 结果缺失/多出 → NULL 逻辑理清
2. 时间比较 `created_at = '2026-01-01'` 漏掉当天数据（DATETIME 含时分秒）→ 用 >= 次日 < 区间
3. 分页翻到最后发现重复/丢行 → 排序不唯一，补唯一键
4. LIMIT 大 offset 越翻越慢 → 见 30

## Checklist

- [ ] 不写 SELECT *
- [ ] 时间范围用 [start, next_start) 区间
- [ ] 分页排序带唯一列
- [ ] 理解 SELECT 逻辑执行顺序并能推导别名可用性
