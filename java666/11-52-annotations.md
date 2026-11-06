# 52 · 注解

## 1. 内置注解

```java
@Override              // 重写检查
@Deprecated            // 标记废弃
@SuppressWarnings("unchecked")   // 抑制警告
@SafeVarargs           // 泛型可变参数安全
@FunctionalInterface   // 函数式接口检查
```

```java
@Deprecated(since = "1.5", forRemoval = true)   // Java 9+
public void oldMethod() { }
```

## 2. 元注解

| 元注解 | 作用 |
|---|---|
| `@Target` | 可用位置（TYPE、FIELD、METHOD、PARAMETER、CONSTRUCTOR、LOCAL_VARIABLE、ANNOTATION_TYPE、PACKAGE、TYPE_PARAMETER、TYPE_USE、RECORD_COMPONENT、MODULE） |
| `@Retention` | 保留策略（SOURCE / CLASS / RUNTIME） |
| `@Documented` | 包含在 Javadoc |
| `@Inherited` | 子类继承父类的注解（仅类注解） |
| `@Repeatable` | 可重复标注 |

### Retention 三级别

| 级别 | 保留到 | 可反射读取 |
|---|---|---|
| `SOURCE` | 源码（编译后丢弃） | ✘ |
| `CLASS`（默认） | class 文件 | ✘ |
| `RUNTIME` | 运行时（内存） | ✔ |

> 自定义框架注解必须是 `RUNTIME`。

## 3. 自定义注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RateLimit {
    String key() default "";
    int permits() default 10;          // 默认值
    int windowSeconds() default 1;
    TimeUnit unit() default TimeUnit.SECONDS;
}
```

使用：

```java
@RateLimit(key = "order:create", permits = 100, windowSeconds = 1)
public Result createOrder(OrderReq req) { ... }
```

### 注解元素的合法类型

- 基本类型、String、Class、枚举、注解、以上类型的数组。
- 不能是包装类型、集合、null。
- 默认值不能为 null。

## 4. 读取注解

```java
Method m = OrderController.class.getMethod("createOrder", OrderReq.class);

RateLimit rl = m.getAnnotation(RateLimit.class);
if (rl != null) {
    System.out.println(rl.permits());
}

// 检查是否存在
m.isAnnotationPresent(RateLimit.class);

// 类/字段/参数
clazz.getAnnotation(X.class);
field.getAnnotation(X.class);
method.getParameters()[0].getAnnotation(X.class);

// 继承注解（@Inherited 只对类有效，且只查父类不查接口）
```

## 5. 可重复注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(Schedules.class)
public @interface Schedule {
    String cron();
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Schedules {
    Schedule[] value();
}

// 使用
@Schedule(cron = "0 0 * * *")
@Schedule(cron = "0 12 * * *")
public void job() { }

// 读取
Schedule[] arr = m.getAnnotationsByType(Schedule.class);
```

## 6. 实战：简易限流器

```java
@Aspect
@Component
public class RateLimitAspect {
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable {
        String key = rateLimit.key().isEmpty()
                ? pjp.getSignature().toShortString()
                : rateLimit.key();

        RateLimiter limiter = limiters.computeIfAbsent(key, k ->
                RateLimiter.create(rateLimit.permits() / (double) rateLimit.windowSeconds()));

        if (!limiter.tryAcquire()) {
            throw new BizException("RATE_LIMITED", "请求过于频繁");
        }
        return pjp.proceed();
    }
}
```

