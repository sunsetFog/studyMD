# 05 · AOP 基础：代理机制与 JDK/CGLIB

## 1. 为什么需要 AOP

把横切关注点（日志、事务、权限、监控、幂等）从业务代码中剥离，集中到切面。

```java
// 没有 AOP：每个方法都要写
public void transfer(...) {
    long t = System.currentTimeMillis();
    log.info("start");
    try {
        tx.begin();
        // ... 业务
        tx.commit();
    } catch (Exception e) { tx.rollback(); throw e; }
    finally { log.info("cost " + (System.currentTimeMillis()-t)); }
}
```

有 AOP：业务方法只剩业务，横切逻辑由切面统一织入。

## 2. 核心术语

| 术语 | 含义 |
|---|---|
| Aspect（切面） | 横切逻辑的模块化（`LoggingAspect`） |
| JoinPoint（连接点） | 可被增强的点（Spring 中仅方法执行） |
| Pointcut（切点） | 匹配连接点的表达式（哪些方法被增强） |
| Advice（通知） | 增强逻辑 + 时机 |
| Weaving（织入） | 把切面应用到目标对象的过程 |
| Target（目标对象） | 被代理的原始对象 |
| Proxy（代理） | 织入后生成的对象，容器里放的是它 |

### 五种通知

| 通知 | 注解 | 时机 |
|---|---|---|
| 前置 | `@Before` | 方法前 |
| 后置（返回后） | `@AfterReturning` | 正常返回后 |
| 异常 | `@AfterThrowing` | 抛异常后 |
| 最终 | `@After` | finally，一定执行 |
| 环绕 | `@Around` | 包住整个方法，**最强，可控制是否执行/改返回值** |

执行顺序（Spring 5.2+）：
```
@Around 前半 → @Before → 目标方法 → @Around 后半
             → @AfterReturning / @AfterThrowing → @After
```

## 3. 两种代理方式

| 方式 | 条件 | 原理 | 限制 |
|---|---|---|---|
| JDK 动态代理 | 目标**实现接口** | 运行时生成实现同一接口的代理类 | 只能代理接口方法 |
| CGLIB | 目标无接口（或强制） | 继承目标类生成子类（ASM 字节码） | 不能代理 `final` 类/`final`/`private` 方法 |

### Spring 的默认选择

- Spring 5 / Boot 2：**有接口用 JDK，无接口用 CGLIB**。
- Spring 6 / Boot 3 起：**默认一律用 CGLIB**（`spring.aop.proxy-target-class=true` 默认）。
- 这样无论有没有接口，都能直接注入实现类（避免 `BeanNotOfRequiredTypeException`）。

```yaml
spring:
  aop:
    proxy-target-class: false   # 想退回 JDK 代理才设
```

## 4. 手写 JDK 动态代理

```java
interface UserService { void create(String name); }
class UserServiceImpl implements UserService {
    public void create(String name) { System.out.println("create " + name); }
}

UserService proxy = (UserService) Proxy.newProxyInstance(
        UserService.class.getClassLoader(),
        new Class[]{UserService.class},
        (p, method, args) -> {
            System.out.println("[log] " + method.getName());
            Object r = method.invoke(new UserServiceImpl(), args);
            System.out.println("[log] done");
            return r;
        });
proxy.create("tom");
```

底层：运行时生成 `$Proxy0` 类（实现接口），方法调用转发到 `InvocationHandler#invoke`。

## 5. 手写 CGLIB（Spring 内置，无需额外依赖）

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(UserServiceImpl.class);
enhancer.setCallback((MethodInterceptor) (obj, method, args, methodProxy) -> {
    System.out.println("[log] " + method.getName());
    return methodProxy.invokeSuper(obj, args);   // ★ 用 invokeSuper，避免无限递归
});
UserServiceImpl p = (UserServiceImpl) enhancer.create();
```

> `FastClass` 机制：CGLIB 通过给每个方法编号，避免反射调用，性能优于 JDK 代理（JDK 8 后 JDK 代理性能已反超）。

## 6. 代理的局限（必考）

| 场景 | 是否生效 | 原因 |
|---|---|---|
| **同类方法自调用** `this.b()` | ❌ | `this` 是原始对象，不是代理 |
| `private` 方法 | ❌ | 无法被子类/接口覆写 |
| `final` 方法/类 | ❌（CGLIB） | 无法继承覆写 |
| `static` 方法 | ❌ | 不属于实例 |
| 构造器 | ❌ | 对象尚未创建 |
| 外部通过代理调用 `a()`，`a()` 内部调 `b()` | ❌ b 不增强 | 自调用 |
| `@Configuration` 类中 `@Bean` 互相调用 | ✅ | 配置类被 CGLIB 代理，`@Bean` 方法走容器单例逻辑 |

```java
// 自调用失效示例
@Service
public class OrderService {
    public void create() { this.pay(); }       // ★ 事务/日志都在 pay 上，但不生效
    @Transactional public void pay() {}
}
```

解决方案（见 [06](06-spring-aop-practice.md)）：注入自身代理、拆类、`AopContext.currentProxy()`。

## 7. 织入时机对比

| 框架 | 织入时机 | 说明 |
|---|---|---|
| AspectJ | 编译期 / 类加载期（LTW） | 功能最强，可拦截字段、构造器、私有方法；需 ajc 编译器 |
| Spring AOP | **运行期代理** | 简单、无侵入，仅支持方法级 |

> Spring AOP **不等于 AspectJ**，只是借用了 AspectJ 的**注解和切点表达式语法**。需要拦截非 public / 构造器时才上 `@EnableAspectJAutoProxy` 之外的 AspectJ LTW。

## 8. AOP 在 Spring 内部的落地

```
@EnableAspectJAutoProxy
   → 注册 AnnotationAwareAspectJAutoProxyCreator（BeanPostProcessor）
   → postProcessAfterInitialization 中判断是否需要代理
   → 生成代理对象放入容器（原对象被替换）
```

调用链（JDK 代理）：`proxy.method()` → `JdkDynamicAopProxy#invoke` → 拦截器链 `ReflectiveMethodInvocation` → 各 Advice → 目标方法。

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 切不中 | 切面无效果 | 检查包扫描、切点表达式、方法是否 public |
| 自调用 | 事务/缓存/异步全失效 | 注入自身代理或拆类 |
| 注入实现类失败 | `BeanNotOfRequiredTypeException` | 注入接口；或确认使用 CGLIB 代理 |
| 循环依赖 + 代理 | 注入了未代理对象 | 保留三级缓存（不要关 `allow-circular-references` 又同时自注入） |
| 切面里 `proceed()` 调用两次 | 目标方法执行两次 | 环绕通知确保只调一次 |

## 10. 面试速答

- **JDK 与 CGLIB 区别**：JDK 基于接口（`Proxy`+`InvocationHandler`），CGLIB 基于继承（`Enhancer`+`MethodInterceptor`）；前者要求实现接口，后者不能代理 final。
- **Spring AOP 默认用哪个**：Spring 6/Boot 3 起默认 CGLIB。
- **为什么同类调用不生效**：因为 `this` 指向目标对象而非代理对象，绕过了拦截器链。
