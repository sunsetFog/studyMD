# 17 - 约束

## 1. 主键（PRIMARY KEY）

- 每表一个，NOT NULL + UNIQUE，InnoDB 中即聚簇索引（见 21）
- 首选自增 BIGINT：顺序插入无页分裂
- 业务键（身份证号）做主键的问题：变长、可能变更、占索引空间 → 用代理键 + 业务键 UNIQUE
- 分布式场景：雪花 ID（趋势递增 BIGINT，见 19）

```sql
PRIMARY KEY (id)
```

## 2. 唯一约束

```sql
UNIQUE KEY uk_username (username),
UNIQUE KEY uk_order_no (order_no),
-- 软删除表防重：删除后允许再次创建 → 把删除标记并入唯一键
UNIQUE KEY uk_username (username, deleted)
```

- 唯一键冲突错误码 1062，应用层要捕获转友好提示
- 与"先 SELECT 再 INSERT"相比，唯一键是并发下唯一可靠的去重手段

## 3. 外键（FOREIGN KEY）争议

```sql
CONSTRAINT fk_order_user FOREIGN KEY (user_id) REFERENCES user(id)
```

互联网大厂普遍**不用物理外键**，理由：
- 插入/更新时校验开销与锁传递（父表删除锁全子表）
- 分库分表后外键无法跨库
- 数据修正（清理脏数据）被卡死

替代：**应用层保证 + 定期对账脚本** + 连接列建索引。
保守建议：中小业务可留外键保数据干净；高并发/分库分表必去。

## 4. NOT NULL 与 DEFAULT

```sql
phone VARCHAR(20) NOT NULL DEFAULT ''
```

- NULL 列可存储/比较语义复杂（见 07 三值逻辑）、索引统计失真、`COUNT(col)` 口径分裂
- 允许"业务上未知"的字段（如实名认证时间）可用 NULL，但查询时必须显式 `IS NULL`
- 8.0.17 前 NULL 列仍额外占一个字节标记位

## 5. CHECK（8.0.16 起真正生效）

```sql
CHECK (age >= 0 AND age <= 150),
CHECK (status IN (0, 1, 2))
```

5.7 中 CHECK 会被解析但忽略（写入不报错）——迁移时的坑。枚举校验优先用 TINYINT+字典，CHECK 做兜底。

## 6. 自增细节

```sql
AUTO_INCREMENT=100000        -- 起始值
-- 持久化：8.0 重启不回退（5.7 重启会取 max(id)+1，回退到已删除的最大值）
innodb_autoinc_lock_mode = 2 -- 8.0 默认交错模式，并发 insert 性能最好
```

自增不连续的来源：批量插入取段、事务回滚、INSERT IGNORE 冲突——**不要用 id 差值推算行数**。

## 常见坑

1. 唯一键与软删除组合错误：`UNIQUE(username)` 删除后无法重建同名用户 → 见 2 的组合写法
2. 依赖外键级联删除（ON DELETE CASCADE）误删大面积数据
3. 字段没 NOT NULL，出现 NULL 导致 NOT IN/比较全错
4. 用 max(id) 判断数据量（自增不连续）

## Checklist

- [ ] 主键 BIGINT 自增或雪花
- [ ] 并发去重靠唯一键而非先查后插
- [ ] 分库分表场景无物理外键
- [ ] 关键字段 NOT NULL DEFAULT
