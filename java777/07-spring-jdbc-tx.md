# 07 · JdbcTemplate 与 Spring 事务抽象

## 1. 为什么还需要 JDBC 抽象

原生 JDBC 样板代码多、异常是受检 `SQLException`、连接管理易泄漏。Spring 提供：

- `JdbcTemplate`：模板方法封装「获取连接→创建语句→执行→关闭资源→异常处理」。
- `DataSourceUtils`：与事务同步的连接获取。
- 统一的 `DataAccessException` 异常体系（**非受检**，跨数据库可移植）。

## 2. JdbcTemplate 基本用法

```java
@Repository
@RequiredArgsConstructor
public class UserDao {
    private final JdbcTemplate jdbc;

    public User findById(Long id) {
        return jdbc.queryForObject(
            "select id,name,age from t_user where id=?",
            new BeanPropertyRowMapper<>(User.class), id);
    }

    public List<User> findAll() {
        return jdbc.query("select * from t_user", new BeanPropertyRowMapper<>(User.class));
    }

    public int insert(User u) {
        return jdbc.update("insert into t_user(name,age) values(?,?)", u.getName(), u.getAge());
    }

    // 主键回填
    public Long insertAndGetKey(User u) {
        KeyHolder kh = new GeneratedKeyHolder();
        jdbc.update(con -> {
            PreparedStatement ps = con.prepareStatement(
                "insert into t_user(name) values(?)", Statement.RETURN_GENERATED_KEYS);
            ps.setString(1, u.getName());
            return ps;
        }, kh);
        return kh.getKey().longValue();
    }

    // 批量
    public void batchInsert(List<User> list) {
        jdbc.batchUpdate("insert into t_user(name,age) values(?,?)",
            new BatchPreparedStatementSetter() {
                public void setValues(PreparedStatement ps, int i) {
                    ps.setString(1, list.get(i).getName());
                    ps.setInt(2, list.get(i).getAge());
                }
                public int getBatchSize() { return list.size(); }
            });
    }

    // NamedParameterJdbcTemplate：命名参数，可读性更好
    public int update(MapSqlParameterSource params) {
        return namedJdbc.update("update t_user set name=:name where id=:id", params);
    }
}
```

> `JdbcTemplate` 是**线程安全**的（无状态），可以单例注入。

## 3. 事务核心接口

```
PlatformTransactionManager   事务管理器（策略接口）
   ├── DataSourceTransactionManager      JDBC / MyBatis
   ├── JpaTransactionManager             JPA
   ├── JtaTransactionManager             分布式 JTA
   └── RabbitTransactionManager 等
TransactionDefinition        传播行为、隔离级别、超时、只读
TransactionStatus            事务运行状态（是否新事务、是否有保存点）
```

```java
// 编程式事务（不常用，但某些场景必需，如大批量分批处理）
@Autowired PlatformTransactionManager txManager;
@Autowired TransactionTemplate txTemplate;

txTemplate.execute(status -> {
    dao.insert(a);
    dao.insert(b);
    return null;      // 抛异常自动回滚
});

// 手动回滚
txTemplate.execute(status -> {
    try { doSomething(); }
    catch (BizException e) { status.setRollbackOnly(); }
    return null;
});
```

## 4. 声明式事务 @Transactional

```java
@Transactional(
    propagation = Propagation.REQUIRED,   // 默认
    isolation = Isolation.DEFAULT,        // 默认用数据库隔离级别
    timeout = 30,                          // 秒，超时回滚
    readOnly = true,                       // 只读优化（MySQL 路由从库）
    rollbackFor = Exception.class,         // ★ 默认只回滚 RuntimeException/Error
    noRollbackFor = IllegalArgumentException.class
)
public void transfer(...) {}
```

> **关键**：`@Transactional` 默认只对 `RuntimeException` 和 `Error` 回滚，检查异常（`IOException` 等）**不回滚**。生产上建议显式写 `rollbackFor = Exception.class`。

## 5. 事务失效的 8 种场景（详见 [18](18-boot-transaction.md)）

| 场景 | 原因 |
|---|---|
| 方法非 public | 代理无法覆写 |
| 同类自调用 | 未走代理 |
| 异常被 catch 掉 | 拦截器感知不到 |
| 抛检查异常 | 默认不回滚 |
| 数据库引擎不支持 | MyISAM 不支持事务 |
| `final`/`static` 方法 | 无法代理 |
| 新线程/异步调用 | 事务上下文绑定 ThreadLocal，跨线程丢失 |
| 未被 Spring 管理 | `new` 出来的对象 |

## 6. 事务同步机制

Spring 事务通过 `TransactionSynchronizationManager`（一堆 `ThreadLocal`）保存：

- 当前连接（`DataSourceUtils` 从此处取，保证同一线程同一连接）
- 当前事务状态、隔离级别、只读标记

```java
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronization() {
        @Override public void afterCommit() {
            // 事务提交后发 MQ / 清缓存 —— 保证一致性
            publisher.publishEvent(new OrderCreatedEvent(id));
        }
    });
```

> **afterCommit 发送 MQ** 是解决「事务未提交消息已消费」的经典方案。

## 7. 只读事务与路由

```java
@Transactional(readOnly = true)
public List<Order> query() {}
```

- Hibernate/JPA：`flush` 模式优化，跳过脏检查。
- MySQL + 读写分离中间件：自动路由到从库。
- 注意：`readOnly=true` 中执行写操作在 MySQL 下不报错但可能被路由到从库导致失败。

## 8. 事务传播行为速查

| 传播行为 | 说明 | 常见用法 |
|---|---|---|
| `REQUIRED`（默认） | 有则加入，无则新建 | 绝大多数 |
| `REQUIRES_NEW` | 挂起当前，另起新事务 | 操作日志（失败不影响主事务） |
| `SUPPORTS` | 有则用，无则非事务 | 查询 |
| `NOT_SUPPORTED` | 挂起事务，非事务执行 | 大批量读取 |
| `MANDATORY` | 必须有事务，否则抛异常 | 强制约束 |
| `NEVER` | 必须无事务，否则抛异常 | 强制约束 |
| `NESTED` | 嵌套事务（保存点），外层回滚内层回滚 | 部分回滚 |

```java
// 日志记录：无论主业务成功与否都要落库
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveLog(LogEntity log) {}
```

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 检查异常不回滚 | 数据不一致 | `rollbackFor = Exception.class` |
| `REQUIRES_NEW` 造成连接翻倍 | 连接池耗尽 | 控制使用，调大连接池 |
| 大事务 | 锁持有时间长、连接池占满 | 拆分、异步化、减小事务粒度 |
| 事务内调远程接口 | 网络抖动导致长事务 | 移到事务外；用 `afterCommit` |
| `readOnly` 下写操作 | 路由到从库失败 | 不要在只读事务里写 |
| 自注入循环依赖 | 启动失败 | 用 `@Lazy` 或事件 |

## 10. 面试速答

- **声明式事务原理**：`@EnableTransactionManagement` 注册 `TransactionInterceptor`（Advice）+ `BeanFactoryTransactionAttributeSourceAdvisor`；代理调用时拦截器开启事务、执行目标、提交/回滚。
- **编程式 vs 声明式**：声明式无侵入、推荐；需要精细控制（分批提交）时用 `TransactionTemplate`。
- **默认回滚规则**：`RuntimeException` / `Error`；检查异常不回滚，除非配置 `rollbackFor`。
