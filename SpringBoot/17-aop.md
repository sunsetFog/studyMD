# 17 · AOP 切面编程

## 一、核心概念

| 术语 | 含义 |
|---|---|
| Aspect（切面） | @Aspect 标注的类 |
| JoinPoint（连接点） | 可被拦截的点（方法执行） |
| Pointcut（切点） | 匹配连接点的表达式 |
| Advice（通知） | Before / After / AfterReturning / AfterThrowing / Around |
| Weaving（织入） | 把切面应用到目标（Spring 用运行时代理） |

## 二、依赖与启用

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

## 三、切点表达式

```java
@Aspect
@Component
public class ServiceAspect {

    // execution：按方法签名
    @Pointcut("execution(* com.example.demo.service..*.*(..))")
    public void serviceLayer() {}

    // @annotation：按注解（最常用，注解即契约）
    @Pointcut("@annotation(com.example.demo.common.annotation.ApiLog)")
    public void apiLog() {}

    // 组合
    @Pointcut("serviceLayer() && !execution(* com.example.demo.service..*.query*(..))")
    public void serviceWrite() {}
}
```

execution 语法：`execution(修饰符 返回类型 包.类.方法(参数))`，`..` 匹配任意层级/任意参数。

## 四、五种通知

```java
@Around("serviceLayer()")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    try {
        Object result = pjp.proceed();                       // 执行目标方法
        log.info("{} 耗时 {}ms", pjp.getSignature(), System.currentTimeMillis() - start);
        return result;
    } catch (Throwable e) {
        log.error("{} 异常", pjp.getSignature(), e);
        throw e;                                             // 必须重抛
    }
}

@Before("serviceLayer()")
public void before(JoinPoint jp) {
    log.info("入参: {}", Arrays.toString(jp.getArgs()));
}

@AfterReturning(pointcut = "serviceLayer()", returning = "result")
public void afterReturn(JoinPoint jp, Object result) { }

@AfterThrowing(pointcut = "serviceLayer()", throwing = "e")
public void afterThrow(JoinPoint jp, Exception e) { }
```

**优先 @Around**：能力覆盖其余四种，且能改入参/改返回值/吞异常。

## 五、实战场景一：注解式接口日志

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ApiLog {
    String value() default "";
}

@Around("@annotation(apiLog)")    // 直接绑定注解参数
public Object log(ProceedingJoinPoint pjp, ApiLog apiLog) throws Throwable {
    String traceId = UUID.randomUUID().toString();
    MDC.put("traceId", traceId);                    // 日志链路（见 23 篇）
    try {
        return pjp.proceed();
    } finally {
        MDC.clear();
    }
}
```

## 六、实战场景二：分布式锁注解

```java
@Around("@annotation(lock)")
public Object lock(ProceedingJoinPoint pjp, DistributedLock lock) throws Throwable {
    String key = lock.key().isEmpty()
            ? "lock:" + pjp.getSignature().toShortString()
            : lock.key();
    String requestId = UUID.randomUUID().toString();
    if (!redisUtil.tryLock(key, requestId, Duration.ofSeconds(lock.ttl()))) {
        throw new BusinessException(429, "操作过于频繁，请稍后再试");
    }
    try {
        return pjp.proceed();
    } finally {
        redisUtil.unlock(key, requestId);
    }
}
```

## 七、Spring AOP vs AspectJ

| 维度 | Spring AOP | AspectJ |
|---|---|---|
| 织入时机 | 运行时代理（JDK/CGLIB） | 编译期/类加载期 |
| 拦截范围 | Spring Bean 的 public 方法 | 任意方法/字段/构造器 |
| 性能 | 够用 | 更优 |

Spring AOP 能满足 95% 场景。两种代理选择：目标实现了接口 → JDK 动态代理；否则 CGLIB。Boot 3 默认 `proxyTargetClass=true`（CGLIB）。

## 八、代理与自调用的坑（呼应 14 篇）

```java
@Service
public class UserService {
    public void a() {
        this.b();        // b 的切面不生效！内部调用绕过代理
    }

    @Log
    public void b() { }
}
// 修复：注入 self 代理 或 拆分类
```

## 常见坑

- **坑 1**：内部自调用切面失效 → 同事务失效原理
- **坑 2**：final 方法/类无法被 CGLIB 代理 → 切面静默失效
- **坑 3**：private 方法不能被代理拦截
- **坑 4**：切面内抛异常被 Around 吞掉 → 业务异常丢失
- **坑 5**：切点写太宽（如 `execution(* *..*.*(..))`）拦截到框架 Bean → 启动报错或性能浪费

## Checklist

- [ ] 横切逻辑（日志/锁/审计/幂等）统一注解 + 切面实现
- [ ] 切点精确到业务包/注解，禁止全匹配
- [ ] Around 中异常必须重抛
- [ ] 自调用场景统一走 self 注入
