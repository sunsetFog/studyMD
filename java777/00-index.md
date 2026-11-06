# Spring 全栈知识体系（Spring Framework → Boot → Cloud）

> 目标：一套可顺序阅读、也可按需查阅的 Spring 生态专家级笔记。
> 约定：示例基于 **Spring Framework 6.x / Spring Boot 3.x（Java 17+，推荐 21）**、**Spring Cloud 2023/2024**。
> 结构：核心 → Boot → 数据/缓存 → 消息/异步 → 微服务 → 响应式 → 可观测 → 工程实践。
> 前置：建议先读完同级目录 `java666/`（Java 语言与 JDK 基础）。

---

## 目录

### 一、Spring Framework 核心
| 编号 | 文件 | 内容 |
|---|---|---|
| 01 | [01-spring-overview-ioc-di.md](01-spring-overview-ioc-di.md) | 生态全景、IoC/DI 思想、模块划分 |
| 02 | [02-spring-core-container.md](02-spring-core-container.md) | BeanFactory / ApplicationContext / BeanDefinition |
| 03 | [03-bean-lifecycle.md](03-bean-lifecycle.md) | 作用域、生命周期回调、BeanPostProcessor |
| 04 | [04-dependency-injection.md](04-dependency-injection.md) | 注入方式、@Autowired、循环依赖三级缓存 |
| 05 | [05-aop-foundation.md](05-aop-foundation.md) | AOP 概念、JDK/CGLIB 动态代理 |
| 06 | [06-spring-aop-practice.md](06-spring-aop-practice.md) | @Aspect、切点表达式、日志/幂等实战 |
| 07 | [07-spring-jdbc-tx.md](07-spring-jdbc-tx.md) | JdbcTemplate、事务抽象与 PlatformTransactionManager |
| 08 | [08-spring-events-validation.md](08-spring-events-validation.md) | 事件机制、Validation 校验 |

### 二、Spring Boot
| 编号 | 文件 | 内容 |
|---|---|---|
| 09 | [09-boot-introduction.md](09-boot-introduction.md) | Starter、约定优于配置、依赖管理 |
| 10 | [10-boot-quickstart.md](10-boot-quickstart.md) | 初始化、目录结构、@SpringBootApplication |
| 11 | [11-boot-auto-configuration.md](11-boot-auto-configuration.md) | 自动配置原理、@Conditional、imports 文件 |
| 12 | [12-boot-configuration.md](12-boot-configuration.md) | yml、多环境、配置优先级、@ConfigurationProperties |
| 13 | [13-boot-logging.md](13-boot-logging.md) | Logback/Log4j2、MDC 链路追踪 |
| 14 | [14-boot-web-mvc.md](14-boot-web-mvc.md) | RestController、参数绑定、拦截器、CORS |
| 15 | [15-boot-exception-handling.md](15-boot-exception-handling.md) | 统一异常处理、@ControllerAdvice |
| 16 | [16-boot-data-jpa.md](16-boot-data-jpa.md) | Spring Data JPA、Repository、审计 |
| 17 | [17-boot-mybatis.md](17-boot-mybatis.md) | MyBatis 集成、动态 SQL、分页 |
| 18 | [18-boot-transaction.md](18-boot-transaction.md) | 传播/隔离、事务失效场景剖析 |
| 19 | [19-boot-aop.md](19-boot-aop.md) | Boot 中 AOP 实战与坑 |
| 20 | [20-boot-security-basics.md](20-boot-security-basics.md) | Spring Security、认证授权、JWT |
| 21 | [21-boot-actuator.md](21-boot-actuator.md) | 健康检查、端点、自定义指标 |
| 22 | [22-boot-custom-starter.md](22-boot-custom-starter.md) | 自定义 Starter、@EnableXXX 模式 |
| 23 | [23-boot-testing.md](23-boot-testing.md) | @SpringBootTest、MockMvc、Testcontainers |
| 24 | [24-boot-common-pitfalls.md](24-boot-common-pitfalls.md) | Boot 常见坑汇总 |

