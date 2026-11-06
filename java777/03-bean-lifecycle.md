# 03 · Bean 作用域与生命周期

## 1. 六种作用域

| 作用域 | 说明 | 适用 |
|---|---|---|
| `singleton` | 每容器一个实例（**默认**） | 无状态服务 |
| `prototype` | 每次 getBean 新建 | 有状态对象 |
| `request` | 每个 HTTP 请求 | Web（需 WebApplicationContext） |
| `session` | 每个会话 | Web |
| `application` | 整个 ServletContext | Web |
| `websocket` | 每个 WebSocket 会话 | Web |

```java
@Scope("prototype")
@Component
public class ReportContext { /* 有状态 */ }

// 注入 prototype 到 singleton：默认只会注入一次！用代理解决
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
```

> Web 作用域在非 Web 环境会报 `No Scope registered for scope name 'request'`。

## 2. 完整生命周期（单例）

```
① 实例化            createBeanInstance（构造器反射）
② 属性填充          populateBean（依赖注入，可能发生循环依赖）
③ Aware 回调        BeanNameAware / BeanFactoryAware / ApplicationContextAware
④ 前置处理          BeanPostProcessor#postProcessBeforeInitialization
                     → @PostConstruct
⑤ 初始化            InitializingBean#afterPropertiesSet()
                     → 自定义 init-method / @Bean(initMethod=)
⑥ 后置处理          BeanPostProcessor#postProcessAfterInitialization
                     → ★ AOP 代理在此生成
⑦ 就绪              放入单例池 singletonObjects，可被使用
────────────────────────────────
⑧ 销毁（容器关闭）   @PreDestroy → DisposableBean#destroy() → destroy-method
```

```java
@Component
public class LifecycleDemo implements InitializingBean, DisposableBean {
    @PostConstruct  void init1() { System.out.println("@PostConstruct"); }
    @Override public void afterPropertiesSet() { System.out.println("InitializingBean"); }
    @PreDestroy void d1() { System.out.println("@PreDestroy"); }
    @Override public void destroy() { System.out.println("DisposableBean"); }
}
// 输出顺序：@PostConstruct → InitializingBean → ... → @PreDestroy → DisposableBean
```

## 3. 三种初始化方式对比

| 方式 | 优点 | 缺点 |
|---|---|---|
| `@PostConstruct` | 标准注解，与框架解耦 | 无参数 |
| `InitializingBean` | 简单 | 耦合 Spring |
| `@Bean(initMethod=)` | 适用第三方类 | 需记住方法名 |

推荐：**`@PostConstruct`**（Java 标准）+ 构造器注入。

## 4. prototype 的销毁陷阱

> `prototype` Bean 创建后**容器不再管理其生命周期**——`@PreDestroy` **不会被调用**。
> 需要手动释放资源，或用 `ObjectProvider` + 显式清理。

## 5. 懒加载 @Lazy

```java
@Lazy            // 首次使用时才创建（不是容器启动时）
@Component
public class HeavyResource { }
```

用途：加快启动速度；**打破构造器循环依赖**（一方加 `@Lazy` 注入代理）。

## 6. 生命周期钩子实战

### 应用级启动/关闭回调

```java
@Component
public class StartupRunner implements ApplicationRunner {
    @Override public void run(ApplicationArguments args) {
        // 容器完全就绪后执行：预热缓存、校验配置
    }
}
// 也可用 CommandLineRunner（参数为原始 String[]）
```

### 容器事件

```java
@Component
public class ReadyListener {
    @EventListener(ApplicationReadyEvent.class)
    public void onReady() { System.out.println("服务可对外提供流量"); }
}
```

事件顺序：`ApplicationStartingEvent` → `EnvironmentPreparedEvent` → `ContextRefreshedEvent` → `ApplicationStartedEvent` → **`ApplicationReadyEvent`** → `ApplicationFailedEvent`（异常时）。

> 需要「预热完成后才注册到注册中心」的场景，用 `ApplicationReadyEvent`，而非 `ContextRefreshedEvent`。

## 7. BeanPostProcessor 实战：自定义初始化标记

```java
public interface Warmable { void warmUp(); }

@Component
public class WarmUpProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String name) {
        if (bean instanceof Warmable w) w.warmUp();
        return bean;
    }
}
```

## 8. 单例池与三级缓存（循环依赖核心）

```java
// DefaultSingletonBeanRegistry
Map<String, Object> singletonObjects;             // 一级：完整 Bean
Map<String, Object> earlySingletonObjects;         // 二级：提前暴露（已从三级升级）
Map<String, ObjectFactory<?>> singletonFactories;  // 三级：ObjectFactory（可生成代理）
Set<String> singletonsCurrentlyInCreation;         // 正在创建中
```

流程：`getSingleton` 依次查一级 → 二级 → 三级；三级拿到 `ObjectFactory.getObject()` 后升级到二级并移除三级。

> 详细循环依赖分析见 [04-dependency-injection.md](04-dependency-injection.md)。

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 单例里放状态 | 并发数据串号 | 单例 Bean 必须无状态，状态放方法局部变量或 DB/Redis |
| prototype 的销毁回调不执行 | 资源泄漏 | 手动释放 |
| `@PostConstruct` 里调远程服务超时 | 启动卡死 | 改为异步或 `ApplicationReadyEvent` 后执行 |
| `@Scope` 与 AOP 混用导致代理类型不符 | `BeanNotOfRequiredTypeException` | 注入接口而非实现类，或用 `TARGET_CLASS` 代理 |
| 循环依赖 + AOP | `BeanCurrentlyInCreationException`（构造器注入时） | 改字段/Setter 注入或 `@Lazy` |

## 10. 面试速答

- **Bean 生命周期**：实例化 → 属性注入 → Aware → @PostConstruct → InitializingBean → init-method → BeanPostProcessor 后置（AOP 代理）→ 使用 → @PreDestroy → DisposableBean。
- **单例 Bean 线程安全吗**：不安全。Spring 只保证 Bean 唯一，不保证线程安全，需自行避免共享可变状态。
- **销毁回调什么时候执行**：单例在容器 `close()`（Boot 为 JVM shutdown hook）时；prototype 不执行。
