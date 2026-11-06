# 45 · 虚拟线程（Virtual Threads）

## 1. 背景：平台线程的瓶颈

传统 Java 线程 = 操作系统线程（1:1）：

- 每个线程默认栈 1MB（`-Xss`），1 万线程 ≈ 10GB。
- 上下文切换成本高（微秒级）。
- 线程数上限通常几千。

后果：高并发 IO 场景下，线程成为瓶颈，只能靠异步回调/响应式（复杂、难调试）。

## 2. 虚拟线程是什么

JDK 19 预览，**JDK 21 正式**（JEP 444）。

- 由 JVM 管理的轻量级线程（M:N 调度）。
- 大量虚拟线程复用少量**载体线程**（carrier，即平台线程）。
- 创建成本极低（几百字节），可创建**百万级**。
- 阻塞时自动**卸载**（unmount），载体线程去执行其他虚拟线程。

```java
Thread vt = Thread.startVirtualThread(() -> {
    System.out.println("running in " + Thread.currentThread());
});

// 或
Thread v2 = Thread.ofVirtual().name("vt-1").start(task);

// 线程工厂
ThreadFactory factory = Thread.ofVirtual().factory();
ExecutorService pool = Executors.newThreadPerTaskExecutor(factory);
```

## 3. 核心用法

### 启动虚拟线程

```java
// 1. 直接启动
Thread.startVirtualThread(() -> doWork());

// 2. Builder
Thread t = Thread.ofVirtual()
        .name("worker-", 0)
        .unstarted(() -> doWork());
t.start();
t.join();

// 3. ExecutorService（推荐用于结构化任务）
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<String> f = executor.submit(() -> fetch(url));
    String result = f.get();
}
```

### 每请求一线程

```java
// Web 服务器（Tomcat 10.1+/Spring Boot 3.2+ 可开启）
// application.properties:
// spring.threads.virtual.enabled=true
```

```java
// 手动处理大量任务
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<Result>> futures = tasks.stream()
            .map(t -> executor.submit(() -> process(t)))
            .toList();

    for (Future<Result> f : futures) {
        results.add(f.get());
    }
}
```

> `newVirtualThreadPerTaskExecutor` 每个任务创建一个虚拟线程，不需要池化（创建太便宜）。

## 4. 原理

```
虚拟线程 V1..Vn
      ↓ 调度
ForkJoinPool（载体线程池，默认 = CPU 核数）
      ↓
操作系统线程
```

### 运行与挂起

```
V1 运行 → 遇到阻塞（sleep/IO/lock）
       → JVM 把 V1 的栈帧（存在堆上的 StackChunk）保存
       → V1 卸载，载体线程去跑 V2
       → V1 阻塞结束 → 重新调度到某个载体线程继续
```

关键：**虚拟线程的栈在堆上**（可增长），所以能创建百万个。

### 调度器

- 默认 `ForkJoinPool`，并行度 = `Runtime.availableProcessors()`。
- 可调：`-Djdk.virtualThreadScheduler.parallelism=N`。
- **不要**用 `ForkJoinPool.commonPool()` 的并行度调整去限制虚拟线程（它们是独立的池）。

## 5. 关键限制：Pinning（钉住）

虚拟线程在以下情况阻塞时**无法卸载**，会一直占住载体线程：

1. `synchronized` 块中阻塞（JDK 21-23；**JDK 24 已修复** JEP 491）。
2. 调用 native 方法/JNI。
3. `Object.wait()`。

```java
// 危险：在 synchronized 中做 IO
synchronized (lock) {
    Thread.sleep(1000);       // 载体线程被钉住
}

// 改：用 ReentrantLock
lock.lock();
try {
    Thread.sleep(1000);       // 可正常卸载
} finally {
    lock.unlock();
}
```

检测 pinning：

```bash
-Djdk.tracePinnedThreads=full      # 打印钉住的栈
```

> JDK 24+ 解决了 synchronized 的 pinning 问题，但老版本需注意。

其他限制：

- `ThreadLocal` 每个虚拟线程独立，百万线程 × 大对象 = OOM（谨慎使用）。
- 不支持 `Thread.stop/suspend`（本来就不该用）。
- `ThreadGroup` 行为退化。
- 线程优先级无效。
- 不是「更快」，而是「更高并发」——CPU 密集任务无收益。

## 6. 适用场景

| 场景 | 适合 | 说明 |
|---|---|---|
| IO 密集（HTTP 调用、DB 查询） | ✔✔✔ | 主要受益者 |
| 每请求一线程模型 | ✔✔✔ | 简化异步代码 |
| 大量并发连接（10 万+） | ✔✔✔ | 替代响应式 |
| CPU 密集计算 | ✘ | 用固定大小平台线程池 |
| 需要线程池限流 | ✘ | 虚拟线程不该池化 |

### 不要池化虚拟线程

