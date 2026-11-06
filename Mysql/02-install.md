# 02 - 安装与配置

## 1. Docker 部署（开发首选）

```bash
docker run -d --name mysql8 \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -e MYSQL_DATABASE=demo \
  -e TZ=Asia/Shanghai \
  -p 3306:3306 \
  -v mysql8-data:/var/lib/mysql \
  -v /path/conf/my.cnf:/etc/mysql/conf.d/my.cnf:ro \
  mysql:8.0.34
```

注意：
- 数据必须挂卷持久化（容器删了数据就没了）
- 时区必须指定，否则容器内 UTC，DATETIME 写入偏差 8 小时

## 2. macOS / Linux 原生安装

```bash
# macOS
brew install mysql@8.0
brew services start mysql@8.0

# Ubuntu
sudo apt install mysql-server
sudo systemctl enable --now mysql
sudo mysql_secure_installation
```

## 3. 核心配置 my.cnf（最小可用模板）

```ini
[mysqld]
# 标识
server_id = 1
port = 3306
character_set_server = utf8mb4
collation_server = utf8mb4_0900_ai_ci

# 存储
datadir = /var/lib/mysql
# InnoDB（详见 31）
innodb_buffer_pool_size = 4G        # 专机建议物理内存 50%~70%
innodb_redo_log_capacity = 1G       # 8.0.30+ 取代 innodb_log_file_size
innodb_flush_log_at_trx_commit = 1  # 生产用 1（默认），1=每次提交刷盘
sync_binlog = 1                     # 与上一条组成"双1"

# 连接
max_connections = 500
wait_timeout = 600

# 日志
log_error = /var/log/mysql/error.log
slow_query_log = ON
long_query_time = 1
# binlog（8.0 默认开启）
log_bin = mysql-bin
binlog_format = ROW
binlog_expire_logs_seconds = 604800 # 7天

[client]
default-character-set = utf8mb4
```

配置优先级：命令行参数 > 配置文件 `!includedir` 中后者覆盖前者 > 系统默认值。
查看实际生效：`SHOW VARIABLES LIKE 'innodb_buffer_pool_size';`

## 4. 目录结构

| 路径 | 内容 |
|---|---|
| datadir/ibdata1 | 系统表空间（8.0 中仅存元数据字典等） |
| datadir/库名/表名.ibd | 每表独立表空间（innodb_file_per_table 默认 ON） |
| datadir/*-bin.000001 | binlog |
| datadir/ib_logfile*（8.0.30 前） | redo log；8.0.30+ 为 `#innodb_redo/` |

## 5. 首次登录与初始化

```bash
mysql -uroot -p
# 创建业务账号（见 37 安全篇，勿直接用 root 跑业务）
CREATE USER 'app'@'%' IDENTIFIED BY 'StrongPass#2026';
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app'@'%';
```

## 常见坑

1. **3306 公网暴露**：默认 bind-address 要绑内网，配合安全组/SSH 隧道
2. **文件句柄不足**：`ulimit -n` 与 `open_files_limit`，报 `Too many open files`
3. **磁盘满导致库锁死**：监控数据盘，binlog 过期清理要配置
4. 修改密码后忘 `FLUSH PRIVILEGES`？——`CREATE USER/GRANT` 语句本身即时生效，只有手动改授权表才需要

## Checklist

- [ ] Docker 部署数据已挂卷、时区已设置
- [ ] my.cnf 已配置 utf8mb4、慢查询日志、binlog 过期
- [ ] 业务应用未使用 root 账号
