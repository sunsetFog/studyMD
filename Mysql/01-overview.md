# 01 - MySQL 整体认知

## 1. MySQL 是什么

开源关系型数据库（RDBMS），现属 Oracle。遵循 C/S 架构：`mysqld` 服务端进程 + 客户端通过 TCP(3306)/Unix Socket 连接。

## 2. 体系结构（面试必考）

```
客户端
  ↓ (连接协议)
连接层：连接管理、认证、线程池
  ↓
服务层：
  - SQL 接口（DDL/DML/存储过程）
  - 解析器（词法/语法分析 → 解析树）
  - 优化器（选索引、JOIN 顺序、改写 SQL）
  - 执行器（调用存储引擎接口）+ 查询缓存（8.0 已移除）
  ↓ (Storage Engine API)
引擎层：InnoDB（默认）、MyISAM、Memory...
  ↓
文件层：数据文件、日志文件（redo/undo/binlog）
```

关键认知：
- **一条 SQL 的旅程**：连接器 → 分析器 → 优化器 → 执行器 → 引擎 → 返回
- **服务层与引擎层分离**：同一 SQL 在不同引擎下行为不同（事务、锁）
- **8.0 移除查询缓存**：命中率低 + 失效代价高（任何写操作清空相关缓存）

## 3. InnoDB 为什么是默认引擎

- 支持事务（ACID）、行级锁、MVCC、崩溃恢复（redo log）
- 聚簇索引组织表，主键访问极快
- MyISAM 仅剩场景：只读归档/系统临时表——生产几乎不用

## 4. 与主流数据库对比

| 维度 | MySQL 8.0 | PostgreSQL | Oracle | MongoDB |
|---|---|---|---|---|
| 定位 | OLTP 互联网标配 | 功能最全的开源 RDBMS | 商业 OLTP/分析 | 文档型 NoSQL |
| 复杂查询/分析 | 一般（无哈希连接 8.0.18 前后差异） | 强 | 强 | 弱 |
| JSON | 函数支持一般 | JSONB 强 | 一般 | 原生 |
| 成本/生态 | 免费云原生生态最广（阿里 RDS/Aurora 兼容系） | 免费 | 昂贵 | 免费 |
| 选型一句话 | 互联网业务高并发读写 → MySQL；复杂分析/地理/数组 → PG |

## 5. MySQL 8.0 关键新特性

| 特性 | 说明 |
|---|---|
| 窗口函数 | ROW_NUMBER/LEAD/LAG，替代自关联（见 12） |
| CTE / 递归查询 | WITH RECURSIVE 查组织树（见 13） |
| 降序索引 | 真正存储降序，混合排序场景受益 |
| 隐藏索引 | invisible index，安全下线索引（见 24） |
| 角色 Role | CREATE ROLE 权限管理（见 37） |
| 通用表表达式 + 直方图统计 | 优化器对倾斜数据更准 |
| utf8mb4 为默认字符集 | 8.0 起默认（见 20） |
| 移除查询缓存 | 需用 Redis 自建缓存 |

## 6. 版本选择

- 8.0.x：新项目唯一推荐。注意 8.0 与 5.7 差异（默认认证插件 caching_sha2_password、默认隔离级别仍 RR、sql_mode 更严格）
- 5.7：存量维护，EOL 已过，勿新选
- MariaDB：谨慎选择，生态与 MySQL 分叉（JSON/窗口实现差异）

## 常见坑

1. 8.0 连接报 `Unknown authentication method` → 客户端老驱动不支持 caching_sha2_password，升级驱动或用户改 `mysql_native_password`（过渡方案）
2. 从 5.7 迁移后 SQL 大量报 `ONLY_FULL_GROUP_BY` 错误 → 见 08
3. `group_concat_max_len` 默认 1024，GROUP_CONCAT 结果被截断

## Checklist

- [ ] 能画出三层体系结构并说出一条 SQL 的执行旅程
- [ ] 知道 8.0 移除查询缓存的原因
- [ ] 新项目使用 8.0.x，未选 5.7/MariaDB
