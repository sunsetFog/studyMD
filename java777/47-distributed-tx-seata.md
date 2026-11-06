# 47 · 分布式事务与 Seata

## 1. 问题

```
订单服务：创建订单（DB1）
库存服务：扣库存（DB2）
支付服务：扣余额（DB3）
```
本地事务无法跨库跨服务保证原子性。

## 2. 理论基础

| 理论 | 内容 |
|---|---|
| CAP | 一致性、可用性、分区容错三选二（分布式必选 P，故在 C/A 间权衡） |
| BASE | 基本可用、软状态、最终一致 —— 微服务的现实选择 |
| 2PC | 两阶段提交（Prepare → Commit/Rollback），强一致但阻塞、性能差 |
| 3PC | 加 CanCommit 阶段，减少阻塞，仍有问题 |
| TCC | Try / Confirm / Cancel，业务侵入 |
| Saga | 长事务拆分为多个本地事务 + 补偿 |

## 3. 方案对比

| 方案 | 一致性 | 侵入 | 性能 | 适用 |
|---|---|---|---|---|
| XA / 2PC | 强 | 低 | 差（锁持有到提交） | 短事务、低并发 |
| Seata AT | 最终（近似强） | **极低**（加注解） | 中 | 主流推荐 |
| TCC | 最终 | **高**（写 3 个方法） | 好 | 金融、性能敏感 |
| Saga | 最终 | 中（写补偿） | 好 | 长流程、跨企业 |
| 本地消息表 | 最终 | 中 | 好 | 异步场景 |
| MQ 事务消息 | 最终 | 低 | 好 | 异步解耦 |

## 4. Seata 架构

```
TC (Transaction Coordinator)   事务协调器，独立部署，维护全局/分支事务状态
TM (Transaction Manager)       事务发起方（@GlobalTransactional 所在服务）
RM (Resource Manager)          分支事务，管理本地资源（DB）
```

四种模式：**AT**（默认，推荐）、TCC、Saga、XA。

## 5. AT 模式（Automatic Transaction）

### 原理

```
一阶段：
  Seata 拦截业务 SQL → 解析 SQL 语义 → 生成 before image（更新前快照）
  → 执行业务 SQL → 生成 after image → 插入 undo_log 行记录（与业务 SQL 同一本地事务提交）
  → 释放本地锁（★ 关键点：一阶段就提交，不长期持有锁）

二阶段-提交：
  TC 通知各 RM 异步删除 undo_log（很快）

二阶段-回滚：
  RM 用 undo_log 的 before image 生成反向 SQL 回滚
  → 校验 after image（脏写检查）→ 成功则删除 undo_log；脏写则人工介入
```

### 部署

1. 下载 Seata Server，配置 `registry.conf`（注册到 Nacos）、`file.conf`（store mode：file/db）。
2. 每个业务库建 `undo_log` 表：

```sql
CREATE TABLE undo_log (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  branch_id BIGINT NOT NULL,
  xid VARCHAR(100) NOT NULL,
  context VARCHAR(128) NOT NULL,
  rollback_info LONGBLOB NOT NULL,
  log_status INT NOT NULL,
  log_created DATETIME NOT NULL,
  log_modified DATETIME NOT NULL,
  UNIQUE KEY ux_undo_log (xid, branch_id)
);
```

### 客户端

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```
```yaml
seata:
  enabled: true
  tx-service-group: order_tx_group       # 事务分组，需与 TC 配置一致
  service:
    vgroup-mapping:
      order_tx_group: default
  registry:
    type: nacos
    nacos: {server-addr: localhost:8848, group: SEATA_GROUP}
  data-source-proxy-mode: AT
