# 20 - 字符集与排序规则

## 1. 基础概念

- **字符集（charset）**：字符 → 字节的编码映射。MySQL 的 `utf8` 是坑：最多 3 字节，**存不了 emoji**（4 字节），真正的 UTF-8 是 `utf8mb4`
- **排序规则（collation）**：同一字符集下字符如何比较/排序

## 2. 铁律：utf8mb4

```ini
[mysqld]
character_set_server = utf8mb4
collation_server = utf8mb4_0900_ai_ci
```

```sql
CREATE TABLE t (...) DEFAULT CHARSET=utf8mb4;
-- 查看链条（任何一环不一致都可能乱码）
SHOW VARIABLES LIKE 'character%';
SHOW VARIABLES LIKE 'collation%';
```

字符集传递链：**客户端 → connection → server → database → table → column**。列级别显式指定可覆盖一切。

## 3. 排序规则选型（8.0）

| Collation | 规则 | 适用 |
|---|---|---|
| utf8mb4_0900_ai_ci | Unicode 9.0，Accent+Case 不敏感（默认） | 一般业务，8.0 默认 |
| utf8mb4_0900_as_cs | 区分大小写/重音 | 密码类、编码类字段 |
| utf8mb4_0900_bin | 按字节二进制比较 | token、哈希值 |
| utf8mb4_general_ci | 5.x 遗留，简陋 | 兼容旧库 |

- `ci` 不区分大小写：`WHERE code='abc'` 能命中 `'ABC'`——对验证码/密钥字段要 `as_cs` 或 `bin`
- **唯一索引在 ci 下 'a' 与 'A' 冲突**（1062 的隐蔽来源）

## 4. 转换与乱码排查

```sql
-- 列级转换
ALTER TABLE t CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

-- 乱码排查顺序
SHOW VARIABLES LIKE 'character_set_%';        -- connection 是否 utf8mb4
-- JDBC 连接串：characterEncoding=UTF-8
-- CLI：mysql --default-character-set=utf8mb4
```

经典乱码链路：客户端 latin1 → 写入 utf8mb4 表 → 显示再错一次。写入前 `SET NAMES utf8mb4` 统一。

## 5. 与索引/性能的关系

- 联合索引、JOIN 两列的**字符集和 collation 必须一致**，否则隐式转换 + 索引失效（见 24）
- VARCHAR(n) 的 n 是字符数，字节占用 = n × 4（utf8mb4 最坏）；索引长度限制 3072 字节按最坏算
- 前缀索引见 22

## 常见坑

1. emoji 写入报 `Incorrect string value` → 库/表/连接任一环还是 utf8(3字节)
2. 手机验证码字段 ci 排序规则下 'aA1B' 与 'Aa1b' 判重 → bin collation
3. 老表 utf8 与新表 utf8mb4 JOIN 报 `Illegal mix of collations`
4. `SET NAMES` 每次 CLI 忘设 → 写进 my.cnf [client]

## Checklist

- [ ] server/database/table 全链 utf8mb4
- [ ] 密码/令牌字段 collation 区分大小写
- [ ] JOIN 列字符集一致
