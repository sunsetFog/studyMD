# 42 · 线程池

## 1. 为什么用线程池

- 复用线程，避免频繁创建/销毁开销（每次创建约 1ms + 1MB 栈）。
- 控制并发数，防止资源耗尽。
- 统一管理、监控、拒绝策略。
- 提供任务排队与延迟/定时能力。

## 2. ThreadPoolExecutor 七大参数

```java
public ThreadPoolExecutor(
    int corePoolSize,                    // 核心线程数
    int maximumPoolSize,                 // 最大线程数
    long keepAliveTime,                  // 非核心线程空闲存活时间
    TimeUnit unit,                       // 时间单位
    BlockingQueue<Runnable> workQueue,   // 任务队列
    ThreadFactory threadFactory,         // 线程工厂（命名、优先级、守护）
    RejectedExecutionHandler handler     // 拒绝策略
) { }
```

### 任务提交流程

```
提交任务
  ├── 当前线程数 < corePoolSize → 新建核心线程执行
  ├── 否则 → 尝试放入队列
  │     ├── 队列未满 → 入队等待
  │     └── 队列已满 → 当前线程数 < maxPoolSize → 新建非核心线程
  │                   └── 已达 maxPoolSize → 执行拒绝策略
```

**关键理解**：先排队，队列满了才扩容到 max（很多人误以为先扩线程）。

### 线程数配置

| 任务类型 | 公式 | 示例（8 核） |
|---|---|---|
| CPU 密集 | `N + 1` | 9 |
| IO 密集 | `N * (1 + W/C)` | 8 * (1 + 9) = 80 |
| 混合 | 拆分两个池 | — |

```java
int cpu = Runtime.getRuntime().availableProcessors();
int core = cpu;
int max = cpu * 2;
```

> 公式只是起点，必须压测。考虑下游承受能力（DB 连接数、第三方限流）。

### 队列选择

| 队列 | 特点 | 风险 |
|---|---|---|
| `ArrayBlockingQueue(n)` | 有界数组 | ✔ 推荐 |
| `LinkedBlockingQueue(n)` | 有界链表，双锁吞吐高 | 必须指定容量 |
| `LinkedBlockingQueue()` | **无界**（Integer.MAX_VALUE） | OOM |
| `SynchronousQueue` | 零容量，直接移交 | 需配合大 maxPoolSize |
| `PriorityBlockingQueue` | 优先级 | 无界 |
| `DelayQueue` | 延迟任务 | 无界 |

### 拒绝策略

| 策略 | 行为 | 适用 |
|---|---|---|
| `AbortPolicy`（默认） | 抛 `RejectedExecutionException` | 需要感知失败 |
| `CallerRunsPolicy` | 调用线程执行任务 | 天然背压（推荐） |
| `DiscardPolicy` | 静默丢弃 | 允许丢任务（日志） |
| `DiscardOldestPolicy` | 丢最老任务再提交 | 只保留最新 |

```java
// 自定义：记录 + 降级
class LoggingRejectPolicy implements RejectedExecutionHandler {
    @Override public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        log.warn("task rejected, poolSize={}, queueSize={}", e.getPoolSize(), e.getQueue().size());
        metrics.counter("threadpool.rejected").increment();
        throw new RejectedExecutionException("系统繁忙");
    }
}
```

### 线程工厂（必须自定义命名）

```java
ThreadFactory factory = new ThreadFactory() {
    private final AtomicInteger seq = new AtomicInteger();
    @Override public Thread newThread(Runnable r) {
        Thread t = new Thread(r, "biz-pool-" + seq.incrementAndGet());
        t.setDaemon(false);
        t.setUncaughtExceptionHandler((th, e) -> log.error("线程 {} 异常", th.getName(), e));
        return t;
    }
};
```

> 默认命名 `pool-1-thread-1` 在排查问题（jstack）时毫无帮助。

## 3. 完整配置示例

```java
@Configuration
public class ThreadPoolConfig {

    @Bean("orderExecutor")
    public ThreadPoolExecutor orderExecutor() {
        int cpu = Runtime.getRuntime().availableProcessors();
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                cpu * 2,                       // core
                cpu * 4,                       // max
                60, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(2000),
                new NamedThreadFactory("order-pool"),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
        executor.setThreadFactory(new NamedThreadFactory("order-pool"));
        executor.allowCoreThreadTimeOut(true);     // 核心线程也可回收
        return executor;
    }
}
```

