# 40 · 服务注册与发现：Nacos / Eureka / Consul

## 1. 原理

```
Provider 启动 → 注册（IP:port, 服务名, 元数据, 健康检查地址）
              → 定时心跳续约（或健康探测）
Consumer 启动 → 从注册中心拉取/订阅服务列表（本地缓存）
              → 本地负载均衡选择一个实例 → 调用
Provider 下线 → 心跳停止 → 注册中心剔除 → 推送变更给 Consumer
```

| 关键机制 | 说明 |
|---|---|
| 服务注册 | 实例启动时注册自身信息 |
| 心跳/续约 | 定时上报存活（默认 5s） |
| 服务剔除 | 超时未心跳则剔除（默认 15s/30s） |
| 自我保护 | 大量实例失联时保留旧数据，防误删（Eureka） |
| 健康检查 | 客户端上报 vs 服务端主动探测 |
| 服务发现 | 拉取 + 推送（长轮询/Watch） |

## 2. CAP 取舍

| 组件 | 一致性模型 | 说明 |
|---|---|---|
| Eureka | **AP** | 允许短暂数据不一致，可用性优先 |
| Consul | **CP** | Raft 强一致 |
| Nacos | **AP/CP 可切换** | 临时实例 AP（Distro），持久实例 CP（Raft） |
| Zookeeper | CP | ZAB |

## 3. Nacos（国内首选）

### 服务端

```bash
# 单机
bin/startup.sh -m standalone
# 访问 http://localhost:8848/nacos  nacos/nacos
# 集群：conf/cluster.conf 配置节点 + 外置 MySQL
```

### 客户端

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: order-service
  cloud:
    nacos:
      server-addr: localhost:8848
      username: nacos
      password: nacos
      discovery:
        namespace: dev            # 环境隔离（注意用 ID 不是名称）
        group: DEFAULT_GROUP
        cluster-name: BJ
        ephemeral: true           # true=临时实例(AP,心跳) false=持久实例(CP,主动探测)
        heart-beat-interval: 5000
        heart-beat-timeout: 15000
        ip-delete-timeout: 30000
        metadata:
          version: 1.0
        # register-enabled: false   # 只消费不注册
```

```java
@SpringBootApplication
@EnableDiscoveryClient        // Spring Cloud 2020+ 可省略（自动开启）
public class OrderApp { }
```

### 服务调用

```java
// 用 DiscoveryClient 手动
List<ServiceInstance> instances = discoveryClient.getInstances("user-service");

// 或 RestTemplate（需 @LoadBalanced）
@Bean @LoadBalanced RestTemplate restTemplate() { return new RestTemplate(); }
restTemplate.getForObject("http://user-service/api/users/1", User.class);   // ★ 用服务名

// 或 OpenFeign（推荐，见 42）
```

### Nacos 关键特性

- **临时/持久实例**：微服务用临时（心跳），数据库/第三方用持久。
- **权重**：`spring.cloud.nacos.discovery.weight`（0~1，0 则不被调用）。
- **命名空间/分组**：环境隔离（dev/test/prod）。
- **保护阈值**：健康实例比例低于阈值时返回全部（含不健康），防雪崩。
- **元数据 + 集群**：用于同集群优先调用（`NacosRule`）。

## 4. Eureka（了解，旧项目多）

```xml
<dependency><groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId></dependency>
```
```java
@SpringBootApplication @EnableEurekaServer
public class EurekaServerApp {}
```
```yaml
# server
server: {port: 8761}
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: false   # 开发关闭自我保护
```
```yaml
# client
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 5
    lease-expiration-duration-in-seconds: 15
```

> 现状：Netflix 组件整体进入维护模式（Hystrix/Zuul/Ribbon/Eureka 均不再有新特性），新项目用 Nacos/Consul。

## 5. Consul

```xml
<dependency><groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-consul-discovery</artifactId></dependency>
```
```yaml
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        service-name: ${spring.application.name}
        health-check-path: /actuator/health
        health-check-interval: 10s
        prefer-ip-address: true
```

Consul 特点：Go 编写、CP、内置 KV 存储、支持多数据中心、服务端主动健康检查。

## 6. K8s 原生服务发现

在 K8s 中可直接使用 **Service + DNS**（`http://user-service:8080`）或 **Spring Cloud Kubernetes**：

```xml
<dependency><groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-kubernetes-fabric8</artifactId></dependency>
```
```yaml
spring:
  cloud:
    kubernetes:
      discovery:
        enabled: true
      reload:
        enabled: true
```

> 用 K8s Service 时，负载均衡由 kube-proxy/Service Mesh 承担，客户端无需 Ribbon/LoadBalancer。

## 7. 优雅上下线

| 环节 | 做法 |
|---|---|
| 优雅下线 | 先调 `/actuator/health` 置 DOWN 或向注册中心反注册 → 等待调用方感知（Ribbon 缓存刷新 30s）→ 再停进程 |
| 优雅上线 | 等 `ApplicationReadyEvent` 后再注册（`spring.cloud.nacos.discovery.register-enabled` + 手动注册），预热后再接流量 |
| K8s | `preStop` sleep + `terminationGracePeriodSeconds`；readiness probe |

```java
// 手动下线（灰度/摘流量）
nacosRegistration.getNacosDiscoveryProperties().setRegisterEnabled(false);
// 或
DiscoveryManager.getInstance().shutdownComponent();     // Eureka
```

> **Ribbon/LoadBalancer 本地缓存**导致下线后 30s 内仍可能被调用 → K8s 场景用 readiness + preStop sleep 保证。

## 8. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 服务名带下划线 | 调用失败 | 服务名用中划线，不要 `_` |
| 调不通但注册中心有 | 本地缓存/网络 | 检查 `@LoadBalanced`、服务名大小写 |
| 下线后仍被调用 | 客户端缓存 | 等待刷新 + preStop sleep |
| Nacos namespace 用名称 | 找不到服务 | 用命名空间 **ID** |
| 多网卡注册错 IP | 调到错误地址 | `spring.cloud.nacos.discovery.ip` 指定 |
| 端口不对 | 容器内端口 | `server.port` 与容器暴露端口一致 |
| 自我保护导致不剔除 | 死实例仍在 | 理解 AP 特性；调阈值或换 CP |
| 注册中心全挂 | 服务仍应可用 | 客户端要有本地缓存兜底 |

## 9. 面试速答

- **注册发现流程**：服务启动注册 → 心跳续约 → 消费者拉取并缓存 → 变更推送 → 下线剔除。
- **Eureka 与 Nacos**：Eureka 是 AP、Netflix 已停止维护；Nacos 支持 AP/CP 切换、兼具配置中心、有控制台、国内主流。
- **自我保护机制**：短时间内大量心跳失败时，注册中心保留实例不剔除，避免因网络抖动把健康服务全摘掉。
- **临时实例 vs 持久实例**：临时靠心跳（微服务，AP）；持久靠服务端探测（CP，如数据库代理）。
