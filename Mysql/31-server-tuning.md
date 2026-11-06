# 31 - 服务端参数调优

## 1. Buffer Pool（最重要）

```ini
innodb_buffer_pool_size = 12G       # 专机物理内存 50%~70%
innodb_buffer_pool_instances = 8    # 默认 8，>1G 才生效，减少锁竞争
innodb_io_capacity = 2000           # SSD 调高（HDD 200）
innodb_io_capacity_max = 4000
```

命中率观察：

```sql
SELECT (1 - Innodb_buffer_pool_reads/Innodb_buffer_pool_read_requests) * 100 AS hit_rate
FROM (SELECT
  MAX(IF(VARIABLE_NAME='Innodb_buffer_pool_reads', VARIABLE_VALUE, 0)) Innodb_buffer_pool_reads,
  MAX(IF(VARIABLE_NAME='Innodb_buffer_pool_read_requests', VARIABLE_VALUE, 0)) Innodb_buffer_pool_read_requests
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN ('Innodb_buffer_pool_reads','Innodb_buffer_pool_read_requests')) t;
-- 目标 > 99%
```

## 2. 双 1 与持久性/性能权衡

```ini
innodb_flush_log_at_trx_commit = 1   # 1=每次提交刷 redo（安全）
sync_binlog = 1                      # 1=每次提交刷 binlog（安全）
```

- 双 1：数据最安全，写性能低
- 2 + 100（或 0）：高吞吐日志场景，宕机丢最近 1 秒事务
- 金融/交易必须双 1；日志表/可重算数据可放宽

## 3. 连接与线程

```ini
max_connections = 500
thread_cache_size = 64
wait_timeout = 600
max_allowed_packet = 256M            # 大批量导入需要
```

> 连接数不是越大越好：每连接线程有内存开销，连接打满 CPU 切换反而更慢（见 03 连接池原则）。

## 4. 排序/分组/临时表

```ini
sort_buffer_size = 2M                # 会话级，别设大（每连接分配）
join_buffer_size = 2M
tmp_table_size = 64M
max_heap_table_size = 64M            # 内存临时表上限，超过转磁盘
```

会话级 buffer 全调大 = 内存爆雷的经典事故，默认值通常够。

## 5. 表空间/文件

```ini
innodb_file_per_table = 1            # 默认
innodb_redo_log_capacity = 2G        # redo 太小 checkpoint 频繁抖动
innodb_flush_method = O_DIRECT       # 避免双写 OS cache（Linux）
```

## 6. 监控指标基线

```sql
-- QPS/TPS
SHOW GLOBAL STATUS LIKE 'Questions';
-- 线程
SHOW GLOBAL STATUS LIKE 'Threads_connected';
-- 临时表落盘（0 理想）
SHOW GLOBAL STATUS LIKE 'Created_tmp_disk_tables';
-- 全表扫描比例
SHOW GLOBAL STATUS LIKE 'Select_scan';
-- 锁等待
SHOW GLOBAL STATUS LIKE 'Innodb_row_lock_%';
```

调优顺序：**先慢 SQL（29/30），再参数**。参数只解决"缓存不够/IO 太慢"类问题，救不了烂 SQL。

## 7. 内存预算估算（专机 32G 示例）

```
buffer pool          20G (65%)
每连接 ~10MB × 500   5G
临时表/排序          2G
OS + 其他            5G
```

容器环境注意 cgroup 限制，`innodb_buffer_pool_size` 要按容器限额配置（参见 Java 系列容器篇）。

## 常见坑

1. sort_buffer/join_buffer 设成 1G，连接一多 OOM
2. buffer pool 小于热数据集，命中率 90% 以下抖动
3. 云盘/网络盘上 O_DIRECT 与平台特性冲突
4. max_connections 抬到 10000 掩盖连接池泄漏

## Checklist

- [ ] buffer pool ≥ 物理内存 50%
- [ ] 生产双 1（或评审放宽）
- [ ] 会话级 buffer 保持默认
- [ ] 命中率与 tmp_disk 指标有监控
