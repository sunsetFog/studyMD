# 53 · Spring WebFlux 与 Reactor

## 1. 为什么需要响应式

| 阻塞模型（MVC） | 非阻塞模型（WebFlux） |
|---|---|
| 一个请求一个线程 | 少量线程（= CPU 核数）处理大量请求 |
| 线程等待 IO 时闲置 | IO 期间线程去处理别的请求 |
| 并发受线程池限制 | 高并发下资源占用低 |
| 编程简单 | 编程复杂（声明式、回调） |

> 注意：WebFlux **不会让单个请求更快**，它提升的是**并发能力**（更高吞吐、更少内存）。CPU 密集型任务收益很小。

## 2. Reactor 核心类型

| 类型 | 元素数 | 说明 |
|---|---|---|
| `Mono<T>` | 0 或 1 | 单个/空 |
| `Flux<T>` | 0..N | 流 |

两者都是**惰性的**：不订阅（subscribe）就不会执行。

```java
Mono<String> m = Mono.just("a").map(String::toUpperCase).delayElement(Duration.ofSeconds(1));
m.subscribe(System.out::println);      // ★ 这一步才触发

Flux<Integer> f = Flux.range(1, 10).filter(i -> i % 2 == 0).map(i -> i * 2);
f.subscribe(i -> log.info("{}", i), err -> log.error("", err), () -> log.info("done"));
```

## 3. 常用操作符

| 类别 | 操作符 |
|---|---|
| 创建 | `just`、`fromIterable`、`range`、`interval`、`empty`、`error`、`defer`、`create` |
| 转换 | `map`、`flatMap`（返回流，异步）、`concatMap`（保序）、`flatMapMany` |
| 过滤 | `filter`、`take`、`skip`、`distinct`、`defaultIfEmpty`、`switchIfEmpty` |
| 组合 | `zip`、`merge`、`concat`、`combineLatest`、`then` |
| 错误处理 | `onErrorResume`、`onErrorReturn`、`onErrorMap`、`retry`、`retryWhen` |
| 副作用 | `doOnNext`、`doOnError`、`doFinally`、`doOnSubscribe` |
| 时间 | `delayElement`、`timeout`、`buffer`、`window` |
| 背压 | `onBackpressureBuffer`、`onBackpressureDrop`、`onBackpressureLatest` |
| 线程 | `publishOn`（影响后续）、`subscribeOn`（影响源头） |

```java
Mono<User> user = userRepo.findById(id)
        .switchIfEmpty(Mono.error(new NotFoundException()))
        .doOnNext(u -> log.info("found {}", u.getId()))
        .timeout(Duration.ofSeconds(2))
        .onErrorResume(TimeoutException.class, e -> Mono.just(User.anonymous()))
        .retryWhen(Retry.backoff(3, Duration.ofMillis(200)));
```

## 4. 背压（Backpressure）

生产者快于消费者时，由**消费者**告诉生产者「我能处理多少」。

- Reactor 默认是无界请求（`Long.MAX_VALUE`），需显式控制。
- `Flux.create` / `Flux.generate` 可感知 request。
- 处理策略：缓冲（可能 OOM）、丢弃最新、保留最新。

