# MySQL 知识体系索引

> 版本基准：MySQL 8.0（8.0.34+），存储引擎 InnoDB。与仓库中 SpringBoot 系列配套（09/13/14 篇引用本系列）。

## 学习路线

```
入门基础 → SQL 基础 → 进阶 SQL → 表设计 → 索引深入 → 事务与锁 → 性能调优 → 运维高可用 → 实战收尾
```

## 目录

### 一、入门基础（01-04）
| 篇目 | 文件 | 核心内容 |
|---|---|---|
| 01 | [01-overview.md](01-overview.md) | MySQL 认知：体系结构、与 Oracle/PG 对比、8.0 新特性 |
| 02 | [02-install.md](02-install.md) | 安装与配置：Docker 部署、my.cnf、目录结构 |
| 03 | [03-client-tools.md](03-client-tools.md) | 客户端工具：mysql CLI、GUI 选型、连接管理 |
| 04 | [04-storage-engines.md](04-storage-engines.md) | 存储引擎：InnoDB vs MyISAM、引擎选择 |

### 二、SQL 基础（05-11）
| 篇目 | 文件 | 核心内容 |
|---|---|---|
| 05 | [05-ddl-database.md](05-ddl-database.md) | 库与表操作：CREATE/ALTER/DROP、表结构变更 |
| 06 | [06-dml-crud.md](06-dml-crud.md) | 增删改：INSERT、UPDATE、DELETE vs TRUNCATE、批量写入 |
| 07 | [07-select-basics.md](07-select-basics.md) | 查询基础：SELECT 执行顺序、WHERE、DISTINCT、LIMIT 分页 |
| 08 | [08-aggregate-group.md](08-aggregate-group.md) | 聚合与分组：GROUP BY、HAVING、聚合陷阱、ONLY_FULL_GROUP_BY |
| 09 | [09-joins.md](09-joins.md) | 多表连接：JOIN 七种结果、执行原理、连接 vs 子查询选型 |
| 10 | [10-subqueries.md](10-subqueries.md) | 子查询：相关/非相关、EXISTS vs IN、派生表 |
| 11 | [11-functions.md](11-functions.md) | 常用函数：字符串、数值、日期、JSON 函数 |

### 三、进阶 SQL（12-15）
| 篇目 | 文件 | 核心内容 |
|---|---|---|
| 12 | [12-window-functions.md](12-window-functions.md) | 窗口函数：ROW_NUMBER/RANK、累计、Top-N 分组 |
| 13 | [13-cte-recursive.md](13-cte-recursive.md) | CTE：WITH 子句、递归查询树形结构 |
| 14 | [14-views.md](14-views.md) | 视图：普通视图、可更新性、物化替代方案 |
| 15 | [15-variables-stored-proc.md](15-variables-stored-proc.md) | 变量/存储过程/触发器：何时用与何时不用 |

### 四、表设计与数据类型（16-20）
| 篇目 | 文件 | 核心内容 |
|---|---|---|
| 16 | [16-data-types.md](16-data-types.md) | 数据类型选型：整型、DECIMAL、VARCHAR vs CHAR、DATETIME vs TIMESTAMP |
| 17 | [17-constraints.md](17-constraints.md) | 约束：主键、唯一、外键争议、CHECK、默认值 |
| 18 | [18-normalization.md](18-normalization.md) | 范式与反范式：1NF/2NF/3NF、什么时候反范式 |
| 19 | [19-schema-design-cases.md](19-schema-design-cases.md) | 设计实战：订单/用户/字典表、软删除、大字段拆分 |
| 20 | [20-charset-collation.md](20-charset-collation.md) | 字符集与排序规则：utf8mb4 铁律、emoji 问题、比较规则 |

### 五、索引深入（21-24）
| 篇目 | 文件 | 核心内容 |
|---|---|---|
| 21 | [21-btree-index-theory.md](21-btree-index-theory.md) | B+树原理：为什么不用哈希/红黑树、树高与 IO |
| 22 | [22-index-types-practice.md](22-index-types-practice.md) | 索引分类实战：聚簇/二级、联合索引最左前缀、覆盖索引、前缀索引 |
| 23 | [23-explain.md](23-explain.md) | 执行计划：EXPLAIN 全字段解读、type 等级、Extra |
| 24 | [24-index-failures.md](24-index-failures.md) | 索引失效：失效场景清单、隐式转换、优化器成本 |

### 六、事务与锁（25-28）
| 篇目 | 文件 | 核心内容 |
|---|---|---|
| 25 | [25-transactions.md](25-transactions.md) | 事务：ACID 实现、隔离级别与并发问题、savepoint |
| 26 | [26-mvcc.md](26-mvcc.md) | MVCC：undo log、ReadView、可见性判断规则 |
| 27 | [27-locks.md](27-locks.md) | 锁体系：行锁/间隙锁/临键锁、死锁排查、热点行 |
| 28 | [28-isolation-practice.md](28-isolation-practice.md) | 隔离级别实战：幻读实验、选型建议、并发扣减案例 |

### 七、性能调优（29-32）
| 篇目 | 文件 | 核心内容 |
|---|---|---|
| 29 | [29-slow-query.md](29-slow-query.md) | 慢查询：slow log 配置、分析三板斧、pt-query-digest |
| 30 | [30-sql-optimization.md](30-sql-optimization.md) | SQL 优化：深分页、count 优化、JOIN 优化、LIMIT 1 |
| 31 | [31-server-tuning.md](31-server-tuning.md) | 参数调优：buffer pool、连接、内存分配公式 |
| 32 | [32-large-tables.md](32-large-tables.md) | 大表优化：分库分表、冷热分离、archiver 策略 |

### 八、运维与高可用（33-37）
| 篇目 | 文件 | 核心内容 |
|---|---|---|
| 33 | [33-log-system.md](33-log-system.md) | 日志体系：redo/undo/binlog 三大日志、两阶段提交 |
| 34 | [34-backup-restore.md](34-backup-restore.md) | 备份恢复：mysqldump、xtrabackup、binlog 恢复、演练 |
| 35 | [35-replication.md](35-replication.md) | 主从复制：原理、GTID、延迟处理、并行复制 |
| 36 | [36-ha-solutions.md](36-ha-solutions.md) | 高可用：MHA/Orchestrator/MGR、读写分离、故障切换 |
| 37 | [37-security-permission.md](37-security-permission.md) | 安全：用户权限、SQL 注入、敏感数据加密 |

### 九、实战收尾（38-39）
| 篇目 | 文件 | 核心内容 |
|---|---|---|
| 38 | [38-interview-questions.md](38-interview-questions.md) | 面试题 30 道 + 答题方法论 |
| 39 | [39-cheatsheet.md](39-cheatsheet.md) | 速查卡：语法、参数、症状反查表 |

## 全局规则（只写一次）

- **utf8mb4** 默认字符集：见 20
- **金额用 DECIMAL** 不用 FLOAT/DOUBLE：见 16
- **索引失效全清单**：见 24
- **间隙锁/临键锁只在 RR 出现**：见 27
- **两阶段提交保证 binlog 与 redo 一致**：见 33
- **大表 DDL 用 pt-osc/gh-ost**：见 05
- **分页优化、深分页**：见 30
