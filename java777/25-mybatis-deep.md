# 25 · MyBatis 深入：执行流程、插件与缓存

## 1. 核心组件

```
SqlSessionFactoryBuilder  →  SqlSessionFactory  →  SqlSession  →  Executor  →  StatementHandler
        （启动时一次）          （单例）           （请求/事务级）
                                       ↓
                          MappedStatement（XML/注解解析后的 SQL 描述）
```

| 组件 | 职责 |
|---|---|
| `Configuration` | 全局配置（所有 MappedStatement、插件、类型处理器） |
| `SqlSession` | 会话（线程不安全），默认实现 `DefaultSqlSession` |
| `Executor` | 执行器：`SimpleExecutor`/`ReuseExecutor`/`BatchExecutor`/`CachingExecutor` |
| `StatementHandler` | 封装 JDBC Statement（含 `RoutingStatementHandler`） |
| `ParameterHandler` | 参数绑定 |
| `ResultSetHandler` | 结果集映射 |
| `TypeHandler` | Java ↔ JDBC 类型转换 |
| `Interceptor` | 插件 |

## 2. 一次查询的完整链路

```
mapper.selectById(1)
  → MapperProxy#invoke（JDK 动态代理，Mapper 接口无实现类）
  → MapperMethod#execute（判断 SQL 类型：SELECT/INSERT/UPDATE/DELETE）
  → SqlSession#selectOne
  → CachingExecutor#query（先查二级缓存）
  → BaseExecutor#query（先查一级缓存 localCache）
  → 查库：SimpleExecutor#doQuery
      → StatementHandler#prepare（预编译）
      → ParameterHandler#setParameters（#{} 赋值）
      → 执行 ResultSetHandler#handleResultSets（映射结果）
  → 结果放入 localCache（一级缓存）
  → 关闭/提交时清空二级缓存暂存区
```

## 3. Mapper 接口为什么不需要实现类

`MapperRegistry` 在 `SqlSession#getMapper` 时用 `MapperProxyFactory` 创建 `MapperProxy`（JDK 代理）。

```java
MapperProxy#invoke → 缓存 MapperMethod → 根据 statementId（namespace + 方法名）找到 MappedStatement → 执行
```

> 因此 Mapper 方法**不能重载**（statementId 会冲突）。

MyBatis-Spring 中 Mapper 通过 `MapperFactoryBean`（`FactoryBean`）注册为 Spring Bean，`@MapperScan` → `MapperScannerRegistrar` → `ClassPathMapperScanner` 批量注册。

## 4. 插件（Interceptor）——四大拦截点

| 接口 | 方法 | 用途 |
|---|---|---|
| `Executor` | `update`/`query`/`commit`/`rollback` | 分页、缓存、SQL 改写 |
| `ParameterHandler` | `setParameters` | 参数加密 |
| `ResultSetHandler` | `handleResultSets` | 结果脱敏 |
| `StatementHandler` | `prepare` | **改写 SQL**（分页、数据权限） |

```java
@Intercepts(@Signature(
    type = StatementHandler.class,
    method = "prepare",
    args = {Connection.class, Integer.class}))
@Component
public class SqlLogInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation inv) throws Throwable {
        StatementHandler sh = (StatementHandler) inv.getTarget();
        BoundSql bs = sh.getBoundSql();
        log.info("SQL: {}", bs.getSql());
        return inv.proceed();          // ★ 必须 proceed
    }
    @Override public Object plugin(Object target) { return Plugin.wrap(target, this); }
    @Override public void setProperties(Properties p) {}
}
```

### 插件原理

`InterceptorChain` 在四大对象创建时用 **JDK 动态代理层层包裹**（责任链）。多个插件顺序 = `Configuration` 中注册顺序，**越后注册的越在外层**。

> 插件是责任链，务必 `inv.proceed()`，否则后续中断。

## 5. 实战：数据权限插件（改写 SQL）

```java
@Intercepts(@Signature(type = StatementHandler.class, method = "prepare",
                       args = {Connection.class, Integer.class}))
public class DataScopeInterceptor implements Interceptor {
    @Override public Object intercept(Invocation inv) throws Throwable {
        StatementHandler sh = (StatementHandler) inv.getTarget();
        MetaObject mo = SystemMetaObject.forObject(sh);
        MappedStatement ms = (MappedStatement) mo.getValue("delegate.mappedStatement");
        if (SqlCommandType.SELECT != ms.getSqlCommandType()) return inv.proceed();

        BoundSql bs = sh.getBoundSql();
        String sql = bs.getSql() + " and dept_id in (" + currentDeptIds() + ")";
        mo.setValue("delegate.boundSql.sql", sql);       // 反射改写
        return inv.proceed();
    }
}
```