## 5. WebFlux 控制器

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {
    private final UserService service;

    @GetMapping("/{id}")
    public Mono<UserVO> get(@PathVariable Long id) { return service.getById(id); }

    @GetMapping
    public Flux<UserVO> list() { return service.findAll(); }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<Long> create(@Valid @RequestBody Mono<UserDTO> dto) {   // ★ Mono 参数
        return dto.flatMap(service::create);
    }

    // SSE 推送
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Long>> stream() {
        return Flux.interval(Duration.ofSeconds(1))
                .map(n -> ServerSentEvent.<Long>builder().data(n).build());
    }
}
```

### 函数式路由（另一种风格）

```java
@Configuration
public class UserRoutes {
    @Bean
    public RouterFunction<ServerResponse> routes(UserHandler h) {
        return route(GET("/api/users/{id}"), h::get)
               .andRoute(GET("/api/users"), h::list)
               .andRoute(POST("/api/users"), h::create);
    }
}
```

## 6. 响应式数据访问

| 组件 | 说明 |
|---|---|
| `R2DBC` | 响应式关系型 DB（MySQL/Postgres/H2） |
| `ReactiveMongoRepository` | MongoDB（原生异步驱动） |
| `ReactiveRedisTemplate` | Redis |
| `WebClient` | HTTP 客户端 |

> **关键**：JDBC 是阻塞的，在 WebFlux 里用 JPA/MyBatis 会**阻塞 EventLoop 线程**，导致吞吐骤降。要用响应式就必须全链路响应式（R2DBC）。

```java
// R2DBC
public interface UserRepository extends ReactiveCrudRepository<User, Long> {
    Mono<User> findByUsername(String username);
}
```
```yaml
spring:
  r2dbc:
    url: r2dbc:mysql://localhost:3306/demo
    username: root
    password: root
```

## 7. WebClient

```java
@Bean
public WebClient webClient() {
    return WebClient.builder()
        .baseUrl("http://user-service")
        .clientConnector(new ReactorClientHttpConnector(
            HttpClient.create().responseTimeout(Duration.ofSeconds(5))))
        .filter(ExchangeFilterFunction.ofRequestProcessor(r -> {
            log.info("{} {}", r.method(), r.url());
            return Mono.just(r);
        }))
        .build();
}

Mono<User> u = webClient.get()
        .uri("/api/users/{id}", id)
        .retrieve()
        .onStatus(HttpStatusCode::is4xxClientError, r -> Mono.error(new BizException("参数错误")))
        .bodyToMono(User.class);

// 并行调用
Mono.zip(orderMono, userMono, (o, u) -> new Detail(o, u));
```

## 8. 阻塞代码怎么办

```java
// 把阻塞调用丢到专门的线程池
Mono.fromCallable(() -> blockingJdbcCall())
    .subscribeOn(Schedulers.boundedElastic());     // ★ 专用弹性线程池
```

> 不要 `block()`（除了测试）；不要在响应式链路里调阻塞库。

## 9. 调度器（Schedulers）

| 调度器 | 用途 |
|---|---|
| `parallel()` | CPU 密集（= 核数） |
| `boundedElastic()` | **阻塞 IO**（有界线程池） |
| `single()` | 单线程 |
| `immediate()` | 当前线程 |
| `fromExecutorService()` | 自定义 |

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 返回 `Mono` 但没订阅 | 什么都没发生 | 框架会订阅；自己写测试要 `block()` |
| 用了阻塞 JDBC | 吞吐比 MVC 还差 | 换 R2DBC 或干脆用 MVC |
| 忘记 `flatMap` 用 `map` | 返回 `Mono<Mono<T>>` | 转换返回流用 flatMap |
| `subscribe` 后主线程结束 | 看不到结果 | 测试用 `StepVerifier` 或 `block()` |
| 在 WebFlux 里 `ThreadLocal` | 取不到 | 用 `Reactor Context`（`contextWrite`） |
| `block()` 在 EventLoop | 报错 `block() is blocking` | 禁止 |
| 异常处理不当 | 流终止 | 用 `onErrorResume` |
| 内存溢出 | 无限流无背压 | 限流/背压策略 |

## 11. 面试速答

- **WebFlux 与 MVC 区别**：MVC 基于 Servlet 阻塞模型（一请求一线程）；WebFlux 基于 Netty + Reactor 非阻塞（少量线程处理海量连接）。
- **Mono 与 Flux**：0~1 个元素 vs 0~N 个元素，都是惰性的，订阅才执行。
- **什么是背压**：消费者向生产者反馈处理能力，防止生产者过快导致消费者 OOM。
- **什么时候用 WebFlux**：高并发、IO 密集、需要长连接（SSE/WebSocket）；**全链路必须响应式**，否则反而不如 MVC。
- **`publishOn` 与 `subscribeOn`**：前者影响后续操作符执行线程，后者影响整个流的订阅/源头线程。
