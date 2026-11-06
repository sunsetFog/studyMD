# 44 · Resilience4j 熔断、限流与隔离

## 1. 为什么需要熔断降级

**服务雪崩**：A→B→C，C 故障 → B 线程等待耗尽 → A 也耗尽 → 全链路崩溃。

应对三板斧：
| 手段 | 说明 |
|---|---|
| **超时** | 快速失败，不无限等待 |
| **熔断** | 下游持续失败时直接拒绝，给下游恢复时间 |
| **隔离（舱壁）** | 限制对某下游的并发/线程数，避免拖垮整个应用 |
| **限流** | 控制进入的 QPS |
| **降级** | 返回兜底结果 |

## 2. 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<!-- 注解方式需要 -->
<dependency><groupId>io.github.resilience4j</groupId><artifactId>resilience4j-spring-boot3</artifactId></dependency>
<dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-aop</artifactId></dependency>
```

> Hystrix 已停止维护，官方推荐 Resilience4j（或阿里的 Sentinel）。

## 3. 熔断器三态

```
CLOSED（关闭，正常）
   │ 失败率 > 阈值(默认 50%)
   ▼
OPEN（打开，直接拒绝请求，走 fallback）
   │ 等待 waitDurationInOpenState(默认 60s)
   ▼
HALF_OPEN（半开，放行部分请求探测）
   │ 成功率高 → CLOSED；仍失败 → OPEN
```

关键参数：
| 参数 | 默认 | 说明 |
|---|---|---|
| `slidingWindowType` | COUNT_BASED | 计数/时间窗口 |
| `slidingWindowSize` | 100 | 统计窗口内请求数 |
| `failureRateThreshold` | 50% | 失败率阈值 |
| `minimumNumberOfCalls` | 100 | 最少调用数才开始计算 |
| `waitDurationInOpenState` | 60s | OPEN 状态持续时间 |
| `permittedNumberOfCallsInHalfOpenState` | 10 | 半开放行数量 |
| `slowCallRateThreshold` | — | 慢调用比例 |
| `slowCallDurationThreshold` | — | 慢调用定义 |

> **熔断器只统计异常**，所以要正确地把超时/4xx/5xx 转成异常抛出。

## 4. 注解式使用

```java
@Service
@Slf4j
public class OrderService {

    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    public UserVO getUser(Long id) {
        return userClient.getById(id);
    }
    // ★ fallback 方法签名必须与目标一致 + 最后一个参数为 Throwable
    private UserVO getUserFallback(Long id, Throwable t) {
        log.warn("降级 id={} reason={}", id, t.getMessage());
        return UserVO.anonymous();
    }

    @TimeLimiter(name = "userService", fallbackMethod = "timeoutFallback")
    public CompletableFuture<UserVO> getUserAsync(Long id) { ... }

    @RateLimiter(name = "queryRate", fallbackMethod = "rateLimitFallback")
    public List<Order> query(Long uid) { ... }

    @Bulkhead(name = "inventory", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<Boolean> deduct(...) { ... }

    @Retry(name = "userService", fallbackMethod = "getUserFallback")
    public UserVO getUserRetry(Long id) { ... }
}
```

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 5
        minimumNumberOfCalls: 20
    instances:
      userService:
        baseConfig: default
        failureRateThreshold: 60
  timelimiter:
    configs:
      default: {timeoutDuration: 3s}
  ratelimiter:
    instances:
      queryRate:
        limitForPeriod: 100
        limitRefreshPeriod: 1s
        timeoutDuration: 0
  bulkhead:
    instances:
      inventory:
        maxConcurrentCalls: 20
        maxWaitDuration: 0
  thread-pool-bulkhead:
    instances:
      inventory:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20
  retry:
    instances:
      userService:
        maxAttempts: 3
        waitDuration: 500ms
        retryExceptions: [ java.net.SocketTimeoutException ]
        ignoreExceptions: [ com.example.BizException ]
```

> 注意注解优先级顺序：Retry → CircuitBreaker → RateLimiter → TimeLimiter → Bulkhead（外层到内层）。

## 5. 编程式（更灵活）

```java
@Service
@RequiredArgsConstructor
public class PayService {
    private final CircuitBreakerRegistry registry;
    private final PayClient client;

    public PayResult pay(PayReq req) {
        CircuitBreaker cb = registry.circuitBreaker("payService");
        Supplier<PayResult> supplier = CircuitBreaker.decorateSupplier(cb, () -> client.pay(req));
        return Try.ofSupplier(supplier)
                .recover(Throwable.class, t -> PayResult.fail("支付暂时不可用"))
                .get();
    }
}
```

## 6. 与 Feign 集成

```yaml
feign:
  circuitbreaker:
    enabled: true
    alphanumeric-ids:
      enabled: true     # 允许服务名带中划线
```
```java
@FeignClient(name = "user-service", fallbackFactory = UserFallbackFactory.class)
```
默认熔断器名为 `UserClient#getById(Long)`，可在 yaml 中配置 `resilience4j.circuitbreaker.instances."UserClient#getById(Long)"`。

## 7. 与 Gateway 集成

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order
          uri: lb://order-service
          filters:
            - name: CircuitBreaker
              args:
                name: orderCb
                fallbackUri: forward:/fallback/order
                statusCodes:
                  - 500
                  - "NOT_FOUND"
```

## 8. Sentinel（阿里，可选对比）

```xml
<dependency><groupId>com.alibaba.cloud</groupId>
           <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId></dependency>
```
```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
```
```java
@SentinelResource(value = "getUser", blockHandler = "block", fallback = "fb")
public UserVO getUser(Long id) { ... }
```

| 对比 | Resilience4j | Sentinel |
|---|---|---|
| 模型 | 函数式、轻量、无控制台 | 有**控制台**、规则动态配置 |
| 功能 | 熔断/限流/隔离/重试 | 熔断/限流/**热点参数**/系统自适应/黑白名单 |
| 生态 | Spring Cloud 官方 | Alibaba 生态 |

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| fallback 不生效 | 方法签名不匹配 | 参数一致 + 末尾 `Throwable` |
| 熔断不触发 | 异常被吞 | 必须抛异常；Feign 要 `dismiss404` |
| 一直熔断 | 阈值/窗口设置不合理 | 调 `minimumNumberOfCalls`、`waitDurationInOpenState` |
| 降级后返回空导致 NPE | 兜底数据缺失 | fallback 返回有意义的默认值 |
| 与 `@Transactional` 混用 | 事务问题 | 降级是 AOP，注意执行顺序 |
| 超时未配置 | 一直等 | 配 `TimeLimiter` 或 HTTP 客户端超时 |
| 注解在 private 方法 | 不生效 | 必须 public |
| 自调用 | 不生效 | 同代理问题 |

## 10. 面试速答

- **熔断三态**：Closed（正常）→ 失败率超阈值 → Open（拒绝）→ 等待后 Half-Open（探测）→ 恢复或再 Open。
- **熔断 vs 降级**：熔断是**触发机制**（自动断开），降级是**处理方式**（返回兜底）。
- **舱壁隔离**：信号量（SemaphoreBulkhead，限制并发数）与线程池（ThreadPoolBulkhead，独立线程池），防止一个下游耗尽所有资源。
- **限流算法**：Resilience4j `RateLimiter` 是令牌桶（`limitForPeriod` + `limitRefreshPeriod`）。
- **为什么 Hystrix 被弃用**：停止维护；Resilience4j 更轻量、函数式、无线程开销（信号量为主）、支持响应式。
