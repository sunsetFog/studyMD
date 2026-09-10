# MySQL 详细学习清单

> 从基础到架构，覆盖 SQL 开发、索引优化、事务锁机制、日志系统、高可用架构等全方位知识点，适合系统化学习和进度跟踪。

---

## 目录

1. [关系型数据库基础](#1-关系型数据库基础)
2. [SQL 基础语法](#2-sql-基础语法)
3. [SQL 进阶查询](#3-sql-进阶查询)
4. [数据类型与表设计](#4-数据类型与表设计)
5. [约束与完整性](#5-约束与完整性)
6. [索引原理与实战](#6-索引原理与实战)
7. [SQL 优化与调优](#7-sql-优化与调优)
8. [存储引擎深入](#8-存储引擎深入)
9. [事务与隔离级别](#9-事务与隔离级别)
10. [MVCC 多版本并发控制](#10-mvcc-多版本并发控制)
11. [锁机制深度](#11-锁机制深度)
12. [日志系统](#12-日志系统)
13. [主从复制与高可用](#13-主从复制与高可用)
14. [分库分表与分布式](#14-分库分表与分布式)
15. [备份恢复与运维](#15-备份恢复与运维)
16. [调优与监控](#16-调优与监控)
17. [实战场景演练](#17-实战场景演练)
18. [进阶路线图](#18-进阶路线图)

---

## 1. 关系型数据库基础

### 1.1 数据库核心概念
- [ ] 理解什么是数据库、数据库管理系统（DBMS）、数据库系统
- [ ] 掌握关系型数据库的核心特征：ACID
- [ ] 理解数据模型的三层结构：物理层、逻辑层、视图层
- [ ] 掌握 ER 模型：实体、属性、关系（1:1、1:N、N:M）
- [ ] 理解范式理论（1NF、2NF、3NF、BCNF）
- [ ] 了解反范式化的场景与权衡

### 1.2 范式详解
- [ ] **1NF**：每个字段原子性，不可再分
- [ ] **2NF**：非主键字段完全函数依赖主键（消除部分依赖）
- [ ] **3NF**：非主键字段不传递依赖主键（消除传递依赖）
- [ ] **BCNF**：所有函数依赖的决定因素都是候选键
- [ ] **练习**：给出表结构，判断属于第几范式，并修正至 3NF

### 1.3 MySQL 架构概览
- [ ] 了解 MySQL 架构分层：连接层 → 服务层 → 引擎层 → 存储层
- [ ] 理解查询执行路径：连接器 → 查询缓存（8.0 已移除）→ 分析器 → 优化器 → 执行器
- [ ] 了解 MySQL 各版本关键差异（5.6 / 5.7 / 8.0）
- [ ] 掌握 MySQL 8.0 重大特性：窗口函数、CTE、原子 DDL、降序索引、隐藏索引、函数索引

---

## 2. SQL 基础语法

### 2.1 DML（数据操作语言）
- [ ] **INSERT**：单行/多行插入、INSERT ... SELECT、INSERT IGNORE、REPLACE
- [ ] **UPDATE**：单表更新、多表关联更新
  ```sql
  UPDATE t1 JOIN t2 ON t1.id = t2.id SET t1.name = t2.name;
  ```
- [ ] **DELETE**：单表删除、多表关联删除、TRUNCATE vs DELETE 区别
  - TRUNCATE：DDL，不可回滚，重置自增主键，不触发触发器
  - DELETE：DML，可回滚，逐行删除，触发触发器

### 2.2 DQL（数据查询语言）
- [ ] **SELECT 基础**：DISTINCT、别名 AS、LIMIT、OFFSET
- [ ] **WHERE 条件**：比较运算符、BETWEEN、IN、LIKE（% _）、REGEXP、IS NULL
- [ ] **ORDER BY**：单列/多列排序、NULL 值排序行为
- [ ] **GROUP BY + HAVING**：分组聚合与过滤
- [ ] **聚合函数**：COUNT、SUM、AVG、MAX、MIN、GROUP_CONCAT

### 2.3 JOIN 全面掌握
- [ ] **INNER JOIN**：内连接，匹配两侧共有的行
- [ ] **LEFT JOIN**：左连接，保留左表所有行
- [ ] **RIGHT JOIN**：右连接，保留右表所有行
- [ ] **FULL OUTER JOIN**：全连接（MySQL 用 UNION 模拟）
- [ ] **CROSS JOIN**：笛卡尔积
- [ ] **SELF JOIN**：自连接，同表关联
- [ ] **JOIN 原理**：Nested-Loop Join / Block Nested-Loop Join / Hash Join (8.0)

**图示记忆**：
```
INNER JOIN：      LEFT JOIN：       RIGHT JOIN：
  ┌──┐ ┌──┐        ┌──┐ ┌──┐         ┌──┐ ┌──┐
  │AA│ │BB│        │AA│ │BB│         │AA│ │BB│
  │AB│ │BD│        │AB│ │BD│         │AB│ │BD│
  └──┘ └──┘        └──┘ └──┘         └──┘ └──┘
   A∩B 交集         A 全部含 B       B 全部含 A
```

### 2.4 UNION / INTERSECT / EXCEPT
- [ ] UNION vs UNION ALL（去重 vs 不去重）
- [ ] 各 SELECT 语句字段数、类型必须一致
- [ ] MySQL 8.0 支持 INTERSECT、EXCEPT

### 2.5 子查询
- [ ] **标量子查询**：返回单个值
- [ ] **行子查询**：返回一行
- [ ] **列子查询**：返回一列，配合 IN/ANY/ALL
- [ ] **表子查询**：FROM 子句中的派生表（必须给别名）
- [ ] **关联子查询 vs 非关联子查询**
- [ ] **EXISTS vs IN**：区别与适用场景

---

## 3. SQL 进阶查询

### 3.1 窗口函数（MySQL 8.0）
- [ ] 理解窗口函数语法：`func() OVER (PARTITION BY ... ORDER BY ...)`
- [ ] **排名函数**：
  - `ROW_NUMBER()`：连续排名，相同值不同排名
  - `RANK()`：跳跃排名，相同值同排名、跳过后续
  - `DENSE_RANK()`：连续排名，相同值同排名、不跳过
  - `NTILE(n)`：分桶排名
- [ ] **聚合窗口函数**：SUM/AVG/COUNT/MAX/MIN OVER
- [ ] **取值函数**：
  - `LAG(expr, n, default)`：向前取 n 行
  - `LEAD(expr, n, default)`：向后取 n 行
  - `FIRST_VALUE(expr)` / `LAST_VALUE(expr)` / `NTH_VALUE(expr, n)`
- [ ] **窗口命名**：WINDOW w AS (...)

### 3.2 CTE（公共表表达式）
- [ ] **普通 CTE**：`WITH cte AS (SELECT ...) SELECT ... FROM cte`
- [ ] **递归 CTE**：`WITH RECURSIVE cte AS (...)`
  - 应用场景：树形结构查询、生成连续日期序列、路径计算

### 3.3 高级语法
- [ ] **CASE WHEN**：简单 CASE、搜索 CASE
  ```sql
  SELECT CASE WHEN score >= 90 THEN 'A'
              WHEN score >= 80 THEN 'B'
              ELSE 'C' END AS grade
  ```
- [ ] **IF / IFNULL / NULLIF / COALESCE**
- [ ] **行转列**：CASE WHEN + GROUP BY / SUM(IF())
- [ ] **列转行**：UNION ALL

### 3.4 高级函数
- [ ] **字符串函数**：CONCAT、SUBSTRING、REPLACE、TRIM、LENGTH vs CHAR_LENGTH、INSTR、FIND_IN_SET
- [ ] **日期函数**：NOW、DATE_FORMAT、STR_TO_DATE、DATEDIFF、TIMESTAMPDIFF、DATE_ADD/DATE_SUB
  - `TIMESTAMPDIFF(DAY, start, end)` 计算时间差
  - `DATE_FORMAT(NOW(), '%Y-%m-%d')` 格式化
  - `UNIX_TIMESTAMP()` / `FROM_UNIXTIME()` 时间戳转换
- [ ] **数学函数**：ROUND、FLOOR、CEIL、ABS、RAND、MOD
- [ ] **JSON 函数**（MySQL 5.7+）：JSON_EXTRACT、JSON_SET、JSON_ARRAY、JSON_OBJECT、JSON_CONTAINS

---

## 4. 数据类型与表设计

### 4.1 整数类型
- [ ] TINYINT（1B）/ SMALLINT（2B）/ MEDIUMINT（3B）/ INT（4B）/ BIGINT（8B）
- [ ] 有符号 vs 无符号（UNSIGNED）：`INT UNSIGNED` 范围 0~42 亿
- [ ] ZEROFILL 补充前导零
- [ ] **选择原则**：在满足业务的前提下选最小的，INT 为主键默认选择

### 4.2 浮点数与定点数
- [ ] FLOAT（4B，单精度） vs DOUBLE（8B，双精度）→ 存在精度丢失
- [ ] **DECIMAL(M, D)**：定点数，精确存储
  - `DECIMAL(10,2)` 共 10 位，2 位小数
  - 金额场景必须用 DECIMAL
- [ ] 精度对比测试：`0.1 + 0.2` 在 FLOAT 和 DECIMAL 下的差异

### 4.3 字符串类型
- [ ] **CHAR(N)**：定长，0~255，适合固定长度（身份证号、MD5）
- [ ] **VARCHAR(N)**：变长，0~65535（需 1~2 字节存储长度），适合变长字段
- [ ] **TEXT 系列**：TINYTEXT / TEXT / MEDIUMTEXT / LONGTEXT
  - TEXT 独立存储，查询不走内存临时表
  - 建议：大文本单独建表，主表只存关联 ID
- [ ] **BLOB 系列**：存储二进制数据
- [ ] **ENUM vs VARCHAR**：ENUM 存储的是数字编号，性能好但扩展难
- [ ] **字符集与排序规则**：utf8 vs utf8mb4（emoji 支持）、`COLLATE utf8mb4_general_ci vs utf8mb4_bin`

### 4.4 日期时间类型
- [ ] **DATE**（3B）：YYYY-MM-DD
- [ ] **TIME**（3B）：HH:MM:SS
- [ ] **DATETIME**（8B）：YYYY-MM-DD HH:MM:SS，范围 1000~9999
- [ ] **TIMESTAMP**（4B）：1970~2038，自动时区转换，自动更新
- [ ] **选择建议**：跨时区用 TIMESTAMP，不需要时区用 DATETIME

### 4.5 表设计规范
- [ ] **命名规范**：
  - 表名：`t_` 前缀 + 下划线小写，如 `t_user_info`
  - 字段名：下划线小写，如 `created_at`
  - 索引名：`idx_` 普通索引、`uk_` 唯一索引、`pk_` 主键
- [ ] **主键设计**：
  - 推荐自增 BIGINT（有序插入、减少页分裂）
  - 避免 UUID 做主键（乱序插入导致页分裂、占用大）
  - 雪花算法 Snowflake ID：有序 + 分布式唯一
- [ ] **NOT NULL + 默认值**：避免 NULL 带来的索引和比较问题
- [ ] **字段类型最小化**：状态用 TINYINT 不用 INT、IP 用 INT UNSIGNED 不用 VARCHAR
- [ ] **预留扩展字段**：JSON 类型存储扩展属性
- [ ] **软删除**：`is_deleted TINYINT DEFAULT 0`
- [ ] **通用字段**：`id`, `create_time`, `update_time`, `create_by`, `update_by`

### 4.6 ER 图绘制
- [ ] 工具：MySQL Workbench / Navicat / draw.io
- [ ] 练习：电商系统 ER 设计（用户→订单→商品→库存→支付）
- [ ] 练习：社交系统 ER 设计（用户→好友→动态→评论→点赞）

---

## 5. 约束与完整性

### 5.1 约束类型
- [ ] **主键约束（PRIMARY KEY）**：非空 + 唯一，一张表只能有一个
- [ ] **唯一约束（UNIQUE）**：可为 NULL（多个 NULL 可并存）
- [ ] **非空约束（NOT NULL）**：必填字段
- [ ] **默认值（DEFAULT）**：`DEFAULT CURRENT_TIMESTAMP`
- [ ] **检查约束（CHECK）**：MySQL 8.0.16+ 支持，`CHECK(age >= 0 AND age <= 150)`
- [ ] **外键约束（FOREIGN KEY）**：
  - `ON DELETE CASCADE / SET NULL / RESTRICT / NO ACTION`
  - `ON UPDATE CASCADE / SET NULL / RESTRICT / NO ACTION`
  - **生产环境建议**：不使用物理外键，用逻辑外键 + 应用层保证

### 5.2 实体完整性
- [ ] 主键唯一标识每一行
- [ ] 主键列不能为 NULL
- [ ] 联合主键的应用场景

### 5.3 参照完整性
- [ ] 外键引用的必须是另一表的主键或唯一键
- [ ] 插入子表时父表必须有对应记录
- [ ] 删除父表时检查子表引用

### 5.4 用户自定义完整性
- [ ] CHECK 约束
- [ ] 应用层校验（字段值域、格式）
- [ ] 触发器校验

---

## 6. 索引原理与实战

### 6.1 索引数据结构
- [ ] **为什么不用 Hash 索引**：不支持范围查询、不支持排序
  - 例外：Memory 引擎支持 Hash 索引，等值查询极快
- [ ] **为什么不用二叉树/红黑树**：高度过高 → 磁盘 IO 次数多
- [ ] **为什么用 B 树**：多叉 → 矮胖 → 减少 IO
- [ ] **为什么用 B+ 树**：
  - 非叶子节点只存 key，叶子节点存完整数据 → 更矮
  - 叶子节点双向链表串联 → 范围查询高效
  - 所有数据在叶子节点 → 查询稳定

### 6.2 B+ 树可视化理解
```
                [10|20|30]          ← 非叶子节点，只存 key
               /    |    \    \
      [1|3|7]  [12|15|18] [22|26|28] [32|35|39]   ← 叶子节点，存完整数据
         ↕          ↕          ↕          ↕
      [双向链表 ─────────────────────────────→]      ← 支持范围扫描
```

### 6.3 InnoDB 索引类型
- [ ] **聚簇索引（Clustered Index）**：
  - 叶子节点存储**完整行数据**
  - 每张表只能有一个（主键索引）
  - 主键查询直接定位 → 最快
- [ ] **二级索引（Secondary Index / 非聚簇索引）**：
  - 叶子节点存储的是**主键值**
  - 查询流程：二级索引 → 获取主键 → 回表 → 聚簇索引查完整数据
- [ ] **覆盖索引（Covering Index）**：
  - 查询的列全在索引中 → 不需要回表 → EXPLAIN Extra 显示 `Using index`
  - 最常用的优化手段之一

### 6.4 索引类型全解
- [ ] **主键索引**：PRIMARY KEY，唯一 + 非空
- [ ] **唯一索引**：UNIQUE INDEX，允许 NULL
- [ ] **普通索引**：INDEX / KEY，加速查询
- [ ] **联合索引**：多列组合，遵循最左前缀原则
- [ ] **前缀索引**：`INDEX idx_email(email(10))`，对长字符串取前缀
- [ ] **全文索引**：FULLTEXT，配合 MATCH ... AGAINST，中文需 ngram parser
- [ ] **空间索引**：SPATIAL INDEX，用于地理位置（GIS）

### 6.5 联合索引与最左前缀原则
- [ ] 理解最左前缀：`INDEX(a, b, c)` 等价于建了 a / (a,b) / (a,b,c) 三个索引
- [ ] **能走索引**：`WHERE a=1` / `WHERE a=1 AND b=2` / `WHERE a=1 AND b=2 AND c=3`
- [ ] **不能走索引**：`WHERE b=2` / `WHERE c=3` / `WHERE b=2 AND c=3`
- [ ] **中间断档场景**：`WHERE a=1 AND c=3` → 只用 a，c 不走索引
- [ ] **范围查询之后的列不走索引**：`WHERE a=1 AND b>2 AND c=3` → a 和 b 走索引，c 不走
- [ ] **ORDER BY 也遵循最左前缀**：`ORDER BY a, b ASC, c DESC` 需注意排序方向

### 6.6 索引失效场景（10 大经典场景）
- [ ] **1. 违反最左前缀**：跳过了联合索引的第一列
- [ ] **2. 在索引列上做运算**：`WHERE id + 1 = 100` → 应用层提前计算
- [ ] **3. 对索引列使用函数**：`WHERE DATE(create_time) = '2024-01-01'` → 改为范围查询
- [ ] **4. 隐式类型转换**：`WHERE phone = 13800138000`，phone 是 VARCHAR → 字符串→数字转换
- [ ] **5. LIKE 以 % 开头**：`LIKE '%abc'`，但 `LIKE 'abc%'` 可以走索引
- [ ] **6. OR 条件不全是索引列**：`WHERE a=1 OR b=2`，需 a 和 b 都有索引
- [ ] **7. 使用 != 或 <> 或 NOT IN**：可能导致全表扫描（数据量大时）
- [ ] **8. IS NULL / IS NOT NULL**：取决于数据分布，NULL 多可能不走索引
- [ ] **9. 联合索引范围查询右边列失效**
- [ ] **10. 优化器认为全表扫描更快**：数据量小或回表代价大

### 6.7 索引设计最佳实践
- [ ] **高选择性的列建索引**：`COUNT(DISTINCT col) / COUNT(*)` 接近 1
- [ ] **选择性低的列**（如性别）不适合单独建索引
- [ ] **WHERE / JOIN / ORDER BY / GROUP BY 涉及的列建索引**
- [ ] **避免冗余索引**：`INDEX(a) 和 INDEX(a,b)` 中前者是冗余的
- [ ] **联合索引列的顺序**：等值条件列在前，范围条件列在后
- [ ] **不宜过多索引**：每多一个索引，INSERT/UPDATE/DELETE 就多维护一个 B+ 树
- [ ] **长字符串用前缀索引**：取区分度够的前 N 个字符

### 6.8 Extra 核心指标
- [ ] `Using index`：覆盖索引，最优
- [ ] `Using index condition`：索引下推（ICP），5.6+ 特性
- [ ] `Using where`：用到了 WHERE 过滤
- [ ] `Using temporary`：需要临时表 → 一般需要优化
- [ ] `Using filesort`：需要额外排序 → 考虑优化索引
- [ ] `Using join buffer`：JOIN 用了连接缓冲

---

## 7. SQL 优化与调优

### 7.1 EXPLAIN 深度掌握
- [ ] **id**：查询序号，id 相同从上往下，id 越大越先执行
- [ ] **select_type**：SIMPLE / PRIMARY / SUBQUERY / DERIVED / UNION / UNION RESULT
- [ ] **table**：访问的表名或别名
- [ ] **partitions**：匹配的分区
- [ ] **type（访问类型，从优到差）**：
  - `system` > `const` > `eq_ref` > `ref` > `range` > `index` > `ALL`
  - `system`：表只有一行
  - `const`：主键或唯一索引等值查询，最多返回一行
  - `eq_ref`：关联查询中，用主键或唯一索引关联
  - `ref`：非唯一索引等值查询
  - `range`：索引范围扫描（BETWEEN、>、<、IN）
  - `index`：全索引扫描（比全表扫描好）
  - `ALL`：全表扫描，最差
- [ ] **possible_keys**：可能使用的索引
- [ ] **key**：实际使用的索引
- [ ] **key_len**：使用的索引长度（可判断联合索引用了几列）
- [ ] **ref**：与索引比较的列或常量
- [ ] **rows**：优化器估计要扫描的行数
- [ ] **filtered**：按条件过滤后剩余行的百分比
- [ ] **Extra**：额外信息（见 6.8）

### 7.2 大分页优化
- [ ] 问题：`LIMIT 1000000, 20` 扫描 100 万 + 20 行再丢弃前 100 万行
- [ ] **方案一：延迟关联**
  ```sql
  SELECT t1.* FROM t t1
  JOIN (SELECT id FROM t WHERE ... LIMIT 1000000, 20) t2
  ON t1.id = t2.id;
  ```
- [ ] **方案二：基于 ID 的范围查询（游标分页）**
  ```sql
  SELECT * FROM t WHERE id > 1000000 ORDER BY id LIMIT 20;
  ```
  - 缺点：需要 ID 连续，不能跳页
- [ ] **方案三：覆盖索引 + 子查询**
  ```sql
  SELECT * FROM t WHERE id >= (SELECT id FROM t ORDER BY id LIMIT 1000000, 1) LIMIT 20;
  ```

### 7.3 IN vs EXISTS 选择
- [ ] **IN**：子查询先执行，结果集小驱动大表
- [ ] **EXISTS**：外层驱动内层，大表驱动小表
- [ ] 8.0 优化器已优化，差异减小但仍需关注

### 7.4 COUNT 优化
- [ ] `COUNT(*)`：统计所有行数（包括 NULL）
- [ ] `COUNT(1)`：效果同 COUNT(*)
- [ ] `COUNT(col)`：统计非 NULL 的行数
- [ ] **性能**：`COUNT(*)` = `COUNT(1)` > `COUNT(col)`
- [ ] **大表 COUNT 优化**：使用冗余计数表或 EXPLAIN 估算

### 7.5 避免索引失效的综合清单
- [ ] 不在索引列上做运算、函数、隐式类型转换
- [ ] LIKE 'abc%' 可以，LIKE '%abc' 不行
- [ ] 联合索引注意最左前缀，范围查询放最后
- [ ] OR 条件各列都要有索引
- [ ] 尽量用覆盖索引避免回表
- [ ] 大范围查询考虑走全表可能更快

### 7.6 慢查询分析
- [ ] **开启慢查询日志**：
  ```sql
  SET GLOBAL slow_query_log = 'ON';
  SET GLOBAL long_query_time = 2;  -- 超过 2s 记录
  SET GLOBAL log_queries_not_using_indexes = 'ON';
  ```
- [ ] **慢查询日志分析工具**：mysqldumpslow / pt-query-digest
- [ ] **重点关注**：Query_time、Lock_time、Rows_sent、Rows_examined

### 7.7 批量操作优化
- [ ] 批量 INSERT：一条 SQL 插入多行 `INSERT INTO t VALUES (...), (...), (...)`
- [ ] 批量 UPDATE：分批更新，每批 1000 条
- [ ] LOAD DATA INFILE：大数据量导入的最快方式
- [ ] 大批量操作时临时关闭 autocommit、unique_checks、foreign_key_checks

---

## 8. 存储引擎深入

### 8.1 InnoDB vs MyISAM 对比
| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | ✅ 完整 ACID | ❌ 不支持 |
| 行级锁 | ✅ | ❌（表级锁） |
| 外键 | ✅ | ❌ |
| MVCC | ✅ | ❌ |
| 崩溃恢复 | ✅（redo log） | ❌（易损坏） |
| 全文索引 | 5.6+ 支持 | ✅ 原生支持 |
| 数据存储 | 表空间 | 独立文件 |
| 适用场景 | 高并发 OLTP | 只读/日志分析 |

### 8.2 InnoDB 存储结构
- [ ] **表空间（Tablespace）** → **段（Segment）** → **区（Extent, 1MB）** → **页（Page, 16KB）** → **行（Row）**
- [ ] **数据页结构**：File Header → Page Header → Infimum + Supremum → User Records → Free Space → Page Directory → File Trailer
- [ ] **页目录**：槽 + 二分查找，快速定位记录所在组
- [ ] **行格式**：COMPACT（5.0）、DYNAMIC（5.7+ 默认）、REDUNDANT、COMPRESSED

### 8.3 行格式深入
- [ ] **COMPACT**：变长字段长度列表 + NULL 值列表 + 记录头 + 列数据
- [ ] **DYNAMIC**：溢出页机制，行存引用 + 前 768 字节前缀
- [ ] **行溢出**：大字段（TEXT/BLOB）超过页大小一半时，存到外部溢出页

### 8.4 其他存储引擎
- [ ] **Memory**：数据存内存，重启丢失，Hash 索引，临时表
- [ ] **Archive**：高压缩比，只支持 INSERT/SELECT，日志归档
- [ ] **CSV**：CSV 文件存储，可直接编辑
- [ ] **NDB**：MySQL Cluster 分布式引擎

---

## 9. 事务与隔离级别

### 9.1 ACID 四大特性
- [ ] **原子性（Atomicity）**：要么全做，要么全不做 → undo log
- [ ] **一致性（Consistency）**：事务前后数据满足所有约束
- [ ] **隔离性（Isolation）**：并发事务互不干扰 → MVCC + 锁
- [ ] **持久性（Durability）**：提交后永久保存 → redo log

### 9.2 并发问题
- [ ] **脏读（Dirty Read）**：读到未提交事务的数据 → ROLLBACK 后读到"不存在"的数据
- [ ] **不可重复读（Non-Repeatable Read）**：同一事务两次读到不同值（UPDATE 导致）
- [ ] **幻读（Phantom Read）**：同一条件两次查询记录数不同（INSERT/DELETE 导致）

### 9.3 四种隔离级别
| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|----------|------|-----------|------|----------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ | 读不加锁 |
| READ COMMITTED | ❌ | ✅ | ✅ | 每次读生成 ReadView |
| REPEATABLE READ（默认） | ❌ | ❌ | ❌⚠️ | 事务开始生成 ReadView + Next-Key Lock |
| SERIALIZABLE | ❌ | ❌ | ❌ | 读加共享锁 |

- [ ] MySQL RR 级别下幻读基本被解决（Next-Key Lock）
- [ ] 特殊情况仍可能出现幻读：快照读（无锁） vs 当前读（加锁）的差异

### 9.4 事务控制语句
- [ ] `BEGIN` / `START TRANSACTION`：开始事务
- [ ] `COMMIT`：提交事务
- [ ] `ROLLBACK`：回滚事务
- [ ] `SAVEPOINT name` / `ROLLBACK TO name`：部分回滚
- [ ] `SET autocommit = 0/1`：关闭/开启自动提交
- [ ] 理解隐式提交的场景（DDL、LOCK TABLES、BEGIN 等）

### 9.5 长事务危害与优化
- [ ] 锁持有时间长 → 阻塞其他事务
- [ ] undo log 不能清理 → 占用大量存储空间
- [ ] 主从延迟加大
- [ ] **监控长事务**：
  ```sql
  SELECT * FROM information_schema.innodb_trx
  WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60;
  ```
- [ ] **`information_schema.innodb_trx`**：查看活跃事务列表

---

## 10. MVCC 多版本并发控制

### 10.1 MVCC 核心概念
- [ ] MVCC = 无锁实现读-写不冲突，提升并发性能
- [ ] **读操作**：通过 undo log 构建历史版本，不加锁（快照读）
- [ ] **写操作**：加锁 + 生成新版本

### 10.2 隐藏列
每行数据在 InnoDB 中有三个隐藏列：

| 隐藏列 | 大小 | 作用 |
|--------|------|------|
| DB_ROW_ID | 6B | 行 ID，无主键时自动生成 |
| DB_TRX_ID | 6B | 最近修改该行的事务 ID |
| DB_ROLL_PTR | 7B | 回滚指针，指向 undo log 中的旧版本 |

### 10.3 undo log 版本链
```
当前行 → {trx_id=100, roll_ptr} → undo_log1 → {trx_id=80, roll_ptr} → undo_log2 → ...
```

### 10.4 ReadView 可见性判断
- [ ] ReadView 包含：`m_ids`（活跃事务 ID 列表）、`min_trx_id`、`max_trx_id`、`creator_trx_id`
- [ ] **RC 级别**：每次 SELECT 生成新的 ReadView
- [ ] **RR 级别**：事务开始时生成一次 ReadView，整个事务复用
- [ ] **可见性规则**：
  - `trx_id < min_trx_id`：已提交 → 可见
  - `trx_id >= max_trx_id`：未来事务 → 不可见
  - `trx_id` 在 `m_ids` 中：活跃事务 → 不可见
  - `trx_id` 是当前事务 → 可见

### 10.5 快照读 vs 当前读
- [ ] **快照读（不加锁）**：普通 SELECT → 读历史版本（MVCC）
- [ ] **当前读（加锁）**：
  - `SELECT ... FOR UPDATE`（排他锁）
  - `SELECT ... LOCK IN SHARE MODE`（共享锁，8.0 是 `FOR SHARE`）
  - `INSERT / UPDATE / DELETE` → 读最新已提交版本
- [ ] RR 级别下，快照读无幻读，当前读通过 Next-Key Lock 防止幻读

### 10.6 MVCC 与 Purge 线程
- [ ] 事务提交后，undo log 不是立即删除
- [ ] Purge 线程：异步清理不再需要的 undo log
- [ ] 长事务阻碍 Purge → undo log 膨胀

---

## 11. 锁机制深度

### 11.1 锁的分类体系
```
锁
├── 按粒度
│   ├── 表级锁（Table Lock）
│   │   ├── 表锁（Table Lock）
│   │   ├── 元数据锁（MDL）
│   │   ├── 意向锁（Intention Lock）
│   │   │   ├── 意向共享锁（IS）
│   │   │   └── 意向排他锁（IX）
│   │   └── AUTO-INC 锁
│   └── 行级锁（Row Lock）
│       ├── 记录锁（Record Lock）
│       ├── 间隙锁（Gap Lock）
│       ├── 临键锁（Next-Key Lock）= Record + Gap
│       └── 插入意向锁（Insert Intention Lock）
├── 按模式
│   ├── 共享锁（S Lock）：读锁
│   └── 排他锁（X Lock）：写锁
└── 按实现
    └── 乐观锁 / 悲观锁
```

### 11.2 行锁类型详解
- [ ] **记录锁（Record Lock）**：锁定索引记录本身
- [ ] **间隙锁（Gap Lock）**：锁定索引记录之间的间隙（开区间）
  - **只在 RR 级别生效**，防止幻读
- [ ] **临键锁（Next-Key Lock）**：Record + Gap，左开右闭
  - RR 级别默认行为
- [ ] **插入意向锁（Insert Intention Lock）**：INSERT 时等待间隙锁释放

### 11.3 锁兼容矩阵
|  | IS | IX | S | X |
|--|-----|-----|----|----|
| IS | ✅ | ✅ | ✅ | ❌ |
| IX | ✅ | ✅ | ❌ | ❌ |
| S  | ✅ | ❌ | ✅ | ❌ |
| X  | ❌ | ❌ | ❌ | ❌ |

### 11.4 加锁规则（RR 级别）
- [ ] 原则 1：基本单位是 Next-Key Lock（左开右闭）
- [ ] 原则 2：查找过程中访问到的对象才加锁
- [ ] 优化 1：等值查询唯一索引 → 退化为 Record Lock
- [ ] 优化 2：等值查询索引向右遍历最后一个不满足条件时 → Next-Key 退化为 Gap Lock
- [ ] Bug：范围查询中唯一索引会向右多锁一个间隙

### 11.5 死锁
- [ ] **条件**：互斥 + 持有并等待 + 不可剥夺 + 循环等待
- [ ] **排查**：`SHOW ENGINE INNODB STATUS` 查看 LATEST DETECTED DEADLOCK
- [ ] **避免**：
  - 按相同顺序访问资源
  - 缩小事务（拆成小事务）
  - 使用低隔离级别
  - 合理的索引 → 减少锁范围
- [ ] 死锁时 MySQL 自动回滚代价小的事务

### 11.6 乐观锁 vs 悲观锁
- [ ] **悲观锁**：`SELECT ... FOR UPDATE`，假设冲突频繁
- [ ] **乐观锁**：版本号 `WHERE version = ?` 或时间戳，假设冲突较少
- [ ] **CAS 实现乐观锁**：`UPDATE t SET count = count - 1, version = version + 1 WHERE id = ? AND version = ?`
- [ ] **适用场景**：读多写少用乐观锁，冲突频繁用悲观锁

### 11.7 元数据锁（MDL）
- [ ] DDL 需要 MDL 写锁，DML 需要 MDL 读锁
- [ ] **问题**：长事务持有 MDL 读锁，阻塞 DDL → 后续 DML 也被阻塞
- [ ] **解决**：`pt-online-schema-change` / 设置 `lock_wait_timeout`

---

## 12. 日志系统

### 12.1 redo log（重做日志）
- [ ] **作用**：保证持久性，崩溃恢复
- [ ] **固定大小**：日志文件组循环写，写满时刷盘
- [ ] **`innodb_flush_log_at_trx_commit`**：
  - `0`：每秒刷盘（性能最好，丢失 1s 数据）
  - `1`：每次提交刷盘（最安全，默认）
  - `2`：每次提交写 OS 缓存，每秒刷盘
- [ ] **WAL（Write-Ahead Log）**：先写日志再写磁盘
- [ ] **redo log 与 binlog 的区别**：

| 对比 | redo log | binlog |
|------|----------|--------|
| 引擎 | InnoDB 独有 | MySQL Server 层 |
| 内容 | 物理日志（页修改） | 逻辑日志（SQL 语句） |
| 写入方式 | 循环写，固定空间 | 追加写，持续增长 |
| 用途 | 崩溃恢复 | 主从复制、数据恢复 |

### 12.2 binlog（归档日志）
- [ ] **三种格式**：
  - `STATEMENT`：记录 SQL 语句（可能主从不一致，如 NOW()）
  - `ROW`：记录行数据修改（推荐，不会出现不一致）
  - `MIXED`：两者混合，默认 STATEMENT，必要时切换 ROW
- [ ] **`sync_binlog`**：
  - `0`：不主动刷盘
  - `1`：每次提交刷盘（最安全）
  - `N`：每 N 次提交刷盘

### 12.3 两阶段提交
```
事务提交流程：
  ┌──────────────────────────────────────────────┐
  │  1. 写 redo log（prepare 状态）              │
  │  2. 写 binlog                                 │
  │  3. 写 redo log（commit 状态）                │
  └──────────────────────────────────────────────┘
```
- [ ] 为什么需要两阶段提交：保证 redo log 和 binlog 逻辑一致
- [ ] 崩溃恢复判断：redo log commit 状态 → 直接提交；redo log prepare + binlog 完整 → 提交；否则回滚

### 12.4 undo log（回滚日志）
- [ ] **作用**：事务回滚 + MVCC 构建历史版本
- [ ] **两种格式**：
  - INSERT undo log：仅用于回滚，提交后直接删除
  - UPDATE undo log：回滚 + MVCC 读版本，Purge 线程清理
- [ ] **存储位置**：独立表空间或共享表空间中的回滚段

### 12.5 其他日志
- [ ] **error log**：错误日志，记录启动/运行/关闭错误
- [ ] **general query log**：通用查询日志（很少开启，影响性能）
- [ ] **slow query log**：慢查询日志
- [ ] **relay log**：从库中继日志，接收主库 binlog 的中转

---

## 13. 主从复制与高可用

### 13.1 复制原理（三步走）
```
Master                            Slave
┌──────────┐                    ┌──────────────┐
│ binlog   │ ── dump thread ─→ │ IO Thread    │
│ (写入)   │                    │  └→ relay log │
└──────────┘                    │ SQL Thread    │
                                │  └→ 回放数据  │
                                └──────────────┘
```
1. Master 写 binlog
2. Slave IO Thread 从 Master 拉取 binlog → 写入 relay log
3. Slave SQL Thread 从 relay log 读取并回放

### 13.2 复制模式
- [ ] **异步复制**：Master 不等待 Slave，可能丢数据（默认）
- [ ] **半同步复制**：Master 等待至少一个 Slave 收到 binlog
  - `rpl_semi_sync_master_wait_point = AFTER_SYNC`（5.7+ 推荐）
- [ ] **全同步复制**：所有 Slave 确认后才提交（MySQL Group Replication）
- [ ] **GTID 复制**（5.6+）：全局事务 ID，切换主库更方便
  ```sql
  CHANGE MASTER TO MASTER_AUTO_POSITION = 1;
  ```

### 13.3 主从延迟
- [ ] **原因**：Slave 是单线程 SQL Thread 回放（5.7 前），Master 高并发
- [ ] **解决方案**：
  - MySQL 5.7：并行复制 `slave_parallel_workers > 0`（按库并行）
  - MySQL 8.0：基于 WRITESET 的并行复制（`binlog_transaction_dependency_tracking = WRITESET`）
  - 应用层：写后读走主库，或延迟一定时间
- [ ] **查看延迟**：`SHOW SLAVE STATUS` → `Seconds_Behind_Master`

### 13.4 一主多从 vs 级联复制
- [ ] **一主多从**：单一 Master，多 Slave 直连
  - 优点：简单，延迟低
  - 缺点：Master 压力大（binlog dump 线程多）
- [ ] **级联复制**：Master → Slave1 → Slave2
  - 优点：减少 Master 压力
  - 缺点：延迟叠加

### 13.5 高可用方案
- [ ] **MHA**（Master High Availability）：自动选主 + 切换
- [ ] **MGR**（MySQL Group Replication，5.7+）：多主高可用
- [ ] **InnoDB Cluster**（8.0）：MGR + MySQL Router + MySQL Shell
- [ ] **Orchestrator**：可视化拓扑管理 + 自动故障转移
- [ ] **ProxySQL / MySQL Router**：读写分离 + 负载均衡

### 13.6 读写分离
- [ ] 优点：分担主库压力，提升读并发
- [ ] 问题：主从延迟导致"写后立刻读"读不到新数据
- [ ] 解决：强制读主库（事务中的写后读）/ 等主从延迟阈值

---

## 14. 分库分表与分布式

### 14.1 为什么要分库分表
- [ ] **单表数据量过大**（如 5000 万 +）→ B+ 树高度增加、查询变慢
- [ ] **单库连接数瓶颈** → 连接数有限
- [ ] **单库磁盘/IO 瓶颈** → 无法扩容

### 14.2 拆分策略
- [ ] **垂直拆分**：
  - 垂直分库：按业务模块（用户库、订单库、商品库）
  - 垂直分表：大字段（TEXT/BLOB）拆到扩展表
- [ ] **水平拆分**：
  - 水平分库：同表数据按分片键分散到多个库
  - 水平分表：同库内按分片键拆到多个表（如 user_0 ~ user_15）

### 14.3 分片算法
- [ ] **范围分片**：`user_id 1~1000 在 shard0，1001~2000 在 shard1`
  - 简单，但热点数据问题
- [ ] **Hash 取模**：`user_id % N`
  - 数据均匀，但扩容困难（rehash 代价大）
- [ ] **一致性 Hash**：环形 Hash 空间，扩容只需迁移部分数据
- [ ] **range + hash 结合**：按时间范围分库 + 按用户 hash 分表

### 14.4 分库分表中间件
- [ ] **ShardingSphere（Sharding-JDBC / Sharding-Proxy）**：无侵入、功能全
- [ ] **MyCat**：基于 Proxy 架构，功能丰富但社区活跃度下降
- [ ] **DBLE**：基于 MyCat 的企业版改进
- [ ] **Vitess**：YouTube 开源，规模化

### 14.5 拆分后的问题与解决
- [ ] **分布式 ID**：
  - 雪花算法（Snowflake）⭐ 推荐
  - 号段模式（美团 Leaf）
  - Redis 自增
- [ ] **跨分片 JOIN**：禁止跨库 JOIN，应用层拼装 / 冗余字段
- [ ] **跨分片排序/分页**：各分片排序后，应用层归并
- [ ] **分布式事务**：Seata AT 模式 / TCC / 本地消息表
- [ ] **全局唯一性约束**：不能依赖唯一索引，需应用层保证

### 14.6 分库分表时机
- [ ] **先优化 SQL 和索引**，再考虑缓存，最后才分库分表
- [ ] 预估 3~5 年数据量，超过单表瓶颈（5000 万 ~ 1 亿）再拆分
- [ ] 设计时预留分片字段，但不一定一开始就启用

---

## 15. 备份恢复与运维

### 15.1 备份类型
- [ ] **逻辑备份**（mysqldump / mysqlpump / mydumper）：
  - SQL 语句形式，可跨版本
  - 速度慢，适合小数据量
- [ ] **物理备份**（xtrabackup / mysqlbackup）：
  - 直接拷贝数据文件，速度快
  - xtrabackup 支持热备份（不锁表）
- [ ] **全量备份 vs 增量备份**
- [ ] **冷备份 vs 热备份**

### 15.2 mysqldump 实战
```bash
# 全库备份
mysqldump -u root -p --all-databases > all.sql

# 单库备份
mysqldump -u root -p --databases db1 > db1.sql

# 单表备份（含结构和数据）
mysqldump -u root -p db1 t_user > t_user.sql

# 只导出表结构
mysqldump -u root -p --no-data db1 > schema.sql

# 只导出数据
mysqldump -u root -p --no-create-info db1 > data.sql

# 一致性快照（InnoDB 单事务导出）
mysqldump --single-transaction --master-data=2
```

### 15.3 xtrabackup 实战
- [ ] **全量备份**：
  ```bash
  xtrabackup --backup --target-dir=/backup/full --user=root --password=xxx
  ```
- [ ] **增量备份**：
  ```bash
  xtrabackup --backup --target-dir=/backup/inc1 \
    --incremental-basedir=/backup/full
  ```
- [ ] **恢复**：
  ```bash
  xtrabackup --prepare --apply-log-only --target-dir=/backup/full
  xtrabackup --prepare --target-dir=/backup/full --incremental-dir=/backup/inc1
  ```

### 15.4 数据恢复
- [ ] **全量恢复**：mysql < dump.sql / xtrabackup 恢复
- [ ] **基于时间点恢复（PITR）**：全量备份 + binlog 增量恢复
  ```bash
  mysqlbinlog --start-datetime="2024-01-01 00:00:00" \
              --stop-datetime="2024-01-01 12:00:00" \
              mysql-bin.000001 | mysql
  ```
- [ ] **基于 GTID 恢复**
- [ ] **闪回工具**：binlog2sql / MyFlash（解析 binlog 生成反向 SQL）

### 15.5 日常运维
- [ ] **版本升级**：小版本 in-place 升级，大版本先 dump 再恢复 / mysql_upgrade
- [ ] **表维护**：
  - `ANALYZE TABLE`：更新索引统计信息
  - `OPTIMIZE TABLE`：重组表空间，回收空间（会锁表，谨慎）
  - `CHECK TABLE`：检查表是否损坏
  - `REPAIR TABLE`：修复表
- [ ] **在线 DDL**：
  - MySQL 5.6+ 支持部分在线 DDL（不阻塞读写）
  - 大表变更推荐 `pt-online-schema-change`：
    ```bash
    pt-online-schema-change --alter "ADD COLUMN age INT" D=db1,t=t_user
    ```
  - 原理：创建新表 → 触发器同步 → 分批复制 → 原子切换

### 15.6 权限管理
- [ ] 最小权限原则，不用 root
- [ ] 创建用户：`CREATE USER 'app'@'%' IDENTIFIED BY 'xxx';`
- [ ] 授权：`GRANT SELECT, INSERT, UPDATE ON db1.* TO 'app'@'%';`
- [ ] 回收：`REVOKE DELETE ON db1.* FROM 'app'@'%';`
- [ ] 查看权限：`SHOW GRANTS FOR 'app'@'%';`
- [ ] `mysql.user` / `mysql.db` / `mysql.tables_priv` 表结构

---

## 16. 调优与监控

### 16.1 配置文件调优（my.cnf）
- [ ] **连接相关**：
  - `max_connections`：最大连接数
  - `thread_cache_size`：线程缓存
  - `back_log`：连接等待队列
- [ ] **InnoDB 相关**：
  - `innodb_buffer_pool_size`：Buffer Pool 大小（物理内存 50%~70%）
  - `innodb_log_file_size`：redo log 文件大小
  - `innodb_flush_log_at_trx_commit`：日志刷盘策略
  - `innodb_io_capacity`：后台 IO 能力
  - `innodb_lock_wait_timeout`：锁等待超时
- [ ] **查询缓存**：8.0 已移除，不再使用
- [ ] **临时表**：`tmp_table_size` / `max_heap_table_size`

### 16.2 Buffer Pool 深入
- [ ] **作用**：缓存数据页和索引页，减少磁盘 IO
- [ ] **LRU 链表冷热分离**：
  - 链表按 5/8 比例分为 Young（热）和 Old（冷）区
  - 新页先进入 Old 区头部（避免污染热区）
  - Old 区停留超过 `innodb_old_blocks_time`（1s）才移到 Young 区
- [ ] **预读（Read Ahead）**：
  - 线性预读：连续 56 页访问 → 预读下个区
  - 随机预读：同区内 13 页 → 预读整个区
- [ ] **Flush 链表**：管理脏页，按照第一次修改时间排序
- [ ] **Free 链表**：管理空闲页
- [ ] **Chunk 机制**：5.7+ 可动态调整 Buffer Pool 大小

### 16.3 Change Buffer
- [ ] **作用**：缓存非唯一二级索引的变更（INSERT/UPDATE/DELETE）
- [ ] 写 Change Buffer → 读时合并到 Buffer Pool
- [ ] **适用场景**：写多读少的二级索引
- [ ] **`innodb_change_buffering`**：all / none / inserts / deletes / changes / purges
- [ ] 唯一索引不能使用 Change Buffer（需要读判断唯一性）

### 16.4 双写缓冲（Doublewrite Buffer）
- [ ] **作用**：防止页部分写入（partial page write）导致的崩溃恢复失败
- [ ] 先顺序写双写缓冲区 → 再随机写数据页
- [ ] **性能影响**：写入量翻倍，但 SSDs 下影响减小
- [ ] `innodb_doublewrite`：可关闭（Fusion-io 设备 / 文件系统原子写）

### 16.5 自适应 Hash 索引（AHI）
- [ ] **作用**：对频繁访问的二级索引页自动建立 Hash 索引
- [ ] 等值查询提升明显，范围查询无用
- [ ] `innodb_adaptive_hash_index`：默认开启

### 16.6 监控指标体系
- [ ] **连接**：Threads_connected、Threads_running、Aborted_connects
- [ ] **TPS/QPS**：`Com_commit + Com_rollback` / `Questions`
- [ ] **InnoDB 行操作**：Innodb_rows_read/inserted/updated/deleted
- [ ] **InnoDB 缓冲池命中率**：
  ```sql
  SELECT (1 - SUM(Innodb_buffer_pool_reads) / SUM(Innodb_buffer_pool_read_requests)) * 100
  FROM information_schema.global_status;
  ```
  - 目标：> 99%
- [ ] **锁等待**：Innodb_row_lock_current_waits
- [ ] **慢查询数量**：Slow_queries
- [ ] **临时表**：Created_tmp_disk_tables / Created_tmp_tables

### 16.7 监控工具
- [ ] **Prometheus + Grafana**：mysqld_exporter 采集指标，Grafana 可视化
- [ ] **PMM（Percona Monitoring and Management）**：全套监控方案
- [ ] **pt-stalk + pt-summary**：触发条件时自动采集诊断数据
- [ ] **MySQL Performance Schema**：
  - `performance_schema.events_statements_summary_by_digest`：SQL 摘要统计
  - `sys.schema_*` 系列视图：便捷查询（sys schema 依赖 performance_schema）

### 16.8 硬件选型
- [ ] **CPU**：MySQL 对单核性能敏感，优先高频
- [ ] **内存**：越大越好，Buffer Pool 占用主要内存
- [ ] **磁盘**：SSD（NVMe）> SSD（SATA）> HDD
- [ ] **网络**：主从复制场景保证带宽和低延迟

---

## 17. 实战场景演练

### 17.1 场景一：从 0 到 1 设计电商数据库
- [ ] 分析业务模块：用户、商品、订单、库存、支付、物流
- [ ] 画出 ER 图
- [ ] 确定核心表结构（字段、类型、索引、约束）
- [ ] 预估数据量：1 年后用户 100 万、订单 1000 万
- [ ] 预留分库分表规划（按 user_id 拆分）
- [ ] 设计冷热数据分离（3 月前订单归档到历史表）

### 17.2 场景二：SQL 从 10s 到 10ms 优化实战
- [ ] **Step 1**：EXPLAIN 分析 → 发现 type=ALL，全表扫描
- [ ] **Step 2**：检查索引 → `WHERE user_id = ? AND status = ?`，建联合索引 `(user_id, status)`
- [ ] **Step 3**：EXPLAIN 再次分析 → type=ref，rows 从 500 万降到 100
- [ ] **Step 4**：覆盖索引优化 → `SELECT user_id, status, amount` 全在索引中，Extra: Using index
- [ ] **Step 5**：验证效果 → 10s → 10ms

### 17.3 场景三：线上 CPU 100% 排查
- [ ] **Top**：找到 mysqld 进程，确认 CPU 高的线程
- [ ] **performance_schema**：
  ```sql
  SELECT * FROM sys.statement_analysis
  ORDER BY avg_timer_wait DESC LIMIT 10;
  ```
- [ ] **进程列表**：
  ```sql
  SHOW FULL PROCESSLIST;
  -- 或
  SELECT * FROM information_schema.processlist WHERE COMMAND != 'Sleep';
  ```
- [ ] **锁等待排查**：
  ```sql
  SELECT * FROM sys.innodb_lock_waits;
  ```
- [ ] 可能原因：无索引的大查询、大量排序、死锁检测开销、连接风暴

### 17.4 场景四：连接数过多处理
- [ ] Step 1：查看当前连接 `SHOW PROCESSLIST` / `SHOW STATUS LIKE 'Threads%'`
- [ ] Step 2：找出阻塞连接：大量 `Sleep` 但未释放 → 检查应用连接池配置
- [ ] Step 3：`KILL <thread_id>` 临时清理（不解决根本问题）
- [ ] Step 4：应用层用连接池（HikariCP），控制最大连接数
- [ ] Step 5：检查事务是否未提交导致连接占用
- [ ] Step 6：紧急情况：`SET GLOBAL max_connections = xxx;`

### 17.5 场景五：双十一高并发数据库设计
- [ ] **读写分离**：主库写 + 多从库读
- [ ] **缓存**：Redis 预热热点数据，缓存扛大部分流量
- [ ] **库存扣减**：Redis + Lua 原子扣减 → 异步刷 MySQL
- [ ] **限流**：令牌桶 + Sentry 限流
- [ ] **降级**：非核心查询关闭，保证核心链路
- [ ] **分库分表**：订单按 `user_id` 水平拆分
- [ ] **消息队列**：削峰填谷，订单写入异步化

### 17.6 场景六：百万级 DDL 在线变更
- [ ] 使用 `pt-online-schema-change`：
  ```bash
  pt-online-schema-change \
    --alter "ADD COLUMN address VARCHAR(200) DEFAULT ''" \
    --chunk-size=1000 \
    --max-load="Threads_running=50" \
    --critical-load="Threads_running=100" \
    D=db1,t=t_user \
    --execute
  ```
- [ ] 备选：8.0 的 INSTANT / INPLACE DDL
- [ ] 绝对不能：直接 ALTER TABLE → 锁全表，线上事故

---

## 18. 进阶路线图

### 阶段一：入门（1-2 周）
- [ ] 掌握 SQL 基础语法（CRUD、JOIN、子查询）
- [ ] 理解基本的表设计规范（范式、类型选择）
- [ ] 会用 EXPLAIN 查看执行计划
- [ ] 了解索引的基本概念和创建方法
- **检验**：从零设计一个博客系统的数据库，写出所有 CRUD SQL

### 阶段二：夯实（1-2 个月）
- [ ] 掌握 InnoDB 核心原理（B+ 树、聚簇索引、页结构）
- [ ] 深入理解索引（联合索引、最左前缀、覆盖索引、索引下推）
- [ ] 熟练进行 SQL 优化（能用 EXPLAIN + 慢查询日志定位问题）
- [ ] 理解事务 ACID 和四种隔离级别的实现
- [ ] 掌握 MVCC 原理（undo log 版本链 + ReadView）
- [ ] 掌握行锁机制（Record Lock、Gap Lock、Next-Key Lock）
- [ ] 理解 redo log / binlog / undo log 作用和配合
- **检验**：能独立优化慢 SQL，能解释为啥某个索引有效，能分析隔离级别案例

### 阶段三：进阶（2-3 个月）
- [ ] 深入 Buffer Pool（LRU 冷热分离、预读、Change Buffer）
- [ ] 搭建主从复制环境（含 GTID、半同步复制）
- [ ] 掌握分库分表设计与中间件使用（ShardingSphere）
- [ ] 熟练备份恢复策略（xtrabackup + binlog 时间点恢复）
- [ ] 掌握 pt 工具集（pt-online-schema-change、pt-query-digest）
- [ ] 搭建监控体系（Prometheus + Grafana + mysqld_exporter）
- [ ] 理解高可用方案（MHA / MGR / InnoDB Cluster）
- **检验**：能独立设计一个百万 QPS 电商数据库方案，含分库分表、读写分离、缓存

### 阶段四：专家（持续精进）
- [ ] 阅读 MySQL 8.0 重要特性源码（如 Hash Join、窗口函数实现）
- [ ] 深入优化器原理（成本模型、统计信息、直方图）
- [ ] 掌握分布式数据库原理（TiDB / OceanBase / PolarDB）
- [ ] 参与 MySQL 社区 / 贡献 Patch
- [ ] 输出技术文章 / 分享 / 培训
- **检验**：能解决团队中遇到的任何 MySQL 疑难问题，能决策技术选型

---

## 推荐资源

### 书籍
- 《MySQL 实战 45 讲》（丁奇）⭐ 必读
- 《高性能 MySQL》第 4 版
- 《MySQL 技术内幕：InnoDB 存储引擎》
- 《数据库系统概念》（基础理论）

### 在线资源
- MySQL 官方文档：https://dev.mysql.com/doc/refman/8.0/en/
- 丁奇极客时间专栏《MySQL 实战 45 讲》
- 德哥 PostgreSQL + MySQL 博客

### 动手实践
- 本地安装 MySQL 8.0
- 使用 Docker 快速搭建主从复制环境
- 建立自己的测试数据集（至少百万级）
- 每个知识点配合 SQL 实验验证

---

> **学习方法建议**：每学一个知识点，打开 MySQL 客户端亲手验证。书上的结论不等于你的理解，只有亲手用 SQL 证明过，才是真正的掌握。
