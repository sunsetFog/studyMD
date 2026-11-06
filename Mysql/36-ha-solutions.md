# 36 - 高可用架构

## 1. 可用性目标

```
99.99% ≈ 年停机 52 分钟
RTO：故障恢复时间    RPO：可接受数据丢失量
```

MySQL 高可用本质：**复制（见 35）+ 故障探测 + 自动切换**。

## 2. 主流方案对比

| 方案 | 原理 | 优缺点 |
|---|---|---|
| Orchestrator | 探测拓扑 + 自动 failover + GTID 修复 | 开源主流，轻量；切换瞬间可能丢未复制事务 |
| MHA | 半同步 + binlog 补偿 | 老牌成熟，需 SSH，维护少 |
| MGR/InnoDB Cluster | 组复制（Paxos 类多数派） | 强一致无脑切换；需 MySQL Shell，运维复杂 |
| 云托管 RDS/Aurora/PolarDB | 平台内置 HA | 最省心；成本与锁定 |
| ProxySQL/Keepalived+VIP | 接入层透明 | 常配合上述方案使用 |

选型建议：自建中小规模 → Orchestrator + 半同步；强一致金融 → MGR；上云 → 托管。

## 3. 半同步配置

```ini
rpl_semi_sync_master_wait_for_slave_count = 1   # 至少 1 从库确认
rpl_semi_sync_master_timeout = 3000             # 3s 未确认降级异步（可用性优先）
```

权衡：timeout 内从库全挂 → 自动降级异步，避免主库不可写；代价是极端情况丢事务（结合双 1 与延迟从库权衡 RPO）。

## 4. MGR（Group Replication）要点

```sql
-- 单主模式（推荐）
SELECT * FROM performance_schema.replication_group_members;
SELECT @@group_replication_primary_member;
```

- 多数派写入（2N+1 节点容忍 N 故障），无脑选主
- 必须_primary_key（每表都要主键）、utf8mb4 下限制较少但外键/级联受限
- MySQL Shell `dba.createCluster()` 一键搭建 InnoDB Cluster + Router

## 5. 接入层

```
应用 → ProxySQL（读写分离/连接池/故障转移路由） → 主/从
应用 → DNS/VIP（Keepalived） → 主库
```

ProxySQL 亮点：SQL 级读写分离规则、连接复用（缓解连接数暴涨）、后端健康检查。

## 6. 故障演练

- 定期 kill 主库进程验证切换时长
- 演练网络分区（iptables）验证脑裂防护
- 切换后核对：应用重连池、长事务恢复、延迟归零

## 7. 容灾分层

```
L1 同机房主从 HA        → 故障秒级切换
L2 跨机房/跨 AZ 复制    → 机房级容灾（半同步跨 AZ）
L3 异地灾备（binlog 异步）→ RPO 分钟级，城市级故障
```

## 常见坑

1. 脑裂：旧主恢复后仍在写（read_only 未设 + VIP 未回收）
2. 切换后应用连接池全是旧连接，恢复窗口比预期长
3. MGR 网络抖动引发频繁选主风暴
4. 以为半同步=零丢失（timeout 降级仍是异步）
5. Orchestrator 探测误判（网络抖动触发切换风暴）→ 探测多源 + 稳定窗口

## Checklist

- [ ] HA 方案经过真实演练
- [ ] 旧主恢复流程有 runbook（先 read_only 再接回）
- [ ] RTO/RPO 有量化目标
- [ ] 接入层有健康检查与只读路由
