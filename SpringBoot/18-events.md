# 18 · 事件机制：解耦的进程内总线

## 一、三要素

```
发布者 → ApplicationEventPublisher → 事件对象 → @EventListener 监听者
```

**适用场景**：同一进程内「一件事发生后，多个不相关模块需要响应」——如订单创建后：发短信、加分、清缓存。比直接注入对方 Service 解耦。

## 二、定义事件

```java
public record OrderCreatedEvent(Long orderId, Long userId, BigDecimal amount) {
}
```

## 三、发布事件

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final ApplicationEventPublisher publisher;

    @Transactional(rollbackFor = Exception.class)
    public Long createOrder(OrderCreateDTO dto) {
        Long orderId = doCreate(dto);
        publisher.publishEvent(new OrderCreatedEvent(orderId, dto.getUserId(), dto.getAmount()));
        return orderId;
    }
}
```

## 四、监听事件

```java
@Component
@Slf4j
public class OrderEventListener {

    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        log.info("订单 {} 创建，发短信给 {}", event.orderId(), event.userId());
    }

    // 监听多个事件
    @EventListener({OrderCreatedEvent.class, OrderPaidEvent.class})
    public void refreshStats(Object event) { }

    // 条件监听
    @EventListener(condition = "#event.amount > 1000")
    public void onBigOrder(OrderCreatedEvent event) { }

    // 返回值作为新事件（链式）
    @EventListener
    public PointEvent onOrderCreatedToPoint(OrderCreatedEvent event) {
        return new PointEvent(event.userId(), event.amount().intValue());
    }
}
```

## 五、事务绑定（关键：默认是同步且事务内的！）

默认 `@EventListener` **同步执行且在当前事务内**——监听器里如果做 RPC，会拖进大事务；事务回滚，监听器已做的副作用无法回滚。

```java
// 事务提交后才执行（最常用）
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderCreated(OrderCreatedEvent event) {
    smsClient.send(...);     // 保证只有订单真落库才发短信
}

// BEFORE_COMMIT / AFTER_ROLLBACK / AFTER_COMPLETION
```

## 六、异步监听

```java
@SpringBootApplication
@EnableAsync      // 开启异步支持
public class App { }

@Async
@EventListener
public void onOrderCreated(OrderCreatedEvent event) {
    // 在线程池中执行（见 19 篇线程池配置）
}
```

## 七、事件 vs MQ 对比决策

| 维度 | Spring Event | MQ |
|---|---|---|
| 范围 | 单 JVM 进程内 | 跨服务/跨进程 |
| 可靠性 | 进程崩丢消息 | 持久化/重试 |
| 顺序 | 同步有序/异步无序 | 分区有序 |
| 适用 | 单体内部解耦 | 微服务集成/削峰 |

**规则**：单体内解耦用 Event；需要跨进程可靠传递用 MQ。单体转微服务时 Event 可平滑替换为 MQ（事件模型不变）。

## 八、内置事件（可监听做启动钩子）

```java
@EventListener
public void onReady(ApplicationReadyEvent event) {
    log.info("服务就绪");        // 比 ApplicationRunner 更晚，全链路完成
}

@EventListener
public void onFail(ApplicationFailedEvent event) { }
```

## 常见坑

- **坑 1**：不知道监听器默认同步且在事务内 → 事务里做重活/事务回滚副作用残留；写事务相关监听一律 `@TransactionalEventListener(AFTER_COMMIT)`
- **坑 2**：@Async 监听器抛异常没人管 → 自定义 AsyncUncaughtExceptionHandler（19 篇）
- **坑 3**：事件对象是可变对象，异步下被监听方修改 → 事件用 record/不可变
- **坑 4**：监听器之间顺序依赖 → @Order 明确顺序，或拆分事件

## Checklist

- [ ] 事件对象不可变（record）
- [ ] 副作用型监听器统一 AFTER_COMMIT
- [ ] 异步监听有独立线程池与异常处理
- [ ] 明确 Event（进程内）与 MQ（跨进程）的边界
