# 18 · 事务深度剖析：传播、隔离与失效场景

## 1. ACID 与隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | MySQL 默认 |
|---|---|---|---|---|
| `READ_UNCOMMITTED` | ❌有 | 有 | 有 | |
| `READ_COMMITTED` (RC) | 无 | 有 | 有 | Oracle/PG 默认 |
| `REPEATABLE_READ` (RR) | 无 | 无 | **基本解决**（MVCC+间隙锁） | ✅ MySQL InnoDB 默认 |
| `SERIALIZABLE` | 无 | 无 | 无 | 性能最差 |

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
```

> Spring 的 `Isolation.DEFAULT` = 使用数据库默认。MySQL 默认 RR，一般无需修改；高并发下对一致性要求低的查询可设 RC 减少锁范围。

## 2. 七大传播行为

| 传播行为 | 当前有事务 | 当前无事务 |
|---|---|---|
| `REQUIRED`（默认） | 加入 | 新建 |
| `SUPPORTS` | 加入 | 非事务 |
| `MANDATORY` | 加入 | **抛异常** |
| `REQUIRES_NEW` | 挂起并新建 | 新建 |
| `NOT_SUPPORTED` | 挂起，非事务执行 | 非事务 |
| `NEVER` | **抛异常** | 非事务 |
| `NESTED` | 嵌套（保存点） | 新建 |

```java
@Service
public class OrderService {
    @Transactional
    public void create() {
        dao.insertOrder();
        try {
            logService.saveLog();          // REQUIRES_NEW：独立事务
        } catch (Exception e) {
            log.warn("日志失败不影响主流程");
        }
        // 主事务继续
    }
}

@Service class LogService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog() { ... }
}
```

### REQUIRES_NEW 的注意事项

- 会**新开一个数据库连接**（原连接被挂起），连接池压力翻倍。
- 与主事务**互不影响**：主事务回滚不影响它提交。
- 死锁风险：两个事务可能互相等待对方持有的锁（尤其是更新同一行）。

### NESTED

- 使用 JDBC **保存点（Savepoint）**，外层回滚则内层一起回滚；内层回滚不影响外层。
- 与 `REQUIRES_NEW` 区别：不新开连接，是同一物理事务。

## 3. 事务失效的 10 种场景（高频）

```java
// ① 方法非 public
@Transactional
void create() {}          // ★ private/protected/package 都不生效（CGLIB 无法覆写）

// ② final / static 方法
@Transactional public final void create() {}   // 无法代理

// ③ 同类自调用
public void a() { this.b(); }
@Transactional public void b() {}             // 不走代理

// ④ 异常被吞
@Transactional public void a() {
    try { dao.insert(); } catch (Exception e) { log.error("", e); }   // 提交！
}

// ⑤ 抛出检查异常
@Transactional public void a() throws IOException { throw new IOException(); }  // 不回滚
// 正解：@Transactional(rollbackFor = Exception.class)

// ⑥ 数据库引擎不支持
// MySQL MyISAM 不支持事务

// ⑦ 未被 Spring 管理
new OrderService().create();       // 自己 new 的对象无代理

// ⑧ 多线程
@Transactional public void a() {
    new Thread(() -> dao.insert()).start();    // 新线程无事务上下文
}

// ⑨ 传播行为配置错误
@Transactional(propagation = Propagation.NOT_SUPPORTED)  // 挂起事务

// ⑩ 错误捕获后又设置 rollbackOnly → UnexpectedRollbackException
```

### 其他隐蔽场景

- `@Transactional` 标注在 **接口** 上但用 **CGLIB 代理**：注解不会被继承读取 → 应标注在实现类上（JDK 代理读接口注解会生效，但不推荐依赖）。
- 事务方法内启动新线程 / `@Async` → 事务上下文基于 ThreadLocal，跨线程丢失。
- 事务方法调用**远程接口**耗时 → 长事务占用连接。

## 4. 自调用解决方案

```java
// 方案1：注入自身（@Lazy 打破循环依赖）
@Service
public class OrderService {
    @Lazy @Autowired private OrderService self;
    public void a() { self.b(); }
    @Transactional public void b() {}
}

// 方案2：AopContext
@EnableAspectJAutoProxy(exposeProxy = true)
((OrderService) AopContext.currentProxy()).b();

// 方案3：拆分为两个 Service（推荐）
```

## 5. 事务拦截器执行流程

```
TransactionInterceptor#invoke
  → createTransactionIfNecessary（获取连接、关闭 autoCommit、设置隔离级别）
  → proceed（执行业务）
  → completeTransactionAfterThrowing（rollback）
     或 commitTransactionAfterReturning（commit）
  → cleanupTransactionInfo（恢复上一事务状态、释放连接）
```

## 6. 只读事务与超时

```java
@Transactional(readOnly = true, timeout = 10)
```
- `readOnly=true`：Spring 会设置 `Connection#setReadOnly(true)`，MySQL 驱动可能路由到从库；JPA 会跳过 flush。
- `timeout`：从事务开始计时，超时抛 `TransactionTimedOutException` 并回滚。默认 `-1`（用数据库默认）。

## 7. 大事务的危害与拆分

| 危害 | 说明 |
|---|---|
| 连接池耗尽 | 长时间占用连接 |
| 锁竞争 | 行锁/间隙锁长期持有 |
| 回滚代价大 | undo log 膨胀 |
| 主从延迟 | 大事务 binlog 传输慢 |

拆分原则：
1. 事务内**只做 DB 操作**，远程调用、消息发送、文件操作移出。
2. 查询操作放到事务外。
3. 批量操作分批提交（每 500 条一批）。
4. 耗时操作用 `TransactionSynchronization#afterCommit`。

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderDTO dto) {
    Long id = dao.insert(dto);            // 仅 DB 操作在事务内
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override public void afterCommit() {
            mq.send(new OrderCreatedEvent(id));    // 提交后再发消息
            cache.evict(id);
        }
    });
}
```

## 8. 分布式事务

本地事务无法跨库/跨服务，方案见 [47](47-distributed-tx-seata.md)：2PC（Seata AT）、TCC、Saga、本地消息表、事务消息。

## 9. 常见坑汇总

| 现象 | 根因 | 解决 |
|---|---|---|
| 数据不一致但无异常 | 吞异常 / 检查异常 | 抛异常或 `rollbackFor` |
| `UnexpectedRollbackException` | 内层标记 rollbackOnly，外层却提交 | 内外层传播行为统一；内层异常上抛 |
| 连接池满 | `REQUIRES_NEW` 滥用 / 大事务 | 减少新事务、拆分 |
| 死锁 | 加锁顺序不一致 | 统一加锁顺序；减小事务粒度 |
| 锁等待超时 | 长事务 | 拆事务、加索引避免锁升级 |
| 只读事务写失败 | 路由到从库 | 只读事务不写 |
| 事务不回滚但日志有异常 | 异常在子线程/被 catch | 检查调用链 |

## 10. 面试速答

- **事务失效原因**：非 public、final/static、自调用、异常被吞、检查异常、引擎不支持、未被容器管理、跨线程。
- **REQUIRED 与 REQUIRES_NEW**：前者加入当前事务（同生共死），后者挂起当前、新建独立事务（互不影响，会多占连接）。
- **NESTED**：基于保存点，外层回滚内层必回滚，内层可单独回滚，不新开连接。
- **长事务怎么优化**：事务内只留 DB 操作、异步化、分批提交、避免远程调用。
