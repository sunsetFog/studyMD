# 18 - 范式与反范式

## 1. 三大范式

- **1NF**：列不可再分（原子性）。地址拆省/市/区即体现 1NF 精神
- **2NF**：非主键列完全依赖主键（消除部分依赖）。组合主键场景：订单明细表以 (order_id, product_id) 为主键时，产品名不能放明细表（只依赖 product_id）
- **3NF**：消除传递依赖。订单表存 user_id 不存 username（否则改昵称要刷全表）

现代意义：范式核心是**消除冗余带来的更新异常**（改一处漏多处）。

## 2. 为什么互联网要反范式

范式化 → 多表 JOIN → 高并发下 JOIN 成本 + 无法分库分表。

常见反范式手段：

```sql
-- 订单表冗余下单时的快照信息
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  user_name VARCHAR(64) NOT NULL,      -- 冗余：下单时昵称（业务语义是"历史值"）
  product_name VARCHAR(128) NOT NULL,  -- 冗余：商品快照
  amount DECIMAL(12,2) NOT NULL
);
```

关键：**冗余的是"快照语义"数据（下单时点），不是"当前值"数据**。当前值冗余（如订单表冗余用户当前手机号）会造成同步地狱。

## 3. 冗余当前值的同步策略（确需时）

1. 同一事务双写（强一致，吞吐受限）
2. 事务 + 消息队列异步刷新（最终一致，见 SpringBoot 25）
3. 定时对账修复（兜底）

## 4. 宽表与汇总表

```sql
-- 汇总表（计数/统计预计算，替代实时 GROUP BY）
CREATE TABLE user_order_stat (
  user_id BIGINT PRIMARY KEY,
  order_count INT NOT NULL DEFAULT 0,
  total_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
  updated_at DATETIME
);
-- 增量更新（事务内）
UPDATE user_order_stat
SET order_count = order_count + 1, total_amount = total_amount + 88.00
WHERE user_id = 1;
```

- 替代视图物化（MySQL 无物化视图，见 14）
- 定时任务全量重算兜底 + 对账

## 5. 决策框架

| 数据 | 方案 |
|---|---|
| 强一致主数据（账户余额） | 3NF 单表，不冗余 |
| 历史快照（订单里的商品名/价格） | 冗余快照，永不回填 |
| 高频读统计 | 汇总表/缓存 |
| 分析报表 | 数仓/ClickHouse，业务库不扛 |

## 常见坑

1. 冗余"当前值"后忘了同步链路，数据漂移
2. 过度反范式：每张表都塞冗余字段，没人知道谁是权威来源
3. 汇总表与明细不一致无对账
4. 快照字段设计成"可变"——快照应视为不可变

## Checklist

- [ ] 明确每个冗余字段是快照还是当前值
- [ ] 当前值冗余有同步+对账机制
- [ ] 统计需求优先汇总表/缓存而非实时聚合
