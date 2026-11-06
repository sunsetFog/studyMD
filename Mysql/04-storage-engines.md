# 04 - 存储引擎

## 1. 引擎层架构

MySQL 是分层设计：服务层（解析/优化）对所有引擎一致，引擎层决定数据怎么存、锁怎么加、事务是否支持。

```sql
SHOW ENGINES;                    -- 查看支持的引擎
SELECT ENGINE, COUNT(*) FROM information_schema.tables GROUP BY ENGINE;
SHOW TABLE STATUS LIKE 'orders'; -- 查看表的引擎/行数/平均行长
```

## 2. InnoDB vs MyISAM

| 维度 | InnoDB（默认） | MyISAM |
|---|---|---|
| 事务 | 支持 ACID | 不支持 |
| 锁 | 行级锁 + 表锁 | 只有表锁 |
| 崩溃恢复 | redo log 安全恢复 | 崩溃可能表损坏 |
| MVCC | 支持 | 不支持 |
| 索引结构 | 聚簇索引（数据即主键树） | 非聚簇（索引与数据分离） |
| count(*) 全表 | 需扫描（MVCC 下无全局计数） | 直接存总数（无 WHERE 时） |
| 适用 | 几乎一切 OLTP | 仅只读归档/日志表 |

> 常见面试题："为什么 InnoDB 没有 MyISAM 那样的精确 count(*) 总数？"——因为 MVCC 下同一时刻不同事务看到的行数不同，全局计数无法对所有人正确。

## 3. InnoDB 关键内部结构

```
内存：Buffer Pool（缓存数据页/索引页）、Change Buffer、Log Buffer、AHI
磁盘：表空间(ibd) → 段 → 区(1MB) → 页(16KB，最小 IO 单元) → 行
日志：redo log（物理日志，崩溃恢复）、undo log（逻辑日志，回滚+MVCC）
后台线程：Master、IO、Purge（清理已提交事务的 undo）
```

**页（16KB）是一切的根本**：一次 IO 读一页、索引树一个节点一页、Buffer Pool 按页缓存。这也解释了行越小越好（一页放下更多行）。

## 4. 其他引擎

| 引擎 | 用途 |
|---|---|
| Memory | 临时数据（重启丢失），哈希索引 |
| Archive | 高压缩比归档，只支持插入/查询 |
| CSV | 数据交换 |
| MyRocks | RocksDB 引擎，高压缩写密集场景（Facebook 系） |

## 5. 修改引擎

```sql
ALTER TABLE t ENGINE = InnoDB;   -- 注意：会重建表，大表要走 pt-osc（见 05）
```

## 6. 与应用层的关系

- SpringBoot `spring.datasource` 不需要指定引擎，建表时 `ENGINE=InnoDB` 显式声明（防止依赖服务器默认）
- JPA `hibernate.dialect` 自动适配

## 常见坑

1. 建表忘写 ENGINE，dump 到 MyISAM 默认的老服务器，失去事务 → 建表语句显式 `ENGINE=InnoDB DEFAULT CHARSET=utf8mb4`
2. `ROW_FORMAT=DYNAMIC`（8.0 默认）下 VARCHAR 超长部分溢出页存储，行内仍占 20 字节指针
3. MyISAM 表迁移到 InnoDB 后发现 count 变慢——预期行为，见 30 的 count 优化

## Checklist

- [ ] 所有业务表 ENGINE=InnoDB
- [ ] 理解页 16KB 与索引/IO 的关系
- [ ] 能解释 InnoDB count(*) 慢的原因
