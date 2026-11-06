# 44 · 同步工具类

## 1. CountDownLatch（一次性倒计数）

```java
CountDownLatch latch = new CountDownLatch(3);

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try { doWork(); }
        finally { latch.countDown(); }     // 必须在 finally
    }).start();
}

latch.await();                             // 阻塞直到计数为 0
latch.await(10, TimeUnit.SECONDS);         // 带超时
System.out.println("all done");
```

特性：

- **一次性**：计数到 0 后不能重置。
- 计数不可增加（`countDown` 减到 0 后无效）。
- `await` 可被中断。

### 典型场景

```java
// 1. 主线程等待多个初始化完成
CountDownLatch initLatch = new CountDownLatch(3);
startCacheLoader(initLatch);
startDbPool(initLatch);
startMqConsumer(initLatch);
initLatch.await(30, SECONDS);

// 2. 模拟并发压测（同时起跑）
CountDownLatch start = new CountDownLatch(1);
for (int i = 0; i < 100; i++) {
    new Thread(() -> {
        start.await();       // 所有线程在此等待
        doRequest();
    }).start();
}
start.countDown();           // 发令枪

// 3. 任务拆分并行执行后汇总
```

## 2. CyclicBarrier（可循环屏障）

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("全部到达，进入下一轮"));

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        for (int round = 0; round < 3; round++) {
            doPhase(round);
            barrier.await();          // 等待其他 2 个线程
        }
    }).start();
}
```

特性：

- **可重用**：所有线程到达后自动重置，进入下一轮。
- 可指定 `Runnable`（所有线程到达后、释放前执行，由最后一个到达的线程执行）。
- 一个线程中断/超时 → 屏障 broken，其他线程抛 `BrokenBarrierException`。

```java
barrier.await(5, TimeUnit.SECONDS);
barrier.reset();              // 手动重置（会导致等待线程抛异常）
barrier.getNumberWaiting();
barrier.isBroken();
```

### CountDownLatch vs CyclicBarrier

| 维度 | CountDownLatch | CyclicBarrier |
|---|---|---|
| 复用 | ✘ 一次性 | ✔ 可循环 |
| 角色 | 一个/多个线程等 N 个事件 | N 个线程互相等待 |
| 计数 | `countDown` 递减 | `await` 递增到 parties |
| 回调 | 无 | 有（barrierAction） |
| 场景 | 主等子、并发起跑 | 分阶段并行计算 |

### 应用：分阶段并行计算

```java
// 阶段 1：并行计算，阶段 2：汇总后继续
CyclicBarrier barrier = new CyclicBarrier(4);
double[][] matrix = new double[4][4];

for (int i = 0; i < 4; i++) {
    final int row = i;
    new Thread(() -> {
        computeRow(matrix, row);
        barrier.await();           // 等所有行算完
        // 下一阶段可以安全读取整个矩阵
    }).start();
}
```

## 3. Semaphore（信号量，限流）

```java
Semaphore sem = new Semaphore(5);              // 最多 5 个并发
Semaphore fair = new Semaphore(5, true);       // 公平模式

sem.acquire();                                  // 获取许可，无可用则阻塞
try {
    doWork();
} finally {
    sem.release();                              // 必须释放
}

// 带超时
if (sem.tryAcquire(1, TimeUnit.SECONDS)) {
    try { doWork(); } finally { sem.release(); }
}

// 一次获取多个
sem.acquire(3);
sem.release(3);

// 立即尝试
if (sem.tryAcquire()) { ... }

// 查询
sem.availablePermits();
sem.getQueueLength();
```

### 应用

```java
// 1. 接口限流（限制并发数）
private final Semaphore limit = new Semaphore(100);

public Result handle(Request req) {
    if (!limit.tryAcquire()) {
        return Result.fail("系统繁忙");
    }
    try { return doHandle(req); }
    finally { limit.release(); }
}

// 2. 数据库连接限流
private final Semaphore dbLimit = new Semaphore(20);

// 3. 资源池（配合 BlockingQueue 更好）
// 4. 控制下游 QPS（令牌桶更适合用 Guava RateLimiter）
```

> 注意：`Semaphore` 控制**并发数**，不控制**速率**。限 QPS 用 `RateLimiter`（Guava）或 Sentinel。

## 4. Phaser（可分阶段、可动态注册）

```java
Phaser phaser = new Phaser(1);      // 注册主线程

for (int i = 0; i < 3; i++) {
    phaser.register();              // 动态注册
    new Thread(() -> {
        doPhase1();
        phaser.arriveAndAwaitAdvance();   // 阶段 1 屏障

        doPhase2();
        phaser.arriveAndAwaitAdvance();   // 阶段 2 屏障

        phaser.arriveAndDeregister();     // 完成后注销
    }).start();
}