## 4. Executors 工厂方法（不推荐）

```java
Executors.newFixedThreadPool(10);       // 无界队列 → OOM 风险
Executors.newSingleThreadExecutor();    // 无界队列
Executors.newCachedThreadPool();        // maxPoolSize = Integer.MAX_VALUE → 线程爆炸
Executors.newScheduledThreadPool(4);    // 无界延迟队列
Executors.newWorkStealingPool();        // ForkJoinPool，可用
```

《阿里巴巴 Java 开发手册》强制要求：**不允许使用 Executors 创建线程池，必须手动 new ThreadPoolExecutor**，原因是无法控制队列与最大线程数。

## 5. 提交任务

```java
// execute：无返回值，异常直接抛到线程（可能静默丢失）
executor.execute(() -> doWork());

// submit：返回 Future，异常封装在 Future 中
Future<Integer> f = executor.submit(() -> compute());
try { f.get(1, TimeUnit.SECONDS); }
catch (ExecutionException e) { log.error("task failed", e.getCause()); }

// invokeAll：批量提交，等待全部完成
List<Future<Integer>> futures = executor.invokeAll(tasks);

// invokeAny：返回第一个成功的结果
Integer first = executor.invokeAny(tasks);
```

**execute vs submit 的异常差异**是高频坑：

```java
executor.execute(() -> { throw new RuntimeException("x"); });   // 线程异常，可能无日志

Future<?> f = executor.submit(() -> { throw new RuntimeException("x"); });
f.get();    // 必须 get 才能看到异常，否则被吞
```

## 6. 生命周期

```java
executor.shutdown();                       // 不再接受新任务，等待已提交任务完成
executor.shutdownNow();                    // 中断所有线程，返回未执行的任务
executor.awaitTermination(30, TimeUnit.SECONDS);
executor.isShutdown();
executor.isTerminated();
```

优雅关闭模板：

```java
@PreDestroy
public void destroy() {
    executor.shutdown();
    try {
        if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
            log.warn("线程池未在 30s 内关闭，强制关闭");
            executor.shutdownNow();
        }
    } catch (InterruptedException e) {
        executor.shutdownNow();
        Thread.currentThread().interrupt();
    }
}
```

## 7. 监控

```java
public Map<String, Object> metrics(ThreadPoolExecutor e) {
    return Map.of(
        "poolSize", e.getPoolSize(),
        "activeCount", e.getActiveCount(),
        "queueSize", e.getQueue().size(),
        "queueRemaining", e.getQueue().remainingCapacity(),
        "completedTasks", e.getCompletedTaskCount(),
        "largestPoolSize", e.getLargestPoolSize(),
        "taskCount", e.getTaskCount()
    );
}
```

关键指标与告警：

| 指标 | 阈值建议 |
|---|---|
| 队列使用率 | > 80% 告警 |
| 活跃线程 / max | > 90% 告警 |
| 拒绝次数 | > 0 告警 |
| 任务耗时 P99 | 业务定义 |

Spring Boot Actuator + Micrometer 可自动暴露。

## 8. 定时任务

```java
ScheduledExecutorService ses = Executors.newScheduledThreadPool(2, factory);

// 固定频率（以上次开始时间为基准）
ses.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS);

// 固定延迟（以上次结束时间为基准）
ses.scheduleWithFixedDelay(task, 0, 1, TimeUnit.SECONDS);

// 一次性延迟
ses.schedule(task, 5, TimeUnit.SECONDS);

// 取消
ScheduledFuture<?> f = ses.scheduleAtFixedRate(task, 0, 1, SECONDS);
f.cancel(false);
```

**关键差异**：

- `scheduleAtFixedRate`：任务执行时间 > 周期时，会连续执行（追赶）。
- `scheduleWithFixedDelay`：任务之间固定间隔，不会堆积。

坑：任务抛异常会导致**后续执行终止**（务必 try-catch）。

```java
ses.scheduleWithFixedDelay(() -> {
    try { doWork(); }
    catch (Throwable e) { log.error("scheduled task failed", e); }   // 必须捕获
}, 0, 60, TimeUnit.SECONDS);
```

## 9. ForkJoinPool