## 7. 实战：自定义校验注解

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneValidator.class)
public @interface Phone {
    String message() default "手机号格式不正确";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PhoneValidator implements ConstraintValidator<Phone, String> {
    private static final Pattern P = Pattern.compile("^1[3-9]\\d{9}$");

    @Override public boolean isValid(String value, ConstraintValidatorContext ctx) {
        return value == null || P.matcher(value).matches();    // null 交给 @NotNull
    }
}

// 使用
public record RegisterReq(
    @Phone String phone,
    @NotBlank @Size(min = 6, max = 20) String password
) { }
```

## 8. 注解处理器（APT）

编译期处理，不依赖反射，性能最好。**Lombok、MapStruct、AutoValue** 都用它。

```java
@SupportedAnnotationTypes("com.acme.GenerateBuilder")
@SupportedSourceVersion(SourceVersion.RELEASE_17)
public class BuilderProcessor extends AbstractProcessor {
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
        for (Element e : env.getElementsAnnotatedWith(GenerateBuilder.class)) {
            // 生成 Java 源文件
            try (Writer w = processingEnv.getFiler()
                    .createSourceFile(e.toString() + "Builder").openWriter()) {
                w.write(generateCode((TypeElement) e));
            } catch (IOException ex) {
                processingEnv.getMessager().printMessage(ERROR, ex.getMessage());
            }
        }
        return true;
    }
}
```

注册：`META-INF/services/javax.annotation.processing.Processor`

> 优点：零运行时开销、编译期报错。缺点：不能修改已有类（只能生成新类）。

## 9. 注解 vs 配置文件 vs 标记接口

| 方式 | 优点 | 缺点 |
|---|---|---|
| 注解 | 类型安全、就近声明、编译期检查 | 无法动态修改、重复代码 |
| XML/配置 | 集中管理、可动态调整 | 无编译检查、冗长 |
| 标记接口 | 可作泛型边界 | 侵入继承结构 |

现代趋势：注解 + 少量配置（Spring Boot 的 `@ConfigurationProperties` 就是两者结合）。

## 10. 常见框架注解速览

```java
// Spring
@Component / @Service / @Repository / @Controller
@Autowired / @Resource / @Qualifier
@Configuration / @Bean
@RequestMapping / @GetMapping / @PostMapping
@Transactional
@Async
@Value / @ConfigurationProperties
@ConditionalOnXxx

// JPA
@Entity / @Table / @Id / @Column / @OneToMany / @ManyToOne

// Jackson
@JsonProperty / @JsonIgnore / @JsonFormat / @JsonValue / @JsonCreator

// Validation
@NotNull / @NotBlank / @Size / @Min / @Max / @Pattern / @Valid

// JUnit
@Test / @BeforeEach / @AfterEach / @ParameterizedTest / @DisplayName

// Lombok
@Data / @Builder / @Getter / @Setter / @NoArgsConstructor / @AllArgsConstructor
```

## 11. 常见坑

1. 自定义注解忘记 `@Retention(RUNTIME)`（反射读不到）。
2. 忘记 `@Target`（默认所有位置，不够精确）。
3. 注解默认值不能为 null。
4. `@Inherited` 只对类有效，方法注解不被继承。
5. 注解值必须是编译期常量。
6. 用 `getAnnotation` 只能拿到本类（不含父类方法重写的注解）。
7. 反射读注解有性能开销（缓存结果）。
8. Lombok 与 IDE/编译器的注解处理顺序冲突（升级版本）。
9. `@Transactional` 同类内部调用失效（代理不生效，与注解无关但相关）。
10. `@Valid` 与 `@Validated` 混用（分组校验用后者）。

```java
// @Transactional 自调用失效
@Service
public class OrderService {
    public void create() { this.pay(); }        // 直接调用，事务注解失效

    @Transactional
    public void pay() { ... }
}
// 解决：拆分到另一个 Bean，或注入自身代理（AopContext.currentProxy()）
```

## 12. 速查

| 需求 | 写法 |
|---|---|
| 定义注解 | `@interface X { String value() default ""; }` |
| 运行时可用 | `@Retention(RUNTIME)` |
| 限定位置 | `@Target(METHOD)` |
| 可重复 | `@Repeatable(Xs.class)` |
| 读取 | `element.getAnnotation(X.class)` |
| 可继承 | `@Inherited`（仅类） |
| 编译期生成 | APT（`AbstractProcessor`） |
| 校验 | `@Constraint(validatedBy=...)` |
| AOP 切注解 | `@Around("@annotation(x)")` |
