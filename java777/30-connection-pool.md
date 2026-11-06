# 30 · 数据库连接池：HikariCP 与 Druid

## 1. 为什么需要连接池

创建 TCP 连接成本很高（三次握手 + 认证 + 初始化）：约 5~20ms。池化后复用连接，避免频繁创建销毁。

| 连接池 | 特点 |
|---|---|
| **HikariCP** | Spring Boot 2+ 默认，**最快**（字节码级优化、FastList、无锁 ConcurrentBag） |
| Druid | 阿里出品，功能丰富（**监控、防 SQL 注入、SQL 防火墙、加解密**） |
| Tomcat JDBC | Tomcat 自带 |
| C3P0 / DBCP | 老旧，不推荐 |

> 选型：追求性能用 HikariCP；需要**内置监控页面/慢 SQL 统计/密码加密**用 Druid。

## 2. HikariCP 配置

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      pool-name: MainPool
      minimum-idle: 5                # 最小空闲（默认 = maximum-pool-size）
      maximum-pool-size: 20          # 最大连接
      auto-commit: true
      idle-timeout: 600000           # 空闲连接超时（10min，仅 > minimum-idle 时生效）
      max-lifetime: 1800000          # 连接最大存活（30min，必须 < MySQL wait_timeout）
      connection-timeout: 30000      # ★ 获取连接超时（30s），超时抛 SQLTransientConnectionException
      validation-timeout: 5000
      leak-detection-threshold: 60000   # ★ 连接泄漏检测（>0 生效，生产建议 60s+）
      connection-test-query: SELECT 1   # 驱动支持 ping 时可不配
      keepalive-time: 0
```

## 3. 池大小怎么定

HikariCP 官方建议：

```
connections = ((core_count * 2) + effective_spindle_count)
```

实践经验：
- 一般 Web 应用 **10~30** 足够；**不是越大越好**。
- 连接数过多会导致数据库上下文切换、锁竞争加剧，反而更慢。
- 若 SQL 平均耗时 20ms，单连接理论 QPS = 50；目标 1000 QPS → 需要 20 连接。
- 有长事务/慢 SQL 时，先优化 SQL 而不是加连接。

监控指标：`hikaricp.connections.pending`（等待连接的线程数）> 0 说明池不够。

## 4. 关键参数原理

| 参数 | 说明 |
|---|---|
| `max-lifetime` | 连接最长存活时间，到期被回收重建。必须**小于**数据库 `wait_timeout`（MySQL 默认 28800s），建议 30 分钟内 |
| `idle-timeout` | 空闲多久被回收（仅当空闲数 > minimum-idle） |
| `connection-timeout` | 从池取连接的最长等待，超时即失败（快速失败优于阻塞） |
| `minimum-idle` | 保持的最小空闲连接，设太小会在突发流量时频繁建连 |
| `leak-detection-threshold` | 借出超过该时间未归还则打 WARN 日志（**排查连接泄漏神器**） |

## 5. 连接泄漏排查

```
WARN HikariPool-1 - Apparent connection leak detected ...
```

常见原因：
- 手动获取 `Connection` 未 `close()`（必须 `try-with-resources`）。
- 事务未结束（长事务、远程调用卡住）。
- `SqlSession` 未关闭（MyBatis 手动 openSession）。
- 流式查询未消费完就返回。

```java
// 正确姿势
try (Connection c = dataSource.getConnection();
     PreparedStatement ps = c.prepareStatement(sql)) { ... }   // 自动归还
```

## 6. Druid

```xml
<dependency><groupId>com.alibaba</groupId><artifactId>druid-spring-boot-3-starter</artifactId>
           <version>1.2.23</version></dependency>
