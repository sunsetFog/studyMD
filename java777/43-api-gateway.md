# 43 · Spring Cloud Gateway

## 1. 定位

网关是微服务的**统一入口**，承担：路由转发、认证鉴权、限流、熔断、日志、跨域、灰度、协议转换。

| 对比 | Zuul 1 | Spring Cloud Gateway |
|---|---|---|
| 模型 | Servlet 阻塞 IO | **WebFlux + Netty 响应式**（非阻塞） |
| 性能 | 一般 | 高 |
| 维护 | 停维护 | 官方主推 |

## 2. 三大核心

```
Route（路由）      = id + uri + predicates + filters
Predicate（断言）  匹配请求的条件（Path、Method、Header、时间…）
Filter（过滤器）   请求/响应的处理（GatewayFilter / GlobalFilter）
```

## 3. 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<!-- 注意：不要引入 spring-boot-starter-web，会冲突 -->
```

```yaml
server:
  port: 8080
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true                  # 开启服务名自动路由（/USER-SERVICE/**）
          lower-case-service-id: true
      routes:
        - id: order-route
          uri: lb://order-service         # ★ lb:// = 负载均衡到注册中心服务
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST
          filters:
            - StripPrefix=1               # 去掉 /api 前缀
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@ipKeyResolver}"
            - AddRequestHeader=X-Gateway, scg
        - id: user-route
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
      default-filters:                   # 全局生效
        - DedupeResponseHeader=Access-Control-Allow-Origin
      httpclient:
        connect-timeout: 3000
        response-timeout: 10s
