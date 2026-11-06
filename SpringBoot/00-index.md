# SpringBoot 完整知识体系 · 总索引

> 版本基准：SpringBoot 3.x（Spring Framework 6 / Java 17+ / Jakarta EE）
> 学习路径：入门 → Web 开发 → 数据访问 → 核心特性 → 安全中间件 → 消息集成 → 工程化 → 原理 → 实战

## 目录

### 一、入门篇
| 篇目 | 内容 |
|---|---|
| [01-framework-overview](01-framework-overview.md) | Spring 全家盘认知：Spring vs SpringBoot vs Spring Cloud |
| [02-quick-start](02-quick-start.md) | 快速上手：环境搭建、首个应用、项目结构 |
| [03-configuration](03-configuration.md) | 配置体系：application.yml、@ConfigurationProperties、Profile |

### 二、Web 开发
| 篇目 | 内容 |
|---|---|
| [04-web-basics](04-web-basics.md) | Spring MVC 基础：@RestController、返回值、静态资源 |
| [05-restful-api](05-restful-api.md) | RESTful API 设计规范与实践 |
| [06-validation](06-validation.md) | 参数校验：Jakarta Validation、自定义校验器 |
| [07-exception-handling](07-exception-handling.md) | 全局异常处理与统一响应体 |
| [08-filter-interceptor](08-filter-interceptor.md) | Filter / Interceptor / AOP 三层拦截体系 |

### 三、数据访问
| 篇目 | 内容 |
|---|---|
| [09-datasource-jdbc](09-datasource-jdbc.md) | 数据源配置与 JdbcTemplate |
| [10-mybatis](10-mybatis.md) | MyBatis 集成：Mapper、动态 SQL、分页 |
| [11-mybatis-plus](11-mybatis-plus.md) | MyBatis-Plus：CRUD 增强、条件构造器、代码生成 |
| [12-jpa-hibernate](12-jpa-hibernate.md) | Spring Data JPA 与 Hibernate |
| [13-redis](13-redis.md) | Redis 集成：RedisTemplate、序列化、分布式锁 |
| [14-transaction](14-transaction.md) | 事务管理：@Transactional 全解、传播机制、失效场景 |

### 四、核心特性
| 篇目 | 内容 |
|---|---|
| [15-ioc-di](15-ioc-di.md) | IoC 容器与依赖注入全景 |
| [16-bean-lifecycle](16-bean-lifecycle.md) | Bean 生命周期与作用域 |
| [17-aop](17-aop.md) | AOP 切面编程：通知、切点、实战场景 |
| [18-events](18-events.md) | 事件机制：ApplicationEvent 与解耦设计 |
| [19-scheduling-async](19-scheduling-async.md) | 定时任务、异步任务与线程池配置 |

### 五、安全与中间件
| 篇目 | 内容 |
|---|---|
| [20-spring-security](20-spring-security.md) | Spring Security 6：过滤器链、认证授权 |
| [21-jwt-auth](21-jwt-auth.md) | JWT 无状态认证完整方案 |
| [22-api-docs](22-api-docs.md) | 接口文档：SpringDoc OpenAPI（Swagger） |
| [23-logging](23-logging.md) | 日志体系：Logback、MDC、链路追踪 |
| [24-caching](24-caching.md) | 缓存抽象：@Cacheable、多级缓存设计 |

### 六、消息与集成
| 篇目 | 内容 |
|---|---|
| [25-rabbitmq](25-rabbitmq.md) | RabbitMQ 集成与可靠投递 |
| [26-kafka](26-kafka.md) | Kafka 集成与消费模式 |
| [27-email-file-oss](27-email-file-oss.md) | 邮件发送、文件上传与对象存储 |
| [28-websocket](28-websocket.md) | WebSocket 实时通信 |

### 七、工程化
| 篇目 | 内容 |
|---|---|
| [29-testing](29-testing.md) | 测试：单元测试、切片测试、集成测试 |
| [30-packaging-deploy](30-packaging-deploy.md) | 打包部署：jar/war、启动脚本、优雅停机 |
| [31-docker](31-docker.md) | Docker 化与容器编排 |
| [32-actuator-monitor](32-actuator-monitor.md) | 监控：Actuator、Prometheus、Grafana |
| [33-multi-environment](33-multi-environment.md) | 多环境与配置中心（Nacos） |
| [34-graalvm-native](34-graalvm-native.md) | GraalVM 原生镜像与 AOT |

### 八、原理深入
| 篇目 | 内容 |
|---|---|
| [35-auto-configuration](35-auto-configuration.md) | 自动配置原理：@EnableAutoConfiguration 深度剖析 |
| [36-startup-flow](36-startup-flow.md) | 启动流程：从 main 方法到容器就绪 |
| [37-embedded-server](37-embedded-server.md) | 内嵌容器：Tomcat 启动机制与调优 |

### 九、实战收尾
| 篇目 | 内容 |
|---|---|
| [38-best-practices-interview](38-best-practices-interview.md) | 最佳实践清单 + 高频面试题 30 道 |
| [39-glossary-cheatsheet](39-glossary-cheatsheet.md) | 术语表与速查卡 |

## 全局约定

- 所有示例基于 **SpringBoot 3.2+ / Java 17+ / Maven**
- 包名规范：`com.example.demo`
- 全局规则只写一次：
  - 事务失效六场景 → [14](14-transaction.md)
  - 循环依赖 → [15](15-ioc-di.md) / [36](36-startup-flow.md)
  - 线程池手动创建 → [19](19-scheduling-async.md)
  - 统一响应与异常 → [07](07-exception-handling.md)
  - 序列化配置 → [13](13-redis.md)
- 每篇结尾有「常见坑」与「Checklist」