phaser.arriveAndDeregister();       // 主线程注销
```

特性：

- 可动态增减参与者（`register`/`arriveAndDeregister`）。
- 支持多阶段（每次 `arriveAndAwaitAdvance` 进入下一 phase）。
- 可重写 `onAdvance` 自定义阶段切换逻辑（甚至终止）。

```java
Phaser phaser = new Phaser(3) {
    @Override protected boolean onAdvance(int phase, int registeredParties) {
        System.out.println("阶段 " + phase + " 完成，剩余 " + registeredParties);
        return registeredParties == 0 || phase >= 2;   // 返回 true 终止
    }
};
```

### 与其他工具对比

| 工具 | 动态参与者 | 多阶段 | 复用 |
|---|---|---|---|
| CountDownLatch | ✘ | ✘ | ✘ |
| CyclicBarrier | ✘ | ✔（循环） | ✔ |
| Phaser | ✔ | ✔ | ✔ |

## 5. Exchanger（两线程交换数据）

```java
Exchanger<List<Integer>> exchanger = new Exchanger<>();

// 生产者
new Thread(() -> {
    List<Integer> buf = new ArrayList<>();
    while (true) {
        buf = exchanger.exchange(buf);     // 交出满的，拿回空的
    }
}).start();

// 消费者
new Thread(() -> {
    List<Integer> buf = new ArrayList<>();
    while (true) {
        buf = exchanger.exchange(buf);     // 交出空的，拿到满的
        process(buf);
        buf.clear();
    }
}).start();
```

- 只支持**两个**线程配对交换。
- 一方到达会阻塞等待另一方。
- 可超时：`exchange(x, 1, SECONDS)`。

应用：双缓冲（流水线处理）、遗传算法配对。

## 6. 工具类对比总表

| 工具 | 作用 | 计数方向 | 复用 | 参与者 |
|---|---|---|---|---|
| `CountDownLatch` | 等待 N 个事件 | 递减到 0 | ✘ | 固定 |
| `CyclicBarrier` | N 个线程互相等待 | 递增到 N | ✔ | 固定 |
| `Phaser` | 多阶段屏障 | 动态 | ✔ | 动态 |
| `Semaphore` | 控制并发数 | 许可增减 | ✔ | — |
| `Exchanger` | 两线程交换数据 | — | ✔ | 2 |

## 7. 实战组合

### 并行任务 + 超时控制

```java
public List<Result> parallelWithTimeout(List<Task> tasks, long timeoutMs) throws Exception {
    ExecutorService pool = Executors.newFixedThreadPool(8);
    CountDownLatch latch = new CountDownLatch(tasks.size());
    List<Result> results = Collections.synchronizedList(new ArrayList<>());

    for (Task t : tasks) {
        pool.submit(() -> {
            try { results.add(t.run()); }
            catch (Exception e) { log.error("task failed", e); }
            finally { latch.countDown(); }
        });
    }

    if (!latch.await(timeoutMs, TimeUnit.MILLISECONDS)) {
        throw new TimeoutException("任务超时");
    }
    pool.shutdown();
    return results;
}
```

> 更好：直接用 `CompletableFuture.allOf(...).orTimeout(...)`。

### 限流 + 重试

```java
public <T> T callWithLimit(Semaphore sem, Supplier<T> action, int maxRetry) {
    for (int i = 0; i <= maxRetry; i++) {
        try {
            if (!sem.tryAcquire(500, TimeUnit.MILLISECONDS)) continue;
            try { return action.get(); }
            finally { sem.release(); }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException(e);
        } catch (Exception e) {
            if (i == maxRetry) throw e;
        }
    }
    throw new IllegalStateException("限流等待超时");
}
```

## 8. 实现原理简述

| 工具 | 底层 |
|---|---|
| `CountDownLatch` | AQS 共享模式，`state` 为计数 |
| `CyclicBarrier` | ReentrantLock + Condition（内部有 `generation` 表示轮次） |
| `Semaphore` | AQS 共享模式，`state` 为许可数 |
| `Phaser` | 自定义 state（phase + parties 位运算） |
| `Exchanger` | 自旋 + `LockSupport.park` |

## 9. 常见坑

1. `CountDownLatch` 的 `countDown` 未放 `finally`（异常导致永久阻塞）。
2. 认为 `CountDownLatch` 可以重置（不能，需新建）。
3. `CyclicBarrier` 中一个线程异常 → 其他线程抛 `BrokenBarrierException`，未处理导致混乱。
4. `Semaphore.release()` 次数多于 `acquire`（许可数增加，限流失效）。
5. `Semaphore` 忘记 `finally release`（许可泄漏，最终永久阻塞）。
6. 用 `Semaphore` 做 QPS 限流（实际是并发数限流）。
7. `Exchanger` 只有一个线程到达（永久阻塞，需超时）。
8. `Phaser` 的 `register` 与 `arriveAndDeregister` 不配对（永远不终止）。
9. 在 UI/请求线程中 `await()` 无超时（卡死）。
10. 这些工具都没有「取消」语义，超时后后台任务仍在跑。

## 10. 选型速查

| 需求 | 工具 |
|---|---|
| 主线程等 N 个任务完成 | `CountDownLatch` |
| N 个线程同步起跑 | `CountDownLatch(1)` |
| 多轮并行计算（每轮同步） | `CyclicBarrier` |
| 动态增减参与者的多阶段任务 | `Phaser` |
| 限制并发访问数 | `Semaphore` |
| 两个线程交换缓冲 | `Exchanger` |
| 一次性异步结果 | `CompletableFuture` |
| 生产者消费者 | `BlockingQueue` |
