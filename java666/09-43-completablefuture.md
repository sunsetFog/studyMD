# 43 · CompletableFuture 异步编程

## 1. Future 的局限

```java
Future<Integer> f = executor.submit(() -> 42);
int v = f.get();          // 阻塞
f.get(1, SECONDS);        // 超时
f.cancel(true);
f.isDone();
```

问题：

- `get()` 阻塞，无法回调。
- 不能组合多个 Future。
- 不能链式处理结果。
- 不能手动完成。

## 2. 创建

```java
// 有返回值
CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(() -> 42);
CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(() -> 42, executor);

// 无返回值
CompletableFuture<Void> f3 = CompletableFuture.runAsync(() -> doWork());
CompletableFuture<Void> f4 = CompletableFuture.runAsync(() -> doWork(), executor);

// 已完成
CompletableFuture<String> done = CompletableFuture.completedFuture("ok");

// 手动完成
CompletableFuture<String> manual = new CompletableFuture<>();
manual.complete("result");             // 或 completeExceptionally(e)
```

> **必须传自定义 executor**。默认用 `ForkJoinPool.commonPool()`（并行度 = CPU-1），IO 密集任务会拖垮全局。

## 3. 链式转换

```java
CompletableFuture<String> f = CompletableFuture
    .supplyAsync(() -> userId, executor)                       // 异步取值
    .thenApply(id -> userService.get(id))                      // 同步转换（同线程）
    .thenApplyAsync(u -> u.getName(), executor)                // 异步转换
    .thenCombine(otherFuture, (a, b) -> a + b)                 // 合并两个
    .exceptionally(e -> "fallback");                           // 异常兜底

f.thenAccept(System.out::println);      // 消费结果
f.thenRun(() -> log.info("done"));      // 不关心结果
```

### 带 Async 后缀的差异

| 方法 | 执行线程 |
|---|---|
| `thenApply` | 上一个任务的线程（或调用线程，如果已完成） |
| `thenApplyAsync` | 默认 ForkJoinPool.commonPool |
| `thenApplyAsync(fn, executor)` | 指定线程池 |

**规则**：涉及阻塞操作（DB、HTTP）必须用 `Async` + 自定义池；纯 CPU 快速转换用非 Async。

## 4. 组合多个任务

```java
// 1. 两个都完成
CompletableFuture<String> a = supplyAsync(this::callA, pool);
CompletableFuture<String> b = supplyAsync(this::callB, pool);
CompletableFuture<String> combined = a.thenCombine(b, (ra, rb) -> ra + rb);

// 2. 等待全部（无返回值）
CompletableFuture<Void> all = CompletableFuture.allOf(a, b, c);
all.join();

// 3. 等待全部并收集结果
List<CompletableFuture<User>> futures = ids.stream()
        .map(id -> CompletableFuture.supplyAsync(() -> userService.get(id), pool))
        .toList();
List<User> users = futures.stream().map(CompletableFuture::join).toList();

// 或
CompletableFuture<List<User>> listFuture = CompletableFuture
        .allOf(futures.toArray(new CompletableFuture[0]))
        .thenApply(v -> futures.stream().map(CompletableFuture::join).toList());

// 4. 任意一个完成
CompletableFuture<Object> any = CompletableFuture.anyOf(a, b);

// 5. 依赖前一个的结果再发起异步任务
supplyAsync(() -> orderService.create(req), pool)
    .thenCompose(order -> supplyAsync(() -> payService.pay(order), pool))
    .thenAccept(res -> log.info("paid {}", res));
```

`thenApply` vs `thenCompose`：

- `thenApply`：`T -> U`（同步转换）。
- `thenCompose`：`T -> CompletableFuture<U>`（扁平化，避免嵌套 Future）。

```java
// 错误：嵌套
CompletableFuture<CompletableFuture<Order>> nested =
    supplyAsync(() -> id).thenApply(id -> supplyAsync(() -> getOrder(id)));

// 正确
CompletableFuture<Order> flat =
    supplyAsync(() -> id).thenCompose(id -> supplyAsync(() -> getOrder(id)));
```

## 5. 异常处理

```java
// 1. exceptionally：只处理异常，返回同类型兜底值
future.exceptionally(e -> {
    log.error("failed", e);
    return defaultValue;
});

// 2. handle：无论成功失败都执行，可转换类型
future.handle((res, ex) -> {
    if (ex != null) return "error: " + ex.getMessage();
    return "ok: " + res;
});

// 3. whenComplete：不改变结果，只做副作用（日志、清理）
future.whenComplete((res, ex) -> {
    if (ex != null) log.error("failed", ex);
    else log.info("result={}", res);
});

// 4. 组合使用
future
    .thenApply(this::transform)
    .exceptionally(e -> fallback())
    .whenComplete((r, e) -> metrics.record(r, e));
```

异常传播规则：

- 链上任何一环抛异常，后续 `thenApply` 等**跳过**，直接进入 `exceptionally`/`handle`。
- `CompletionException` 是包装异常，`getCause()` 才是真实原因。

```java
try {
    future.join();
} catch (CompletionException e) {
    Throwable real = e.getCause();      // 真实业务异常
}
```

> `join()` 抛 `CompletionException`（非受检），`get()` 抛 `ExecutionException`（受检）。

## 6. 超时（Java 9+）

