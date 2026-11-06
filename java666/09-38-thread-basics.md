# 38 · 线程基础

## 1. 进程 vs 线程

| 维度 | 进程 | 线程 |
|---|---|---|
| 资源 | 独立地址空间 | 共享进程内存 |
| 创建开销 | 大 | 小 |
| 通信 | IPC（管道、socket、共享内存） | 共享变量 |
| 崩溃影响 | 隔离 | 影响整个进程 |

Java 线程与操作系统线程是 **1:1** 映射（Java 21 虚拟线程是 M:N）。

## 2. 创建线程的四种方式

```java
// 1. 继承 Thread（不推荐，占用继承名额）
class MyThread extends Thread {
    @Override public void run() { System.out.println("run"); }
}
new MyThread().start();

// 2. 实现 Runnable（推荐）
Runnable task = () -> System.out.println("run");
new Thread(task).start();

// 3. 实现 Callable + FutureTask（有返回值）
Callable<Integer> c = () -> 42;
FutureTask<Integer> ft = new FutureTask<>(c);
new Thread(ft).start();
int v = ft.get();

// 4. 线程池（生产环境唯一正确方式）
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.execute(task);
Future<Integer> f = pool.submit(c);
```

> **本质只有一种**：`Thread.start()` 创建 OS 线程并执行 `run()`。Runnable/Callable 只是任务抽象。

## 3. start vs run

```java
Thread t = new Thread(() -> System.out.println(Thread.currentThread().getName()));
t.run();      // 当前线程直接调用，与普通方法无异
t.start();    // 新线程执行，JVM 调用 run()
```

`start()` 只能调用一次，重复调用抛 `IllegalThreadStateException`。

## 4. 线程生命周期

```
NEW ──start()──→ RUNNABLE ──┬──→ BLOCKED（等 synchronized 锁）
                             ├──→ WAITING（wait/join/park 无超时）
                             ├──→ TIMED_WAITING（sleep/wait(n)/join(n)）
                             └──→ TERMINATED（run 结束或异常）
```

`getState()` 返回 `Thread.State` 枚举。

注意：Java 的 RUNNABLE 包含「正在运行」与「等待 CPU 调度」两种 OS 状态。

## 5. Thread 常用 API

```java
Thread t = new Thread(task, "worker-1");

t.setName("biz-worker");
t.getName();
t.getId();                       // 已废弃，用 threadId()（Java 19+）
t.getPriority();                 // 1-10，默认 5（不可靠，别依赖）
t.setDaemon(true);               // 守护线程
t.isDaemon();
t.isAlive();
t.getState();
t.interrupt();                   // 设置中断标志
t.isInterrupted();
Thread.interrupted();            // 静态：清除并返回当前标志（慎用）
t.join();                        // 等待 t 结束
t.join(1000);                    // 最多等 1s
Thread.sleep(1000);              // 睡眠，抛 InterruptedException
Thread.yield();                  // 让出 CPU（提示，不保证）
Thread.onSpinWait();             // 自旋提示（Java 9+）
Thread.currentThread();

Thread.getAllStackTraces();      // 所有线程栈
Thread.activeCount();
```

### 守护线程

```java
Thread t = new Thread(() -> { while (true) { } });
t.setDaemon(true);
t.start();
// main 结束后 JVM 退出，守护线程被强制终止
```

- 守护线程不阻止 JVM 退出。
- `finally` 不保证执行。
- 不能用于需要可靠清理的任务（文件写入、事务提交）。
- 必须在 `start()` 前设置，否则抛 `IllegalThreadStateException`。

## 6. 线程中断（协作式停止）

Java **没有**安全的强制停止方式（`stop()` 已废弃，会释放锁导致数据不一致）。

```java
// 方式 1：中断标志
Thread t = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        doWork();
    }
});
t.start();
t.interrupt();       // 设置中断标志

// 方式 2：阻塞方法响应中断
try {
    Thread.sleep(10000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // 恢复标志
    return;                                // 退出
}

// 方式 3：volatile 标志（适合自定义循环）
private volatile boolean running = true;
public void shutdown() { running = false; }
```

### 中断规则

| 场景 | 行为 |
|---|---|
| `sleep/wait/join` 时中断 | 抛 `InterruptedException`，**清除**中断标志 |
| 普通运行中中断 | 仅设置标志，代码需自行检查 |
| IO 阻塞（`InputStream.read`） | 不一定响应（NIO 的 `InterruptibleChannel` 会关闭通道） |
| `Lock.lockInterruptibly()` | 响应中断 |
| `Lock.lock()` | 不响应 |

