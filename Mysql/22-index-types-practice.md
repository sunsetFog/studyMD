# 22 - 索引分类与实战

## 1. 索引全景

```
按数据结构：B+树索引（主）、FULLTEXT、空间索引(R-Tree)、哈希(AHI 内部)
按物理存储：聚簇索引（主键）、二级索引（辅助）
按列数：    单列、联合（复合）
按唯一性：  普通、唯一（PRIMARY/UNIQUE）
按用途：    前缀索引、覆盖索引（用法而非类型）、降序索引、函数索引（8.0.13+）、不可见索引
```

## 2. 联合索引与最左前缀

```sql
KEY idx_user_status_created (user_id, status, created_at)
```

- 本质：按 (user_id, status, created_at) **字典序**排序存储
- 可命中：`user_id=?`、`user_id=? AND status=?`、三者全用、`user_id=? AND created_at BETWEEN`（跳过中间列只用到 user_id，范围列之后的列失效）
- 不可命中：只查 `status` / `created_at`（没有最左列）

规则细化：
1. **等值列在前，范围列在后**：`(status, created_at)` 中 created_at 放最后
2. **范围列之后的索引列失效**（只能用于过滤，不能用于定位）
3. 排序：`ORDER BY created_at` 需要等值部分封顶后排序可直接用索引（免 filesort）
4. **索引即排序**：联合索引对首列相同的数据按后续列有序——这是设计"免排序"索引的原理

## 3. 覆盖索引

```sql
-- 只查索引内字段：免回表
SELECT id, status FROM orders WHERE user_id = 1;        -- idx(user_id, status) 覆盖
SELECT * FROM orders WHERE user_id = 1;                 -- 需回表
```

`EXPLAIN` 中 Extra 显示 `Using index`。高频列表页（只取几个字段）设计覆盖索引收益巨大。

## 4. 前缀索引

```sql
-- 长字符串只索引前 N 个字符，省空间（但无法覆盖索引/排序）
ALTER TABLE user ADD INDEX idx_email_p10 (email(10));
-- 选 N：区分度接近全列即可
SELECT COUNT(DISTINCT LEFT(email, 10)) / COUNT(DISTINCT email) FROM user;
```

## 5. 其他实用索引

```sql
-- 降序索引（8.0 真降序存储）：混合排序
CREATE INDEX idx_created_desc_id ON orders (created_at DESC, id DESC);

-- 函数索引：不改 SQL 即可对函数结果索引
CREATE INDEX idx_month ON orders ((DATE_FORMAT(created_at, '%Y-%m')));

-- 隐藏索引：下线灰度（优化器不可见，仍维护）
ALTER TABLE t ALTER INDEX idx_a INVISIBLE;

-- FULLTEXT：中文需 ngram
ALTER TABLE article ADD FULLTEXT INDEX ft_title (title) WITH PARSER ngram;
SELECT * FROM article WHERE MATCH(title) AGAINST ('数据库');
```

> 全文检索/模糊搜索生产一般选 Elasticsearch，MySQL FULLTEXT 仅轻量场景。

## 6. 索引设计工作流

1. 从**查询清单**出发（不是从表出发）：列出高频 WHERE/ORDER BY/JOIN 列组合
2. 每个高频查询设计一个联合索引（等值列在前、范围/排序列在后、尽量覆盖 SELECT 列）
3. 用最左前缀检查是否可被已有索引覆盖，删掉冗余索引
4. 上线用 pt-index-usage/slowlog 复核，长期未用索引下线（INVISIBLE → 观察期 → DROP）

## 常见坑

1. 索引列在等值与范围之间插错位置（范围列放中间）
2. `WHERE a=1 AND b>10 AND c=3`：c 用不上 (a,b,c) 索引定位
3. 为每个 WHERE 单列各建一个索引，而不是一个联合索引
4. 前缀索引后想覆盖索引落空（前缀索引存的是截断值）
5. 唯一索引与普通索引的选择：写入无大差异，**优先普通索引**（change buffer，见 31）除非业务要求唯一

## Checklist

- [ ] 索引按查询清单设计而非乱加
- [ ] 联合索引：等值 → 范围 → 排序
- [ ] 高频列表查询有覆盖索引
- [ ] 知道 INVISIBLE 安全下线索索
