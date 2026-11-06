# 23 · 线程池

## 一、为什么必须用线程池

- 复用线程，避免创建/销毁开销（每线程默认 1MB 栈）
- 控制并发上限，防止资源耗尽
- 统一管理：命名、监控、拒绝策略

## 二、ThreadPoolExecutor 七参数（必背）

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        4,                                  // corePoolSize 核心线程数
        16,                                 // maximumPoolSize 最大线程数
        60, TimeUnit.SECONDS,               // 空闲线程存活时间
        new ArrayBlockingQueue<>(100),      // 工作队列
        new ThreadFactory() {               // 命名工厂（排查必备）
            private final AtomicInteger seq = new AtomicInteger();
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r, "biz-pool-" + seq.incrementAndGet());
                t.setDaemon(false);
                return t;
            }
        },
        new ThreadPoolExecutor.CallerRunsPolicy()   // 拒绝策略
);
```

## 三、任务提交流程（面试核心）

```
submit 任务
  → 核心线程未满？→ 创建核心线程执行
  → 满？→ 队列未满？→ 入队等待
  → 队满？→ 最大线程未满？→ 创建非核心线程执行
  → 最大也满？→ 执行拒绝策略
```

**记忆点：先入队再扩线程**（与直觉相反），队列是「缓冲优先」。

### 拒绝策略四种

| 策略 | 行为 | 适用 |
|---|---|---|
| AbortPolicy（默认） | 抛 RejectedExecutionException | 可感知失败，调用方降级 |
| CallerRunsPolicy | 调用线程自己执行 | 不丢任务（上游变慢形成背压） |
| DiscardPolicy | 静默丢弃 | 可容忍丢失（日志/指标） |
| DiscardOldestPolicy | 丢队首最老的再入队 | 只关心最新数据 |

自定义：实现 RejectedExecutionHandler，落 MQ/日志告警。

## 四、参数怎么定

```
CPU 密集型（计算/加解密）：core = 核数 + 1
IO 密集型（RPC/DB/HTTP）：  core = 核数 × 2 或 核数 × (1 + 等待时间/计算时间)
混合型：压测定（唯一可靠方法）

队列容量：有界！容量 = 预估突发 × 可接受延迟
maxPoolSize：兜底扩容空间
```

**CPU 核数**：`Runtime.getRuntime().availableProcessors()`（容器内注意配额）。

## 五、禁止使用 Executors 工厂（阿里规约 + 面试高频）

| 工厂方法 | 隐患 |
|---|---|
| newFixedThreadPool / newSingleThreadExecutor | **无界队列 LinkedBlockingQueue** → 任务堆积 OOM |
| newCachedThreadPool | **最大线程 Integer.MAX_VALUE** → 无限创建线程 OOM |
| newScheduledThreadPool | 无界队列 |

结论：**手动 new ThreadPoolExecutor**，七个参数全部显式，杜绝默认值陷阱。

## 六、提交与获取结果

```java
// execute：无返回值（Runnable）
executor.execute(() -> log());

// submit：返回 Future（Callable/Runnable）
Future<Integer> f = executor.submit(() -> compute());
Integer r = f.get(3, TimeUnit.SECONDS);       // 带超时！

// invokeAll / invokeAny（批量）
List<Future<Integer>> all = executor.invokeAll(tasks, 5, TimeUnit.SECONDS);

// 延迟/周期任务：ScheduledThreadPoolExecutor
ScheduledExecutorService ses = new ScheduledThreadPoolExecutor(2);
ses.schedule(task, 5, TimeUnit.SECONDS);                          // 延迟
ses.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS);            // 固定频率
ses.scheduleWithFixedDelay(task, 0, 1, TimeUnit.SECONDS);         // 固定间隔（上次结束算起）
// 周期任务抛异常 → 后续不再执行！任务体必须自己 catch
```

## 七、优雅关闭

```java
executor.shutdown();               // 不收新任务，处理完存量
if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
    executor.shutdownNow();        // interrupt 中断在跑的任务
}
// Spring 场景：setWaitForTasksToCompleteOnShutdown(true)（衔接 SpringBoot 系列 19 篇）
```

## 八、监控与诊断

```java
executor.getActiveCount();          // 活跃线程
executor.getQueue().size();         // 积压数（关键指标！）
executor.getCompletedTaskCount();
executor.getLargestPoolSize();
// 队列积压持续增长 = 下游处理不动 → 告警
```

排查命令：`jstack` 看线程名（所以必须命名）。

## 九、Spring 环境整合（衔接）

```java
@Bean(destroyMethod = "shutdown")
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
    ex.setCorePoolSize(8);
    ex.setMaxPoolSize(16);
    ex.setQueueCapacity(500);
    ex.setThreadNamePrefix("async-");
    ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    ex.setWaitForTasksToCompleteOnShutdown(true);
    ex.setAwaitTerminationSeconds(30);
    return ex;
}
```

## 常见坑

- **坑 1**：Executors.newFixedThreadPool 无界队列 → OOM（必考）
- **坑 2**：队列无界 + maxPool 永远不生效 → 扩容逻辑死代码
- **坑 3**：get() 不带超时 → 永久阻塞
- **坑 4**：任务抛异常无 Future 接收 → execute 模式下静默丢失；包装 try/catch 或 UncaughtExceptionHandler
- **坑 5**：scheduleAtFixedRate 任务体不 catch 异常 → 周期任务静默死亡
- **坑 6**：线程数拍脑袋 200 → 上下文切换开销；按压测

## Checklist

- [ ] 手动 new ThreadPoolExecutor 七参数齐全
- [ ] 有界队列 + 命名 ThreadFactory
- [ ] 拒绝策略按「可否丢任务」选择
- [ ] get 带超时；周期任务体 catch
- [ ] 队列积压纳入监控
