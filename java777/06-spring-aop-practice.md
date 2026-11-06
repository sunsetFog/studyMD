# 06 · Spring AOP 实战：@Aspect 与切点表达式

## 1. 开启与依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

Boot 下 `AopAutoConfiguration` 已自动开启（等价于 `@EnableAspectJAutoProxy(proxyTargetClass=true)`）。

## 2. 切点表达式语法

```java
execution(修饰符? 返回类型 包名.类名.方法名(参数) 异常?)
```

| 示例 | 含义 |
|---|---|
| `execution(* com.x.service.*.*(..))` | service 包下所有类的所有方法 |
| `execution(public * *(..))` | 所有 public 方法 |
| `execution(* com.x..*Service.*(..))` | `com.x` 及其子包下以 Service 结尾的类 |
| `execution(* save*(..))` | 所有 save 开头的方法 |
| `execution(* *(..) throws java.io.IOException)` | 抛指定异常 |
| `execution(* com.x.Svc.*(String, ..))` | 第一个参数为 String |

其他指示器：

| 指示器 | 用法 |
|---|---|
| `@annotation` | `@annotation(com.x.Loggable)` —— 打了该注解的方法 |
| `@within` / `@target` | 类上有某注解 |
| `within` | `within(com.x.service..*)` 限定包 |
| `args` | `args(java.lang.String)` 参数类型匹配并可绑定 |
| `bean` | `bean(*Service)` 按 Bean 名匹配（Spring 特有） |
| `@args` | 参数运行时类型上有某注解 |

组合：`&&`、`||`、`!`（XML 中用 `and/or/not`）。

## 3. 第一个切面

```java
@Slf4j
@Aspect
@Component
@Order(1)                       // 值越小优先级越高
public class LogAspect {

    @Pointcut("execution(* com.example..service..*(..))")
    public void serviceMethods() {}     // 空方法，仅作切点签名

    @Around("serviceMethods()")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            log.info("→ {}({})", pjp.getSignature().toShortString(), Arrays.toString(pjp.getArgs()));
            return pjp.proceed();       // ★ 必须调用，否则目标方法不执行
        } catch (Throwable t) {
            log.error("← {} ex={}", pjp.getSignature(), t.getMessage());
            throw t;                    // ★ 不要吞异常
        } finally {
            log.info("← cost={}ms", System.currentTimeMillis() - start);
        }
    }
}
```

## 4. JoinPoint 常用 API

```java
pjp.getSignature().toShortString();   // UserServiceImpl.create(..)
pjp.getTarget();                      // 目标对象
pjp.getThis();                        // 代理对象
pjp.getArgs();                        // 参数数组
((MethodSignature) pjp.getSignature()).getMethod();   // 反射 Method，可取注解
```

## 5. 注解驱动切面（最常用）

```java
@Target(ElementType.METHOD) @Retention(RetentionPolicy.RUNTIME)
public @interface Audit { String value() default ""; }

@Aspect @Component
public class AuditAspect {
    @AfterReturning(pointcut = "@annotation(audit)", returning = "ret")
    public void audit(Audit audit, Object ret) {
        log.info("[audit] {} ret={}", audit.value(), ret);
    }
}
// 使用
@Audit("创建订单")
public Order create(...) {}
```

> 绑定技巧：切点里写 `@annotation(audit)`，通知参数同名即可自动注入注解实例。

## 6. 实战一：接口幂等切面

```java
@Target(ElementType.METHOD) @Retention(RUNTIME)
public @interface Idempotent { int expire() default 60; String key(); }

@Aspect @Component
@RequiredArgsConstructor
public class IdempotentAspect {
    private final StringRedisTemplate redis;

    @Around("@annotation(idem)")
    public Object around(ProceedingJoinPoint pjp, Idempotent idem) throws Throwable {
        String key = "idem:" + SpELUtil.parse(idem.key(), pjp);   // 支持 #orderId 表达式
        Boolean ok = redis.opsForValue()
                .setIfAbsent(key, "1", Duration.ofSeconds(idem.expire()));
        if (!Boolean.TRUE.equals(ok)) {
            throw new BizException("请勿重复提交");
        }
        try { return pjp.proceed(); }
        catch (Throwable t) { redis.delete(key); throw t; }   // 失败允许重试
    }
}
```

## 7. 实战二：分布式锁切面

```java
@Around("@annotation(lock)")
public Object around(ProceedingJoinPoint pjp, DistributedLock lock) throws Throwable {
    String key = parseKey(lock.key(), pjp);
    RLock rLock = redissonClient.getLock(key);
    boolean acquired = rLock.tryLock(lock.waitTime(), lock.leaseTime(), TimeUnit.SECONDS);
    if (!acquired) throw new BizException("系统繁忙，请稍后再试");
    try { return pjp.proceed(); } finally { rLock.unlock(); }
}
```

## 8. 实战三：多数据源动态切换

```java
@Target({METHOD, TYPE}) @Retention(RUNTIME)
public @interface DS { String value(); }

@Aspect @Component
public class DynamicDataSourceAspect {
    @Before("@annotation(ds)")
    public void before(DS ds) { DataSourceContextHolder.set(ds.value()); }
    @After("@annotation(ds)")
    public void after(DS ds) { DataSourceContextHolder.clear(); }
}
```

> 注意：**数据源切换必须在事务开启之前**，即切面 `@Order` 要小于事务切面（`@EnableTransactionManagement(order=..)`）。

## 9. 切面顺序控制

```java
@Order(Ordered.HIGHEST_PRECEDENCE)   // 最先执行（外层）
@Order(Ordered.LOWEST_PRECEDENCE)    // 最后执行（内层）
```

| 切面 | 建议顺序 |
|---|---|
| 数据源切换 / 限流 | 高优先级（先执行） |
| 事务 | 默认 `Ordered.LOWEST_PRECEDENCE` |
| 日志 | 低优先级 |

> 环绕通知「先执行的外层，后结束」——像洋葱一样层层包裹。

## 10. 自调用失效的三种解法

```java
// 方案1：注入自身代理（推荐）
@Service
public class OrderService {
    @Lazy @Autowired private OrderService self;
    public void create() { self.pay(); }     // 走代理
    @Transactional public void pay() {}
}

// 方案2：AopContext（需 exposeProxy=true）
@EnableAspectJAutoProxy(exposeProxy = true)
((OrderService) AopContext.currentProxy()).pay();

// 方案3：拆分到另一个 Service（最干净，推荐）
@Service class OrderService { @Autowired PayService payService; }
```

## 11. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 忘记 `pjp.proceed()` | 方法不执行、返回 null | 环绕必须 proceed |
| 吞掉异常 | 事务不回滚、上层无感知 | `catch` 后 `throw` |
| 切面顺序错误 | 数据源切换在事务之后，切换无效 | 调小 `@Order` |
| 切到 `Object` 方法 | `toString`/`equals` 被代理异常 | 限定 `execution(* com.x..*(..))` 具体包 |
| 过度切 | 切 `execution(* *(..))` 拖慢全局 | 精确切点或用 `@annotation` |
| 切面内再次调用被切方法 | 递归套娃 | 注意切点覆盖范围 |

## 12. 面试速答

- **五种通知**：Before / AfterReturning / AfterThrowing / After / Around；Around 最强，可决定是否执行目标方法。
- **切点表达式**：`execution(返回类型 包名.类.方法(参数))`，常用 `@annotation` 做注解驱动。
- **多个切面顺序**：`@Order` 或实现 `Ordered`，值小者在外层先执行。