> 用 `MetaObject`/`SystemMetaObject` 是 MyBatis 官方推荐访问内部属性的方式（避免硬反射）。

## 6. 一级缓存（SqlSession 级）

- 默认开启，`BaseExecutor.localCache`（PerpetualCache）。
- 命中条件：同一 SqlSession + 同 statementId + 同参数 + 同 SQL + 未执行过增删改 + 未手动清空。
- 增删改会 `clearLocalCache()`；也可 `sqlSession.clearCache()`。

```java
try (SqlSession s = factory.openSession()) {
    UserMapper m = s.getMapper(UserMapper.class);
    m.selectById(1);      // 查库
    m.selectById(1);      // ★ 命中一级缓存
}
```

> Spring 集成后，一级缓存只在**同一事务**内有效（因为 SqlSession 与事务绑定）。无事务时每次 mapper 调用都会新建 SqlSession，一级缓存形同虚设。
> 作用域：`LOCAL_CACHE`（默认）/ `STATEMENT`（每次查询即清空，等于关闭）。

```yaml
mybatis:
  configuration:
    local-cache-scope: statement     # 分布式环境建议这样，避免脏读
```

## 7. 二级缓存（namespace 级）

- 需要 XML 中 `<cache/>` 或 `@CacheNamespace`，且实体可序列化。
- 存储在 `MappedStatement` 所属的 namespace 下，跨 SqlSession 共享。
- **分布式下极易脏读**（各节点缓存不一致）→ **生产建议关闭**。

```yaml
mybatis:
  configuration:
    cache-enabled: false
```

自定义二级缓存（接 Redis）：

```java
@CacheNamespace(implementation = RedisCache.class)   // 实现 org.apache.ibatis.cache.Cache
```

> 一般更推荐用 Spring Cache + Redis（见 [27](27-redis-patterns.md)），而不是 MyBatis 二级缓存。

## 8. 批量执行

```java
// 方式1：BatchExecutor
try (SqlSession s = factory.openSession(ExecutorType.BATCH)) {
    UserMapper m = s.getMapper(UserMapper.class);
    for (int i = 0; i < 1000; i++) {
        m.insert(new User("u" + i));
        if (i % 500 == 0) s.flushStatements();
    }
    s.commit();
}
// 方式2（更常用）：foreach 拼接 + MySQL rewriteBatchedStatements=true
```

## 9. 类型处理器与枚举

```java
public class EnumTypeHandler<E extends Enum<E> & CodeEnum> extends BaseTypeHandler<E> {
    private final Class<E> type;
    public void setNonNullParameter(PreparedStatement ps, int i, E p, JdbcType t) throws SQLException {
        ps.setInt(i, p.getCode());
    }
    public E getNullableResult(ResultSet rs, String col) throws SQLException {
        int code = rs.getInt(col);
        return rs.wasNull() ? null : CodeEnum.of(type, code);
    }
}
```

## 10. 与 Spring 事务的整合

- `SqlSessionTemplate`（线程安全代理）：每次操作从 `SqlSessionHolder`（绑定到事务）取或新建。
- 事务开启 → 创建 SqlSession 并绑定到 `TransactionSynchronizationManager`；事务结束 → 提交/回滚并关闭。
- 因此：**无事务时，两次 mapper 调用是两个 SqlSession、两条连接**（连接池压力）。

## 11. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| Mapper 方法重载 | 启动报 statementId 冲突 | 禁止重载 |
| 插件不生效 | 无拦截 | `@Intercepts` 签名写对；插件注册顺序 |
| 插件改 SQL 导致参数错位 | SQL 报错 | 改写 SQL 同时要同步 `ParameterMapping` |
| 一级缓存脏读 | 同事务内读到旧值 | 改 `local-cache-scope: statement` |
| 二级缓存脏数据 | 分布式不一致 | 关闭，用 Redis |
| `flushStatements` 未调 | 内存堆积 | 分批 flush |
| 大量 `MappedStatement` 内存 | 启动慢、内存大 | 拆分模块 |
| `#{}` 在 `order by` 失效 | 排序无效 | 用 `${}` + 白名单 |

## 12. 面试速答

- **MyBatis 执行流程**：`MapperProxy` → `SqlSession` → `Executor`（一二级缓存）→ `StatementHandler` → `ParameterHandler`/`ResultSetHandler`。
- **一级/二级缓存**：一级是 SqlSession（Spring 下同事务）级，默认开；二级是 namespace 级，需显式开启，分布式下建议关闭。
- **插件能拦截什么**：Executor、StatementHandler、ParameterHandler、ResultSetHandler（分页插件拦截 `prepare` 改 SQL）。
- **Mapper 接口为什么能直接调用**：JDK 动态代理 `MapperProxy` + Spring 的 `MapperFactoryBean`。