### 三、数据层与缓存
| 编号 | 文件 | 内容 |
|---|---|---|
| 25 | [25-mybatis-deep.md](25-mybatis-deep.md) | MyBatis 源码流程、插件、一二级缓存 |
| 26 | [26-spring-data-redis.md](26-spring-data-redis.md) | RedisTemplate、序列化、连接池 |
| 27 | [27-redis-patterns.md](27-redis-patterns.md) | 穿透/击穿/雪崩、分布式锁、布隆过滤器 |
| 28 | [28-spring-data-mongodb.md](28-spring-data-mongodb.md) | MongoDB 文档建模与聚合 |
| 29 | [29-spring-data-elasticsearch.md](29-spring-data-elasticsearch.md) | ES 检索、高亮、分页 |
| 30 | [30-connection-pool.md](30-connection-pool.md) | HikariCP / Druid 调优 |
| 31 | [31-sharding-readwrite-split.md](31-sharding-readwrite-split.md) | ShardingSphere 分库分表、读写分离 |
| 32 | [32-distributed-id.md](32-distributed-id.md) | 雪花算法、Leaf、UUID、号段 |

### 四、消息与异步
| 编号 | 文件 | 内容 |
|---|---|---|
| 33 | [33-spring-async.md](33-spring-async.md) | @Async、线程池、异常与上下文传递 |
| 34 | [34-rabbitmq.md](34-rabbitmq.md) | 交换机、可靠投递、死信队列 |
| 35 | [35-kafka.md](35-kafka.md) | 分区、消费组、幂等与顺序性 |
| 36 | [36-rocketmq.md](36-rocketmq.md) | RocketMQ 模型、延时/事务消息 |
| 37 | [37-spring-cloud-stream.md](37-spring-cloud-stream.md) | Binder 抽象与函数式模型 |
| 38 | [38-transactional-message.md](38-transactional-message.md) | 事务消息、本地消息表、最终一致性 |

### 五、Spring Cloud 微服务
| 编号 | 文件 | 内容 |
|---|---|---|
| 39 | [39-microservices-overview.md](39-microservices-overview.md) | 微服务概念、演进、版本对照 |
| 40 | [40-service-registry-discovery.md](40-service-registry-discovery.md) | Nacos / Eureka / Consul |
| 41 | [41-load-balancer.md](41-load-balancer.md) | LoadBalancer（含 Ribbon 历史） |
| 42 | [42-openfeign.md](42-openfeign.md) | 声明式调用、超时、重试、降级 |
| 43 | [43-api-gateway.md](43-api-gateway.md) | Spring Cloud Gateway、路由、限流 |
| 44 | [44-circuit-breaker.md](44-circuit-breaker.md) | Resilience4j 熔断/隔离/限流 |
| 45 | [45-config-center.md](45-config-center.md) | Nacos Config / Spring Cloud Config |
| 46 | [46-distributed-lock.md](46-distributed-lock.md) | Redis/ZK 分布式锁 |
| 47 | [47-distributed-tx-seata.md](47-distributed-tx-seata.md) | Seata AT/TCC/Saga |
| 48 | [48-oauth2-resource-server.md](48-oauth2-resource-server.md) | OAuth2、JWT、资源服务器 |
| 49 | [49-service-mesh-istio.md](49-service-mesh-istio.md) | 服务网格与 Istio 概念 |
| 50 | [50-microservice-patterns.md](50-microservice-patterns.md) | BFF/CQRS/Saga/事件驱动 |
| 51 | [51-microservice-pitfalls.md](51-microservice-pitfalls.md) | 拆分原则与常见坑 |
| 52 | [52-spring-cloud-alibaba.md](52-spring-cloud-alibaba.md) | Alibaba 全家桶整合 |

