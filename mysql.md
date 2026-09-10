# MySQL 知识点总结

## 1. 简介

MySQL 是最流行的开源关系型数据库，使用 SQL 语言操作，数据以表的形式存储，支持事务、索引、视图等。

特点：
- 开源免费、性能高
- 支持多种存储引擎（InnoDB 默认）
- 支持事务（ACID）
- 跨平台

## 2. 数据类型

| 类型 | 说明 | 示例 |
|------|------|------|
| INT | 整数 | 100 |
| BIGINT | 长整数 | 1000000000 |
| DECIMAL | 精确小数 | DECIMAL(10,2) |
| VARCHAR | 变长字符串 | VARCHAR(255) |
| CHAR | 定长字符串 | CHAR(10) |
| TEXT | 长文本 | - |
| DATE | 日期 | 2026-08-26 |
| DATETIME | 日期时间 | 2026-08-26 10:30:00 |
| TIMESTAMP | 时间戳 | - |
| BOOLEAN | 布尔（TINYINT） | 0/1 |

## 3. 数据库操作

```
-- 创建数据库
CREATE DATABASE mydb;

-- 创建数据库（指定字符集）
CREATE DATABASE mydb DEFAULT CHARACTER SET utf8mb4;

-- 查看所有数据库
SHOW DATABASES;

-- 使用数据库
USE mydb;

-- 删除数据库
DROP DATABASE mydb;
```

## 4. 创建表

```
CREATE TABLE user (
    id INT PRIMARY KEY AUTO_INCREMENT,   -- 主键，自增
    name VARCHAR(50) NOT NULL,           -- 非空
    age INT DEFAULT 0,                   -- 默认值
    email VARCHAR(100) UNIQUE,           -- 唯一约束
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 查看表结构
DESC user;

-- 查看建表语句
SHOW CREATE TABLE user;

-- 删除表
DROP TABLE user;
```

## 5. 约束

```
-- 主键：唯一标识一行
id INT PRIMARY KEY

-- 外键：引用另一张表的主键
user_id INT,
FOREIGN KEY (user_id) REFERENCES user(id)

-- 唯一：值不能重复
email VARCHAR(100) UNIQUE

-- 非空
name VARCHAR(50) NOT NULL

-- 默认值
status INT DEFAULT 0

-- 检查（MySQL 8.0+ 支持）
age INT CHECK (age >= 0)
```

## 6. 插入数据 INSERT

```
-- 插入单行
INSERT INTO user (name, age) VALUES ('tom', 20);

-- 插入多行
INSERT INTO user (name, age) VALUES
    ('jerry', 21),
    ('lucy', 22);

-- 插入所有字段（可省略列名）
INSERT INTO user VALUES (1, 'tom', 20, 'tom@test.com', NOW());

-- 从查询结果插入
INSERT INTO user_backup SELECT * FROM user;
```

## 7. 更新数据 UPDATE

```
-- 更新单行
UPDATE user SET age = 21 WHERE id = 1;

-- 更新多列
UPDATE user SET age = 21, email = 'new@test.com' WHERE id = 1;

-- 批量更新（不加 WHERE 会更新所有行，危险）
UPDATE user SET status = 1 WHERE age > 20;
```

注意：UPDATE 和 DELETE 一定要带 WHERE，否则会操作整张表。

## 8. 删除数据 DELETE

```
-- 删除指定行
DELETE FROM user WHERE id = 1;

-- 删除所有行（保留表结构）
DELETE FROM user;

-- 清空表（比 DELETE 快，重置自增）
TRUNCATE TABLE user;
```

## 9. 查询 SELECT

```
-- 查询所有列
SELECT * FROM user;

-- 查询指定列
SELECT name, age FROM user;

-- 列别名
SELECT name AS 姓名, age AS 年龄 FROM user;

-- 去重
SELECT DISTINCT age FROM user;

-- 计算列
SELECT name, age * 2 AS double_age FROM user;
```

## 10. WHERE 条件

```
-- 比较运算
SELECT * FROM user WHERE age > 20;
SELECT * FROM user WHERE age >= 20 AND age <= 30;

-- 逻辑运算
SELECT * FROM user WHERE age > 20 AND status = 1;
SELECT * FROM user WHERE age < 18 OR age > 60;

-- IN：匹配多个值
SELECT * FROM user WHERE age IN (20, 21, 22);

-- BETWEEN：范围
SELECT * FROM user WHERE age BETWEEN 20 AND 30;

-- IS NULL：判空
SELECT * FROM user WHERE email IS NULL;
SELECT * FROM user WHERE email IS NOT NULL;

-- LIKE：模糊匹配
SELECT * FROM user WHERE name LIKE 't%';    -- 以 t 开头
SELECT * FROM user WHERE name LIKE '%om';   -- 以 om 结尾
SELECT * FROM user WHERE name LIKE '%m%';   -- 包含 m
SELECT * FROM user WHERE name LIKE '_om';   -- 单字符占位
```

