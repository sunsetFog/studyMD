# 02 · 核心容器：BeanFactory、ApplicationContext 与 BeanDefinition

## 1. 容器全景

```
ApplicationContext
  ├── BeanFactory          获取 Bean
  ├── ResourceLoader       加载 classpath/file/URL 资源
  ├── MessageSource        i18n
  ├── ApplicationEventPublisher  事件发布
  └── Environment          profiles + properties
```

实现类：

| 实现 | 场景 |
|---|---|
| `AnnotationConfigApplicationContext` | 纯注解（普通 Java 应用） |
| `ClassPathXmlApplicationContext` | XML（遗留项目） |
| `AnnotationConfigWebApplicationContext` | 传统 Web（Boot 内部用） |
| `GenericApplicationContext` | 编程式注册，轻量、Boot 大量使用 |

## 2. BeanDefinition：Bean 的「图纸」

容器不是直接 new 对象，而是先把类解析成 `BeanDefinition`（元数据），再按定义创建实例。

```java
// 编程式注册（Boot 自动配置底层就是这么干的）
GenericApplicationContext ctx = new GenericApplicationContext();
ctx.registerBean("userService", UserService.class,
        bd -> bd.setScope(BeanDefinition.SCOPE_PROTOTYPE));
ctx.refresh();     // 必须 refresh，触发实例化所有非懒加载单例
```

常用属性：

| 属性 | 含义 |
|---|---|
| `scope` | singleton / prototype / request / session / application |
| `lazyInit` | 是否懒加载 |
| `primary` | 多实现时的首选 |
| `dependsOn` | 强制初始化顺序（不解决循环依赖） |
| `initMethod` / `destroyMethod` | 生命周期回调 |
| `autowireMode` | byType / byName / constructor |

> **`@Bean` 方法、`@Component` 类、XML `<bean>` 最终都会变成 BeanDefinition**，这是 Spring 可扩展的根本。

## 3. BeanFactory vs FactoryBean（高频面试题）

| 名称 | 角色 |
|---|---|
| `BeanFactory` | **容器本身**，负责管理所有 Bean |
| `FactoryBean<T>` | **一个特殊 Bean**，它生产的不是自己，而是 `getObject()` 的返回值 |

```java
@Component
public class MyClientFactoryBean implements FactoryBean<MyClient> {
    @Override public MyClient getObject() { return new MyClient(...); }  // 复杂构建逻辑
    @Override public Class<?> getObjectType() { return MyClient.class; }
    @Override public boolean isSingleton() { return true; }
}
// 取产品：ctx.getBean("myClientFactoryBean") → MyClient
// 取工厂本身：ctx.getBean("&myClientFactoryBean")
```

典型应用：MyBatis 的 `MapperFactoryBean`（为接口生成代理）、Feign 的 `FeignClientFactoryBean`。

## 4. 容器启动流程（refresh 十二步，简化版）

```
1  prepareRefresh          准备环境、校验必填属性
2  obtainFreshBeanFactory  创建 BeanFactory、加载 BeanDefinition
3  prepareBeanFactory      注册环境 Bean、后置处理器
4  postProcessBeanFactory  子类扩展（Web 上下文注册作用域）
5  invokeBeanFactoryPostProcessors   ★ 执行 BeanDefinitionRegistryPostProcessor
                                     （ConfigurationClassPostProcessor 在这里解析 @Configuration/@Import/@Bean）
6  registerBeanPostProcessors        ★ 注册 BeanPostProcessor（AOP、@Autowired 处理器）
7  initMessageSource        i18n
8  initApplicationEventMulticaster   事件广播器
9  onRefresh               子类：Web 服务器启动（Tomcat）、WebFlux
10 registerListeners        注册监听器
11 finishBeanFactoryInitialization ★ 实例化所有非懒加载单例（getBean 循环）
12 finishRefresh           发布 ContextRefreshedEvent、Lifecycle 启动
```

记住 **5 / 6 / 11** 三步即可覆盖 90% 面试题。