```

## 4. 常用 Predicate

| 断言 | 示例 |
|---|---|
| `Path` | `- Path=/api/**`，支持 `{seg}` 变量 |
| `Method` | `- Method=GET,POST` |
| `Header` | `- Header=X-Token, \d+` |
| `Query` | `- Query=name` 或 `Query=name,zh.*` |
| `Host` | `- Host=**.example.com` |
| `Cookie` | `- Cookie=session,.*` |
| `RemoteAddr` | `- RemoteAddr=192.168.1.0/24` |
| `After/Before/Between` | 时间窗口（可用于灰度/定时开放） |
| `Weight` | `- Weight=group1, 80`（**权重灰度**） |

## 5. 常用 Filter

| Filter | 作用 |
|---|---|
| `StripPrefix=n` | 去掉前 n 段路径 |
| `PrefixPath=/api` | 加前缀 |
| `RewritePath=/a/(?<x>.*), /${x}` | 路径重写 |
| `AddRequestHeader` / `AddRequestParameter` | 加请求头/参数 |
| `RemoveRequestHeader` / `RemoveResponseHeader` | 删除头 |
| `RequestRateLimiter` | 限流（Redis 令牌桶） |
| `CircuitBreaker` | 熔断降级 |
| `Retry` | 重试 |
| `Hystrix`（旧） | 熔断 |
| `SetStatus` | 设置响应码 |
| `RedirectTo` | 重定向 |
| `RequestSize` | 限制请求体大小 |
| `PreserveHostHeader` | 保留 Host |

## 6. 全局过滤器（认证鉴权）

```java
@Component
@Slf4j
@Order(-100)                       // 值越小越早执行
public class AuthGlobalFilter implements GlobalFilter {

    private static final List<String> WHITE = List.of("/api/auth/login", "/actuator/health");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest req = exchange.getRequest();
        String path = req.getPath().value();
        if (WHITE.contains(path)) return chain.filter(exchange);

        String token = req.getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (token == null || !JwtUtil.validate(token)) {
            return unauthorized(exchange);
        }
        // 传递用户信息给下游
        ServerHttpRequest mutated = req.mutate()
                .header("X-User-Id", String.valueOf(JwtUtil.getUserId(token)))
                .build();
        return chain.filter(exchange.mutate().request(mutated).build());
    }

    private Mono<Void> unauthorized(ServerWebExchange e) {
        ServerHttpResponse res = e.getResponse();
        res.setStatusCode(HttpStatus.UNAUTHORIZED);
        res.getHeaders().setContentType(MediaType.APPLICATION_JSON);
        String body = "{\"code\":40100,\"msg\":\"未登录\"}";
        return res.writeWith(Mono.just(res.bufferFactory().wrap(body.getBytes(StandardCharsets.UTF_8))));
    }
}
```

> 网关是**响应式**编程模型，全程 `Mono`/`Flux`，**不要**在里面做阻塞操作（如 JDBC、同步 HTTP）。

## 7. 限流

```java
@Bean
public KeyResolver ipKeyResolver() {
    return exchange -> Mono.just(
        Optional.ofNullable(exchange.getRequest().getRemoteAddress())
                .map(InetSocketAddress::getAddress).map(InetAddress::getHostAddress)
                .orElse("unknown"));
}
// 用户维度：exchange.getRequest().getHeaders().getFirst("X-User-Id")
```
```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 100     # 每秒生成令牌数
      redis-rate-limiter.burstCapacity: 200     # 桶容量（突发）
      redis-rate-limiter.requestedTokens: 1
      key-resolver: "#{@ipKeyResolver}"
```
需要 `spring-boot-starter-data-redis-reactive`。

## 8. 熔断降级

```yaml
filters:
  - name: CircuitBreaker
    args:
      name: orderCircuit
      fallbackUri: forward:/fallback/order
```
```java
@RestController
public class FallbackController {
    @RequestMapping("/fallback/order")
    public Mono<Result<Void>> fb() {
        return Mono.just(Result.fail(50300, "服务暂时不可用"));
    }
}
```
需 `spring-cloud-starter-circuitbreaker-reactor-resilience4j`。

## 9. 跨域

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOriginPatterns: "*"
            allowedMethods: "*"
            allowedHeaders: "*"
            allowCredentials: true
```

## 10. 灰度发布（权重路由）

```yaml
routes:
  - id: v1
    uri: lb://order-service-v1
    predicates:
      - Path=/api/orders/**
      - Weight=group1, 90
  - id: v2
    uri: lb://order-service-v2
    predicates:
      - Path=/api/orders/**
      - Weight=group1, 10
```
或按 header：`- Header=X-Canary, true`。

## 11. 生产要点

- **不要**在网关写业务逻辑（鉴权、限流就够）。
- 网关是单点，**必须集群 + 高可用**（前面挂 SLB/Nginx）。
- 配置超时：`connect-timeout`、`response-timeout` 必须设置。
- 开启 `management.endpoint.gateway` 可动态查看路由。
- 请求体大小限制：默认无限制，设 `RequestSize` filter。

## 12. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 引入 `starter-web` | 启动冲突报 `Spring MVC found` | 移除 web，只用 webflux |
| 404 | 路由未匹配 | 检查 Path、StripPrefix、服务名 |
| 阻塞操作 | 吞吐骤降 | 响应式禁止阻塞（用 WebClient） |
| 限流不生效 | 无 Redis | 加 reactive redis + keyResolver |
| 认证后下游拿不到用户 | 未透传 | 加 `AddRequestHeader` |
| 大文件上传失败 | 超时/限流 | 调 `RequestSize` 与超时 |
| WebSocket 不通 | 需特殊配置 | `spring.cloud.gateway` 支持 ws:// uri |
| 路由刷新 | 需重启 | 用 `/actuator/gateway/refresh` 或配置中心 |

## 13. 面试速答

- **Gateway 三要素**：Route（id + uri + predicates + filters）、Predicate（匹配条件）、Filter（处理逻辑）。
- **为什么用 WebFlux**：非阻塞 IO，单机可支撑更高并发，资源占用低。
- **限流实现**：`RequestRateLimiter` + Redis + Lua 令牌桶，`KeyResolver` 决定维度（IP/用户/接口）。
- **与 Nginx 区别**：Nginx 是反向代理/L4-L7 负载均衡（服务端），Gateway 是应用层网关，做业务相关的路由、鉴权、限流、灰度。
