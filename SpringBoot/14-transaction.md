# 14 · 事务管理全解

## 一、基础用法

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderMapper orderMapper;
    private final StockMapper stockMapper;

    @Transactional(rollbackFor = Exception.class)   // 推荐显式写 rollbackFor
    public void createOrder(OrderCreateDTO dto) {
        orderMapper.insert(buildOrder(dto));
        int rows = stockMapper.deduct(dto.getSkuId(), dto.getCount());
        if (rows == 0) {
            throw new BusinessException(30001, "库存不足");   // RuntimeException 自动回滚
        }
    }
}
```

- 默认只回滚 `RuntimeException` 和 `Error`；**显式 `rollbackFor = Exception.class` 防 checked 异常漏网**
- @Transactional 加在**实现类的方法**上（public 方法），Service 层而非 Controller 层

## 二、传播机制（propagation）

| 传播行为 | 语义 | 典型场景 |
|---|---|---|
| REQUIRED（默认） | 有事务加入，没有新建 | 绝大多数场景 |
| REQUIRES_NEW | 挂起当前事务，新开独立事务 | **记录操作日志（主事务回滚日志也要留）** |
| NESTED | 嵌套事务，Savepoint 可部分回滚 | 批量处理允许部分失败 |
| SUPPORTS | 有则加入，没有非事务运行 | 查询 |
| NOT_SUPPORTED | 挂起事务，非事务执行 | 大批量读取 |
| MANDATORY / NEVER | 强制要求/强制无事务 | 防御性 API |

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void writeAuditLog(String action) { ... }   // 主流程回滚，日志保留
```

## 三、隔离级别与只读

```java
@Transactional(isolation = Isolation.READ_COMMITTED)  // MySQL 默认 REPEATABLE_READ
@Transactional(readOnly = true)   // 查询方法标注，驱动层/ORM 可优化
```

## 四、@Transactional 失效六大场景（面试与实战双高频）

### 1. 同类内自调用

```java
public void methodA() {
    methodB();        // this.methodB()，未走代理 → 事务失效！
}

@Transactional
public void methodB() { ... }

// 修复：注入自身代理
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderService self;   // 或 AopContext.currentProxy()（需 exposeProxy）
    public void methodA() { self.methodB(); }
}
```

### 2. 方法非 public
事务拦截器只处理 public 方法。

### 3. 异常被 catch 吞掉

```java
@Transactional
public void update() {
    try {
        doUpdate();
    } catch (Exception e) {
        log.error(...);   // 异常没抛出 → 不回滚！
    }
}
// 修复：catch 后 throw，或 TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()
```

### 4. 抛出 checked 异常
默认不回滚 → 写 `rollbackFor = Exception.class`。

### 5. 类没被 Spring 管理
忘了 @Service → 无代理无事务。

### 6. 数据库引擎不支持
MyISAM 不支持事务 → 用 InnoDB。

**额外场景**：多线程（子线程不在同一事务上下文）、propagation 配 NOT_SUPPORTED、finally 中 return（吞异常）。

## 五、大事务优化

大事务的危害：连接长时间占用、锁范围大、undo log 膨胀。

```java
// 反例：RPC/IO 在事务内
@Transactional
public void create(OrderDTO dto) {
    orderMapper.insert(order);
    smsClient.send(...);            // 外部调用放事务内 → 大事务
    fileService.upload(...);
}

// 正例：缩小事务范围，事务只包 DB 操作
public void create(OrderDTO dto) {
    Long orderId = self.saveOrderWithTx(dto);   // 事务只含 DB
    smsClient.send(...);                        // 事务外
}
```

辅助手段：
- `@Transactional(readOnly = true)` 标注查询
- 事务内只做 DB 写，消息发 MQ 用 `TransactionSynchronizationManager` 事务提交后发送，或本地消息表

## 六、事务与缓存/MQ 一致性

```java
// 事务提交后再删缓存
@Transactional
public void update(UserDTO dto) {
    userMapper.updateById(...);
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            redisTemplate.delete("user:" + dto.getId());
        }
    });
}
```

## 常见坑

- **坑 1**：自调用失效（最高频）→ self 注入或拆类
- **坑 2**：rollbackFor 缺失 + checked 异常 → 数据不一致
- **坑 3**：@Transactional 加在类上导致所有方法（含查询）都开事务 → 连接占用上升
- **坑 4**：多数据源下 @Transactional 用错事务管理器 → 指定 transactionManager 名
- **坑 5**：嵌套 REQUIRES_NEW 引起连接池耗尽（外层连接被挂起占用）→ 控制嵌套深度

## Checklist

- [ ] 写事务统一 `@Transactional(rollbackFor = Exception.class)`
- [ ] 无同类自调用（code review 重点检查）
- [ ] 事务内无 RPC/IO，大事务已拆分
- [ ] 查询方法 readOnly 或不加事务
