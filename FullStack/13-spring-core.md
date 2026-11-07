# 13 - Spring / SpringBoot 核心

## 1. IoC 与 DI（必考⭐）

- IoC 控制反转：对象创建与依赖装配交给容器；DI 是实现方式
- 注入方式：构造器注入（**推荐**：不可变、可发现依赖缺失、便于测试）vs setter vs 字段（@Autowired 直接注字段，无法 final、隐藏依赖）
- @Autowired vs @Resource：前者 Spring 提供、byType + @Qualifier；后者 JDK 标准、默认 byName
- Bean 作用域：singleton（默认）/prototype/request/session/application；singleton 注入 prototype 用 ObjectProvider

## 2. Bean 生命周期（必考⭐⭐）

```
实例化（反射构造）→ 属性填充（依赖注入）
→ Aware 回调（BeanNameAware/BeanFactoryAware/ApplicationContextAware）
→ BeanPostProcessor#postProcessBeforeInitialization
→ 初始化：@PostConstruct → InitializingBean#afterPropertiesSet → init-method
→ BeanPostProcessor#postProcessAfterInitialization   ← AOP 代理在此生成！
→ 使用
→ 销毁：@PreDestroy → DisposableBean#destroy → destroy-method
```

高频考点：**代理对象在 afterInitialization 阶段由 AOP 后置处理器（AbstractAutoProxyCreator）创建**，所以容器里拿到的是代理。

## 3. AOP（必考⭐⭐）

- 原理：JDK 动态代理（接口）+ CGLIB 子类代理（无接口，Boot 2.x 起默认 CGLIB）
- 切面组成：Pointcut（execution 表达式）+ Advice（@Before/@After/@Around/@AfterReturning/@AfterThrowing）
- 执行顺序（@Around 包裹一切）：@Around → @Before → 方法 → @AfterReturning/@AfterThrowing → @After

**自调用失效（经典）**：

```java
@Service
public class OrderService {
    public void a() { this.b(); }        // b() 的事务/切面失效！
    @Transactional public void b() {}
}
// 原因：this 是原始对象不是代理；绕过代理直接调
// 解法：拆类 / 注入自身代理（ObjectProvider<OrderService>）/ AopContext.currentProxy()
```

## 4. 事务传播与失效（必考⭐⭐）

传播行为：REQUIRED（默认，有则加入）、REQUIRES_NEW（挂起当前开新事务）、NESTED（savepoint）、SUPPORTS/NOT_SUPPORTED/NEVER/MANDATORY

**@Transactional 失效场景清单**：
1. 方法非 public
2. 自调用（见上）
3. 异常被 try/catch 吞掉
4. 默认只回滚 RuntimeException/Error，受检异常不回滚 → `rollbackFor = Exception.class`
5. 多线程（事务绑定 ThreadLocal 连接，子线程不在事务内）
6. 传播行为配错（NOT_SUPPORTED）
7. 类没被 Spring 管理
8. 数据库引擎不支持事务（MyISAM）

底层：AOP 环绕 + `PlatformTransactionManager` + 连接绑定 ThreadLocal。

## 5. 循环依赖三级缓存（必考⭐⭐）

```
一级 singletonObjects     ：成品 Bean
二级 earlySingletonObjects：提前暴露的半成品（已实例化未填充属性）
三级 singletonFactories   ：ObjectFactory（可提前生成代理）
```

流程：A 实例化后把工厂放三级缓存 → 填充属性需要 B → B 创建 → B 注入 A（从三级缓存拿到 A 的早期引用，若是代理则升级到二级）→ B 完成 → A 完成。

- 为什么需要三级而非二级：保证**代理在正常生命周期生成**的同时支持循环依赖（工厂决定返回原对象还是代理）
- 无法解决：构造器循环依赖、prototype 循环依赖、`@Async` 代理场景

## 6. SpringBoot 自动配置（必考⭐⭐）

```
@SpringBootApplication = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
@EnableAutoConfiguration → AutoConfigurationImportSelector
→ 读取 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
→ 按条件装配：@ConditionalOnClass / @ConditionalOnMissingBean / @ConditionalOnProperty
→ @ConfigurationProperties 绑定配置
```

- 自定义 Bean 优先：`@ConditionalOnMissingBean` 让用户配置覆盖默认
- starter 三件套：autoconfigure 模块 + imports 文件 + 条件注解

## 7. SpringMVC 请求流程

```
DispatcherServlet → HandlerMapping（找 Handler）→ HandlerAdapter
→ 参数解析（ArgumentResolver）/消息转换（HttpMessageConverter：Jackson）
→ Controller → 返回值处理（ReturnValueHandler）→ 视图/JSON
→ 拦截器 preHandle → postHandle → afterCompletion
```

- Filter（Servlet 规范）vs Interceptor（Spring）vs AOP（方法级）：作用范围与粒度不同，见 SpringBoot 专题 08

## 常见追问

1. @Configuration 的 CGLIB 增强干什么？——拦截 @Bean 方法调用，保证单例语义（full mode）
2. BeanFactory vs ApplicationContext：后者是前者的超集（事件/国际化/AOP 集成）
3. Spring 事件机制：ApplicationEvent + @EventListener + @Async + 事务后发布（@TransactionalEventListener）

## Checklist

- [ ] Bean 生命周期 10 步能默写（AOP 生成时机必提）
- [ ] 事务失效 8 场景随口说 5 个
- [ ] 三级缓存为什么不是二级能讲清