```

```java
@GlobalTransactional(name = "create-order", rollbackFor = Exception.class)
public void createOrder(OrderDTO dto) {
    orderMapper.insert(order);                       // 分支1：订单库
    inventoryClient.deduct(dto.getSkuId(), dto.getQty());  // 分支2：库存服务（HTTP）
    accountClient.debit(dto.getUserId(), dto.getAmount()); // 分支3：账户服务
}
// 任一分支抛异常 → 全局回滚
```

> **XID 传播**：通过 Feign/RestTemplate 拦截器传递 `TX_XID` header（starter 已自动集成）。

### AT 的限制

- 只支持**关系型数据库**且支持本地事务（MySQL/Oracle/PG）。
- SQL 需能被 Seata 解析（不支持复杂嵌套/部分 DDL）。
- **隔离性**：默认读未提交（一阶段已提交），需 `@GlobalLock` + `SELECT FOR UPDATE` 实现读已提交。
- 性能：比本地事务慢（多了镜像生成、undo_log 写入、全局锁）。

## 6. TCC 模式

```java
@LocalTCC
public interface InventoryTccAction {
    @TwoPhaseBusinessAction(name = "deduct", commitMethod = "commit", rollbackMethod = "rollback")
    boolean prepare(BusinessActionContext ctx,
                    @BusinessActionContextParameter(paramName = "skuId") Long skuId,
                    @BusinessActionContextParameter(paramName = "num") int num);

    boolean commit(BusinessActionContext ctx);
    boolean rollback(BusinessActionContext ctx);
}
```

```java
// Try：冻结资源（不是真扣）
UPDATE t_inventory SET frozen = frozen + #{num} WHERE sku_id=? AND available >= #{num};
// Confirm：确认扣减
UPDATE t_inventory SET used = used + #{num}, frozen = frozen - #{num} WHERE sku_id=?;
// Cancel：释放冻结
UPDATE t_inventory SET frozen = frozen - #{num} WHERE sku_id=?;
```

三个必须注意的点：
1. **幂等**：Confirm/Cancel 可能被重复调用（网络重试）。
2. **空回滚**：Try 未执行却收到 Cancel → 需记录 Try 状态，识别后直接返回成功。
3. **悬挂**：Cancel 早于 Try 到达 → Try 执行时应拒绝（查回滚记录）。

## 7. Saga 模式

长事务拆成一串本地事务，每步都有补偿操作；失败时**反向执行补偿**。

- 适合：流程长、无法锁定资源（如跨企业、订机票+酒店）。
- Seata 提供状态机编排（JSON 定义），无需代码写分支。
- 不保证隔离性（可能出现脏写），需业务容忍或加锁。

## 8. XA 模式

标准 XA 协议，`TM` 协调多个 `RM`，两阶段**不提交本地事务直到二阶段**。
- 优点：强一致，无需 undo_log。
- 缺点：锁持有时间长，性能最差。

## 9. 实践建议

| 原则 | 说明 |
|---|---|
| **能不用就不用** | 优先用「本地事务 + MQ 最终一致」 |
| 优先最终一致 | 大多数业务可接受 |
| 避免大事务 | 分支越少越好 |
| 必须设超时 | 全局事务超时后回滚 |
| 监控 undo_log | 大量残留说明有异常 |
| 隔离性要评估 | AT 默认读未提交 |

> 现实中最常用的其实是：**本地消息表 / MQ 事务消息 + 幂等 + 对账**（见 [38](38-transactional-message.md)），Seata 适合对一致性要求更高且改造成本敏感的场景。

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| undo_log 表缺失 | 回滚失败 | 每个业务库都要建 |
| 数据源未被代理 | 事务不生效 | 使用 Seata 的 `DataSourceProxy`（starter 自动） |
| XID 未传播 | 分支未注册 | 检查 Feign/RestTemplate 拦截器；用 seata starter |
| 分支异常被吞 | 不回滚 | 必须抛异常 |
| 脏写 | 回滚失败 | 用 `@GlobalLock` + `for update` |
| 全局事务超时 | 部分提交 | 配置合理超时 + 告警 |
| 与其它数据源冲突 | 代理失败 | 多数据源时手动配置代理 |
| AT 不支持的 SQL | 报错 | 改 SQL 或换 TCC |

## 11. 面试速答

- **Seata 三种角色**：TC（协调器）、TM（发起方）、RM（分支资源管理）。
- **AT 模式流程**：一阶段记录 before/after image 并写 undo_log 后**立即提交本地事务**（释放锁）；二阶段提交则删 undo_log，回滚则用 before image 反向补偿。
- **AT 与 TCC 区别**：AT 无侵入（自动生成 SQL 补偿），依赖数据库本地事务；TCC 需手写 Try/Confirm/Cancel，性能好、可控性强，但要处理幂等/空回滚/悬挂。
- **TCC 三大问题**：幂等、空回滚、悬挂。
- **分布式事务选型**：优先最终一致（消息）；强一致且简单用 Seata AT；性能与可控性要求高用 TCC；长流程用 Saga。
