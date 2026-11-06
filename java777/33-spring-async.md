# 33 · @Async 异步与线程池

## 1. 启用

```java
@Configuration
@EnableAsync
public class AsyncConfig { }
```

```yaml
# Boot 3 可用自定义 TaskExecutor Bean，或配置默认池
spring:
  task:
    execution:
      pool:
        core-size: 8
        max-size: 16
        queue-capacity: 500
        keep-alive: 60s
      thread-name-prefix: task-
```

> Boot 会自动配置 `applicationTaskExecutor`（`ThreadPoolTaskExecutor`），但**默认队列是 `Integer.MAX_VALUE`**，等同于 max-size 无效，务必自定义。

## 2. 自定义线程池（必须）

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("bizExecutor")
    public ThreadPoolTaskExecutor bizExecutor() {
        ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
        e.setCorePoolSize(8);
        e.setMaxPoolSize(16);
        e.setQueueCapacity(500);                 // ★ 有界队列，防止 OOM
        e.setKeepAliveSeconds(60);
        e.setThreadNamePrefix("biz-");
        e.setTaskDecorator(mdcDecorator());       // 传递 MDC / SecurityContext
        e.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());  // ★ 降级
        e.setWaitForTasksToCompleteOnShutdown(true);
        e.setAwaitTerminationSeconds(30);         // 优雅停机
        e.initialize();
        return e;
    }

    @Bean
    public TaskDecorator mdcDecorator() {
        return r -> {
            Map<String, String> ctx = MDC.getCopyOfContextMap();
            SecurityContext sc = SecurityContextHolder.getContext();
            return () -> {
                if (ctx != null) MDC.setContextMap(ctx);
                SecurityContextHolder.setContext(sc);
                try { r.run(); } finally { MDC.clear(); SecurityContextHolder.clearContext(); }
            };
        };
    }

    @Override-ish // 异常处理
    @Bean
    public AsyncConfigurer asyncConfigurer() {
        return new AsyncConfigurer() {
            @Override public Executor getAsyncExecutor() { return bizExecutor(); }
            @Override public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
                return (ex, method, params) -> log.error("异步异常 method={}", method.getName(), ex);
            }
        };
    }
}
```

## 3. 使用

```java
@Async("bizExecutor")          // ★ 指定线程池，别用默认
public void sendSms(String phone) { ... }

@Async
public CompletableFuture<Report> genReport(Long id) {
    return CompletableFuture.completedFuture(build(id));   // 有返回值必须 CompletableFuture
}

// 调用方
CompletableFuture<Report> f = service.genReport(1);
f.get(10, TimeUnit.SECONDS);                 // ★ 必须设超时
CompletableFuture.allOf(f1, f2).join();      // 等待多个
```

> 返回值还可以是 `ListenableFuture`（已废弃）/ `Future`。

## 4. 七大失效场景

| 场景 | 原因 |
|---|---|
| 未加 `@EnableAsync` | 无后置处理器 |
| 方法非 `public` | 无法代理 |
| 同类自调用 | 未走代理 |
| 方法 `final`/`static` | 无法代理 |
| 返回值不是 `void`/`Future` | 异步结果拿不到 |
| 异步方法与调用者在同一类 | 自调用 |
| 对象不是 Spring Bean | 无代理 |

> 与 `@Transactional` 一样是代理机制，同一套坑。

## 5. 线程池参数怎么定

| 类型 | 公式 | 说明 |
|---|---|---|
| CPU 密集型 | `N(cpu) 或 N+1` | 计算为主 |
| IO 密集型 | `N × (1 + 等待时间/计算时间)`，常取 `2N ~ 4N` | 远程调用/DB 为主 |

实践：
- 队列：**必须设界**；`LinkedBlockingQueue` 无界会 OOM。
- 拒绝策略：
  - `AbortPolicy`（默认，抛异常）
  - `CallerRunsPolicy`（**推荐**，让调用线程执行，天然限流）
  - `DiscardPolicy` / `DiscardOldestPolicy`（丢任务，慎选）
- 线程名要有意义（排障靠它）。

## 6. 异步的上下文问题

| 上下文 | 是否传递 | 解决 |
|---|---|---|
| `ThreadLocal` | ❌ | `TaskDecorator` |
| MDC（日志 traceId） | ❌ | `TaskDecorator` |
| `SecurityContext` | ❌ | `DelegatingSecurityContextExecutor` 或 `SecurityContextHolder` 手动传 |
| `RequestContext` | ❌ | 提前取值传入 |
| **事务** | ❌ | 新线程无事务（`@Transactional` 与 `@Async` 不要同方法） |

```java
// 安全上下文传递
Executor delegate = new DelegatingSecurityContextExecutor(bizExecutor);
```

## 7. 定时任务 @Scheduled

```java
@Configuration @EnableScheduling
public class ScheduleConfig { }

