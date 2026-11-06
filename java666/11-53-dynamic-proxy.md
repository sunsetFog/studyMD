# 53 · 动态代理

## 1. 静态代理 vs 动态代理

```java
// 静态代理：手写代理类，编译期确定
class UserServiceProxy implements UserService {
    private final UserService target;
    UserServiceProxy(UserService t) { this.target = t; }

    @Override public User get(Long id) {
        log.info("before");
        try { return target.get(id); }
        finally { log.info("after"); }
    }
}
```

问题：每个接口都要写一个代理类，无法通用。

**动态代理**：运行时生成代理类，统一处理横切逻辑（日志、事务、权限、缓存）。

## 2. JDK 动态代理

### 核心 API

```java
public interface InvocationHandler {
    Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}

public static Object Proxy.newProxyInstance(
    ClassLoader loader, Class<?>[] interfaces, InvocationHandler h);
```

### 完整示例

```java
public interface UserService {
    User get(Long id);
    void save(User user);
}

public class UserServiceImpl implements UserService {
    @Override public User get(Long id) { return new User(id, "Tom"); }
    @Override public void save(User u) { System.out.println("saved " + u); }
}

public class LogHandler implements InvocationHandler {
    private final Object target;

    public LogHandler(Object target) { this.target = target; }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("调用 " + method.getName() + " 参数 " + Arrays.toString(args));
        try {
            Object result = method.invoke(target, args);      // 反射调用目标
            System.out.println("返回 " + result);
            return result;
        } catch (InvocationTargetException e) {
            throw e.getTargetException();                     // 剥离包装
        }
    }
}

UserService target = new UserServiceImpl();
UserService proxy = (UserService) Proxy.newProxyInstance(
        target.getClass().getClassLoader(),
        target.getClass().getInterfaces(),
        new LogHandler(target));

proxy.get(1L);
```

### 生成代理类的特征

```java
proxy.getClass().getName();     // jdk.proxy1.$Proxy0
proxy.getClass().isProxyClass(); // 需要 Class 判断：Proxy.isProxyClass(proxy.getClass())
Proxy.getInvocationHandler(proxy);
```

- 代理类继承 `java.lang.reflect.Proxy`，实现指定接口。
- 类名 `$Proxy0`、`$Proxy1`...
- 方法调用转发到 `InvocationHandler.invoke`。
- `equals/hashCode/toString` 也会转发（需注意）。

### 限制

1. **只能代理接口**（代理类已继承 Proxy，无法再继承目标类）。
2. 目标方法必须是 public（接口方法）。
3. 性能有反射开销（JDK 8+ 有优化，接近直接调用）。
4. 无法拦截 final 方法（接口方法本身不是 final）。

## 3. CGLIB 动态代理

### 原理

通过 **ASM 生成目标类的子类**，重写非 final 方法，在方法中插入拦截逻辑。

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(UserServiceImpl.class);       // 继承目标类
enhancer.setCallback((MethodInterceptor) (obj, method, args, proxy) -> {
    System.out.println("before " + method.getName());
    Object result = proxy.invokeSuper(obj, args);    // 调用父类（原）方法
    System.out.println("after");
    return result;
});

UserServiceImpl proxy = (UserServiceImpl) enhancer.create();
proxy.get(1L);
```

### 限制

1. **不能代理 final 类**。
2. **不能拦截 final/private/static 方法**。
3. 需要目标类有无参构造器（或通过 `create(Class[], Object[])` 指定）。
4. 生成类在子类中调用 `super` 方法，因此**自调用无法被拦截**（内部 `this.method()` 不走代理）。
5. 需要引入依赖（Spring Core 已内置 repackaged 版本）。

## 4. JDK vs CGLIB

| 维度 | JDK 动态代理 | CGLIB |
|---|---|---|
| 前提 | 目标实现接口 | 目标可继承（非 final） |
| 原理 | 生成实现接口的类 | 生成子类 |
| 速度（创建） | 快 | 慢（生成字节码） |
| 速度（调用） | JDK 8+ 较快 | 快（直接调用 super） |
| 依赖 | JDK 内置 | 第三方（Spring 内置） |
| final 方法 | 不涉及 | 无法代理 |
| 默认选择 | 有接口时 | 无接口时 |

Spring AOP 规则：

- 目标实现接口 → 默认 JDK 代理。
- 无接口 → CGLIB。
- `proxy-target-class=true` 强制 CGLIB（Spring Boot 2.x 起默认 true）。

## 5. Spring AOP 原理

```
@Transactional / @Cacheable / @Async
        ↓
BeanPostProcessor 在初始化后创建代理
        ↓
调用方持有代理对象
        ↓