```java
// 错误：虚拟线程不需要池
ExecutorService pool = Executors.newFixedThreadPool(1000);   // 仍是平台线程

// 正确
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

虚拟线程创建成本极低，池化反而限制了并发。

### 限流要用 Semaphore

```java
private final Semaphore limit = new Semaphore(100);   // 限制对下游的并发

void handle() {
    limit.acquire();
    try { callDownstream(); }
    finally { limit.release(); }
}
```

## 7. 结构化并发（预览）

JDK 21 预览（JEP 453），JDK 25 仍为预览/演进中。

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Supplier<User> user = scope.fork(() -> userService.get(id));
    Supplier<Order> order = scope.fork(() -> orderService.get(id));

    scope.join();                 // 等待全部
    scope.throwIfFailed();        // 任一失败则抛出

    return new Detail(user.get(), order.get());
}
// 作用域结束自动取消未完成任务
```

优势：

- 子任务生命周期绑定父作用域（不会泄漏）。
- 任一失败自动取消其他（`ShutdownOnFailure`）。
- 任一成功即返回（`ShutdownOnSuccess`）。
- 堆栈清晰（父子关系可见）。

## 8. 与响应式编程对比

| 维度 | 虚拟线程 | 响应式（WebFlux/Reactor） |
|---|---|---|
| 编程模型 | 同步阻塞写法 | 异步流式 |
| 学习成本 | 低 | 高 |
| 调试 | 普通堆栈 | 复杂（需要 hook） |
| 吞吐 | 高 | 高 |
| 内存 | 每线程几百字节 | 更低（事件循环） |
| 生态兼容 | 完全兼容（JDBC 等） | 需要响应式驱动 |
| CPU 密集 | 不适用 | 不适用 |

**结论**：虚拟线程让「同步代码获得异步性能」，大多数业务系统不必上响应式。

## 9. 迁移建议

1. 升级到 JDK 21+。
2. 替换 `Executors.newFixedThreadPool` 为 `newVirtualThreadPerTaskExecutor`（IO 密集处）。
3. 检查 `synchronized` 中的阻塞调用（改 `ReentrantLock`）。
4. 检查 `ThreadLocal` 使用（尤其大对象、连接池绑定）。
5. 用 `Semaphore` 或信号量限流替代线程池大小限制。
6. 压测验证（不要假设一定更快）。

```java
// 旧代码
ExecutorService pool = Executors.newFixedThreadPool(200);
for (Task t : tasks) pool.submit(() -> process(t));

// 新代码
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Task t : tasks) executor.submit(() -> process(t));
}
```

## 10. 观测与诊断

```bash
jcmd <pid> Thread.dump_to_file -format=json /tmp/threads.json
```

- 虚拟线程默认不出现在 `jstack` 中（数量太多）。
- `Thread.getAllStackTraces()` 不包含虚拟线程。
- JFR 事件：`jdk.VirtualThreadStart`、`jdk.VirtualThreadEnd`、`jdk.VirtualThreadPinned`。

```java
// 代码中判断
Thread.currentThread().isVirtual();
```

## 11. 性能数据（参考）

| 场景 | 平台线程 | 虚拟线程 |
|---|---|---|
| 创建 100 万 | OOM | 秒级完成，约 1GB 内存 |
| 10 万并发 HTTP 调用 | 不可行 | 可行 |
| CPU 密集任务 | 相当 | 相当（无优势） |
| 简单计算吞吐 | 略高 | 略低（调度开销） |

## 12. 常见坑

1. 池化虚拟线程（失去意义）。
2. 在 `synchronized` 中阻塞（pinning，JDK 24 前）。
3. 用虚拟线程做 CPU 密集任务（无收益）。
4. 依赖 `ThreadLocal` 缓存大对象（OOM）。
5. 认为虚拟线程一定更快（高并发才体现）。
6. 用 `Thread.currentThread().getThreadGroup()`（行为异常）。
7. 虚拟线程中调用 `Thread.stop`（不支持）。
8. 忘记虚拟线程也需要限制下游并发（打垮 DB）。
9. 在虚拟线程中用固定大小 `ForkJoinPool` 做并行计算（调度冲突）。
10. 监控工具未适配（看不到线程数暴增）。

## 13. 速查

| 需求 | 写法 |
|---|---|
| 启动虚拟线程 | `Thread.startVirtualThread(r)` |
| Builder | `Thread.ofVirtual().name("x").start(r)` |
| 执行器 | `Executors.newVirtualThreadPerTaskExecutor()` |
| 判断 | `Thread.currentThread().isVirtual()` |
| Spring Boot | `spring.threads.virtual.enabled=true` |
| 检测 pinning | `-Djdk.tracePinnedThreads=full` |
| 调度并行度 | `-Djdk.virtualThreadScheduler.parallelism=N` |
| 限流 | `Semaphore` |
| 结构化并发 | `StructuredTaskScope`（预览） |