@Component
public class Jobs {
    @Scheduled(fixedRate = 60000)                     // 上次开始后 60s（不管是否完成）
    @Scheduled(fixedDelay = 60000)                    // 上次结束后 60s
    @Scheduled(initialDelay = 10000, fixedRate = 30000)
    @Scheduled(cron = "0 0/5 * * * ?")                // 每 5 分钟
    @Scheduled(cron = "${jobs.clean.cron:0 0 2 * * ?}")
    public void clean() { }
}
```

> **默认单线程**（`ScheduledThreadPoolExecutor` core=1），多个任务会互相阻塞。自定义：

```java
@Bean
public TaskScheduler taskScheduler() {
    ThreadPoolTaskScheduler s = new ThreadPoolTaskScheduler();
    s.setPoolSize(5);
    s.setThreadNamePrefix("sched-");
    s.setErrorHandler(t -> log.error("定时任务异常", t));
    s.setRemoveOnCancelPolicy(true);
    return s;
}
```

**分布式环境**多实例会重复执行 → 用分布式锁（Redisson/ShedLock）或独立调度中心（XXL-Job / ElasticJob / PowerJob）。

```xml
<dependency><groupId>net.javacrumbs.shedlock</groupId>
           <artifactId>shedlock-spring</artifactId><version>5.10.2</version></dependency>
```
```java
@SchedulerLock(name = "cleanJob", lockAtMostFor = "10m", lockAtLeastFor = "1m")
@Scheduled(cron = "0 0 2 * * ?")
public void clean() {}
```

## 8. 异步编排（CompletableFuture）

```java
CompletableFuture<User> u = CompletableFuture.supplyAsync(() -> userService.get(id), executor);
CompletableFuture<List<Order>> o = CompletableFuture.supplyAsync(() -> orderService.list(id), executor);

CompletableFuture<VO> vo = u.thenCombine(o, (user, orders) -> new VO(user, orders))
        .orTimeout(3, TimeUnit.SECONDS)                 // JDK 9+
        .exceptionally(ex -> VO.empty());

// 多个并行后汇聚
CompletableFuture.allOf(f1, f2, f3).join();
```

> 注意：`supplyAsync` 默认用 `ForkJoinPool.commonPool`，**必须传入自定义线程池**。

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 默认队列无界 | OOM | 自定义池，`queue-capacity` 设界 |
| 异步异常丢失 | 日志无堆栈 | `AsyncUncaughtExceptionHandler`；有返回值时 `get()` 会抛 |
| 事务不生效 | 数据不一致 | `@Async` 方法内自己开事务，别混用 |
| traceId 丢失 | 日志无链路 | `TaskDecorator` 传递 MDC |
| 多线程并发上下文 | SecurityContext 丢失 | 手动传递 |
| 优雅停机丢任务 | 任务被中断 | `waitForTasksToCompleteOnShutdown=true` |
| 线程池参数拍脑袋 | 压测才发现问题 | 按 CPU/IO 类型估 + 压测 |
| 定时任务重复执行 | 多实例重复 | ShedLock / XXL-Job |

## 10. 面试速答

- **@Async 原理**：`@EnableAsync` 注册 `AsyncAnnotationBeanPostProcessor`，为 Bean 生成代理，调用时把方法提交到 `TaskExecutor`。
- **为什么线程池要自定义**：Boot 默认队列无界（`Integer.MAX_VALUE`），高并发下会堆积导致 OOM；自定义有界队列 + 合理拒绝策略。
- **拒绝策略怎么选**：核心业务用 `CallerRunsPolicy`（降级不丢），非核心用 `AbortPolicy` + 告警。
- **异步丢失上下文怎么办**：`TaskDecorator` 传递 MDC/ThreadLocal；事务与 `@Async` 不要放同一方法。
