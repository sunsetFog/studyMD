# 11 - 常用函数

## 1. 字符串

```sql
CONCAT('a', 'b')                    -- 任一参数 NULL → 结果 NULL！用 CONCAT_WS(',','a','b') 忽略 NULL
LEFT(name, 1), RIGHT(name, 1), SUBSTRING(name, 2, 3)
LENGTH(name)                        -- 字节数！utf8mb4 中文 3~4 字节
CHAR_LENGTH(name)                   -- 字符数（业务一般用这个）
TRIM(name), UPPER(), LOWER()
REPLACE(str, 'from', 'to')
INSTR(str, 'sub'), LOCATE('sub', str)
LPAD(id, 6, '0')                    -- 补零
FORMAT(1234567.891, 2)              -- 千分位（返回字符串，金额展示别用它算数）
REPEAT('ab', 3)
```

典型场景：脱敏 `CONCAT(LEFT(phone,3), '****', RIGHT(phone,4))`。

## 2. 数值

```sql
ROUND(1.567, 2)     -- 四舍五入
TRUNCATE(1.567, 2)  -- 直接截断
CEIL(1.2), FLOOR(1.8)
MOD(10, 3)
RAND()
ABS(), POW(), SQRT()
```

**精度铁律**：金额运算一律 DECIMAL（见 16）；FLOAT/DOUBLE 有误差，`SUM(double_col)` 结果可能 0.30000000000000004。
判断相等避免 `=` 浮点：`ABS(a-b) < 0.0001`。

## 3. 日期时间（重点）

```sql
NOW(), CURDATE(), CURRENT_TIMESTAMP      -- 当前日期时间（会话时区）
DATE(), TIME(), YEAR(), MONTH(), DAY()
DATE_ADD(created_at, INTERVAL 7 DAY)     -- 加 7 天
DATE_SUB(now(), INTERVAL 1 HOUR)
DATEDIFF(d1, d2)                         -- 天数差
TIMESTAMPDIFF(SECOND, t1, t2)            -- 指定单位差值（更常用）
DATE_FORMAT(now(), '%Y-%m-%d %H:%i:%s')  -- 格式化
STR_TO_DATE('2026-01-01', '%Y-%m-%d')    -- 解析
UNIX_TIMESTAMP(now()), FROM_UNIXTIME(1700000000)   -- 时间戳互转
LAST_DAY(now())                          -- 月末
```

时区规则：
- **DATETIME 存的是"字面量"**，与时区无关；应用存什么取什么
- **TIMESTAMP 受 time_zone 影响**：存储转 UTC，读取转回会话时区，范围到 2038 年
- 建议：业务统一 `DATETIME + Asia/Shanghai`（连接串 serverTimezone），跨时区产品才考虑 TIMESTAMP/UTC 策略（详见 16）

## 4. 条件函数

```sql
IF(cond, a, b)
IFNULL(col, 0)                 -- NULL → 0
NULLIF(a, b)                   -- a=b 返回 NULL（防除零：x/NULLIF(y,0)）
COALESCE(a, b, c)              -- 第一个非 NULL（可变参数，优先于 IFNULL）
CASE WHEN status = 1 THEN '正常'
     WHEN status = 0 THEN '禁用'
     ELSE '未知' END
```

## 5. JSON 函数（8.0 常用）

```sql
SELECT JSON_EXTRACT(info, '$.address.city') FROM t;   -- 结果带引号
SELECT info->>'$.address.city' FROM t;                -- 去引号（等价 JSON_UNQUOTE）
SELECT info->'$.tags' FROM t;
JSON_OBJECT('k', v), JSON_ARRAY(1,2)
JSON_CONTAINS(info, '"vip"')
-- JSON 列可建多值索引（8.0.17+）
CREATE INDEX idx_tags ON t ((CAST(info->'$.tags' AS UNSIGNED ARRAY)));
SELECT * FROM t WHERE 3 MEMBER OF (info->'$.tags');
```

设计原则：**JSON 列适合"读多改少、结构不稳定"的扩展字段**；核心查询字段一定拍平成列（JSON 内字段无法像列一样高效索引与约束）。

## 6. 窗口/聚合之外的杂项

```sql
UUID()               -- 无序，不适合聚簇主键（见 21）
UUID_SHORT()
MD5(), SHA2(x, 256)  -- 8.0 也支持加密函数，敏感数据加密见 37
CONNECTION_ID(), DATABASE(), VERSION()
```

## 常见坑

1. `CONCAT` 中任意 NULL 导致整串 NULL → 用户名 NULL 时地址拼接为空
2. `LENGTH` vs `CHAR_LENGTH` 混淆，中文长度校验出错
3. `DATE_FORMAT(created_at, '%Y-%m')` 用于 WHERE/GROUP 使索引失效（见 24）
4. 日期函数用于 WHERE 条件列 → 改写为范围查询
5. TIMESTAMP 2038 上限 + 时区耦合

## Checklist

- [ ] 字符长度判断用 CHAR_LENGTH
- [ ] WHERE 中不对索引列套函数
- [ ] 金额全程 DECIMAL
- [ ] JSON 只做扩展存储
