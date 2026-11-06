# 34 - 备份与恢复

## 1. 备份分类

| 维度 | 类型 |
|---|---|
| 物理 vs 逻辑 | xtrabackup（物理热备）vs mysqldump/SELECT INTO（逻辑） |
| 全量 vs 增量 | 全量周期 + binlog 增量（标准组合） |
| 热备 | InnoDB + xtrabackup = 不锁库备份 |

生产标准：**每周物理全备 + 每日 binlog 备份**，异机/异地存放。

## 2. mysqldump（小库/逻辑迁移）

```bash
# 全库
mysqldump -uroot -p --single-transaction --routines --triggers --set-gtid-purged=ON \
  --all-databases > all.sql

# 单库
mysqldump -uroot -p --single-transaction appdb > appdb.sql

# 恢复
mysql -uroot -p appdb < appdb.sql
```

- `--single-transaction`：InnoDB 一致性快照，不锁库（必加）
- 缺点：大库恢复极慢（逐条重放）

## 3. XtraBackup（物理热备，大库标准）

```bash
# 全备
xtrabackup --backup --target-dir=/backup/full -uroot -p...
# 增量
xtrabackup --backup --target-dir=/backup/inc1 --incremental-basedir=/backup/full

# 恢复：prepare → 复制回 datadir
xtrabackup --prepare --target-dir=/backup/full
xtrabackup --copy-back --target-dir=/backup/full
```

备份同时记录了 binlog 位点/GTID → 可衔接后续增量重放。

## 4. binlog 恢复到任意时间点（PITR）

```bash
# 场景：周二 15:00 误删表。恢复链 = 周一全备 + 周二 binlog 重放到误操作前
mysqlbinlog --start-datetime='2026-11-03 03:00:00' \
            --stop-datetime='2026-11-04 14:59:00' \
            mysql-bin.000123 | mysql -uroot -p
```

- ROW 格式下误删整表恢复成本高 → **核心表有延迟从库（delayed replica）兜底**
- 8.0.32+ 有 `binlog_transaction_dependency_tracking` 相关优化，GTID 定位更方便：`--include-gtids=...`

## 5. 延迟从库（防误操作的保险）

```sql
CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 3600;   -- 从库延迟 1 小时（8.0.22+ 写法）
STOP REPLICA; START REPLICA;
```

误删后：从库还保留 1 小时前的数据 → 导出误删数据回灌主库。

## 6. 恢复演练（必须做）

> "没演练过的备份等于没有备份。"

季度演练清单：
1. 恢复全备到隔离环境，业务冒烟
2. 模拟误删表 → binlog PITR 恢复，核对行数
3. 记录实际恢复时长（RTO）与数据丢失窗口（RPO），写入容灾文档

## 7. 监控备份健康度

- 备份任务完成告警（失败立即报）
- 定期校验：备份文件大小趋势、`xtrabackup --verify`（部分版本）
- 备份集保留策略：全备 4 周 + 每月归档一份

## 常见坑

1. mysqldump 没加 --single-transaction 锁库（FTWRL）
2. binlog 只在本机，机器挂了全丢 → binlog 远程同步（rds 自动化）
3. 恢复时 max_allowed_packet 太小，大行导入失败
4. 只备份不演练，真出事才知道备份是坏的
5. TRUNCATE/DROP 走 binlog 也会被重放 → PITR 时要跳过该事务（GTID 排除）

## Checklist

- [ ] 全备 + binlog 双轨且异机存放
- [ ] 核心库有延迟从库
- [ ] 每季度 PITR 演练
- [ ] RTO/RPO 有明确数字