### 六、响应式与高级
| 编号 | 文件 | 内容 |
|---|---|---|
| 53 | [53-spring-webflux.md](53-spring-webflux.md) | WebFlux、Reactor、背压 |
| 54 | [54-spring-websocket.md](54-spring-websocket.md) | WebSocket、STOMP 消息 |
| 55 | [55-spring-batch.md](55-spring-batch.md) | 批处理 Job/Step、分片 |
| 56 | [56-spring-integration.md](56-spring-integration.md) | EIP 企业集成模式 |
| 57 | [57-grpc-protobuf.md](57-grpc-protobuf.md) | gRPC + Protobuf 集成 |
| 58 | [58-netty-basics.md](58-netty-basics.md) | Netty 线程模型与编解码 |

### 七、可观测性与运维
| 编号 | 文件 | 内容 |
|---|---|---|
| 59 | [59-micrometer-metrics.md](59-micrometer-metrics.md) | 指标埋点、MeterRegistry |
| 60 | [60-distributed-tracing.md](60-distributed-tracing.md) | Sleuth/Micrometer Tracing、SkyWalking |
| 61 | [61-logging-elk.md](61-logging-elk.md) | 日志规范、ELK/Loki |
| 62 | [62-prometheus-grafana.md](62-prometheus-grafana.md) | 监控告警与看板 |
| 63 | [63-docker.md](63-docker.md) | 多阶段构建、镜像瘦身 |
| 64 | [64-kubernetes.md](64-kubernetes.md) | K8s 部署、探针、Helm |

### 八、工程实践与面试
| 编号 | 文件 | 内容 |
|---|---|---|
| 65 | [65-ci-cd.md](65-ci-cd.md) | GitHub Actions / Jenkins 流水线 |
| 66 | [66-monolith-to-microservices.md](66-monolith-to-microservices.md) | 单体到微服务演进 |
| 67 | [67-spring-performance-tuning.md](67-spring-performance-tuning.md) | JVM+Spring 性能调优 |
| 68 | [68-security-best-practices.md](68-security-best-practices.md) | 注入/XSS/CSRF/越权防护 |
| 69 | [69-system-design-cases.md](69-system-design-cases.md) | 秒杀、短链、IM 案例 |
| 70 | [70-interview-cheatsheet.md](70-interview-cheatsheet.md) | 面试速查清单 |

---

## 学习路径（8 周）

| 周次 | 主题 | 篇目 |
|---|---|---|
| W1 | IoC/DI 与容器 | 01-04 |
| W2 | AOP、JDBC、事件 | 05-08 |
| W3 | Boot 基础与自动配置 | 09-13 |
| W4 | Web、数据、事务、安全 | 14-20 |
| W5 | Boot 进阶与测试 | 21-24，数据层 25-32 |
| W6 | 消息与异步 | 33-38 |
| W7 | 微服务全家桶 | 39-52 |
| W8 | 响应式、可观测、工程实践 | 53-70 |

## 版本对照（选型必看）

| 组件 | 推荐版本 | 说明 |
|---|---|---|
| JDK | 17 / 21 (LTS) | Boot 3.x 强制 Java 17+ |
| Spring Framework | 6.1 / 6.2 | 支持 AOT、虚拟线程、ProblemDetail |
| Spring Boot | 3.2 / 3.3 / 3.4 | Jakarta EE 9+（`jakarta.*` 包名） |
| Spring Cloud | 2023.0.x (Leyton) / 2024.0.x | 与 Boot 版本一一对应，不可混用 |
| Spring Cloud Alibaba | 2023.0.1.0+ | 适配 Boot 3.x |
| MyBatis | 3.5.x + mybatis-spring-boot 3.x | |
| Seata | 2.x | |
| Nacos | 2.3+ | |

> **铁律**：Spring Cloud 与 Spring Boot 版本必须按官方 Release Train 对应，否则启动报 `NoSuchMethodError`/`ClassNotFoundException`。

## 阅读约定

- 代码块中 `// ...` 表示省略的无关代码。
- 「坑」小节列的是真实项目中高频事故点。
- 「面试」小节给出标准回答要点。