代理拦截 → 执行通知链 → 调用目标方法
```

```java
// 简化版 AOP 代理
public class AopProxy implements InvocationHandler {
    private final Object target;
    private final List<MethodInterceptor> interceptors;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        MethodInvocation invocation = new MethodInvocation(target, method, args);
        return new ReflectiveMethodInvocation(invocation, interceptors).proceed();
    }
}
```

### 自调用失效问题

```java
@Service
public class OrderService {
    public void create() {
        this.pay();          // this 是原始对象，不是代理 → @Transactional 失效
    }

    @Transactional
    public void pay() { }
}
```

解决方案：

1. **拆分类**（推荐）：把 `pay` 放到另一个 Bean。
2. 注入自身：`@Autowired private OrderService self;` 然后 `self.pay()`。
3. `AopContext.currentProxy()`：需开启 `@EnableAspectJAutoProxy(exposeProxy = true)`。
4. 用 `TransactionTemplate` 编程式事务。

## 6. 性能对比

| 方式 | 相对耗时（调用 1 亿次） |
|---|---|
| 直接调用 | 1x |
| JDK 代理（JDK 8+） | ~1.5-2x |
| CGLIB | ~1.2-1.5x |
| 反射 `Method.invoke` | ~5-20x |

JDK 8 对 `Method.invoke` 做了 `LambdaForm` 优化，JDK 代理性能大幅提升。

## 7. 手写简易 AOP 框架

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable { }

public class LoggingProxyFactory {
    @SuppressWarnings("unchecked")
    public static <T> T wrap(T target) {
        Class<?>[] interfaces = target.getClass().getInterfaces();
        return (T) Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                interfaces,
                (proxy, method, args) -> {
                    boolean loggable = method.isAnnotationPresent(Loggable.class);
                    long start = System.nanoTime();
                    try {
                        Object r = method.invoke(target, args);
                        if (loggable) {
                            System.out.printf("%s cost %.2fms%n",
                                    method.getName(), (System.nanoTime() - start) / 1e6);
                        }
                        return r;
                    } catch (InvocationTargetException e) {
                        throw e.getTargetException();
                    }
                });
    }
}
```

## 8. ByteBuddy（现代方案）

```java
Class<? extends UserService> dynamicType = new ByteBuddy()
        .subclass(UserServiceImpl.class)
        .method(named("get")).intercept(MethodDelegation.to(Interceptor.class))
        .make()
        .load(UserServiceImpl.class.getClassLoader())
        .getLoaded();

UserService proxy = dynamicType.getDeclaredConstructor().newInstance();
```

优点：API 更友好、性能好、支持复杂场景。Spring Boot 3 / Mockito 5 已用 ByteBuddy 替代 CGLIB。

## 9. 动态代理的应用

| 场景 | 说明 |
|---|---|
| Spring AOP | 事务、缓存、日志、权限 |
| MyBatis Mapper | 接口无需实现，代理转发到 SQL |
| RPC 框架 | 客户端代理发起远程调用 |
| Mock 框架 | Mockito 生成 mock 对象 |
| 延迟加载 | Hibernate 代理 |
| 权限校验 | 方法级注解拦截 |
| 重试/熔断 | Resilience4j |

### MyBatis Mapper 原理

```java
// 你只写接口，MyBatis 用 JDK 代理生成实现
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
User u = mapper.selectById(1L);

// 内部：MapperProxy implements InvocationHandler
// invoke() 根据 方法 → MappedStatement → 执行 SQL → 映射结果
```

## 10. 常见坑

1. JDK 代理只能代理接口（目标类强转失败）。
2. CGLIB 无法代理 final 类/方法。
3. 自调用导致 AOP 失效。
4. `InvocationHandler` 中未剥离 `InvocationTargetException`（异常类型被包装）。
5. 代理对象的 `equals/hashCode` 被拦截（可能出错）。
6. 在 `invoke` 中调用 `proxy.method()` 导致无限递归（应调用 `target`）。
7. Spring 中 `@Autowired` 注入的是代理，`getClass()` 返回代理类（判类型用 `AopUtils`）。
8. 代理对象的字段为 null（CGLIB 子类未走构造器时）。
9. 大量动态生成类导致 Metaspace 增长。
10. 认为 CGLIB 一定比 JDK 快（JDK 8+ 已接近）。

```java
// 判断/获取真实类型
if (AopUtils.isAopProxy(bean)) {
    Class<?> targetClass = AopUtils.getTargetClass(bean);
}
if (AopUtils.isJdkDynamicProxy(bean)) { }
if (AopUtils.isCglibProxy(bean)) { }
```

## 11. 速查

| 需求 | 做法 |
|---|---|
| 代理接口 | `Proxy.newProxyInstance` |
| 代理类 | CGLIB `Enhancer` / ByteBuddy |
| 拦截逻辑 | `InvocationHandler.invoke` |
| Spring AOP | 自动代理，注意自调用 |
| 强制 CGLIB | `spring.aop.proxy-target-class=true` |
| 获取真实类 | `AopUtils.getTargetClass` |
| 高性能字节码 | ByteBuddy / ASM |
