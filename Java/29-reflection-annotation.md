# 29 · 反射与注解

## 一、反射：运行时审查与操作类

```java
// 获取 Class 对象三种方式
Class<User> c1 = User.class;
Class<?> c2 = user.getClass();
Class<?> c3 = Class.forName("com.example.User");

// 成员探查
c.getName(); c.getSimpleName();
c.getDeclaredFields();       // 所有字段（含私有）
c.getFields();               // public（含继承）
c.getDeclaredMethods();
c.getDeclaredConstructors();
c.getSuperclass(); c.getInterfaces();
c.getAnnotations();

// 实例化
User u = c.getDeclaredConstructor().newInstance();   // Java 9+ 写法
User u2 = c.getConstructor(String.class).newInstance("tom");

// 访问字段（含私有）
Field f = c.getDeclaredField("name");
f.setAccessible(true);            // 破开 private（安全管理器/模块系统下受限）
Object val = f.get(u);
f.set(u, "newName");

// 调用方法
Method m = c.getDeclaredMethod("setName", String.class);
m.setAccessible(true);
m.invoke(u, "jerry");

// 泛型信息（擦除后仍可从声明处拿到）
ParameterizedType pt = (ParameterizedType) c.getGenericSuperclass();
pt.getActualTypeArguments()[0];   // 父类的 T 实参（MyBatis/JPA 基础设施原理）
```

## 二、反射的应用场景与代价

```
场景：框架基础设施（Spring IoC 按 Bean 定义实例化、MyBatis 映射、
      JSON 字段映射、JUnit 发现 @Test、依赖注入）
代价：比直接调用慢（JIT 优化受限）、绕过编译期检查、破坏封装、
      模块系统/原生镜像下需显式开放
```

**原则**：业务代码不写反射；写反射的是框架与工具。原生镜像下反射类要注册 hints（衔接 SpringBoot 系列 34 篇）。

## 三、注解

### 内置注解

```java
@Override            // 编译期校验重写
@Deprecated          // 标记废弃
@SuppressWarnings("unchecked")   // 压制警告
@SafeVarargs @FunctionalInterface
```

### 元注解（注解的注解）

```java
@Target(ElementType.METHOD)          // 可标注位置：TYPE/FIELD/METHOD/PARAMETER/PACKAGE
@Retention(RetentionPolicy.RUNTIME)  // 保留策略：SOURCE(编译丢弃)/CLASS(默认)/RUNTIME(反射可读)
@Documented                          // 进 javadoc
@Inherited                           // 子类继承
@Repeatable                          // 可重复标注
```

### 自定义注解 + 反射消费（框架原理闭环）

```java
// 1. 定义
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Audit {
    String action();
    boolean required() default true;
}

// 2. 标注
class OrderService {
    @Audit(action = "createOrder")
    public void create(Order o) { }
}

// 3. 运行时读取（AOP/拦截器/调度器的原理）
Method m = OrderService.class.getMethod("create", Order.class);
if (m.isAnnotationPresent(Audit.class)) {
    Audit audit = m.getAnnotation(Audit.class);
    audit.action();        // "createOrder"
    audit.required();
}
```

## 四、动态代理：反射的集大成应用

### JDK 动态代理（接口代理）

```java
interface PayService { void pay(String order); }

class Handler implements InvocationHandler {
    private final Object target;
    Handler(Object t) { target = t; }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        log.info("before {}", method.getName());       // 前置增强
        Object result = method.invoke(target, args);   // 委托真实对象
        log.info("after {}", method.getName());
        return result;
    }
}

PayService proxy = (PayService) Proxy.newProxyInstance(
        loader, new Class[]{PayService.class}, new Handler(real));
proxy.pay("O001");       // 每次调用都走 invoke
```

### CGLIB 子类代理（无接口）

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(OrderService.class);       // 生成子类
enhancer.setCallback((MethodInterceptor) (obj, method, args, proxy) -> {
    log.info("before");
    Object r = proxy.invokeSuper(obj, args);      // 调父类（原方法）
    log.info("after");
    return r;
});
OrderService p = (OrderService) enhancer.create();
```

**Spring AOP 选择**：目标有接口→JDK 代理，无接口→CGLIB（Boot 默认 CGLIB）。这也解释了「内部自调用不走代理」的切面/事务失效（衔接 SpringBoot 系列 14/17 篇）。

## 五、注解处理器（编译期）

```java
// Lombok 就是注解处理器：编译期改 AST，生成 getter/setter（运行时零反射）
@Getter @Setter @Builder
public class UserDTO { ... }

// 自研注解处理器需 javax.annotation.processing.AbstractProcessor（工具链场景）
```

**RUNTIME vs 编译期处理**：需要运行时行为（AOP/发现）→ RUNTIME 注解；代码生成 → 注解处理器（零运行时开销）。

## 六、实战：注解驱动的路由（迷你框架思路）

```java
@Target(METHOD) @Retention(RUNTIME)
@interface Route { String path(); }

class Dispatcher {
    private final Map<String, Method> routes = new HashMap<>();

    void scan(Object controller) {          // 启动时注册
        for (Method m : controller.getClass().getDeclaredMethods()) {
            if (m.isAnnotationPresent(Route.class)) {
                routes.put(m.getAnnotation(Route.class).path(), m);
            }
        }
    }

    void dispatch(String path, Object... args) throws Exception {   // 请求时反射调用
        routes.get(path).invoke(controller, args);
    }
}
```

## 常见坑

- **坑 1**：@Retention 用默认 CLASS → 反射读不到注解；运行时消费必须 RUNTIME
- **坑 2**：反射调用性能敏感热路径 → 缓存 Method/Field 对象；或 MethodHandles
- **坑 3**：setAccessible 绕过模块封装 → JPMS 下 InaccessibleObjectException
- **坑 4**：getAnnotation 继承误判 → @Inherited 只对类注解生效，方法注解不继承
- **坑 5**：泛型反射拿不到 T 的运行时类型 → 从 genericSuperclass/参数 Class<T> 传
- **坑 6**：代理对象内的 this 调用 → 增强失效（自调用）

## Checklist

- [ ] Class 对象三获取方式与成员探查 API 熟练
- [ ] 自定义注解三要素：@Target/@Retention(RUNTIME)/属性
- [ ] JDK vs CGLIB 代理区别与 Spring AOP 的关联
- [ ] 反射只出现在框架/工具层，热路径缓存元数据
