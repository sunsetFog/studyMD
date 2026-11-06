# 05 - 库与表操作（DDL）

## 1. 数据库操作

```sql
CREATE DATABASE appdb DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
ALTER DATABASE appdb CHARACTER SET utf8mb4;   -- 只影响之后新建的表
DROP DATABASE appdb;                          -- 危险：不可回滚
```

## 2. 建表标准模板

```sql
CREATE TABLE `user` (
  `id`          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
  `username`    VARCHAR(64)     NOT NULL COMMENT '用户名',
  `phone`       CHAR(11)        NOT NULL DEFAULT '' COMMENT '手机号',
  `status`      TINYINT         NOT NULL DEFAULT 1 COMMENT '1正常 0禁用',
  `balance`     DECIMAL(12,2)   NOT NULL DEFAULT 0.00 COMMENT '余额',
  `created_at`  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at`  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted`     TINYINT         NOT NULL DEFAULT 0 COMMENT '软删除标记',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_username` (`username`),
  KEY `idx_phone` (`phone`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

铁律：
- 显式 `ENGINE=InnoDB CHARSET=utf8mb4`（见 04/20）
- `id BIGINT` 自增主键（见 21 聚簇索引）；雪花 ID 场景用 BIGINT 无自增
- created_at/updated_at 由数据库维护，应用层别再赋值
- 字段都写 COMMENT 和 NOT NULL DEFAULT（NULL 三值逻辑坑多）

## 3. 修改表

```sql
ALTER TABLE `user`
  ADD COLUMN `email` VARCHAR(128) NOT NULL DEFAULT '' COMMENT '邮箱' AFTER `phone`,
  MODIFY COLUMN `username` VARCHAR(128) NOT NULL,
  CHANGE COLUMN `phone` `mobile` CHAR(11) NOT NULL DEFAULT '',
  DROP INDEX idx_phone,
  ADD INDEX idx_mobile (`mobile`);
```

- `ADD/MODIFY/CHANGE` 区别：CHANGE 改名；MODIFY 只改类型
- 加列 8.0 的 INSTANT 算法：秒级完成（5.7 要重建表）

## 4. 大表 DDL 的血泪

ALTER 在大表（千万级）上会长时间锁表（部分算法）或耗时巨长。生产必须用在线工具：

```bash
# pt-online-schema-change：建影子表+触发器同步+原子改名
pt-online-schema-change --alter "ADD COLUMN email varchar(128) NOT NULL DEFAULT ''" \
  D=appdb,t=user --execute

# gh-ost（无触发器，binlog 模拟）：更推荐
gh-ost --host=... --database=appdb --table=user \
  --alter="ADD COLUMN email varchar(128)" --execute
```

经验：开发环境随意，生产千万级以上表变更一律走 pt-osc/gh-ost，且在业务低峰执行。

## 5. 表信息查询

```sql
SHOW CREATE TABLE t\G
DESC t;
SELECT table_name, table_rows, data_length/1024/1024 AS data_mb, index_length/1024/1024 AS idx_mb
FROM information_schema.tables WHERE table_schema='appdb' ORDER BY data_mb DESC;
```

## 常见坑

1. DROP/TRUNCATE 无法闪回，误操作只能靠备份恢复（见 34）→ 生产 DROP 前先备份或改名 `_bak`
2. `ALTER` 遇到 `ERROR 1062 Duplicate entry`（改唯一键时）→ 旧数据已有重复
3. 字段类型改窄（VARCHAR(128)→VARCHAR(64)）会截断报错
4. pt-osc 期间原表有外键 → 用 `--no-check-alter` 系列选项前先理清外键，最好别用外键（见 17）

## Checklist

- [ ] 建表模板包含 COMMENT/NOT NULL/DEFAULT/时间戳列
- [ ] 生产大表变更使用 pt-osc 或 gh-ost
- [ ] DROP 前有备份或改名过渡
