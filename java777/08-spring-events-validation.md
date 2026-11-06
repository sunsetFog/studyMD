# 08 · 事件机制与数据校验

## 1. Spring 事件三要素

```
ApplicationEvent       事件对象（Spring 4.2+ 可以传任意 POJO，无需继承）
ApplicationListener    监听器
ApplicationEventMulticaster  广播器（同步，默认 SimpleApplicationEventMulticaster）
```

```java
// 1. 定义事件（推荐 record 不可变对象）
public record OrderCreatedEvent(Long orderId, Long userId, BigDecimal amount) {}

// 2. 发布
@Service
@RequiredArgsConstructor
public class OrderService {
    private final ApplicationEventPublisher publisher;
    public Long create(Order o) {
        Long id = dao.insert(o);
        publisher.publishEvent(new OrderCreatedEvent(id, o.getUserId(), o.getAmount()));
        return id;
    }
}

// 3. 监听
@Component
@Slf4j
public class CouponListener {
    @EventListener
    public void handle(OrderCreatedEvent e) { log.info("发券 {}", e.orderId()); }
}
```

## 2. 四种监听写法

| 写法 | 说明 |
|---|---|
| `@EventListener` 方法 | 最常用，支持条件、异步 |
| `@TransactionalEventListener` | **事务提交后**才触发 |
| 实现 `ApplicationListener<E>` | 传统方式 |
| `SmartApplicationListener` | 支持顺序、类型判断 |

```java
// 条件监听（SpEL）
@EventListener(condition = "#event.amount.compareTo(new java.math.BigDecimal('1000')) > 0")
public void onBigOrder(OrderCreatedEvent event) { /* 风控 */ }

// 事务提交后（有 phase 可选：BEFORE_COMMIT / AFTER_COMMIT(默认) / AFTER_ROLLBACK / AFTER_COMPLETION）
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void afterCommit(OrderCreatedEvent e) {
    mqSender.send(e);   // 保证数据已落库再发消息
}
```

> `@TransactionalEventListener` 若当前**没有事务**，默认不执行（除非 `fallbackExecution=true`）。

## 3. 异步事件

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean("applicationEventMulticaster")
    public ApplicationEventMulticaster multicaster() {
        SimpleApplicationEventMulticaster m = new SimpleApplicationEventMulticaster();
        m.setTaskExecutor(new SimpleAsyncTaskExecutor("event-"));   // 或自定义线程池
        m.setErrorHandler(t -> log.error("事件处理失败", t));
        return m;
    }
}
```

或单个监听器加 `@Async`（需 `@EnableAsync`）。

> 注意：设为异步广播器后**所有事件都异步**，且 `@TransactionalEventListener` 语义会变得微妙（事务可能已提交）。生产更推荐在监听器上单独加 `@Async`。

## 4. 监听器顺序与异常

```java
@EventListener @Order(1) public void a(Event e) {}
@EventListener @Order(2) public void b(Event e) {}
```

默认**同步**执行：监听器抛异常会**中断后续监听器**，且会**回滚发布者所在事务**（因为在同一线程事务内）。

解决：加 `@Async`、或 try-catch 内部消化、或设置 `multicaster.setErrorHandler(...)`。

## 5. 内置事件（Boot）

| 事件 | 时机 |
|---|---|
| `ApplicationStartingEvent` | 启动最开始 |
| `ApplicationEnvironmentPreparedEvent` | Environment 就绪 |
| `ApplicationContextInitializedEvent` | 上下文准备 |
| `ApplicationPreparedEvent` | refresh 前 |
| `ContextRefreshedEvent` | 容器刷新完成 |
| `ApplicationStartedEvent` | 刷新后、Runner 前 |
| `ApplicationReadyEvent` | **完全就绪**（Runner 执行完） |
| `ApplicationFailedEvent` | 启动失败 |

## 6. 与 MQ 的边界

| 场景 | 选择 |
|---|---|
| 同进程、允许丢失 | Spring 事件 |
| 跨服务、必须可靠 | MQ |
| 同进程、必须异步且可靠 | 事件 + 落库 + 补偿 |

> Spring 事件**没有持久化、没有重试**，进程重启即丢失。别拿它当 MQ 用。

## 7. 数据校验（Jakarta Validation）

### 依赖与启用

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

`spring-boot-starter-web` 在 Boot 2.3+ 后**不再自带** validation，需单独引入。

### 常用约束注解

| 注解 | 含义 |
|---|---|
| `@NotNull` / `@Null` | 非 null / 必须为 null |
| `@NotBlank` | 字符串非 null 且 `trim` 后非空 |
| `@NotEmpty` | 集合/字符串非空 |
| `@Size(min,max)` | 长度范围 |
| `@Min` / `@Max` / `@DecimalMin` | 数值范围 |
| `@Pattern(regexp)` | 正则 |
| `@Email` | 邮箱格式 |
| `@Positive` / `@PositiveOrZero` | 正数 |
| `@Past` / `@Future` | 时间 |
| `@Valid` | 级联校验嵌套对象 |

### 使用

```java
@Data
public class UserDTO {
    @NotBlank(message = "姓名不能为空")
    @Size(max = 32)
    private String name;

