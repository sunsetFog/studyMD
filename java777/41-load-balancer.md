# 41 · 负载均衡：Spring Cloud LoadBalancer

## 1. 客户端 vs 服务端负载均衡

| 类型 | 位置 | 例子 |
|---|---|---|
| 客户端 | 调用方本地选实例 | Ribbon、Spring Cloud LoadBalancer、Dubbo |
| 服务端 | 独立组件转发 | Nginx、LVS、F5、K8s Service、Envoy |

Spring Cloud 使用**客户端负载均衡**：从注册中心拉实例列表，本地选择后直连。

## 2. Ribbon（已退役）

Netflix Ribbon 进入维护状态，Spring Cloud 2020 起被 **`spring-cloud-loadbalancer`** 取代。

| Ribbon 概念 | LoadBalancer 对应 |
|---|---|
| `IRule` | `ReactorServiceInstanceLoadBalancer` |
| `IPing` | 由注册中心健康检查替代 |
| `ServerList` | `DiscoveryClientServiceInstanceListSupplier` |
| `ILoadBalancer` | `LoadBalancerClientFactory` |

## 3. 快速使用

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

```java
@Bean
@LoadBalanced                     // ★ 关键：给 RestTemplate 加拦截器，解析服务名
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// 使用
User u = restTemplate.getForObject("http://user-service/api/users/1", User.class);

// WebClient（响应式）
@Bean @LoadBalanced WebClient.Builder lbBuilder() { return WebClient.builder(); }
webClient.get().uri("http://user-service/api/users/1").retrieve().bodyToMono(User.class);
```

底层：`LoadBalancerInterceptor` → `LoadBalancerClient#execute` → `ReactiveLoadBalancer#choose` → 替换 URL 为实际 IP。

## 4. 两种负载均衡算法

| 算法 | 说明 | 配置 |
|---|---|---|
| **轮询**（默认） | `RoundRobinLoadBalancer` | 默认 |
| **随机** | `RandomLoadBalancer` | 显式配置 |

```java
@Configuration
@LoadBalancerClient(name = "user-service", configuration = RandomLbConfig.class)
public class LbConfig { }

public class RandomLbConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment env,
            LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

> 默认**没有**权重/最少连接/一致性哈希实现，需自定义（或引入 Alibaba Nacos 的 `NacosLoadBalancer`）。

## 5. 全局默认配置

```java
@Configuration
@LoadBalancerClients(defaultConfiguration = GlobalLbConfig.class)
public class GlobalConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> lb(Environment env, LoadBalancerClientFactory f) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(f.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

## 6. 实例列表来源与缓存

```java
// 默认：DiscoveryClientServiceInstanceListSupplier（带缓存，默认 35s 缓存过期）
// 可选：
// - SameInstancePreferenceServiceInstanceListSupplier（同实例优先，会话保持）
// - ZonePreferenceServiceInstanceListSupplier（同 zone 优先）
// - HealthCheckServiceInstanceListSupplier（主动健康检查）
// - CachingServiceInstanceListSupplier + 缓存 TTL
```

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 35s
        capactity: 256
      health-check:
        initial-delay: 0
        interval: 25s
```

## 7. 自定义：灰度/权重路由

```java
public class GrayLoadBalancer implements ReactorServiceInstanceLoadBalancer {
    private final ObjectProvider<ServiceInstanceListSupplier> supplier;
    private final String serviceId;

    public Mono<Response<ServiceInstance>> choose(Request request) {
        return supplier.getIfAvailable().get(request).next().map(list -> {
            String version = RequestContextHolder.currentHeader("x-version");   // 从请求上下文取
            List<ServiceInstance> matched = list.stream()
                    .filter(i -> version.equals(i.getMetadata().get("version"))).toList();
            List<ServiceInstance> target = matched.isEmpty() ? list : matched;
            ServiceInstance i = target.get(ThreadLocalRandom.current().nextInt(target.size()));
            return new DefaultResponse(i);
        });
    }
}
```

## 8. 与 OpenFeign 集成

Feign 默认使用 LoadBalancer（引入 `spring-cloud-openfeign-core` 即可，`url` 不填时用服务名）。

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connect-timeout: 3000
            read-timeout: 10000
```

## 9. 常见问题与坑

| 坑 | 现象 | 解决 |
|---|---|---|
| `UnknownHostException: user-service` | 服务名没解析 | 加 `@LoadBalanced`；检查依赖 |
| 调用到已下线实例 | 缓存未刷新 | 调小 `cache.ttl`；K8s 用 readiness + preStop |
| 只调用同一实例 | 缓存/会话保持 | 检查 SameInstancePreference |
| 负载不均 | 轮询在长连接下失效 | 换随机或一致性哈希 |
| 没走灰度 | 元数据未传 | 检查 metadata 与请求头透传 |
| Ribbon 配置残留 | 冲突 | 移除 ribbon 依赖与配置 |

## 10. 面试速答

- **Ribbon 与 LoadBalancer**：Ribbon 已退役，Spring Cloud LoadBalancer 是官方替代，基于 Reactive（`ReactorServiceInstanceLoadBalancer`）。
- **`@LoadBalanced` 原理**：为 `RestTemplate` 添加 `LoadBalancerInterceptor`，拦截请求把服务名替换为实际实例 IP。
- **客户端负载均衡的优缺点**：少一跳、性能好；但配置分散在各服务、升级需各服务重启。
- **负载均衡算法**：轮询（默认）、随机、最少连接、一致性哈希（需自定义或使用 Nacos/Service Mesh）。
