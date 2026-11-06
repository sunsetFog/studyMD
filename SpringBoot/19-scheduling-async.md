# 19 · 定时任务与异步任务

## 一、@Scheduled 定时任务

```java
@Component
@Slf4j
public class ReportTask {

    // cron：秒 分 时 日 月 周
    @Scheduled(cron = "0 0 2 * * ?")        // 每天凌晨 2 点
    public void dailyReport() { }

    @Scheduled(fixedRate = 5, timeUnit = TimeUnit.MINUTES)   // 每 5 分钟（按起始时间）
    public void sync() { }

    @Scheduled(fixedDelay = 60, timeUnit = TimeUnit.SECONDS) // 上次执行完再等 60s（推荐防重叠）
    public void poll() { }
}
```

启动：启动类加 `@EnableScheduling`。

**Cron 速记**：
```
0 0/30 * * * ?       每 30 分钟
0 0 9-18 * * MON-FRI 工作日 9-18 点整点
0 0 0 1 * ?          每月 1 号零点
```

## 二、单线程问题与线程池配置

默认 @Scheduled 是**单线程调度器**——一个任务卡住，全部任务停摆：

```yaml
spring:
  task:
    scheduling:
      pool:
        size: 5
      thread-name-prefix: sched-
```

或代码配置：

```java
@Configuration
@EnableScheduling
public class ScheduleConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.setThreadNamePrefix("sched-");
        scheduler.initialize();
        registrar.setTaskScheduler(scheduler);
    }
}
```

## 三、@Async 异步任务

```java
@SpringBootApplication
@EnableAsync
public class App { }

@Service
public class NotifyService {

    @Async("taskExecutor")                     // 指定线程池
    public void sendSms(Long userId, String content) { }
}
```

## 四、线程池手动创建（必须）

`@Async` 默认走 SimpleAsyncTaskExecutor（不复用线程！）。**必须自定义 ThreadPoolTaskExecutor**：

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(500);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("async-");
        // 队列满 + 线程满：由调用线程自己执行（不丢任务），或 ABORT 抛异常走降级
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);   // 优雅停机
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }
}
```

线程数经验：CPU 密集 = 核数 + 1；IO 密集 = 核数 * 2（按压测调优）。

## 五、异步异常处理

```java
// 无返回值的 @Async 异常 → AsyncUncaughtExceptionHandler
@Configuration
public class AsyncExceptionConfig implements AsyncConfigurer {
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> log.error("异步任务异常 method={}", method.getName(), ex);
    }
}
```

```java
// 有返回值 → 调用方处理
@Async
public CompletableFuture<String> fetch() { ... }

CompletableFuture<String> f = service.fetch();
f.whenComplete((r, e) -> { if (e != null) log.error("失败", e); });
```

## 六、并发执行与批量并行

```java
// 并行调用多个下游，汇总结果
List<CompletableFuture<Detail>> futures = skuIds.stream()
        .map(id -> CompletableFuture.supplyAsync(() -> detailService.load(id), executor))
        .toList();

CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
List<Detail> details = futures.stream().map(CompletableFuture::join).toList();
```

## 七、分布式环境：定时任务防重复执行

多实例部署下 @Scheduled 每个节点都会执行：

| 方案 | 说明 |
|---|---|
| Redis 分布式锁（17 篇切面） | 注解式 tryLock，简单通用 |
| ShedLock | 库表记录执行状态，轻量 |
| XXL-JOB / PowerJob | 分布式调度平台，支持分片、失败重试、可视化 |

```java
@Scheduled(cron = "0 0 3 * * ?")
@DistributedLock(key = "job:daily-settle", ttl = 3600)   // 自定义切面
public void settle() { }
```

> 复杂调度需求直接上 XXL-JOB，不要在业务代码里造调度轮子。

## 常见坑

- **坑 1**：忘了 @EnableAsync → @Async 变同步执行
- **坑 2**：@Async 同类自调用 → 失效（同事务/AOP）
- **坑 3**：@Scheduled 任务抛异常 → 该任务后续仍执行，但异常被吞；方法内要 catch + log
- **坑 4**：默认调度单线程 → 任务互相阻塞；配 pool
- **坑 5**：多实例重复执行 → 分布式锁或调度平台
- **坑 6**：异步方法返回 void 但抛异常 → 静默丢失，必须配全局异常处理器

## Checklist

- [ ] 自定义 ThreadPoolTaskExecutor，拒绝策略明确
- [ ] @Async 指定线程池名，不自调用
- [ ] 调度任务线程池隔离 + 分布式防重
- [ ] 任务异常全部记录日志并可告警