## 11. 排序 ORDER BY

```
-- 升序（默认）
SELECT * FROM user ORDER BY age;

-- 降序
SELECT * FROM user ORDER BY age DESC;

-- 多列排序
SELECT * FROM user ORDER BY age DESC, id ASC;
```

## 12. 分组 GROUP BY / HAVING

```
-- 按部门分组统计人数
SELECT dept, COUNT(*) FROM user GROUP BY dept;

-- 分组后过滤（HAVING 在分组后，WHERE 在分组前）
SELECT dept, COUNT(*) AS cnt
FROM user
WHERE status = 1          -- 先过滤
GROUP BY dept             -- 再分组
HAVING cnt > 5            -- 分组后过滤
ORDER BY cnt DESC;        -- 排序
```

## 13. 聚合函数

```
COUNT(*)        -- 行数
SUM(column)     -- 求和
AVG(column)     -- 平均值
MAX(column)     -- 最大值
MIN(column)     -- 最小值
```

```
SELECT COUNT(*) FROM user;          -- 总行数
SELECT AVG(age) FROM user;          -- 平均年龄
SELECT MAX(age), MIN(age) FROM user; -- 最大最小年龄
SELECT SUM(salary) FROM user;       -- 工资总和
```

## 14. 多表连接 JOIN

```
-- INNER JOIN：取两表匹配的行
SELECT u.name, o.order_no
FROM user u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN：左表全保留，右表匹配不到为 NULL
SELECT u.name, o.order_no
FROM user u
LEFT JOIN orders o ON u.id = o.user_id;

-- RIGHT JOIN：右表全保留
SELECT u.name, o.order_no
FROM user u
RIGHT JOIN orders o ON u.id = o.user_id;

-- 多表连接
SELECT u.name, o.order_no, p.product_name
FROM user u
JOIN orders o ON u.id = o.user_id
JOIN product p ON o.product_id = p.id;
```

## 15. 子查询

```
-- WHERE 中的子查询
SELECT * FROM user
WHERE age > (SELECT AVG(age) FROM user);

-- IN 子查询
SELECT * FROM user
WHERE dept IN (SELECT dept FROM dept WHERE status = 1);

-- FROM 中的子查询（派生表）
SELECT * FROM (
    SELECT name, age FROM user WHERE age > 20
) AS t
WHERE t.age < 30;
```

## 16. 分页 LIMIT

```
-- 前 10 条
SELECT * FROM user LIMIT 10;

-- 跳过 10 条，取 10 条（第 2 页）
SELECT * FROM user LIMIT 10, 10;

-- 等价写法（推荐，更清晰）
SELECT * FROM user LIMIT 10 OFFSET 10;

-- 分页公式：LIMIT (page-1)*size, size
```

## 17. 索引

索引加快查询速度，但会降低写入性能和占用空间。

```
-- 普通索引
CREATE INDEX idx_name ON user(name);

-- 唯一索引
CREATE UNIQUE INDEX idx_email ON user(email);

-- 组合索引
CREATE INDEX idx_name_age ON user(name, age);

-- 建表时定义索引
CREATE TABLE user (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    INDEX idx_name (name)
);

-- 查看索引
SHOW INDEX FROM user;

-- 删除索引
DROP INDEX idx_name ON user;
```

## 18. 事务

事务保证一组操作要么全部成功，要么全部失败（ACID）。

```
-- 开始事务
START TRANSACTION;

UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;

-- 提交（成功）
COMMIT;

-- 回滚（失败）
ROLLBACK;
```

ACID：
- 原子性（Atomicity）：操作不可分割
- 一致性（Consistency）：数据保持一致
- 隔离性（Isolation）：事务互不干扰
- 持久性（Durability）：提交后永久保存

## 19. 隔离级别

```
-- 查看隔离级别
SELECT @@transaction_isolation;

-- 设置隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

| 级别 | 脏读 | 不可重复读 | 幻读 |
|------|------|-----------|------|
| READ UNCOMMITTED | 会 | 会 | 会 |
| READ COMMITTED | 不会 | 会 | 会 |
| REPEATABLE READ（默认） | 不会 | 不会 | 会 |
| SERIALIZABLE | 不会 | 不会 | 不会 |

## 20. 视图 VIEW

视图是虚拟表，封装复杂查询。

```
-- 创建视图
CREATE VIEW user_view AS
SELECT id, name, age FROM user WHERE age > 18;

-- 查询视图（像查表一样）
SELECT * FROM user_view;

