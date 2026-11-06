# 39 · 微服务概览与版本演进

## 1. 什么是微服务

一种架构风格：把单体拆分为一组**围绕业务能力构建、独立部署、去中心化治理**的小服务，通过轻量通信（HTTP/gRPC/MQ）协作。

核心特征：
- 单一职责（围绕业务领域，而非技术分层）
- 独立进程、独立部署、独立扩缩容
- 独立数据库（**Database per Service**）
- 去中心化治理与技术栈
- 自动化基础设施（CI/CD、容器化、监控）

## 2. 单体 vs 微服务

| 维度 | 单体 | 微服务 |
|---|---|---|
| 开发效率 | 初期快 | 初期慢 |
| 部署 | 整体发布，风险高 | 单服务发布，风险低 |
| 扩展 | 整体扩容 | 按需扩容 |
| 技术栈 | 统一 | 可异构 |
| 故障隔离 | 一处崩溃全站挂 | 隔离（需熔断） |
| 运维 | 简单 | **复杂**（链路、监控、部署） |
| 一致性 | 本地事务 | **分布式事务难题** |
| 适用 | 小团队、早期业务 | 大团队、复杂业务、高弹性 |

> 忠告：**不要为了微服务而微服务**。团队 < 20 人、业务未稳定时，单体（或模块化单体）更合适。

## 3. 拆分原则

- **DDD 限界上下文**（Bounded Context）：按业务能力/领域边界拆分。
- **单一职责 + 高内聚低耦合**。
- **康威定律**：组织结构决定系统架构。
- 避免**分布式单体**（拆了但每次都要一起发布）。
- 拆分顺序：先按业务垂直拆，再对热点服务进一步水平拆。

常见拆分维度：用户、商品、订单、库存、支付、营销、搜索、通知、风控、结算。

## 4. 微服务需要的基础设施

| 能力 | 组件 |
|---|---|
| 服务注册与发现 | Nacos / Eureka / Consul / K8s Service |
| 配置中心 | Nacos Config / Apollo / Spring Cloud Config |
| 远程调用 | OpenFeign / gRPC / Dubbo |
| 负载均衡 | Spring Cloud LoadBalancer / Ribbon(旧) |
| API 网关 | Spring Cloud Gateway / Kong / APISIX |
| 熔断限流 | Resilience4j / Sentinel / Hystrix(旧) |
| 分布式事务 | Seata / 消息最终一致 |
| 链路追踪 | Micrometer Tracing / SkyWalking / Zipkin |
| 日志 | ELK / Loki |
| 监控 | Prometheus + Grafana |
| 部署 | Docker + K8s + Helm |
| 安全 | OAuth2 / JWT |

## 5. Spring Cloud 生态

```
Spring Cloud Netflix（Eureka/Hystrix/Zuul/Ribbon）  ← 已进入维护，逐渐淘汰
Spring Cloud Alibaba（Nacos/Sentinel/Seata/RocketMQ）← 国内主流
Spring Cloud 官方（Gateway/LoadBalancer/OpenFeign/Config/Resilience4j）
Spring Cloud Kubernetes
Spring Cloud Stream / Function
```

### 版本对照（Release Train）

| Spring Cloud | Spring Boot | 说明 |
|---|---|---|
| 2021.0.x (Jubilee) | 2.6 / 2.7 | |
| 2022.0.x (Kilburn) | 3.0 / 3.1 | |
| 2023.0.x (Leyton) | 3.2 / 3.3 | Jakarta EE 10 |
| 2024.0.x (Moorgate) | 3.4 | |

| Spring Cloud Alibaba | Spring Cloud | Spring Boot |
|---|---|---|
| 2022.0.0.0 | 2022.0.0 | 3.0.x |
| 2023.0.1.0 | 2023.0.1 | 3.2.x |
| 2023.0.3.2 | 2023.0.3 | 3.3.x |

> **版本必须严格对应**，否则 `NoSuchMethodError`、启动失败。用 `dependencyManagement` 统一管理。

## 6. 依赖管理模板

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2023.0.3</version><type>pom</type><scope>import</scope>
    </dependency>
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-alibaba-dependencies</artifactId>
      <version>2023.0.1.0</version><type>pom</type><scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

## 7. 服务间通信

| 方式 | 协议 | 场景 |
|---|---|---|
| HTTP REST (OpenFeign) | HTTP/JSON | 外部 API、跨语言、简单 |
| gRPC | HTTP/2 + Protobuf | **内部高频调用、高性能** |
| MQ | 异步 | 解耦、削峰、事件驱动 |
| Dubbo | RPC | 国内 Java 生态 |

选型建议：对外 HTTP，内部高频 gRPC/Dubbo，异步事件用 MQ。

## 8. 演进路线（推荐）

```
阶段0  单体（快速验证）
阶段1  模块化单体（清晰的包边界）
阶段2  垂直拆分核心域（订单/支付独立）
阶段3  引入注册中心、配置中心、网关
阶段4  引入熔断限流、链路追踪、服务网格
阶段5  平台化（DevOps、Service Mesh、多租户）
```

## 9. 微服务的成本（先想清楚）

- 分布式事务、数据一致性
- 网络延迟、重试、幂等、超时
- 排查问题难（一条请求跨 N 个服务）
- 测试与部署复杂
- 运维成本高（几十上百个服务）
- 团队需要 DevOps 能力

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 过度拆分 | 一个接口跨 5 个服务 | 合并过小的服务 |
| 分布式单体 | 改一个字段要发 8 个服务 | 重新划分边界 |
| 共享数据库 | 耦合、改表互相影响 | 独立库，接口化访问 |
| 版本不匹配 | 启动报错 | 严格按 Release Train |
| 无熔断 | 雪崩 | Resilience4j/Sentinel |
| 无超时 | 线程池打满 | 每个调用都设超时 |
| 无幂等 | 重试导致重复 | 幂等设计 |
| 链路无追踪 | 排障靠猜 | 引入 traceId |

## 11. 面试速答

- **微服务是什么**：按业务能力拆分、独立部署与扩展、去中心化治理的架构风格。
- **与单体对比**：微服务在可扩展性、故障隔离、团队协作上更优，但带来分布式一致性、运维复杂度成本。
- **Spring Cloud 是什么**：基于 Spring Boot 的微服务工具集（注册发现、配置、网关、熔断、调用、追踪）。
- **什么时候不该用微服务**：小团队、业务早期、对一致性要求极高且无法接受最终一致的场景。
