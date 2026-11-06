# 35 - 主从复制

## 1. 原理

```
主库：事务提交 → binlog 落盘 → dump 线程推送
从库：IO 线程接收 → relay log（中继日志）→ SQL/协调线程回放
```

- 异步复制（默认）：主库不等从库 → 主库崩溃可能丢最新事务
- 半同步（semi-sync）：至少一个从库确认收到 binlog 才返回客户端 → 丢失窗口缩小

```ini
# 主库
server_id = 1
log_bin = mysql-bin
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON

# 从库
server_id = 2
relay_log = relay-bin
read_only = ON        # 8.0 用 super_read_only 防超级用户写
```

## 2. GTID 复制（标配）

每个事务有全局唯一 ID（server_uuid:seq），复制位点不再靠文件名+offset：

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='master', SOURCE_USER='repl', SOURCE_PASSWORD='...',
  SOURCE_AUTO_POSITION=1;
START REPLICA;
```

故障切换时新主上 `SOURCE_AUTO_POSITION` 自动对齐缺口，简单可靠。

## 3. 查看复制状态

```sql
SHOW REPLICA STATUS\G   -- 8.0.22+ 写法（旧 SHOW SLAVE STATUS）
-- Replica_IO_Running / Replica_SQL_Running 都要 Yes
-- Seconds_Behind_Source：延迟秒数
SELECT * FROM performance_schema.replication_applier_status_by_worker;
```

## 4. 主从延迟：成因与解法

成因：
1. 大事务（一次回放一大坨）→ 事务拆小（见 06）
2. 单线程回放跟不上并行写 → **并行复制**：

```ini
replica_parallel_workers = 8
replica_parallel_type = LOGICAL_CLOCK     # 基于 binlog 组提交并行
```

3. 从库机器规格低 / 单线程 DDL
4. 长期延迟：升级从库规格、拆分读流量

## 5. 读写分离与延迟问题

读走从库的风险：写后立即读 → 从库还没回放 → 读到旧数据。

应对（按代价排序）：
1. 业务规避：写后重定向主库读（关键路径）
2. GTID 等待：`SELECT WAIT_FOR_EXECUTED_GTID_SET('uuid:1', 1);` 确认再读
3. 中间件（ShardingSphere/ProxySQL）读一致性路由
4. 强一致读一律主库

## 6. 主从切换流程

1. 确认新主追平（GTID 对齐）
2. 应用停写 / VIP 摘除
3. 提升从库：`STOP REPLICA; RESET REPLICA ALL;` 关 read_only
4. 其余从库指向新主
5. 应用/代理切换，验证写入

手动切换可配合 Orchestrator 自动化（见 36）。

## 7. 数据一致性校验

```bash
pt-table-checksum --host=master --databases=appdb   # 主从数据校验
pt-table-sync                                       # 修复小范围差异（谨慎）
```

## 常见坑

1. 从库被误写（未设 read_only）导致主从不一致
2. Seconds_Behind_Source=0 但实际 relay 堆积（心跳间隙）
3. 大 DDL 在从库回放数小时，读流量异常
4. 无 GTID 的传统复制切换丢事务

## Checklist

- [ ] GTID + ROW 格式 + 并行复制开启
- [ ] 从库 read_only
- [ ] 延迟监控与告警（阈值如 >10s）
- [ ] 关键读路径明确主从路由策略
