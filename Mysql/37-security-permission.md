# 37 - 安全与权限

## 1. 用户管理（8.0）

```sql
CREATE USER 'app'@'10.0.%' IDENTIFIED BY 'StrongPass#2026'
  PASSWORD EXPIRE INTERVAL 180 DAY
  FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 1;      -- 8.0.19+ 防爆破

ALTER USER 'app'@'%' IDENTIFIED BY 'NewPass#2027';
DROP USER 'app'@'%';
SELECT user, host FROM mysql.user;
```

- host 尽量收窄（内网网段），禁止 `'app'@'%'` 暴露公网
- 8.0 默认认证插件 caching_sha2_password（见 02 兼容性坑）

## 2. 权限模型（最小权限）

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app'@'%';
GRANT SELECT ON appdb.* TO 'report'@'%';                    -- 报表只读
GRANT SELECT ON appdb.v_active_user TO 'bi'@'%';            -- 通过视图裁剪敏感列（见 14）
REVOKE DELETE ON appdb.* FROM 'app'@'%';
SHOW GRANTS FOR 'app'@'%';
```

- 权限粒度：全局 > 库 > 表 > 列；生产业务账号不给 DDL/DROP
- 8.0 Role：

```sql
CREATE ROLE r_app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO r_app_rw;
GRANT r_app_rw TO 'app'@'%';
```

## 3. 传输与存储加密

```ini
# 传输：TLS
require_secure_transport = ON
# JDBC: useSSL=true&verifyServerCertificate=...
```

```sql
-- 静态加密（表空间加密，密钥管理需 KMS）
CREATE TABLE t (...) ENCRYPTION='Y';

-- 应用层字段加密（推荐对敏感 PII）
-- 手机号/身份证：AES_ENCRYPT + 应用侧 KMS 管理 key；DBA 看不到明文
INSERT INTO user(phone_enc) VALUES (AES_ENCRYPT('13800000001', ?));
SELECT AES_DECRYPT(phone_enc, ?) FROM user;
-- 或应用层加密（Jasypt/阿里 KMS），DB 内只有密文
```

- 数据脱敏展示：见 11 函数（`CONCAT(LEFT(phone,3),'****',RIGHT(phone,4))`）或 8.0.30+ 动态脱敏组件

## 4. SQL 注入防护

```java
// 错误：字符串拼接
String sql = "SELECT * FROM user WHERE name = '" + name + "'";   // ' OR '1'='1

// 正确：参数化
jdbc.update("SELECT * FROM user WHERE name = ?", name);
// MyBatis：#{} 预编译；${} 拼接（只用于表名/排序白名单场景）
```

- MyBatis `${}` 是注入重灾区：动态表名/ORDER BY 必须白名单校验
- 权限隔离兜底：即使注入也不该 DROP（业务账号无 DDL）

## 5. 审计与加固

```ini
log_error_verbosity = 3
general_log = OFF            # 排查时临时开，量大
# 企业审计：MariaDB Audit Plugin / 云审计日志
```

加固清单：
- 3306 不暴露公网；堡垒机/SSH 隧道
- 定期清理无用账号与老密码
- `mysql_secure_installation`（匿名用户、test 库）
- 敏感操作双人复核（DROP/TRUNCATE 走工单）

## 6. 备份安全

- 备份文件含全量数据：加密存放、访问控制
- 恢复演练环境同样受权限管理（见 34）

## 常见坑

1. root 远程可连（`root@'%'`）+ 弱密码 = 被勒索病毒删库的经典起点
2. JDBC 连接串带明文密码进 git → 环境变量/密钥管理
3. MyBatis `${ORDER BY}` 排序列来自前端直接注入
4. 唯一/校验依赖 ci 排序规则造成的"密码都一样"（见 20）

## Checklist

- [ ] 业务账号最小权限，无 DDL
- [ ] host 网段收窄，公网不可达
- [ ] 所有 SQL 参数化，${} 白名单
- [ ] PII 字段加密存储、展示脱敏
