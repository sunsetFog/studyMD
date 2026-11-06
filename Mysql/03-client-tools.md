# 03 - 客户端工具与连接管理

## 1. CLI：mysql

```bash
mysql -h 127.0.0.1 -P 3306 -u app -p appdb

# 常用操作
SHOW DATABASES; USE appdb; SHOW TABLES;
SHOW CREATE TABLE orders\G          # \G 纵向输出，看长结构必备
SELECT ... \G
status;                             # 连接与版本信息
tee /tmp/session.log;               # 记录本次会话所有输出
source /tmp/init.sql;               # 执行脚本
```

导入大数据：`mysql -uapp -p appdb < dump.sql`，超时加 `--max-allowed-packet=512M`。

## 2. GUI 选型

| 工具 | 特点 | 适用 |
|---|---|---|
| DataGrip | 功能最强、收费 | 日常重度 |
| DBeaver | 免费开源、跨库 | 通用 |
| Navicat | 稳定收费 | 通用 |
| MySQL Workbench | 官方免费、建模 | 偶尔用 |
| TablePlus | 轻快 | macOS |

## 3. 连接的底层：会话与线程

- 每个客户端连接 = mysqld 的一个线程（旧版每连接一线程；企业版有线程池）
- 查看当前连接：

```sql
SHOW PROCESSLIST;          -- 或 SELECT * FROM performance_schema.threads
-- Id User Host db Command Time State Info
```

- `Command=Sleep` 表示空闲连接；超过 `wait_timeout`（默认 8h）被服务端踢掉

## 4. 连接数相关的三个参数

| 参数 | 含义 | 经验值 |
|---|---|---|
| max_connections | 最大连接数 | 物理机 500~1000，规格小则低 |
| wait_timeout | 空闲多久被断开 | 600~3600 |
| max_connect_errors | 主机连续失败次数（防爆破） | 遇 `Host blocked` 时 FLUSH HOSTS |

**重要**：应用层连接池大小 ≠ max_connections。业务并发 100 的服务，连接池 10~20 就够（连接昂贵，参见 HikariCP 经验公式：`cores*2 + 磁盘数`）。连接池最大值之和要小于 max_connections。

## 5. 长连接与断线重连

- 应用必须用连接池，并配置连接有效性检测（HikariCP `maxLifetime` 小于服务端 `wait_timeout`）
- MySQL 没有 ping 接口，用 `SELECT 1` 校验
- MySQL 8 的 `useSSL=false` 等连接串参数按驱动版本调整（JDBC 8.x：`serverTimezone=Asia/Shanghai`）

## 常见坑

1. `Communications link failure` → 服务端超时踢连接，连接池 lifetime 未配置
2. `Host 'x.x.x.x' is blocked because of many connection errors` → 网络抖动重试过多，`FLUSH HOSTS` 或调 max_connect_errors
3. GUI 工具长期挂着大量 Sleep 连接占满 max_connections → 设置 GUI 空闲断开
4. JDBC 不写 `serverTimezone` 导致时间差 8 小时（老驱动）

## Checklist

- [ ] 应用使用连接池且 maxLifetime < wait_timeout
- [ ] 连接池大小远小于 max_connections
- [ ] 排查问题熟练使用 SHOW PROCESSLIST