    @NotNull @Min(0) @Max(150)
    private Integer age;

    @Email
    private String email;

    @Valid                 // ★ 级联
    @NotNull
    private AddressDTO address;
}

@PostMapping("/users")
public Result<Void> create(@Valid @RequestBody UserDTO dto) { ... }

// 校验 List
public void batch(@Valid @RequestBody List<@Valid UserDTO> list) {}

// 校验 query 参数（需类上加 @Validated）
@GetMapping("/u")
public User get(@NotNull @RequestParam Long id) {}
```

### 分组校验

```java
public interface CreateGroup {}   // 标记接口
public interface UpdateGroup {}

@NotNull(groups = UpdateGroup.class) private Long id;
@NotBlank(groups = {CreateGroup.class, UpdateGroup.class}) private String name;

@PostMapping public void create(@Validated(CreateGroup.class) @RequestBody UserDTO dto) {}
@PutMapping  public void update(@Validated(UpdateGroup.class) @RequestBody UserDTO dto) {}
```

### 方法级校验（Service 层）

```java
@Service
@Validated          // ★ 类上加
public class UserService {
    public void update(@NotNull Long id, @NotBlank String name) {}
}
// 违规抛 ConstraintViolationException
```

### 自定义校验注解

```java
@Target({FIELD, PARAMETER}) @Retention(RUNTIME)
@Constraint(validatedBy = PhoneValidator.class)
public @interface Phone {
    String message() default "手机号格式错误";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
public class PhoneValidator implements ConstraintValidator<Phone, String> {
    private static final Pattern P = Pattern.compile("^1[3-9]\\d{9}$");
    public boolean isValid(String v, ConstraintValidatorContext ctx) {
        return v == null || P.matcher(v).matches();   // null 交给 @NotNull
    }
}
```

### 编程式校验

```java
@Autowired Validator validator;
Set<ConstraintViolation<UserDTO>> violations = validator.validate(dto);
```

### 快速失败模式（fail-fast）

```java
@Bean
public Validator validator() {
    return Validation.byProvider(HibernateValidator.class).configure()
            .failFast(true)     // 第一个错误即返回，省性能
            .buildValidatorFactory().getValidator();
}
```

## 8. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| `@Valid` 不生效 | 参数校验没触发 | 检查是否引入 starter-validation；`@Valid` 与 `@Validated` 区别 |
| 嵌套对象未校验 | 内部字段全放过 | 字段上加 `@Valid` |
| `@NotBlank` 用在 Integer | 报错 `No validator could be found` | 数字用 `@NotNull` |
| 事件同步阻塞请求 | 接口响应变慢 | `@Async` 或 `@TransactionalEventListener` |
| `@Async` 事件丢失 traceId | 日志无链路 | 用 `TaskDecorator` 传递 MDC |
| 校验异常返回 500 | 前端看不懂 | 全局异常处理 `MethodArgumentNotValidException`（见 [15](15-boot-exception-handling.md)） |

## 9. 面试速答

- **Spring 事件是观察者模式**：`ApplicationEventPublisher` + `ApplicationEventMulticaster` + `@EventListener`，默认同步。
- **@EventListener 与 @TransactionalEventListener**：后者绑到事务阶段（默认 AFTER_COMMIT），适合「提交后再发消息/清缓存」。
- **事件 vs MQ**：事件是进程内、无持久化；MQ 跨进程、可靠但重。
- **@Valid 与 @Validated**：前者是 Jakarta 标准、支持级联；后者是 Spring 的、支持分组与方法级校验。
