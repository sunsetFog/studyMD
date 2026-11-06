# 22 · 并发工具：CountDownLatch、Semaphore、AQS

## 一、CountDownLatch：倒计时门闩（等待 N 件事完成）

```java
CountDownLatch latch = new CountDownLatch(3);     // 计数 3

// 工作线程
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        doTask();
        latch.countDown();        // 计数减 1（不可重置）
    }).start();
}

// 主线程
latch.await();                     // 阻塞到计数归零（带超时版本：await(5, TimeUnit.SECONDS)）
runAfterAllDone();

// 典型场景：并行任务汇聚、启动依赖（等 N 个服务 ready）
```

## 二、CyclicBarrier：循环栅栏（N 方互相等待）

```java
CyclicBarrier barrier = new CyclicBarrier(4, () ->
        System.out.println("全员就位，进入下一阶段"));   // 可选的栅栏动作

// 每个线程
barrier.await();                   // 到齐才放行，且**可复用**（区分 Latch）

// 场景：分片计算到齐后合并、多阶段并行仿真
```

| | CountDownLatch | CyclicBarrier |
|---|---|---|
| 语义 | 等事件发生 N 次 | N 个线程互相等 |
| 复用 | 否（一次性） | 是 |
| 计数操作 | 由事件方 countDown | 参与者 await |

## 三、Semaphore：信号量（限流/资源池）

```java
Semaphore permits = new Semaphore(10);        // 10 个许可

public void handle() {
    if (!permits.tryAcquire()) {              // 非阻塞获取
        throw new BizException("并发超限");
    }
    try {
        // 受限并发段（如调用限速的第三方 API）
    } finally {
        permits.release();
    }
}

// 公平信号量：new Semaphore(10, true) 防插队
```

## 四、Exchanger 与 Phaser（了解）

```java
// Exchanger：两线程间交换数据
Exchanger<String> ex = new Exchanger<>();
// thread A: String b = ex.exchange("A-data");
// thread B: String a = ex.exchange("B-data");

// Phaser：多阶段栅栏，替代 CyclicBarrier 的多阶段版本（更灵活，少用）
```

## 五、AQS：并发工具的地基（原理核心）

**AbstractQueuedSynchronizer**：ReentrantLock/Semaphore/CountDownLatch/线程池同步器的共同底层。

```
核心 = volatile int state（同步状态） + CLH 变体 FIFO 等待队列 + CAS

- 获取：CAS 修改 state，失败入队挂起（park）
- 释放：修改 state，唤醒队头后继（unpark）
- 独占模式（ReentrantLock）/ 共享模式（Semaphore/Latch）两种模板
```

### 自定义 AQS 组件示例

```java
class SimpleLock extends AbstractQueuedSynchronizer {
    @Override
    protected boolean tryAcquire(int arg) {
        return compareAndSetState(0, 1);          // state 0=未锁 1=已锁
    }
    @Override
    protected boolean tryRelease(int arg) {
        setState(0);
        return true;
    }
}
// 只需实现 tryAcquire/tryRelease，排队/挂起/唤醒全由 AQS 托管
```

**面试话术**：AQS 把「同步状态管理 + 线程排队 + 阻塞唤醒」封装为模板，子类只实现状态获取/释放的语义（state 的含义由子类定义：ReentrantLock=重入次数、Semaphore=剩余许可、Latch=剩余计数）。

## 六、CompletableFuture：异步编排（重点）

```java
// 创建
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> fetchUser(id), executor);
CompletableFuture<Void> f2 = CompletableFuture.runAsync(() -> log.info("done"), executor);

// 转换与消费
f1.thenApply(user -> user.getName())              // 同步转换
  .thenApplyAsync(name -> enrich(name), executor) // 指定池异步
  .thenAccept(name -> save(name))
  .thenRun(() -> cleanup())
  .exceptionally(ex -> { log.error("", ex); return "fallback"; })   // 兜底
  .handle((r, ex) -> ex == null ? r : "default")  // 成功失败都过
  .whenComplete((r, ex) -> audit(r, ex));

// 组合
CompletableFuture<Profile> p = f1.thenCompose(user -> fetchProfile(user));   // 串行依赖
CompletableFuture<Result> r = f1.thenCombine(f2, (user, order) -> merge(user, order));  // 两路合并
f1.thenAcceptBoth(f2, (a, b) -> ...);
f1.applyToEither(f2, first -> first);            // 谁快用谁（竞速）

// 汇总多个
CompletableFuture.allOf(f1, f2, f3).join();      // 等全部
CompletableFuture.anyOf(f1, f2).join();          // 任一完成

// 结果收集惯用法
List<CompletableFuture<User>> futures = ids.stream()
        .map(id -> CompletableFuture.supplyAsync(() -> fetch(id), executor))
        .toList();
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
List<User> users = futures.stream().map(CompletableFuture::join).toList();
```

**铁律**：
- 永远显式传 executor（默认 ForkJoinPool.commonPool 全局共享）
- join/get 必带超时意识（get(t, unit)）；join 抛非受检 CompletionException
- 链中异常会被吞进 CompletionException → 用 exceptionally/handle 兜底

## 七、选型表

| 需求 | 工具 |
|---|---|
| 等 N 个任务完成 | CountDownLatch |
| N 方互相等、多阶段 | CyclicBarrier / Phaser |
| 限并发数/资源池 | Semaphore |
| 两线程交换 | Exchanger |
| 异步链式编排/并行聚合 | CompletableFuture |
| 生产者消费者 | BlockingQueue（24 篇） |
| 倒计时后一次性动作 | Latch + Runnable |

## 常见坑

- **坑 1**：latch.countDown() 不在 finally → 异常时主线程永久等待
- **坑 2**：CompletableFuture 全用默认池 → commonPool 被阻塞任务拖垮
- **坑 3**：忘记 join/get → 异步根本没执行完主流程就继续
- **坑 4**：Semaphore release 多次 → 许可泄漏并发爆炸
- **坑 5**：allOf 后直接拿到的是 List<CompletableFuture> 忘了逐个 join → 半成品数据

## Checklist

- [ ] Latch/Barrier/Semaphore 三件套能按场景选择
- [ ] CompletableFuture 显式传池 + 超时 + 异常兜底
- [ ] 能用 AQS 话术解释 Lock/Semaphore 底层
- [ ] 异步任务异常全链路可见（不静默）