**铁律**：捕获 `InterruptedException` 后要么继续抛，要么 `Thread.currentThread().interrupt()` 恢复标志，绝不静默吞掉。

## 7. 线程通信

### wait/notify（配合 synchronized）

```java
class Buffer {
    private final Queue<Integer> q = new LinkedList<>();
    private final int cap = 10;

    public synchronized void put(int v) throws InterruptedException {
        while (q.size() == cap) wait();      // 必须 while（防虚假唤醒）
        q.add(v);
        notifyAll();
    }

    public synchronized int take() throws InterruptedException {
        while (q.isEmpty()) wait();
        int v = q.poll();
        notifyAll();
        return v;
    }
}
```

要点：

- `wait/notify` 必须在 `synchronized` 块中，否则 `IllegalMonitorStateException`。
- `wait()` 释放锁并进入 WAITING；被唤醒后**重新竞争锁**。
- 用 `while` 而非 `if`（虚假唤醒 + 多消费者竞争）。
- 优先 `notifyAll()`（`notify` 可能唤醒错误的线程导致死锁）。

现代替代：`BlockingQueue`、`Condition`、`CompletableFuture`。

### join 与 CountDownLatch

```java
Thread a = new Thread(taskA);
Thread b = new Thread(taskB);
a.start(); b.start();
a.join(); b.join();       // 等两个都结束
System.out.println("all done");

// 更好：CountDownLatch
CountDownLatch latch = new CountDownLatch(2);
// 每个任务 finally { latch.countDown(); }
latch.await();
```

## 8. 线程优先级

```java
t.setPriority(Thread.MAX_PRIORITY);   // 10
```

- 平台相关，Linux 上基本无效。
- 不要用优先级控制执行顺序（不可靠）。
- 依赖优先级 = 依赖运气。

## 9. 线程栈与异常

```java
// 未捕获异常处理器
Thread.setDefaultUncaughtExceptionHandler((thread, e) -> {
    log.error("线程 {} 异常退出", thread.getName(), e);
});

// 单个线程
t.setUncaughtExceptionHandler((th, e) -> {
    System.err.println(th.getName() + " failed: " + e);
});
```

线程池中未捕获异常会导致**线程被回收并新建**，但任务静默丢失。务必在任务内部 try-catch 或使用 `Future.get()`。

## 10. 线程数量规划

| 任务类型 | 线程数 |
|---|---|
| CPU 密集 | `N + 1`（N = 核数） |
| IO 密集 | `N * (1 + 等待时间/计算时间)` |
| 混合 | 拆分后用不同线程池 |

```java
int cpu = Runtime.getRuntime().availableProcessors();
int ioThreads = cpu * 2;      // 经验值，需压测
```

线程过多的问题：上下文切换开销、内存占用（每线程默认 1MB 栈）、GC 压力。

## 11. 完整示例：可停止的工作线程

```java
public class Worker implements AutoCloseable {
    private final Thread thread;
    private final BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);
    private volatile boolean running = true;

    public Worker() {
        this.thread = new Thread(this::loop, "worker");
        this.thread.setDaemon(false);
        this.thread.start();
    }

    private void loop() {
        while (running && !Thread.currentThread().isInterrupted()) {
            try {
                Task task = queue.poll(500, TimeUnit.MILLISECONDS);
                if (task != null) task.run();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            } catch (Exception e) {
                log.error("task failed", e);       // 单任务异常不影响循环
            }
        }
    }

    public boolean submit(Task t) { return queue.offer(t); }

    @Override
    public void close() {
        running = false;
        thread.interrupt();
        try { thread.join(5000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

## 12. 常见坑

1. 调用 `run()` 而不是 `start()`（在调用线程执行）。
2. 重复 `start()`。
3. 用 `stop()/suspend()/resume()`（已废弃，不安全）。
4. 吞掉 `InterruptedException` 不恢复标志。
5. `wait` 用 `if` 不用 `while`。
6. `wait/notify` 未加 `synchronized`。
7. 依赖线程优先级。
8. 守护线程写文件（JVM 退出时数据丢失）。
9. 在 `finally` 中做耗时操作（中断后仍阻塞退出）。
10. 无限创建线程（OOM: unable to create new native thread）。

## 13. 速查

| 需求 | API |
|---|---|
| 创建线程 | `new Thread(runnable).start()` |
| 线程池 | `Executors` / `ThreadPoolExecutor` |
| 停止 | `interrupt()` + 标志检查 |
| 等待结束 | `join()` / `CountDownLatch` |
| 睡眠 | `Thread.sleep()` |
| 线程名 | `Thread.currentThread().getName()` |
| 异常处理 | `setUncaughtExceptionHandler` |
| 守护 | `setDaemon(true)`（start 前） |