```

```yaml
spring:
  datasource:
    druid:
      url: jdbc:mysql://...
      username: root
      password: ${ENCRYPTED}      # 配合 config.decrypt=true
      initial-size: 5
      min-idle: 5
      max-active: 20
      max-wait: 30000
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000
      validation-query: SELECT 1
      test-while-idle: true
      test-on-borrow: false
      filters: stat,wall,slf4j
      filter:
        stat:
          slow-sql-millis: 1000
          log-slow-sql: true
        wall:
          config:
            multi-statement-allow: true
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
        login-username: admin
        login-password: admin       # ★ 生产务必改，且不要暴露公网
      web-stat-filter:
        enabled: true
```

Druid 独有：`/druid/index.html` 监控页、SQL 统计、URI 监控、Session 监控、Spring 监控、密码加密（`ConfigTools.encrypt`）。

> HikariCP + Prometheus 也能拿到 `hikaricp.*` 指标，不必为监控硬上 Druid。

## 7. MySQL 侧配合

```sql
-- 连接相关
SHOW VARIABLES LIKE 'max_connections';      -- 默认 151，注意多实例总和
SHOW STATUS LIKE 'Threads_connected';
SHOW PROCESSLIST;                            -- 查看当前连接与慢 SQL
SET GLOBAL wait_timeout = 600;               -- 空闲连接超时
```

- 应用实例数 × 池大小 < `max_connections`（留余量给管理连接）。
- 长连接被 MySQL 服务端断开会导致 `Communications link failure`，靠 `max-lifetime` 与健康检查规避。

## 8. 慢 SQL 与监控

```yaml
# p6spy（打印真实 SQL 与耗时，开发用）
<dependency><groupId>p6spy</groupId><artifactId>p6spy</artifactId><version>3.9.1</version></dependency>
```

生产推荐：MySQL slow query log + Prometheus + Grafana；或 `micrometer` 的 `hikaricp.*`、`spring.data.repository.invocations`。

关键指标：
| 指标 | 告警线 |
|---|---|
| `hikaricp.connections.pending` | > 0 持续 |
| `hikaricp.connections.usage` | > 0.8 |
| `hikaricp.connections.timeout` | 出现即告警 |
| 慢 SQL 数 | 突增 |

## 9. 多数据源

```java
@Configuration
public class MultiDataSourceConfig {
    @Bean @Primary
    @ConfigurationProperties("spring.datasource.master")
    public DataSourceProperties masterProps() { return new DataSourceProperties(); }

    @Bean @Primary
    public DataSource masterDataSource() {
        return masterProps().initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.slave")
    public DataSource slaveDataSource() { return DataSourceBuilder.create().build(); }
}
```

> 多数据源 + 事务：每个数据源需要各自的 `DataSourceTransactionManager`（用 `@Bean` 并指定 `@Qualifier`），或 `ChainedTransactionManager`（已废弃，慎用）。跨库事务见 [47](47-distributed-tx-seata.md)。

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| `Connection is not available, request timed out` | 池耗尽 | 查慢 SQL/长事务；适度调大池；缩短 connection-timeout 快速失败 |
| `Communications link failure` | 连接被服务端断开 | `max-lifetime` < `wait_timeout`；开 keepalive |
| `Too many connections` | 超过 MySQL 上限 | 减少实例池大小或调大 `max_connections` |
| 连接泄漏 | pending 持续增长 | 开 `leak-detection-threshold`；检查未关闭的 Connection |
| 池过大反而慢 | CPU 飙高 | 减到合理值 |
| Druid 监控页暴露 | 安全事件 | 改密码、限制 IP、生产关闭 |
| 密码明文 | 泄露 | Druid 加密或配置中心/Secrets |

## 11. 面试速答

- **HikariCP 为什么快**：精简代码（字节码级优化）、`FastList` 替代 ArrayList 减少边界检查、`ConcurrentBag` 无锁借还、避免不必要的代理与同步。
- **连接池大小如何设置**：不是越大越好；公式 `(CPU核数 × 2) + 磁盘数`，实践中 10~30，以 `pending` 指标为准。
- **`max-lifetime` 为什么要小于数据库 `wait_timeout`**：避免连接被服务端单方面断开后应用还在使用，导致 `Communications link failure`。
