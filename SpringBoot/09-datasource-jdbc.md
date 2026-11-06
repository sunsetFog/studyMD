# 09 · 数据源与 JdbcTemplate

## 一、连接池配置（HikariCP 默认）

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo?useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver   # Boot 3 可省略（自动推断）
    hikari:
      minimum-idle: 5            # 最小空闲连接
      maximum-pool-size: 20      # 最大连接数
      idle-timeout: 600000       # 空闲超时 10min
      max-lifetime: 1800000      # 连接最大存活（须小于 MySQL wait_timeout）
      connection-timeout: 30000  # 获取连接超时
      pool-name: DemoHikariPool
```

**连接池大小经验公式**：`connections = 核心数 * 2 + 有效磁盘数`，一般 10-20 足够，不是越大越好。

## 二、多数据源

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.second")
    public DataSource secondDataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

配合 `@MapperScan` 按包指定 SqlSessionFactory，或基于 AbstractRoutingDataSource 实现读写分离（框架场景建议直接用 dynamic-datasource-spring-boot-starter）。

## 三、JdbcTemplate

```java
@Service
@RequiredArgsConstructor
public class UserDao {

    private final JdbcTemplate jdbcTemplate;

    public User findById(Long id) {
        return jdbcTemplate.queryForObject(
                "SELECT id, username, age FROM users WHERE id = ?",
                (rs, rowNum) -> {
                    User u = new User();
                    u.setId(rs.getLong("id"));
                    u.setUsername(rs.getString("username"));
                    u.setAge(rs.getInt("age"));
                    return u;
                }, id);
    }

    public int updateAge(Long id, int age) {
        return jdbcTemplate.update("UPDATE users SET age = ? WHERE id = ?", age, id);
    }

    // 批量插入（性能关键：一次网络往返）
    public int[] batchInsert(List<User> users) {
        return jdbcTemplate.batchInsert(
                "INSERT INTO users(username, age) VALUES (?, ?)",
                users,
                1000,   // 批大小
                (ps, u) -> {
                    ps.setString(1, u.getUsername());
                    ps.setInt(2, u.getAge());
                });
    }
}
```

## 四、命名参数模板

```java
private final NamedParameterJdbcTemplate npTemplate;

Map<String, Object> params = Map.of("name", "tom", "minAge", 18);
List<User> list = npTemplate.query(
        "SELECT * FROM users WHERE username = :name AND age >= :minAge",
        params, userRowMapper);
```

## 五、MyBatis vs JPA vs JdbcTemplate 选型

| 维度 | MyBatis(-Plus) | JPA | JdbcTemplate |
|---|---|---|---|
| SQL 控制力 | 强 | 弱（自动生成） | 最强 |
| 开发效率 | 高 | 最高 | 低 |
| 复杂查询 | 好 | 差 | 好 |
| 国内主流度 | 第一 | 中 | 补充 |
| 适用 | 绝大多数业务 | 简单 CRUD 域模型 | 特殊场景/工具类 |

国内生产环境主力 **MyBatis-Plus**，本文档以它为主线（10/11 篇），JPA 作为了解（12 篇）。

## 六、数据库连接 URL 安全清单

- 生产禁用 `useSSL=false`（除非内网）
- 必须指定 `serverTimezone`，避免时间偏移
- 密码走环境变量/配置中心，不进代码库

## 常见坑

- **坑 1**：`HikariPool-1 - Connection is not available` → 连接池耗尽：慢 SQL 或连接泄漏（未释放）
- **坑 2**：max-lifetime > MySQL `wait_timeout` → 死连接报 `CommunicationsException`
- **坑 3**：批量插入一条条 insert → 性能差；用 batchInsert 或 `rewriteBatchedStatements=true`
- **坑 4**：`queryForObject` 查不到数据抛 EmptyResultDataAccessException → 用 `query().stream().findFirst()` 或捕获

## Checklist

- [ ] Hikari 参数按生产规模调整，max-lifetime < MySQL 超时
- [ ] 批量操作使用 batch 能力
- [ ] 多数据源有明确路由方案
- [ ] 慢 SQL 有监控（慢查询日志/Druid 监控）