-- 修改视图
ALTER VIEW user_view AS SELECT id, name FROM user;

-- 删除视图
DROP VIEW user_view;
```

## 21. 存储过程

封装一组 SQL，可复用。

```
DELIMITER $$                    -- 修改结束符

CREATE PROCEDURE get_users(IN min_age INT)
BEGIN
    SELECT * FROM user WHERE age > min_age;
END$$

DELIMITER ;                     -- 恢复结束符

-- 调用
CALL get_users(20);

-- 删除
DROP PROCEDURE get_users;
```

## 22. 触发器

在表发生 INSERT/UPDATE/DELETE 时自动执行。

```
DELIMITER $$

CREATE TRIGGER before_user_insert
BEFORE INSERT ON user
FOR EACH ROW
BEGIN
    SET NEW.created_at = NOW();   -- 自动填充时间
END$$

DELIMITER ;

-- 删除触发器
DROP TRIGGER before_user_insert;
```

## 23. 常用字符串函数

```
CONCAT('a', 'b')         -- 'ab'，拼接
UPPER('abc')             -- 'ABC'
LOWER('ABC')             -- 'abc'
LENGTH('hello')          -- 5，长度
SUBSTRING('hello', 1, 3) -- 'hel'，截取
TRIM('  hi  ')           -- 'hi'，去空格
REPLACE('abc', 'b', 'x') -- 'axc'，替换
LEFT('hello', 2)         -- 'he'
RIGHT('hello', 3)        -- 'llo'
```

## 24. 常用日期函数

```
NOW()                    -- 当前日期时间
CURDATE()                -- 当前日期
CURTIME()                -- 当前时间
YEAR('2026-08-26')       -- 2026
MONTH('2026-08-26')      -- 8
DAY('2026-08-26')        -- 26
DATE_FORMAT(NOW(), '%Y-%m-%d')  -- '2026-08-26'
DATEDIFF('2026-08-26', '2026-08-20')  -- 6，天数差
DATE_ADD(NOW(), INTERVAL 1 DAY)   -- 加一天
```

## 25. 常用数学函数

```
ABS(-5)      -- 5，绝对值
ROUND(3.56)  -- 4，四舍五入
ROUND(3.56, 1)  -- 3.6，保留 1 位小数
CEIL(3.1)    -- 4，向上取整
FLOOR(3.9)   -- 3，向下取整
MOD(10, 3)   -- 1，取余
RAND()       -- 0~1 随机数
```

## 26. 表结构修改 ALTER

```
-- 添加列
ALTER TABLE user ADD COLUMN phone VARCHAR(20);

-- 修改列类型
ALTER TABLE user MODIFY COLUMN phone VARCHAR(30);

-- 修改列名和类型
ALTER TABLE user CHANGE COLUMN phone mobile VARCHAR(20);

-- 删除列
ALTER TABLE user DROP COLUMN phone;

-- 重命名表
ALTER TABLE user RENAME TO user_info;
```

## 27. SQL 优化

```
-- 1. 用 EXPLAIN 分析查询
EXPLAIN SELECT * FROM user WHERE name = 'tom';

-- 2. 避免 SELECT *，只查需要的列
SELECT id, name FROM user;

-- 3. 使用索引（WHERE/ORDER BY/JOIN 的字段）

-- 4. 避免在 WHERE 中对列做函数运算（索引失效）
-- 错误：SELECT * FROM user WHERE YEAR(created_at) = 2026
-- 正确：SELECT * FROM user WHERE created_at >= '2026-01-01'

-- 5. 避免使用 % 开头的模糊查询（索引失效）
-- SELECT * FROM user WHERE name LIKE '%tom'

-- 6. 使用 LIMIT 限制返回行数
SELECT * FROM user LIMIT 100;
```

## 28. UNION 合并结果

```
-- UNION：合并去重
SELECT name FROM user_a
UNION
SELECT name FROM user_b;

-- UNION ALL：合并不去重（更快）
SELECT name FROM user_a
UNION ALL
SELECT name FROM user_b;
```

## 29. 常用命令速查

| 命令 | 作用 |
|------|------|
| SHOW DATABASES | 查看数据库 |
| SHOW TABLES | 查看表 |
| DESC table | 查看表结构 |
| SHOW INDEX | 查看索引 |
| SHOW PROCESSLIST | 查看连接 |
| SHOW VARIABLES | 查看配置 |
| SHOW STATUS | 查看状态 |

## 30. 备份与恢复

```
# 备份整个数据库
mysqldump -u root -p mydb > mydb.sql

# 备份指定表
mysqldump -u root -p mydb user > user.sql

# 恢复
mysql -u root -p mydb < mydb.sql
```