## 5. 后置处理器（Spring 的扩展灵魂）

| 接口 | 作用时机 | 典型实现 |
|---|---|---|
| `BeanDefinitionRegistryPostProcessor` | 所有 BeanDefinition 注册前 | `ConfigurationClassPostProcessor` |
| `BeanFactoryPostProcessor` | BeanDefinition 加载后、实例化前 | 修改属性占位符 |
| `BeanPostProcessor#postProcessBeforeInitialization` | init 前 | `@PostConstruct` 处理 |
| `BeanPostProcessor#postProcessAfterInitialization` | init 后 | **AOP 代理在这里生成** |
| `InstantiationAwareBeanPostProcessor` | 实例化前后 | 属性注入（`AutowiredAnnotationBeanPostProcessor`） |

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String name) {
        if (bean instanceof MyAware a) a.setContainerName(name);
        return bean;   // ★ 一定要返回 bean，返回 null 会截断链条
    }
}
```

## 6. 常用 Aware 接口

| 接口 | 注入内容 |
|---|---|
| `ApplicationContextAware` | 上下文（不推荐，耦合容器） |
| `EnvironmentAware` | 环境变量 |
| `BeanNameAware` | Bean 名称 |
| `ResourceLoaderAware` | 资源加载器 |

> 能用 DI 注入的（如 `Environment`）就不要用 Aware，减少与 Spring 的耦合。

## 7. 获取 Bean 的几种方式

```java
@Autowired ApplicationContext ctx;
ctx.getBean(UserService.class);           // 按类型
ctx.getBean("userService", UserService.class);
ctx.getBeansOfType(MessageSender.class);  // 策略模式：一次拿到所有实现
ctx.getBeanProvider<Foo>(Foo.class);      // 延迟/多实例获取（ObjectProvider）
```

**策略模式实战**（企业常用）：

```java
@Service
public class PayRouter {
    private final Map<String, PayHandler> handlers;
    public PayRouter(Map<String, PayHandler> handlers) { this.handlers = handlers; } // key=beanName
    public void pay(String channel, BigDecimal amt) {
        handlers.get(channel).handle(amt);
    }
}
// 定义：@Component("alipay") class AliPayHandler implements PayHandler {}
```

## 8. 环境与环境隔离

```java
Environment env = ctx.getEnvironment();
env.getActiveProfiles();                 // 激活的 profile
env.getProperty("server.port", Integer.class, 8080);
ctx.getEnvironment().setActiveProfiles("dev");
```

资源加载：

```java
Resource r = new ClassPathResource("config/app.yml");
Resource r2 = ctx.getResource("classpath:data/*.json");   // 支持 ant 通配符
```

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| `refresh()` 后注册 Bean 无效 | 无报错但不生效 | 必须在 refresh 前注册，或用 `registerBean` + 手动 refresh |
| `BeanCurrentlyInCreationException` | 构造器循环依赖 | 改 Setter/字段注入，或抽取第三方类，或 `@Lazy` |
| `BeanPostProcessor` 返回 null | 后续 Bean 全变 null，NPE 满天飞 | 必须 `return bean` |
| 在 `BeanPostProcessor` 里 `getBean` | 导致目标 Bean 过早初始化、跳过 AOP | 延迟到 `afterSingletonsInstantiated` |
| `getBean` 拿到的是原始对象 | `@Transactional` 失效 | 说明该 Bean 未被代理，检查是否自调用或 final 类 |

## 10. 面试速答

- **BeanFactory 与 ApplicationContext 区别**：前者是基础容器，后者在其上加了 i18n、事件、资源、环境，且默认预实例化单例（前者懒加载）。
- **BeanDefinition 是什么**：描述 Bean 的元数据（类、作用域、初始化方法、构造参数等），是容器创建对象的依据。
- **FactoryBean 用途**：封装复杂对象的创建（如 MyBatis Mapper 代理、Feign 客户端），取产品用 beanName，取工厂用 `&beanName`。