```java
ForkJoinPool pool = new ForkJoinPool();     // 默认并行度 = CPU 核数

// 递归任务
class SumTask extends RecursiveTask<Long> {
    private final long[] arr;
    private final int lo, hi;
    private static final int THRESHOLD = 10_000;

    SumTask(long[] arr, int lo, int hi) { this.arr = arr; this.lo = lo; this.hi = hi; }

    @Override protected Long compute() {
        if (hi - lo <= THRESHOLD) {
            long s = 0;
            for (int i = lo; i < hi; i++) s += arr[i];
            return s;
        }
        int mid = (lo + hi) >>> 1;
        SumTask left = new SumTask(arr, lo, mid);
        SumTask right = new SumTask(arr, mid, hi);
        left.fork();
        long r = right.compute();       // 一个 fork 一个 compute
        return left.join() + r;
    }
}

long sum = pool.invoke(new SumTask(arr, 0, arr.length));

// RecursiveAction：无返回值
```

工作窃取（work-stealing）：空闲线程从其他线程队列尾部偷任务，提高利用率。

并行流底层就是 ForkJoinPool.commonPool()：

```java
list.parallelStream()...                    // 共享 commonPool（默认 CPU-1 个线程）
// 自定义池
ForkJoinPool custom = new ForkJoinPool(8);
custom.submit(() -> list.parallelStream().forEach(...)).get();
```

> 并行流不适合 IO 密集任务，且共享 commonPool 会干扰其他并行流。

## 10. 线程池隔离

不同业务用不同线程池，避免相互影响：

```java
// 订单、支付、通知各一个池
// 好处：支付池打满不影响订单；便于独立监控与调参
```

微服务中的舱壁模式（Bulkhead）：每个下游依赖独立线程池/信号量。

## 11. 常见坑

1. 用 `Executors.newFixedThreadPool`（无界队列 OOM）。
2. 用 `newCachedThreadPool`（线程数无上限）。
3. 队列无界 + 任务堆积（延迟剧增、内存溢出）。
4. 线程池不命名（排查困难）。
5. `submit` 后不 `get`（异常被吞）。
6. 任务中未捕获异常（定时任务停止、线程重建）。
7. 忘记 `shutdown`（JVM 不退出）。
8. 在池内任务中提交任务并等待（死锁，尤其单线程池）。
9. 线程池中做阻塞 IO 且池太小（吞吐低）。
10. `ThreadLocal` 未清理（脏数据 + 内存泄漏）。
11. 使用 `InheritableThreadLocal` 配合线程池（复用线程不会继承新值）。
12. 核心线程数配置未考虑下游承受能力（打垮 DB）。

```java
// 死锁示例
ExecutorService single = Executors.newSingleThreadExecutor();
Future<?> outer = single.submit(() -> {
    Future<?> inner = single.submit(() -> System.out.println("inner"));
    inner.get();      // 永远等不到：唯一线程被 outer 占用
});
```

## 12. 完整工具类

```java
public final class ThreadPools {
    private ThreadPools() { }

    public static ThreadPoolExecutor newPool(String name, int core, int max, int queueSize) {
        ThreadPoolExecutor e = new ThreadPoolExecutor(
                core, max, 60L, TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(queueSize),
                new NamedThreadFactory(name),
                new ThreadPoolExecutor.CallerRunsPolicy());
        e.allowCoreThreadTimeOut(true);
        return e;
    }

    static class NamedThreadFactory implements ThreadFactory {
        private final String prefix;
        private final AtomicInteger seq = new AtomicInteger();
        private final ThreadFactory delegate = Executors.defaultThreadFactory();

        NamedThreadFactory(String prefix) { this.prefix = prefix; }

        @Override public Thread newThread(Runnable r) {
            Thread t = delegate.newThread(r);
            t.setName(prefix + "-" + seq.incrementAndGet());
            t.setUncaughtExceptionHandler((th, ex) ->
                LoggerFactory.getLogger(ThreadPools.class)
                    .error("线程 {} 未捕获异常", th.getName(), ex));
            return t;
        }
    }
}
```

## 13. 速查

| 需求 | 做法 |
|---|---|
| 创建池 | `new ThreadPoolExecutor(...)` |
| 队列 | 有界 `ArrayBlockingQueue` |
| 拒绝 | `CallerRunsPolicy` 或自定义 |
| 命名 | 自定义 `ThreadFactory` |
| 关闭 | `shutdown` + `awaitTermination` |
| 定时 | `ScheduledExecutorService` |
| 分治 | `ForkJoinPool` |
| 监控 | `getActiveCount` / `getQueue().size()` |
| 隔离 | 每业务独立池 |
