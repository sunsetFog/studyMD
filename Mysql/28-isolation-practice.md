# 28 - 隔离级别实战

## 1. 幻读实验（RR 下）

```sql
-- 会话 A (RR)
START TRANSACTION;
SELECT * FROM t WHERE id > 10;            -- 快照读：3 行
-- 会话 B：INSERT INTO t(id) VALUES(20); COMMIT;

SELECT * FROM t WHERE id > 10;            -- 快照读：仍 3 行（MVCC）
SELECT * FROM t WHERE id > 10 FOR UPDATE; -- 当前读：4 行！（看到新行 = 幻读）
```

结论：**RR 挡住"快照读"的幻读，挡不住"当前读"的幻读**。当前读幻读由临键锁在会话 A 加锁时阻止他人插入——若 A 先做当前读再做快照读顺序不同，现象有差异。所以标准说法是"RR 基本解决幻读，SERIALIZABLE 彻底解决"。

## 2. RC vs RR 选型

| 维度 | RC | RR（默认） |
|---|---|---|
| 幻读 | 存在 | 基本不存在 |
| Gap Lock | 无 | 有 → 死锁率更高 |
| 语句级 binlog 兼容 | 需 ROW 格式 | 任意 |
| 互联网大厂 | 阿里系常用 RC | 通用默认 |

经验：
- 默认跟 RR 走，无特殊理由不改
- 高并发写、死锁频繁、能接受不可重复读（配合乐观锁）→ RC + binlog_format=ROW
- RC + 每次提交可见 = 天然最新读，配合原子条件更新做并发控制很顺

## 3. 并发扣减库存完整案例

```sql
-- 方案 A：原子条件更新（无显式锁）
UPDATE stock SET num = num - 1
WHERE sku_id = 100 AND num > 0;
-- affected = 1 成功；0 售罄。天然并发安全（当前读+行锁排队）
```

```java
@Transactional
public boolean deduct(long skuId) {
    int n = jdbc.update("UPDATE stock SET num=num-1 WHERE sku_id=? AND num>0", skuId);
    if (n == 0) throw new BizException("售罄");
    jdbc.update("INSERT INTO stock_flow(sku_id, change_num) VALUES (?, -1)", skuId);
    return true;
}
```

```sql
-- 方案 B：SELECT FOR UPDATE 再扣（锁窗口更长，无必要则不用）
START TRANSACTION;
SELECT num FROM stock WHERE sku_id=100 FOR UPDATE;
UPDATE stock SET num = num - 1 WHERE sku_id = 100;
COMMIT;
```

方案 A 是标准答案：条件更新把"检查+扣减"合并为一个原子操作。

## 4. 防超卖 / 防重 / 幂等三件套

| 问题 | 机制 |
|---|---|
| 超卖 | 原子条件更新 `num > 0` / 乐观锁 version |
| 重复提交 | 唯一键 uk_biz_no + 捕获 1062 |
| 并发重复扣款 | 幂等表：INSERT IGNORE，成功才继续 |

三者配合构成支付/交易类系统的并发骨架。

## 5. 隔离级别相关参数

```sql
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- 新连接生效
innodb_lock_wait_timeout = 50        -- 锁等待超时（秒）
transaction_isolation = 'READ-COMMITTED'   -- my.cnf 写法
```

## 6. 验证当前级别与行为

```sql
SELECT @@transaction_isolation, @@binlog_format;
-- 两个会话跑 28.1 的实验验证幻读边界
```

## 常见坑

1. 先 SELECT 判断库存再 UPDATE → 并发下判断失效，必须条件更新
2. 乐观锁失败后不重试不提示，业务悄悄丢了
3. 以为切到 RC 就没间隙锁/没死锁——RC 死锁少但仍有锁等待
4. 事务里"读旧值做业务判断"（快照读）后用旧值写库 → 数据漂移，关键判断用当前读或原子更新

## Checklist

- [ ] 库存/余额用原子条件更新
- [ ] 幂等靠唯一键而非应用判重
- [ ] 乐观锁失败路径有重试/告警
- [ ] 修改默认隔离级别有评审记录
