# 52 · Spring Cloud Alibaba 全家桶

## 1. 组件地图

| 组件 | 作用 | 替代的 Netflix 组件 |
|---|---|---|
| **Nacos Discovery** | 服务注册发现 | Eureka |
| **Nacos Config** | 配置中心 | Spring Cloud Config / Bus |
| **Sentinel** | 熔断、限流、系统自适应 | Hystrix |
| **Seata** | 分布式事务 | 无 |
| **RocketMQ** | 消息（事务消息） | — |
| **Spring Cloud Gateway** | 网关（官方） | Zuul |
| **OpenFeign + LoadBalancer** | 调用与负载均衡（官方） | Ribbon |
| **SkyWalking**（生态） | 链路追踪 | Sleuth/Zipkin |
| **Dubbo**（可选） | RPC | — |

## 2. 依赖管理

```xml
<dependencyManagement>
  <dependencies>
    <dependency><groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-alibaba-dependencies</artifactId>
      <version>2023.0.1.0</version><type>pom</type><scope>import</scope></dependency>
    <dependency><groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2023.0.1</version><type>pom</type><scope>import</scope></dependency>
  </dependencies>
</dependencyManagement>
```

> 版本对应关系见 [39](39-microservices-overview.md)。

## 3. 典型聚合依赖

```xml
<!-- 注册发现 -->
<dependency><groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId></dependency>
<!-- 配置中心 -->
<dependency><groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId></dependency>
<!-- 熔断限流 -->
<dependency><groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId></dependency>
<!-- 分布式事务 -->
<dependency><groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-seata</artifactId></dependency>
```

## 4. Nacos 双中心

```yaml
spring:
  application: {name: order-service}
  cloud:
    nacos:
      server-addr: 127.0.0.1:8848
      username: nacos
      password: nacos
      discovery:
        namespace: ${NACOS_NS}
        group: DEFAULT_GROUP
      config:
        namespace: ${NACOS_NS}
        group: DEFAULT_GROUP
        file-extension: yaml
        shared-configs:
          - {data-id: common.yaml, refresh: true}
```

详见 [40](40-service-registry-discovery.md) 与 [45](45-config-center.md)。

## 5. Sentinel

### 依赖与控制台

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080      # sentinel-dashboard
        port: 8719                     # 与控制台通信端口
      eager: true                      # 饥饿加载（避免首次调用才初始化）
      web-context-unify: false
```

### 注解使用

```java
@SentinelResource(value = "getUser",
                  blockHandler = "blockHandler",   // 限流/熔断（BlockException）
                  fallback = "fallback",           // 业务异常
                  exceptionsToIgnore = {IllegalArgumentException.class})
public UserVO getUser(Long id) { ... }

public UserVO blockHandler(Long id, BlockException e) { return UserVO.anonymous(); }
public UserVO fallback(Long id, Throwable t) { return UserVO.empty(); }
```

### 规则类型

| 规则 | 说明 |
|---|---|
| 流控（QPS/并发） | 直接/关联/链路；快速失败/Warm Up/排队等待 |
| 熔断（降级） | 慢调用比例 / 异常比例 / 异常数 |
| 热点参数 | 对参数值单独限流（如热点商品 ID） |
| 系统规则 | CPU/Load/入口 QPS 自适应，保护系统整体 |
| 授权规则 | 黑白名单（按来源 app） |
| 集群流控 | 需要 Token Server |

```java
// 编程式定义规则（一般在控制台配更方便）
FlowRule rule = new FlowRule("getUser")
        .setCount(100)                       // QPS 阈值
        .setGrade(RuleConstant.FLOW_GRADE_QPS)
        .setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP)
        .setWarmUpPeriodSec(10);
FlowRuleManager.loadRules(List.of(rule));
```

### 与 Feign / Gateway 集成

```yaml
feign:
  sentinel:
    enabled: true
```
```java
@FeignClient(name = "user-service", fallback = UserFallback.class)
```

```xml
<dependency><groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId></dependency>
```
```yaml
spring.cloud.sentinel.filter.enabled: true
```

### 规则持久化

默认规则存在内存，**重启丢失**。持久化方案：
- 推模式（推荐）：`DataSource` 接入 Nacos/Apollo/ZK。
- 拉模式：文件、RDBMS。

```java
// Nacos 数据源
ReadableDataSource<String, List<FlowRule>> ds = new NacosDataSource<>(
        serverAddr, groupId, dataId,
        source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {}));
FlowRuleManager.register2Property(ds.getProperty());
```

## 6. Seata

见 [47](47-distributed-tx-seata.md)。关键配置：

```yaml
seata:
  tx-service-group: my_tx_group
  service:
    vgroup-mapping: {my_tx_group: default}
  registry:
    type: nacos
    nacos: {server-addr: 127.0.0.1:8848, group: SEATA_GROUP}
  config:
    type: nacos
```

## 7. RocketMQ

见 [36](36-rocketmq.md)。与 Spring Cloud Stream 也可用 `spring-cloud-stream-binder-rocketmq`（阿里云版）。

## 8. 与官方 Spring Cloud 的取舍

| 场景 | 建议 |
|---|---|
| 国内项目、需要控制台 | Alibaba 全家桶（Nacos + Sentinel + Seata） |
| 国际化 / 轻依赖 | 官方（Consul/Config + Resilience4j） |
| 已在 K8s | K8s Service + ConfigMap + Gateway + Resilience4j（甚至 Mesh） |

> 不要为了「全家桶」而引入不需要的组件：例如已有 MQ 就别硬上 RocketMQ。

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 版本不对应 | 启动报类不匹配 | 严格按官方 Release Train 表 |
| Nacos 配置改了不生效 | 放在 application.yml | 放 bootstrap.yml 或用 `spring.config.import` |
| Sentinel 规则重启丢失 | 规则没了 | 配 Nacos 持久化数据源 |
| Seata 不回滚 | undo_log 缺失或异常被吞 | 建表 + 抛异常 |
| `@SentinelResource` 不生效 | 无 starter 或 AOP | 加依赖；方法是 public；别自调用 |
| Feign fallback 冲突 | 与 Resilience4j 同时启用 | 只启用一种（`feign.circuitbreaker.enabled` 或 `feign.sentinel.enabled`） |
| Nacos 集群数据不一致 | 未外置 MySQL | 集群必须配外部数据库 |
| 服务名冲突 | 跨环境串调用 | 用 namespace 隔离 |

## 10. 面试速答

- **SCA 包含哪些组件**：Nacos（注册+配置）、Sentinel（限流熔断）、Seata（分布式事务）、RocketMQ（消息）。
- **Sentinel 与 Hystrix/Resilience4j 区别**：Sentinel 有控制台、支持热点参数限流与系统自适应保护、规则可动态配置；Hystrix 已停维护。
- **Nacos 为什么能同时做注册中心和配置中心**：内嵌存储 + 长轮询推送 + 控制台，一个组件解决两件事，运维成本低。
- **Sentinel 与 Resilience4j 如何选**：要控制台与热点限流选 Sentinel；要轻量、函数式、响应式选 Resilience4j。
