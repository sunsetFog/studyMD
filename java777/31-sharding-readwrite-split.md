# 31 · 分库分表与读写分离（ShardingSphere）

## 1. 演进顺序

```
单库单表 → 读写分离（主从） → 垂直拆分（按业务分库） → 水平拆分（分库分表）
```

> 不要一上来就分库分表：先做**索引优化、缓存、归档冷数据、读写分离**，单表 2000 万行以内通常都能扛。

## 2. 读写分离

### 原理

主库写、从库读，通过 binlog 主从复制（异步，有延迟）。

```yaml
# ShardingSphere-JDBC 配置
spring:
  shardingsphere:
    datasource:
      names: master,slave0,slave1
      master: {type: com.zaxxer.hikari.HikariDataSource, driver-class-name: com.mysql.cj.jdbc.Driver, jdbc-url: ..., username: root, password: root}
      slave0: {..., jdbc-url: ...}
      slave1: {..., jdbc-url: ...}
    rules:
      readwrite-splitting:
        data-sources:
          rw:
            type: Static
            props:
              write-data-source-name: master
              read-data-source-names: slave0,slave1
            load-balancer-name: round_robin
        load-balancers:
          round_robin:
            type: ROUND_ROBIN
```

**强制走主库**（解决主从延迟导致「刚写就读不到」）：

```java
HintManager.getInstance().setWriteRouteOnly();     // ShardingSphere
// 或自定义注解 + AOP 切数据源
@Master public void create() {}
```

其他解决延迟的方案：写后读主（同一会话短期内读主）、缓存写入后回填、半同步复制。

## 3. 分库分表

| 类型 | 说明 |
|---|---|
| 垂直分库 | 按业务拆（订单库、用户库） |
| 垂直分表 | 大表拆主表+扩展表（热点字段与冷字段分离） |
| 水平分库 | 同一表数据按规则分散到多个库 |
| 水平分表 | 同一库内拆成 `order_0..order_15` |

### 分片键选择

- 选**查询最频繁**的维度（如 `user_id`、`order_id`）。
- 保证数据均匀分布（避免热点）。
- 尽量让查询都带分片键，避免全库扫描（`UNION` 归并，性能差）。

### 分片算法

| 算法 | 说明 |
|---|---|
| 取模 | `user_id % 16`，均匀但扩容需迁移 |
| 范围 | 按时间/ID 区间，易扩容但可能热点 |
| 一致性哈希 | 扩容迁移少 |
| 哈希取模 + 范围（复合） | 先按 user 分库，再按月分表 |
| 自定义（SPI） | 实现 `StandardShardingAlgorithm` |

### 配置示例

```yaml
spring:
  shardingsphere:
    rules:
      sharding:
        tables:
          t_order:
            actual-data-nodes: ds${0..1}.t_order_${0..15}
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db-mod
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: tbl-mod
        sharding-algorithms:
          db-mod:
            type: MOD
            props: {sharding-count: 2}
          tbl-mod:
            type: MOD
            props: {sharding-count: 16}
```

## 4. 分布式主键

不能用数据库自增（多库冲突）。方案见 [32](32-distributed-id.md)：雪花算法、号段（Leaf-segment）、UUID。

```yaml
key-generators:
  snowflake:
    type: SNOWFLAKE
    props: {worker-id: 1}
```

## 5. 分库分表带来的问题

| 问题 | 方案 |
|---|---|
| 跨库 JOIN | 字段冗余（反范式）、多次查询内存组装、宽表/ES 聚合 |
| 跨库分页/排序 |  forbid 大 offset；用 ES 或全局二级索引表 |
| 跨库事务 | Seata / 消息最终一致（见 47） |
| 分布式主键 | 雪花 / 号段 |
| 扩容迁移 | 一致性哈希、双写迁移、停机或在线（Canal 追增量） |
| 全局唯一索引 | 用「路由表」或全局索引表 |
| 统计/报表 | 同步到数仓（Hive/ClickHouse） |

## 6. 广播表与绑定表

```yaml
broadcast-tables: t_dict, t_region      # 小表每个库一份，JOIN 本地化
binding-tables: t_order,t_order_item    # 同规则分片，JOIN 时避免笛卡尔积
```

## 7. ShardingSphere 三种形态

| 形态 | 部署 | 说明 |
|---|---|---|
| ShardingSphere-JDBC | 应用内 jar | 轻量级，性能最好，语言绑定 Java |
| ShardingSphere-Proxy | 独立服务（MySQL 协议） | 多语言可用，运维友好，多一跳 |
| Sidecar | 网格 | 规划中 |

选型：Java 单体/微服务 → JDBC；多语言/DBA 独立运维 → Proxy。

## 8. 平滑扩容（在线迁移）

```
① 双写：新旧库同时写（新库异步补偿）
② 存量迁移：按分片键分批 copy（Canal/DataX）
③ 数据对比校验（行数 + 抽样 checksum）
④ 灰度读：先读旧，逐步切新
⑤ 停写旧库，清理
```

## 9. 其他方案对比

| 方案 | 说明 |
|---|---|
| ShardingSphere | 开源主流 |
| MyCat / DBLE | 中间件代理 |
| 客户端自研 | 简单场景自己按年/月分表 |
| TiDB / OceanBase / PolarDB | **分布式数据库**，对应用透明，无需分片中间件（推荐新项目考虑） |

> 新项目如果不想折腾分库分表，优先考虑 **TiDB/PolarDB-X** 这类原生分布式数据库。

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 主从延迟读到旧数据 | 刚下单查不到 | 关键读走主库 / 缓存回填 |
| 无分片键查询 | 全库路由，慢 | 禁掉或走 ES |
| 分页错误 | 结果重复/缺失 | 归并排序；深分页禁用 |
| 分片键选错 | 数据倾斜、热点 | 重新评估 |
| `Binding table` 未配置 | 笛卡尔积 | 配 binding-tables |
| 扩容成本高 | 停机迁移 | 提前规划分片数（如 16 库 × 16 表） |
| 事务跨库 | 不一致 | 避免跨库事务；用 Seata/消息 |

## 11. 面试速答

- **什么时候分库分表**：单表数据量过大（千万/亿级）导致索引膨胀、写入瓶颈；先考虑索引、缓存、归档、读写分离。
- **分片键怎么选**：高频查询字段、分布均匀、尽量避免跨片查询。
- **跨库 JOIN 怎么做**：字段冗余、应用层组装、全局表（广播表）、ES 宽表。
- **如何平滑扩容**：双写 + 存量迁移 + 校验 + 灰度切流。
