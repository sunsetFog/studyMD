# 19 · Boot 中的 AOP 实战

## 1. 自动配置带来的差异

Boot 中 `AopAutoConfiguration` 已做：

```java
@AutoConfiguration
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {
    @Configuration
    @EnableAspectJAutoProxy(proxyTargetClass = true)    // ★ Boot 3 默认 CGLIB
    @ConditionalOnProperty(prefix="spring.aop", name="proxy-target-class", havingValue="true", matchIfMissing=true)
    static class CglibAutoProxyConfiguration {}
}
```

即：**无需手动 `@EnableAspectJAutoProxy`**，引入 starter-aop 即可。

```yaml
spring:
  aop:
    auto: true
    proxy-target-class: true     # true=CGLIB(默认)，false=JDK（需接口）
```

## 2. 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<!-- 内部引入 aspectjweaver -->
```

## 3. 实战一：接口耗时 + 慢请求告警

```java
@Aspect @Component @Slf4j
@Order(100)
public class CostAspect {
    private static final long SLOW = 1000;

    @Around("execution(* com.example..controller..*(..))")
    public Object cost(ProceedingJoinPoint pjp) throws Throwable {
        long s = System.nanoTime();
        try { return pjp.proceed(); }
        finally {
            long ms = (System.nanoTime() - s) / 1_000_000;
            String sig = pjp.getSignature().toShortString();
            if (ms > SLOW) log.warn("SLOW {} cost={}ms args={}", sig, ms, mask(pjp.getArgs()));
            Metrics.timer("api.cost", "method", sig).record(ms, TimeUnit.MILLISECONDS);
        }
    }
}
```

## 4. 实战二：防重复提交（幂等）

```java
@Target(METHOD) @Retention(RUNTIME)
public @interface NoRepeatSubmit { int seconds() default 5; }

@Aspect @Component
public class NoRepeatAspect {
    private final StringRedisTemplate redis;
    @Around("@annotation(ann)")
    public Object around(ProceedingJoinPoint pjp, NoRepeatSubmit ann) throws Throwable {
        HttpServletRequest req = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        String key = "nrs:" + req.getRequestURI() + ":" + LoginUtil.userId() + ":" + DigestUtils.md5DigestAsHex(
                JsonUtil.toJson(pjp.getArgs()).getBytes(StandardCharsets.UTF_8));
        if (!Boolean.TRUE.equals(redis.opsForValue().setIfAbsent(key, "1", Duration.ofSeconds(ann.seconds())))) {
            throw new BizException("请勿重复提交");
        }
        return pjp.proceed();
    }
}
```

## 5. 实战三：操作日志（含变更前后对比）

```java
@Target(METHOD) @Retention(RUNTIME)
public @interface OperateLog { String module(); String action(); }

@Aspect @Component
public class OperateLogAspect {
    @Autowired private ApplicationEventPublisher publisher;

    @AfterReturning(pointcut = "@annotation(l)", returning = "ret")
    public void save(OperateLog l, Object ret) {
        publisher.publishEvent(new AuditEvent(l.module(), l.action(), LoginUtil.userId(), JsonUtil.toJson(ret)));
    }
}
// 监听端异步落库（避免拖慢主流程、避免日志失败影响业务）
```

## 6. 实战四：数据权限（行级过滤）

```java
@Aspect @Component
public class DataScopeAspect {
    @Before("execution(* com.example..mapper.*.select*(..)) && @annotation(scope)")
    public void setScope(DataScope scope) {
        DataScopeContext.set(buildSqlByRole(LoginUtil.currentUser(), scope.value()));
    }
    @After("execution(* com.example..mapper.*.select*(..)) && @annotation(scope)")
    public void clear(DataScope scope) { DataScopeContext.clear(); }
}
```

> 更通用的做法是 MyBatis 拦截器改写 SQL（见 [25](25-mybatis-deep.md)）。

## 7. 实战五：统一返回包装 + 脱敏

```java
@Aspect
public class DesensitizeAspect {
    @AfterReturning(pointcut = "execution(* com.example..controller..*(..))", returning = "ret")
    public Object mask(Object ret) { return Desensitizer.mask(ret); }
}
```

## 8. 多切面顺序（Boot 内建切面的优先级）

| 切面 | 默认 order |
|---|---|
| 缓存 `CacheInterceptor` | `Ordered.LOWEST_PRECEDENCE` |
| 事务 `TransactionInterceptor` | `Ordered.LOWEST_PRECEDENCE` |
| Async `AsyncAnnotationBeanPostProcessor` | `Ordered.LOWEST_PRECEDENCE` |
| 重试 `@Retryable` | `Ordered.LOWEST_PRECEDENCE - 1` |
| 你的日志/限流/数据源切面 | 应设更小值（更高优先级） |

```java
@EnableTransactionManagement(order = 100)   // 显式指定，避免不确定性
```

## 9. 与 @Async / @Cacheable / @Transactional 协作的坑

| 组合 | 现象 | 说明 |
|---|---|---|
| `@Transactional` + `@Cacheable` 同方法 | 缓存可能在事务提交前写入 | 拆开或用 `@TransactionalEventListener` 清缓存 |
| `@Async` + `@Transactional` | 事务失效（新线程） | 不要在同一方法用；Async 方法内自己开事务 |
| `@Cacheable` 自调用 | 缓存不生效 | 同代理问题 |
| `@Retryable` + `@Transactional` | 重试时事务重新开启（OK） | 注意幂等 |
| 切面抛异常 | 事务回滚 | 环绕通知要正确传递异常 |

## 10. 性能建议

- 切点尽量精确：避免 `execution(* *(..))`。
- 高频接口慎用 `@Around` 里的序列化（JSON 序列化参数开销大）；可用 `if (log.isDebugEnabled())` 保护。
- 切面数不宜过多，每个方法调用都要走拦截器链。
- 反射取注解结果可缓存（`Method` → 注解实例）。

## 11. 调试技巧

```java
// 查看对象是否被代理、是哪种代理
System.out.println(AopUtils.isAopProxy(bean));
System.out.println(AopUtils.isCglibProxy(bean));
System.out.println(AopUtils.isJdkDynamicProxy(bean));
System.out.println(AopProxyUtils.ultimateTargetClass(bean));
```

```yaml
# 打印代理创建过程
logging.level.org.springframework.aop: debug
```

## 12. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 切面不生效 | 无增强 | 检查 starter-aop、切点范围、方法 public、是否被同类调用 |
| Boot 3 注入实现类失败 | `BeanNotOfRequiredTypeException` | 默认已是 CGLIB，若手动设了 `proxy-target-class: false` 改回 |
| 事务+缓存顺序错 | 脏读缓存 | 显式指定 order |
| 环绕吞异常 | 事务提交 | `throw t` |
| 切面中取 `HttpServletRequest` 报异常 | 非 Web 线程 | 判空或用 `RequestContextHolder` 前检查 |

## 13. 面试速答

- **Boot 里还要不要 `@EnableAspectJAutoProxy`**：不需要，`AopAutoConfiguration` 已自动开启，且默认 CGLIB。
- **AOP 与事务谁先执行**：取决于 `@Order`，默认事务优先级最低（最内层），所以自定义切面（限流、数据源）要设更小的 order 才能包住事务。
- **如何排查 AOP 不生效**：先看对象是不是代理（`AopUtils.isAopProxy`），再看切点是否匹配（`/actuator/beans`），最后看是否被自调用绕过。
