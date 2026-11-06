# 16 - 数据类型选型

## 选型原则

**够用的最小类型 + 精确类型 + 永不用 FLOAT 存钱。** 小类型 → 更小索引、一页更多行、更少 IO。

## 1. 整型

| 类型 | 字节 | 范围（有符号） | 用途 |
|---|---|---|---|
| TINYINT | 1 | -128~127 | 状态、布尔（0/1） |
| SMALLINT | 2 | ±3.2万 | 小枚举 |
| MEDIUMINT | 3 | ±838万 | 少用 |
| INT | 4 | ±21亿 | 中间量 |
| BIGINT | 8 | ±9.2×10^18 | 主键、订单号、计数器 |

- 手机号/订单号/雪花 ID 一律 **BIGINT**（INT 上限 21 亿不够且无扩展空间）
- 布尔用 `TINYINT(1)`；BOOL 是 TINYINT(1) 别名
- 显示宽度 `INT(11)` 已废弃语义（8.0.17 起忽略），不必写

## 2. 小数：DECIMAL 铁律

```sql
DECIMAL(12,2)    -- 共 12 位，2 位小数。金额标准
```

- FLOAT/DOUBLE 是近似值：`0.1+0.2 != 0.3`，SUM 会漂移——**任何涉及钱的字段禁用**
- 百分比/汇率精度高也用 DECIMAL
- 科学计算用 DOUBLE；FLOAT 基本不用

## 3. 字符串：VARCHAR vs CHAR

```sql
VARCHAR(64)   -- 变长，实际按内容存储 + 1~2 字节长度
CHAR(11)      -- 定长，不足补空格；适合固定长度
```

- **长度必须从业务反推**：用户名 64、昵称 128、手机号 11（CHAR(11)）、身份证 18、摘要 255
- VARCHAR 上限 65535 字节（行级限制，与 utf8mb4 无关，utf8mb4 下 1 字符最多 4 字节）
- 超长文本：TEXT（64KB）/MEDIUMTEXT(16MB)/LONGTEXT——**大字段拆独立表**（见 19），避免拖累主查询的 IO
- ENUM：新增值要 DDL，扩展性差，用 TINYINT + 字典表或应用枚举

## 4. 日期时间

| 类型 | 字节 | 范围 | 时区 | 用途 |
|---|---|---|---|---|
| DATETIME | 8 | 1000-9999 | 无关 | **业务首选** |
| TIMESTAMP | 4 | 1970~2038 | 随时区转换 | 跨时区/短时间戳 |
| DATE | 3 | 日期 | - | 生日 |
| TIME | 3 | 时长 | - | 时长 |
| BIGINT 时间戳 | 8 | 无限 | - | 对外 API/日志场景 |

决策：
- 业务表 `DATETIME` + `DEFAULT CURRENT_TIMESTAMP`（写字面量，时区由应用规范管理）
- TIMESTAMP 2038 上限是硬伤
- 精确到微秒：`DATETIME(6)`
- 只精确到天：DATE

## 5. 布尔/枚举/JSON

```sql
status TINYINT NOT NULL DEFAULT 1 COMMENT '1正常 0禁用'
ext_info JSON NULL                     -- 扩展属性，核心字段不进 JSON（见 11）
is_deleted TINYINT NOT NULL DEFAULT 0  -- 软删除
```

## 6. 类型转换成本

改类型（尤其改窄、改字段位置）都要重建表 → 大表 DDL 走 pt-osc（见 05）。**上线前把类型定对，比上线后改便宜 100 倍。**

## 常见坑

1. DOUBLE 存金额，对账差一分钱
2. 手机号存 INT：首位 0 丢失、INT 上限不够 → CHAR(11) 或 BIGINT（不带 +86 语义时）
3. VARCHAR(255) 滥用：联合索引受 3072 字节限制，前缀变短
4. TIMESTAMP 跨 2038 报错
5. `created_at` 用字符串 VARCHAR 存 → 无法范围索引、比较错乱

## Checklist

- [ ] 金额 DECIMAL、主键 BIGINT、状态 TINYINT
- [ ] 字符长度业务反推，固定长度用 CHAR
- [ ] DATETIME 做业务时间，大文本拆表