```java
// 1. orTimeout：超时抛 TimeoutException
future.orTimeout(3, TimeUnit.SECONDS);

// 2. completeOnTimeout：超时返回默认值
future.completeOnTimeout("default", 3, TimeUnit.SECONDS);

// 3. 完整示例
String result = supplyAsync(() -> remoteCall(), pool)
        .orTimeout(2, TimeUnit.SECONDS)
        .exceptionally(e -> "timeout-fallback")
        .join();
```

Java 8 需要手写：

```java
public static <T> CompletableFuture<T> withTimeout(
        CompletableFuture<T> f, long timeout, TimeUnit unit, ScheduledExecutorService scheduler) {
    CompletableFuture<T> result = new CompletableFuture<>();
    scheduler.schedule(() -> result.completeExceptionally(new TimeoutException()), timeout, unit);
    f.whenComplete((v, e) -> {
        if (e != null) result.completeExceptionally(e);
        else result.complete(v);
    });
    return result;
}
```

## 7. 实战：并行调用多个服务

```java
public OrderDetail getOrderDetail(Long orderId) {
    CompletableFuture<Order> orderF = supplyAsync(() -> orderService.get(orderId), pool);
    CompletableFuture<User> userF = supplyAsync(() -> userService.get(orderId), pool);
    CompletableFuture<List<Item>> itemsF = supplyAsync(() -> itemService.list(orderId), pool);

    return CompletableFuture
            .allOf(orderF, userF, itemsF)
            .thenApply(v -> new OrderDetail(
                    orderF.join(), userF.join(), itemsF.join()))
            .orTimeout(3, TimeUnit.SECONDS)
            .exceptionally(e -> {
                log.error("获取订单详情失败 orderId={}", orderId, e);
                throw new BizException("查询失败");
            })
            .join();
}
```

> 注意：`allOf` 后仍用 `join()` 取各结果——因为 `allOf` 只保证全部完成，不收集结果。

## 8. 批量并行（限流）

```java
public List<User> batchGet(List<Long> ids, int concurrency) throws Exception {
    Semaphore sem = new Semaphore(concurrency);
    List<CompletableFuture<User>> futures = ids.stream()
            .map(id -> supplyAsync(() -> {
                sem.acquire();
                try { return userService.get(id); }
                finally { sem.release(); }
            }, pool))
            .toList();
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    return futures.stream().map(CompletableFuture::join).toList();
}
```

更优雅：用 `ThreadPoolExecutor` 的固定线程数天然限流。

## 9. 与其他框架集成

```java
// Spring 的 @Async 返回 CompletableFuture
@Async("taskExecutor")
public CompletableFuture<Result> asyncCall() {
    return CompletableFuture.completedFuture(doCall());
}

// WebFlux / Reactor 互转
Mono.fromFuture(completableFuture);
completableFuture.toCompletableFuture();   // Reactor 的 Mono.toFuture()

// RPC 框架（Dubbo 3）支持返回 CompletableFuture
```

## 10. 性能与设计建议

1. **始终传自定义 Executor**（隔离 + 防止 commonPool 饥饿）。
2. IO 任务用大池，CPU 任务用小池。
3. 链式调用保持在同一个池（避免线程跳跃开销）。
4. 避免在链中做阻塞操作（`.get()`、`.join()`）——会退化为同步。
5. 超时必设（`orTimeout`）。
6. 异常必须有兜底（`exceptionally`）。
7. 不要创建过长的链（可读性差，改用同步代码 + 少量并行）。
8. `thenApply` 内不要做耗时操作（占住线程）。
9. 谨慎使用 `anyOf`（其他任务不会被取消，继续占用资源）。
10. 在 Web 请求线程中 `join()` 会阻塞容器线程（高并发下需 async servlet 或 WebFlux）。

## 11. 常见坑

1. 不传 Executor（commonPool 被 IO 任务占满，影响并行流）。
2. `thenApply` 中做阻塞调用。
3. 忘记 `join/get` 导致任务未执行完就返回（异步方法直接结束）。
4. 异常被静默吞掉（没有 `exceptionally`/`whenComplete`）。
5. `allOf` 后直接取值（必须 `join` 各个 future）。
6. 使用 `anyOf` 后其他任务泄漏。
7. 在链中 `join()` 自己的 future（死锁）。
8. 超时后未取消底层任务（资源浪费，需配合 `cancel`）。
9. 把 `CompletableFuture` 作为字段共享（多次 `complete` 只有第一次生效）。
10. 线程池队列满导致 `RejectedExecutionException` 未处理。

## 12. API 速查

| 分类 | 方法 |
|---|---|
| 创建 | `supplyAsync` `runAsync` `completedFuture` `new CompletableFuture` |
| 转换 | `thenApply` `thenApplyAsync` |
| 扁平化 | `thenCompose` |
| 消费 | `thenAccept` `thenRun` |
| 组合 | `thenCombine` `thenAcceptBoth` `runAfterBoth` |
| 择一 | `applyToEither` `acceptEither` `runAfterEither` `anyOf` |
| 全部 | `allOf` |
| 异常 | `exceptionally` `handle` `whenComplete` |
| 超时 | `orTimeout` `completeOnTimeout` |
| 取值 | `join`（非受检）`get`（受检）`getNow` |
| 状态 | `isDone` `isCompletedExceptionally` `isCancelled` |
| 完成 | `complete` `completeExceptionally` `cancel` |
