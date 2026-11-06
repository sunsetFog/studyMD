# 01 · Spring 生态全景与 IoC/DI 思想

## 1. Spring 是什么

Spring 不是「一个框架」，而是一个**分层的技术栈**，核心是让 Java 企业级开发更简单、松耦合、可测试。

| 层次 | 模块 | 作用 |
|---|---|---|
| 核心容器 | spring-core / spring-beans / spring-context / spring-expression | IoC 容器、SpEL |
| AOP | spring-aop / spring-aspects | 切面、代理 |
| 数据访问 | spring-jdbc / spring-tx / spring-orm | JdbcTemplate、事务抽象、ORM 整合 |
| Web | spring-web / spring-webmvc / spring-webflux | Servlet 栈与响应式栈 |
| 集成 | spring-messaging / spring-jms / spring-amqp | 消息 |
| 测试 | spring-test | MockMvc、上下文测试 |
| 上层 | Spring Boot / Spring Cloud / Security / Data / Batch | 全家桶 |

```xml
<!-- 仅核心容器（非 Boot 项目） -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.1.6</version>
</dependency>
```

## 2. 为什么需要 IoC

### 传统写法（对象自己 new 依赖，硬耦合）

```java
public class OrderService {
    // 直接依赖具体实现，无法替换、无法单测
    private OrderDao dao = new MySqlOrderDao();
}
```

问题：实现类一换就要改源码；单元测试无法注入 mock；生命周期（单例、池化）无法统一管理。

### IoC 写法（容器负责创建与装配）

```java
@Service
public class OrderService {
    private final OrderDao dao;          // 只依赖抽象
    public OrderService(OrderDao dao) { this.dao = dao; }  // 构造器注入
}
```

> **IoC（Inversion of Control，控制反转）**：把「对象的创建与依赖装配」控制权从业务代码反转给容器。
> **DI（Dependency Injection，依赖注入）**：IoC 的具体实现方式——容器把依赖「注入」进去。

## 3. 三种依赖注入方式对比

| 方式 | 写法 | 优点 | 缺点 |
|---|---|---|---|
| 构造器注入 | 构造方法参数 | 依赖不可变（`final`）、**可发现循环依赖**、便于单测 | 参数多时略冗长 |
| Setter 注入 | `setXxx()` | 可选依赖、可重新配置 | 对象可能处于不完整状态 |
| 字段注入 | `@Autowired` 字段 | 最省代码 | **不推荐**：无法 final、难单测、掩盖循环依赖、NPE 隐患 |

```java
// 推荐：构造器 + final（Spring 4.3+ 单构造器时可省略 @Autowired）
@Service
public class PayService {
    private final AlipayClient client;
    private final RiskEngine engine;
    public PayService(AlipayClient client, RiskEngine engine) {
        this.client = client; this.engine = engine;
    }
}
```

## 4. Spring 的三种配置方式

| 方式 | 形态 | 现状 |
|---|---|---|
| XML | `<bean id=".." class=".."/>` | 遗留项目多，新项目基本不用 |
| Java Config | `@Configuration` + `@Bean` | 主流，类型安全 |
| 注解扫描 | `@Component` + `@ComponentScan` | 主流，配合 Java Config |

```java
@Configuration                 // 本身就是 @Component
@ComponentScan("com.example")  // 扫描 @Component/@Service/@Repository/@Controller
@PropertySource("classpath:app.properties")
public class AppConfig {

    @Bean                       // 方法名即 Bean 名
    @Scope("prototype")         // 默认 singleton
    public ObjectMapper objectMapper() {
        return JsonMapper.builder()
                .addModule(new JavaTimeModule())
                .build();
    }
}

// 启动
var ctx = new AnnotationConfigApplicationContext(AppConfig.class);
OrderService s = ctx.getBean(OrderService.class);
```

## 5. 容器接口体系

```
BeanFactory (最基础：getBean / containsBean / isSingleton)
   └── ListableBeanFactory（枚举所有 Bean）
         └── ApplicationContext（= BeanFactory + 企业级能力）
                ├── AnnotationConfigApplicationContext
                ├── ClassPathXmlApplicationContext
                └── WebApplicationContext
```

ApplicationContext 额外能力：国际化 `MessageSource`、资源加载 `ResourceLoader`、事件发布 `ApplicationEventPublisher`、环境配置 `Environment`。

## 6. 第一个完整例子

```java
public interface MessageSender { void send(String to, String text); }

@Repository
class SmsSender implements MessageSender {
    public void send(String to, String text) { System.out.println("SMS->" + to + ":" + text); }
}

@Service
class NotifyService {
    private final MessageSender sender;
    NotifyService(MessageSender sender) { this.sender = sender; }
    public void notifyUser() { sender.send("138****", "验证码 9527"); }
}

public class Main {
    public static void main(String[] args) {
        var ctx = new AnnotationConfigApplicationContext("com.example");
        ctx.getBean(NotifyService.class).notifyUser();
        ctx.close();  // 触发销毁回调
    }
}
```

## 7. Spring 5/6 的重要演进

| 版本 | 关键变化 |
|---|---|
| 4.x | Java 8 支持、条件注解 `@Conditional`、泛型注入 |
| 5.x | WebFlux 响应式、函数式 Bean 注册、Kotlin 支持 |
| 6.x | **Java 17 基线**、**`javax.*` → `jakarta.*`**、AOT 与 GraalVM 原生镜像、ProblemDetail（RFC 7807）、虚拟线程集成 |

## 8. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| Bean 扫不到 | `NoSuchBeanDefinitionException` | 启动类包路径必须覆盖业务包（Boot 默认扫描启动类所在包及其子包） |
| 字段注入 NPE | 单元测试 `new XxxService()` 后调用报 NPE | 改用构造器注入，测试里手动传 mock |
| 接口多实现冲突 | `NoUniqueBeanDefinitionException` | `@Qualifier("beanName")` 或 `@Primary` |
| 误用 `new` | `new Service()` 得到的对象的 `@Transactional` 不生效 | 必须取容器里的代理对象 |
| `javax` 包找不到 | Boot 3 启动报 `ClassNotFoundException: javax.servlet.*` | 全部替换为 `jakarta.servlet.*` |

## 9. 面试速答

- **IoC 和 DI 的关系**：IoC 是设计思想（控制权反转），DI 是实现手段（容器注入依赖）。
- **Spring 如何解耦**：通过反射 + 配置元数据（注解/XML）创建 Bean，按类型/名称装配，业务代码只面向接口。
- **为什么推荐构造器注入**：保证依赖不可变与完全初始化、能在启动时暴露循环依赖、无需反射即可单测。
