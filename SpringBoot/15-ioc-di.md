# 15 · IoC 容器与依赖注入

## 一、核心理念

**IoC（控制反转）**：对象的创建与依赖装配交给容器。你只需要声明「我需要什么」，容器负责「给你什么」。

**DI（依赖注入）是实现手段**：容器把依赖注入到 Bean 中。

```java
// 传统：自己 new，强耦合
UserService s = new UserService(new UserMapper(...));

// IoC：声明依赖，容器装配
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserMapper userMapper;   // 构造器注入
}
```

## 二、注册 Bean 的方式

| 方式 | 场景 |
|---|---|
| @Component / @Service / @Repository / @Controller | 自己写的分层类 |
| @Configuration + @Bean | 第三方对象（如 RestTemplate、拦截器链） |
| @Import(XxxConfig.class) | 引入配置类 |
| @ComponentScan | 批量扫描（启动类默认） |
| spring.factories / AutoConfiguration.imports | 自动配置（35 篇） |

**注解语义区分**（本质都是 @Component）：
- @Service：业务层
- @Repository：数据层，附带异常翻译（SQLException → DataAccessException 体系）
- @Controller/@RestController：Web 层

## 三、三种注入方式

```java
// 1. 构造器注入（唯一推荐）—— 可 final、可空指针排查、便于单测
@Service
@RequiredArgsConstructor           // Lombok 生成构造器
public class OrderService {
    private final OrderMapper orderMapper;
    private final PayService payService;
}

// 2. Setter 注入 —— 可选依赖场景
@Autowired
public void setPayService(PayService p) { ... }

// 3. 字段注入 —— 不推荐：无法 final、隐藏依赖、单测要反射
@Autowired
private PayService payService;
```

## 四、注入与歧义消解

多个同类型 Bean 时：

```java
@Bean public PayService alipayPayService() { ... }
@Bean public PayService wechatPayService() { ... }

// 方式 1：@Primary 默认
// 方式 2：@Qualifier("wechatPayService") 按名注入
// 方式 3：@Resource(name = "wechatPayService")（JSR-250，先按名后按类型）
// 方式 4：集合注入，拿全部实现
private final Map<String, PayService> payServices;      // key = beanName
private final List<PayService> allPayServices;

// 策略模式首选：Map 注入按渠道 key 路由
payServices.get("alipay").pay(order);
```

## 五、条件装配

```java
@Bean
@ConditionalOnProperty(name = "app.sms.enabled", havingValue = "true")
public SmsClient smsClient() { ... }

@ConditionalOnMissingBean    // 自动配置标配：用户自定义了就不重复注册
@ConditionalOnClass(Redisson.class)
@Profile("prod")
```

## 六、Bean 作用域

| 作用域 | 说明 |
|---|---|
| singleton（默认） | 容器内单例。**必须无状态** |
| prototype | 每次获取新建实例 |
| request / session | Web 场景 |

```java
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class TaskContext { ... }

// prototype 注入单例只会注入一个实例 → 需要时用 ObjectFactory
@Autowired
private ObjectFactory<TaskContext> taskContextFactory;
```

## 七、循环依赖

```
A 创建中 → 依赖 B → 创建 B → 依赖 A → ...
```

Spring 通过**三级缓存**解决单例 Setter/字段注入的循环依赖：
1. singletonObjects：成品 Bean
2. earlySingletonObjects：半成品（提前暴露引用）
3. singletonFactories：工厂（生成早期引用，处理 AOP 代理）

**限制**：
- 构造器注入循环依赖无法解决（尚未实例化完，无法提前暴露）→ `BeanCurrentlyInCreationException`
- prototype 循依赖无法解决

**正确姿势**：出现循环依赖优先重构（职责拆分），实在不行用 `@Lazy`：

```java
public OrderService(@Lazy PayService payService) { ... }
```

## 八、获取容器中的 Bean（工具类场景）

```java
@Component
public class SpringContextHolder implements ApplicationContextAware {
    private static ApplicationContext ctx;
    @Override
    public void setApplicationContext(ApplicationContext c) { ctx = c; }
    public static <T> T getBean(Class<T> clazz) { return ctx.getBean(clazz); }
}
```

> 慎用：getBean 是服务定位器反模式，只在框架级代码（如 SPI 路由）使用。

## 常见坑

- **坑 1**：字段注入的类做单元测试无法构造 → 构造器注入是可测试性的基石
- **坑 2**：单例 Bean 保存可变成员变量 → 并发污染；状态放方法参数/ThreadLocal
- **坑 3**：@Configuration 类中 @Bean 方法互相调用出现意外单例/多例 → @Configuration 是 CGLIB 代理的，方法调用被拦截返回同一单例
- **坑 4**：找不到 Bean → 类不在扫描包内 / 没加注解 / 条件装配没命中

## Checklist

- [ ] 统一构造器注入 + final 字段
- [ ] 单例 Bean 无状态
- [ ] 多实现用 Map 集合注入实现策略路由
- [ ] 循环依赖通过重构消除而非 @Lazy 兜底
