# 70 · Spring 面试速查清单

> 按主题组织的「一句话标准答案」，用于临阵速记。

## 一、IoC / DI

- **IoC**：把对象创建与依赖装配的控制权交给容器；**DI** 是 IoC 的实现方式（注入依赖）。
- **注入方式**：构造器（推荐，可 final、暴露循环依赖、易测）、Setter（可选依赖）、字段（不推荐）。
- **容器**：`BeanFactory` 基础；`ApplicationContext` 增加 i18n、事件、资源、环境。
- **BeanDefinition**：描述 Bean 的元数据（类、作用域、初始化方法…），是创建 Bean 的依据。
- **`BeanFactory` vs `FactoryBean`**：前者是容器；后者是特殊 Bean，产品是 `getObject()`，取工厂用 `&beanName`。
- **refresh 关键三步**：⑤ 执行 `BeanFactoryPostProcessor`（解析 @Configuration/@Import/@Bean）→ ⑥ 注册 `BeanPostProcessor` → ⑪ 实例化所有非懒加载单例。

## 二、Bean 生命周期与循环依赖

- **生命周期**：实例化 → 属性注入 → Aware → `@PostConstruct` → `InitializingBean` → `init-method` → `BeanPostProcessor` 后置（**AOP 代理在此生成**）→ 使用 → `@PreDestroy` → `DisposableBean` → `destroy-method`。
- **作用域**：singleton（默认）、prototype（不执行销毁）、request/session/application/websocket。
- **循环依赖**：字段/Setter 注入可解决（三级缓存）；**构造器注入不可**；prototype 不可。
- **三级缓存**：一级成品、二级半成品、三级 `ObjectFactory`；三级存在的意义是**保证注入 AOP 代理**。
- **解决方式**：抽公共类、`@Lazy`、改 Setter、事件解耦、`ObjectProvider`。

## 三、AOP

- **术语**：Aspect、JoinPoint、Pointcut、Advice、Weaving、Target、Proxy。
- **五种通知**：Before / AfterReturning / AfterThrowing / After / Around（Around 最强，必须 `proceed()`）。
- **JDK vs CGLIB**：JDK 基于接口，CGLIB 基于继承（不能 final）；**Spring 6/Boot 3 默认 CGLIB**。
- **失效场景**：同类自调用、非 public、final/static、未被容器管理。
- **自调用解决**：注入自身代理、拆类、`AopContext.currentProxy()`。
- **顺序**：`@Order` 值小的在外层先执行；事务默认最低优先级。

## 四、事务

- **ACID**：原子、一致、隔离、持久。隔离级别：RC / RR（MySQL 默认）/ Serializable。
- **传播行为**：REQUIRED（默认）、REQUIRES_NEW（新连接、互不影响）、NESTED（保存点）、SUPPORTS、NOT_SUPPORTED、MANDATORY、NEVER。
- **默认回滚**：`RuntimeException` / `Error`；**检查异常不回滚** → 写 `rollbackFor = Exception.class`。
- **失效 8 种**：非 public、final/static、自调用、吞异常、检查异常、引擎不支持、未被容器管理、跨线程。
- **原理**：`TransactionInterceptor` + 代理；连接绑定 ThreadLocal（`TransactionSynchronizationManager`）。
- **大事务优化**：事务内只做 DB、异步化、分批、远程调用移出、`afterCommit` 后发消息。

## 五、Spring Boot

- **三大特性**：自动配置、Starter 依赖管理、内嵌容器。
- **`@SpringBootApplication`** = `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`。
- **自动配置原理**：`AutoConfigurationImportSelector` 读取 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（Boot 3；Boot 2 用 `spring.factories`），经 `@Conditional*` 过滤后注册。
- **用户配置优先**：`DeferredImportSelector` 延迟执行 + `@ConditionalOnMissingBean`。
- **配置优先级**：命令行 > 环境变量 > 配置文件；同目录 properties > yml。
- **`@Value` vs `@ConfigurationProperties`**：后者支持松散绑定、校验、IDE 提示，推荐。

## 六、MVC / Web

- **参数注解**：`@PathVariable`、`@RequestParam`、`@RequestBody`、`@RequestHeader`、`@RequestPart`、`@ModelAttribute`。
- **Filter vs Interceptor vs AOP**：Servlet 层 / MVC 层（有 Handler、可注入 Bean）/ 方法层。
- **全局异常**：`@RestControllerAdvice` + `@ExceptionHandler`；校验异常 `MethodArgumentNotValidException`。
- **Boot 3 新特性**：`ProblemDetail`（RFC 7807）、`jakarta.*`、Java 17 基线、AOT/Native。

## 七、数据访问

