# 04 · 依赖注入详解与循环依赖三级缓存

## 1. @Autowired 的装配规则

```
1. 按类型（byType）在容器中查找候选
2. 候选数 == 0 → 抛 NoSuchBeanDefinitionException（除非 required=false）
3. 候选数 == 1 → 直接注入
4. 候选数 > 1 → 按字段名/参数名（byName）匹配；仍冲突则报错
```

```java
@Autowired(required = false)        // 找不到就注入 null（谨慎使用）
private Optional<Metrics> metrics;  // 也可用 Optional 表达可选

@Autowired @Qualifier("aliPay")     // 指定 Bean 名
private PayHandler handler;
```

## 2. @Resource vs @Autowired vs @Inject

| 注解 | 来源 | 默认方式 | 支持 required | 支持 @Primary |
|---|---|---|---|---|
| `@Autowired` | Spring | byType → byName | 是 | 是 |
| `@Resource` | JSR-250（JDK） | byName → byType | 否 | 否 |
| `@Inject` | JSR-330 | byType | 否 | 是（@Qualifier） |

> 建议统一用 **构造器注入**（可省注解）；必须字段注入时用 `@Resource` 可避免与 Spring 强耦合。

## 3. 集合注入与 @Primary

```java
@Primary                        // 多实现时的默认选择
@Component("defaultPay")
class DefaultPay implements PayHandler {}

@Autowired List<PayHandler> all;      // 注入所有实现，按 @Order 排序
@Autowired Map<String, PayHandler> m; // key = beanName
```

`@Order` / `Ordered` 控制集合注入顺序（值越小越靠前）。

## 4. 循环依赖

### 三种场景

| 场景 | Spring 能否解决 |
|---|---|
| 字段注入 / Setter 注入的循环依赖 | ✅ 能（三级缓存） |
| 构造器注入的循环依赖 | ❌ 不能，直接抛异常 |
| prototype 之间的循环依赖 | ❌ 不能（不缓存） |

```java
// 失败案例：构造器循环
@Service class A { A(B b) {} }
@Service class B { B(A a) {} }
// 启动时报：
// BeanCurrentlyInCreationException: Error creating bean with name 'a':
// Requested bean is currently in creation: Is there an unresolvable circular reference?
```

## 5. 三级缓存原理（源码级）

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object o = singletonObjects.get(beanName);            // 一级：成品
    if (o == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (singletonObjects) {
            o = earlySingletonObjects.get(beanName);      // 二级：半成品
            if (o == null && allowEarlyReference) {
                ObjectFactory<?> f = singletonFactories.get(beanName); // 三级：工厂
                if (f != null) {
                    o = f.getObject();                    // ★ 可能返回 AOP 代理
                    earlySingletonObjects.put(beanName, o);
                    singletonFactories.remove(beanName);
                }
            }
        }
    }
    return o;
}
```

### 完整时序（A ↔ B，字段注入）

```
1 getBean(A) → 实例化 A（构造器）→ 未完成
2 addSingletonFactory(A, () -> getEarlyBeanReference(A))   ★ 三级缓存放入工厂
3 populateBean(A)：发现需要 B
4   getBean(B) → 实例化 B → addSingletonFactory(B)
5   populateBean(B)：发现需要 A
6     getSingleton(A)：一级无 → 三级有 → 执行工厂拿到 A 的早期引用（可能是代理）
7     B 拿到 A（半成品），B 完成初始化 → 放入一级缓存
8 A 继续注入 B → A 完成 → 放入一级缓存
```

### 为什么需要三级而不是两级？

- 二级也能解决普通循环依赖，**但无法正确处理 AOP**。
- 若 A 需要被代理，B 注入的必须是 **A 的代理对象**。三级缓存存的 `ObjectFactory`（`getEarlyBeanReference`）内部会调用 `SmartInstantiationAwareBeanPostProcessor`，在「提前暴露」时就生成代理，保证 B 注入的是代理。
- 两级缓存方案需要「无论是否循环都提前创建代理」，违背「代理在初始化后创建」的设计。

> 关键源码类：`DefaultSingletonBeanRegistry#getSingleton`、`AbstractAutowireCapableBeanFactory#getEarlyBeanReference`、`AbstractAutoProxyCreator`。

## 6. 关闭循环依赖支持

```java
SpringApplicationBuilder b = new SpringApplicationBuilder(App.class);
b.initializers(...);
// Boot 2.6+ 默认已关闭；也可配置文件关闭：
```

```yaml
spring:
  main:
    allow-circular-references: false   # Boot 2.6+ 默认 false
    allow-bean-definition-overriding: false   # 禁止 Bean 覆盖
```

> 关闭后任何循环依赖都会启动失败——这是一件**好事**，它逼迫你改善设计。

## 7. 解决循环依赖的正确姿势

| 方案 | 说明 |
|---|---|
| 抽取第三方 | A、B 共同依赖的逻辑抽成 C，A→C、B→C |
| `@Lazy` | 注入方加 `@Lazy`，注入代理而非真实对象（打破即时解析） |
| 改成 Setter/字段注入 | 让三级缓存生效 |
| 事件解耦 | A 不直接依赖 B，改为发布事件由 B 监听 |
| `ApplicationContextAware` 延迟获取 | 运行时再 `getBean` |

```java
// 推荐：事件解耦
@Service class OrderService {
    private final ApplicationEventPublisher publisher;
    void create(Order o) { publisher.publishEvent(new OrderCreatedEvent(o)); }
}
@Service class CouponService {
    @EventListener void on(OrderCreatedEvent e) { /* 发券 */ }
}
```

## 8. 注入其他特殊类型

```java
@Value("${app.name:default}")  private String appName;   // 占位符 + 默认值
@Value("#{T(java.time.LocalDate).now()}") private LocalDate d;  // SpEL

@Autowired Environment env;
@Autowired ObjectProvider<Foo> provider;   // 延迟/多实例安全获取
@Autowired ObjectFactory<Bar> factory;
```

`@Lazy` + `ObjectProvider` 是打破循环依赖的最优雅组合。

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 构造器注入循环 | 启动失败 | 抽公共类 / `@Lazy` / 事件 |
| 同类自调用导致注入失效 | `@Transactional` 不回滚 | 注入自己（`@Lazy`）或 `AopContext.currentProxy()`，或拆 Service |
| `@Autowired` 静态字段 | 注入为 null | Spring 不注入 static；改用 `@PostConstruct` 赋值 |
| 泛型注入错误 | 类型擦除导致匹配错 | `ResolvableType` / 显式 `@Qualifier` |
| Bean 覆盖 | 同名 Bean 后注册覆盖前者 | 关闭 `allow-bean-definition-overriding`，重命名 |

## 10. 面试速答

- **Spring 如何解决循环依赖**：通过三级缓存 + 「提前暴露未完成初始化的对象引用」。一级存成品，二级存半成品，三级存 `ObjectFactory`（用于生成可能的 AOP 代理）。
- **为什么构造器注入无法解决**：对象还没实例化就无法放入三级缓存，依赖方无从获取引用。
- **三级缓存存在的意义**：在循环依赖场景下保证注入的是 AOP 代理对象，而非原始对象。
