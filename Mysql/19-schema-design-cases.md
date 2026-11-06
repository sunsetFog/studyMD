# 19 - 表设计实战

## 1. 通用规范（阿里规约精要）

- 库名/表名/字段名：小写下划线，见名知义，禁保留字
- 每表必备：`id BIGINT` 主键、`created_at`、`updated_at`
- 软删除用 `deleted TINYINT`（不物理删，配合唯一键见 17）
- 单表行数预警：5000 万行或表容量 2GB 建议分库分表（经验值，重点看增长曲线与索引效率）
- 单表字段 ≤ 50，大字段拆分

## 2. 订单系统（经典案例）

```sql
CREATE TABLE orders (
  id            BIGINT UNSIGNED NOT NULL COMMENT '雪花ID',
  order_no      VARCHAR(32)  NOT NULL COMMENT '对外单号',
  user_id       BIGINT       NOT NULL,
  status        TINYINT      NOT NULL DEFAULT 0 COMMENT '0待支付 1已支付 2已发货 3完成 4取消',
  total_amount  DECIMAL(12,2) NOT NULL DEFAULT 0,
  pay_time      DATETIME     NULL,
  created_at    DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_order_no (order_no),
  KEY idx_user_status (user_id, status),        -- "我的订单"页查询
  KEY idx_created (created_at)                  -- 运营按时间查
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE order_item (
  id          BIGINT UNSIGNED NOT NULL,
  order_id    BIGINT       NOT NULL,
  product_id  BIGINT       NOT NULL,
  product_name VARCHAR(128) NOT NULL,   -- 快照
  price       DECIMAL(12,2) NOT NULL,   -- 快照
  quantity    INT          NOT NULL,
  PRIMARY KEY (id),
  KEY idx_order (order_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

要点：order_item 不用 orders.order_no 关联（省空间，id 快）；商品信息全部快照冗余；状态机字段 + CHECK 兜底。

## 3. 用户/账户

```sql
CREATE TABLE account (
  id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id    BIGINT        NOT NULL,
  balance    DECIMAL(16,2) NOT NULL DEFAULT 0,
  version    INT           NOT NULL DEFAULT 0,   -- 乐观锁（见 28）
  updated_at DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_user (user_id)
) ENGINE=InnoDB;
```

金额变动三原则：
1. `balance = balance ± x` 原子更新，不读改写
2. 变动必须落流水表（可审计可恢复）
3. 条件更新 `WHERE balance >= x` 防负余额

```sql
CREATE TABLE account_flow (
  id          BIGINT UNSIGNED NOT NULL,
  account_id  BIGINT      NOT NULL,
  change_type TINYINT     NOT NULL,
  amount      DECIMAL(16,2) NOT NULL,
  balance_after DECIMAL(16,2) NOT NULL,
  biz_no      VARCHAR(64) NOT NULL,
  created_at  DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_biz (biz_no),        -- 幂等：同一业务单只入账一次
  KEY idx_account (account_id, created_at)
) ENGINE=InnoDB;
```

## 4. 字典/枚举表

```sql
CREATE TABLE dict (
  id INT NOT NULL,
  type VARCHAR(32) NOT NULL,
  code VARCHAR(32) NOT NULL,
  name VARCHAR(64) NOT NULL,
  sort INT NOT NULL DEFAULT 0,
  status TINYINT NOT NULL DEFAULT 1,
  PRIMARY KEY (id),
  UNIQUE KEY uk_type_code (type, code)
) ENGINE=InnoDB;
```

业务表存 code（TINYINT/VARCHAR），显示层 join 或缓存字典——修改文案不改业务表。

## 5. 分布式 ID 选型

| 方案 | 特点 |
|---|---|
| 自增 AUTO_INCREMENT | 单库简单；分库后冲突 |
| 雪花 Snowflake | 趋势递增 BIGINT，时钟回拨需处理 |
| 号段模式（DB 发段） | 强依赖 DB 但实现简单 |
| UUID | 无序，聚簇索引页分裂（见 21），禁做主键 |

## 常见坑

1. 订单表状态无状态机约束，出现"已完成又发货"
2. 金额更新先 SELECT 再 UPDATE（并发丢更新）→ 原子条件更新或乐观锁
3. 大 TEXT 与常用列同表，每页行数骤降 → 拆详情表
4. 分库分表键选择：订单按 user_id 分（C 端查询均匀），交易查询另建按 order_no 的映射

## Checklist

- [ ] 主表含 id/created_at/updated_at/软删
- [ ] 金额变动有流水 + 幂等唯一键
- [ ] 业务快照冗余，当前值不冗余
- [ ] 主键用分布式 ID 或自增 BIGINT（非 UUID）
