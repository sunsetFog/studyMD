# 33 - 日志体系：redo / undo / binlog

## 1. 三大日志定位

| 日志 | 层级 | 内容 | 作用 |
|---|---|---|---|
| redo log | InnoDB 引擎 | 物理/页级修改 | 崩溃恢复（持久性），WAL |
| undo log | InnoDB 引擎 | 逻辑反向操作 | 回滚（原子性）+ MVCC |
| binlog | Server 层 | 逻辑语句/行事件 | 复制、备份恢复、审计 |

## 2. redo log：WAL 原理

Write-Ahead Logging：先写日志（顺序 IO）再刷数据页（随机 IO）。

```
事务提交：修改 buffer pool 页 → 写 redo log buffer → 持久化 redo log → COMMIT 成功
数据页（脏页）由后台线程异步刷盘
```

- 循环写：固定容量（innodb_redo_log_capacity），写满 checkpoint 停顿 → 容量配置见 31
- `innodb_flush_log_at_trx_commit`：1=提交刷盘（默认）、0=每秒、2=提交写 OS cache
- 崩溃恢复：重放 redo 确保已提交的修改在页上生效

## 3. binlog：三种格式

```ini
binlog_format = ROW        # 生产标配
binlog_row_image = FULL    # FULL/MINIMAL
binlog_expire_logs_seconds = 604800
```

| 格式 | 记录 | 问题 |
|---|---|---|
| STATEMENT | SQL 原文 | NOW()/UUID() 等不确定函数主从不一致 |
| ROW | 行级变更前后镜像 | 量大（但最安全） |
| MIXED | 自动切换 | 遗留 |

ROW 是复现/MQ CDC（Canal/Flink CDC）/审计的基础。

## 4. 两阶段提交（2PC）——为什么需要

一次 UPDATE 的提交顺序：

```
1. 写 redo log，状态 prepare
2. 写 binlog
3. redo log 状态 commit
```

崩溃恢复判定：
- redo prepare + binlog 完整 → 提交（binlog 已复制给从库，不能丢）
- redo prepare + binlog 不完整 → 回滚

目的：**保证 redo（引擎）与 binlog（复制）的一致性**。不做 2PC，主库恢复后与从库数据分叉。

## 5. 组提交与性能

- binlog 组提交：多个并发事务的 flush 合并一次刷盘，`binlog_group_commit_sync_delay` 可微调
- sync_binlog=N：攒 N 个事务刷一次（性能↑，崩溃丢 N 个事务）

## 6. 与应用侧联动

- CDC：Canal/Debezium 订阅 ROW binlog → 同步 ES/缓存/数仓
- 复制：主从就是 binlog 重放（见 35）
- 误删数据恢复：备份 + binlog 位置回放（见 34）

## 常见坑

1. redo 太小，写入高峰 checkpoint 频繁，性能锯齿
2. STATEMENT 格式 + NOW() 导致主从不一致（老库遗留）
3. binlog 无过期清理撑爆磁盘
4. 双 1 改 0/2 后忘记评估丢失窗口
5. 认为 undo log 只是回滚用——它还是 MVCC 的版本仓库（见 26）

## Checklist

- [ ] binlog_format=ROW 常开
- [ ] redo 容量 ≥ 1 小时峰值写入
- [ ] 能讲清 2PC 与崩溃恢复判定