- **`#{}` vs `${}`**：预编译 vs 拼接（有注入风险，仅动态表名/排序 + 白名单）。
- **MyBatis 缓存**：一级（SqlSession/同事务）、二级（namespace，分布式建议关闭）。
- **MyBatis 插件**：拦截 `Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler`。
- **JPA N+1**：`join fetch`、`@EntityGraph`、DTO 投影；关闭 `open-in-view`。
- **HikariCP**：`maximum-pool-size` 10~30；`max-lifetime` < 数据库 `wait_timeout`；`leak-detection-threshold` 查泄漏。
- **连接池大小**：`(CPU×2) + 磁盘数`，不是越大越好。

## 八、缓存与 Redis

- **三大问题**：穿透（布隆/空值）→ 击穿（互斥锁/逻辑过期）→ 雪崩（随机 TTL/多级缓存）。
- **Cache Aside**：读命中返回、未命中读 DB 写缓存；写则**更新 DB 后删缓存**。
- **分布式锁**：`SET NX EX` 原子加锁、唯一 value、Lua 解锁、看门狗续期、finally 释放。
- **序列化**：默认 JDK 序列化乱码 → 改 JSON/String。
- **限流**：固定窗口、滑动窗口、令牌桶（允许突发）、漏桶（匀速）。

## 九、消息

- **可靠性**：生产者 Confirm + 持久化（交换机/队列/消息）+ 消费者**手动 ACK**。
- **重复消费**：MQ 只保证 at least once → 业务**幂等**（唯一键/乐观锁/状态机）。
- **RabbitMQ**：Direct/Topic/Fanout/Headers；DLX 实现延迟与死信；prefetch 限流。
- **Kafka**：分区内有序、消费组、offset 手动提交、`enable.idempotence`、EOS 事务。
- **RocketMQ**：NameServer、Tag 过滤、**事务消息**（半消息 + 回查）、18 级延时、DLQ。
- **本地事务与消息一致**：事务消息 or 本地消息表（业务 + 消息同库事务，定时重投）。

## 十、Spring Cloud

- **注册发现**：Nacos（AP/CP、兼具配置）、Eureka（AP，停维护）、Consul（CP）。
- **LoadBalancer**：取代 Ribbon；`@LoadBalanced` 给 RestTemplate 加拦截器解析服务名。
- **OpenFeign**：JDK 动态代理 + 注解解析；必须配超时；`fallbackFactory` 降级。
- **Gateway**：WebFlux/Netty；Route = uri + Predicate + Filter；限流用 Redis 令牌桶。
- **熔断三态**：Closed → Open（拒绝）→ Half-Open（探测）→ 恢复/再 Open。
- **Resilience4j**：CircuitBreaker、RateLimiter、Bulkhead（信号量/线程池）、Retry、TimeLimiter。
- **Seata AT**：一阶段写 undo_log 后**立即提交本地事务**（释放锁），二阶段提交删日志/回滚反向补偿。
- **TCC 三问题**：幂等、空回滚、悬挂。
- **分布式 ID**：UUID（无序）、雪花（41 时间戳 + 10 机器 + 12 序列，时钟回拨）、号段（Leaf）。

## 十一、可观测与运维

- **Actuator**：health/info/metrics/conditions/beans/loggers；生产不暴露 env/heapdump。
- **Micrometer**：Counter、Gauge、Timer、DistributionSummary、LongTaskTimer；**避免高基数 tag**。
- **链路**：traceId/spanId/parentId；Boot 3 用 Micrometer Tracing（Sleuth 已废）；异步靠 `TaskDecorator` 传 MDC。
- **日志**：SLF4J + Logback；MDC 传 traceId；`finally` 清理 MDC。
- **K8s 探针**：startup（保护启动）、readiness（摘流量）、liveness（重启）。
- **优雅停机**：preStop sleep + `terminationGracePeriodSeconds` + `server.shutdown=graceful`。

## 十二、高频追问速答

| 问题 | 答 |
|---|---|
| 为什么构造器注入好 | 依赖不可变、启动即暴露循环依赖、便于单测 |
| 三级缓存为什么不能两级 | 两级无法在循环依赖时保证注入的是 AOP 代理 |
| 事务和 AOP 都在代理后置生成 | 事务由 `TransactionInterceptor` 在 AOP 链中执行 |
| Boot 3 与 2 的最大差异 | Jakarta 包名、Java 17、`.imports` 注册文件、Observability 重构 |
| 微服务最大代价 | 分布式一致性 + 运维复杂度 |
| 如何排查线上 CPU 高 | top -Hp → jstack（nid 转 16 进制）；Arthas `thread -n 5`；火焰图 |
| 如何排查内存泄漏 | heapdump → MAT 看 dominator tree；查 ThreadLocal、静态集合、未关闭资源 |
| 接口慢怎么排查 | 链路追踪定位哪一跳 → 慢 SQL → 缓存命中 → GC → 外部依赖 |

## 十三、答题套路

1. **先定义**（是什么）→ 2. **原理/流程**（怎么做的）→ 3. **对比**（与别的方案）→ 4. **实践**（踩过什么坑、怎么选）→ 5. **总结一句话**。

> 面试加分项：主动说出**版本差异**（Boot 2 vs 3）、**失效场景**、**替代方案与选型理由**。
