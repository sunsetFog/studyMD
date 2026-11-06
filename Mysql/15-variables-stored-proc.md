# 15 - 变量、存储过程与触发器

## 1. 三类变量

```sql
-- 用户变量（会话级，@开头）
SET @total = (SELECT COUNT(*) FROM user);
SELECT @total;

-- 局部变量（存储过程内，DECLARE）
DECLARE cnt INT DEFAULT 0;

-- 系统变量
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SET SESSION sort_buffer_size = 2 * 1024 * 1024;   -- 会话级
SET GLOBAL max_connections = 800;                 -- 全局（重启失效，需写入 my.cnf）
```

## 2. 存储过程

```sql
DELIMITER //
CREATE PROCEDURE transfer(IN from_id BIGINT, IN to_id BIGINT, IN amt DECIMAL(12,2))
BEGIN
  DECLARE insufficient CONDITION FOR SQLSTATE '45000';
  START TRANSACTION;
    UPDATE account SET balance = balance - amt WHERE id = from_id AND balance >= amt;
    IF ROW_COUNT() = 0 THEN
      SIGNAL insufficient SET MESSAGE_TEXT = '余额不足';
    END IF;
    UPDATE account SET balance = balance + amt WHERE id = to_id;
  COMMIT;
END //
DELIMITER ;

CALL transfer(1, 2, 100.00);
```

要点：`SIGNAL` 抛业务错误；`ROW_COUNT()` 取上一语句影响行数；`START TRANSACTION`/`COMMIT` 内嵌事务。

## 3. 何时用、何时不用

| 用存储过程 | 不用（互联网主流） |
|---|---|
| 报表库/数据分析、DBA 维护任务 | 核心业务逻辑（难以版本管理/测试/水平扩展） |
| 数据修复批处理脚本 | 需要随应用发布演进的功能 |

原则：业务逻辑在应用层，库内逻辑只做"数据自治"（约束、触发式维护）。

## 4. 触发器

```sql
CREATE TRIGGER trg_salary_check BEFORE INSERT ON emp
FOR EACH ROW
BEGIN
  IF NEW.salary < 0 THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = '薪资不能为负';
  END IF;
END;
```

- `NEW`/`OLD` 引用新旧行；BEFORE/AFTER × INSERT/UPDATE/DELETE 组合
- 限制：一个表每事件最多一个触发器；**触发器内不能操作本表**（会递归）
- 隐蔽性差、拖慢写入、主从复制行为复杂 → 业务校验放应用层，触发器仅兜底

## 5. 事件调度器（定时任务）

```sql
SET GLOBAL event_scheduler = ON;

CREATE EVENT ev_archive_log
ON SCHEDULE EVERY 1 DAY STARTS '2026-01-01 02:00:00'
DO DELETE FROM log WHERE created_at < NOW() - INTERVAL 90 DAY;
```

注意：MySQL 事件无分布式协调，多实例部署会重复执行 → 用应用层调度（xxl-job/Quartz）+ 分布式锁替代。

## 常见坑

1. DELIMITER 忘记改回 `;`
2. 存储过程里的长事务 + 外部调用（如发 HTTP，不可能）——确保过程内只做 DB 操作
3. 触发器把简单 INSERT 拖慢，且故障难排查
4. `SET GLOBAL` 重启丢失 → 同时写入 my.cnf

## Checklist

- [ ] 业务逻辑不放存储过程
- [ ] 触发器只做兜底校验
- [ ] 定时任务用应用调度器而非 EVENT
