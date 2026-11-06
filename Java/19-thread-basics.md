# 19 · 线程基础

## 一、进程与线程

- **进程**：资源分配的最小单位（独立内存空间）
- **线程**：CPU 调度的最小单位（共享进程内存：堆/方法区，私有：栈/程序计数器）
- 并发：单核时间片轮换「宏观并行」；并行：多核同时执行

## 二、创建线程的方式

### 1. Runnable + Lambda（推荐）

```java
Thread t = new Thread(() -> {
    System.out.println("running in " + Thread.currentThread().getName());
}, "worker-1");
t.start();      // start() 才是新线程；直接 run() 是普通方法调用！
```

### 2. Callable + FutureTask（有返回值/可抛异常）

```java
Callable<Integer> task = () -> {
    Thread.sleep(100);
    return 42;
};
FutureTask<Integer> ft = new FutureTask<>(task);
new Thread(ft, "calc").start();
Integer result = ft.get();      // 阻塞取结果（get 不设超时会永久阻塞！）
```

### 3. 线程池（生产唯一正确方式）

```java
// 直接 new Thread 每请求一线程 = 资源灾难
// 统一走线程池（23 篇）
executor.submit(() -> doWork());
```

> 实现 Runnable / 继承 Thread 的区别：Runnable 解耦任务与执行载体，还能配合池；继承 Thread 无额外价值。**永远 Runnable**。

## 三、线程生命周期

```
NEW → start() → RUNNABLE ⇄ BLOCKED/WAITING/TIMED_WAITING → TERMINATED
```

| 状态 | 触发 |
|---|---|
| NEW | new 之后未 start |
| RUNNABLE | 可运行（等待 CPU 或运行中） |
| BLOCKED | 等锁（synchronized 未抢到） |
| WAITING | wait()/join() 无参、LockSupport.park |
| TIMED_WAITING | sleep(ms)、wait(ms)、join(ms) |
| TERMINATED | run 结束 |

线程终止后不可重启（IllegalThreadStateException）。

## 四、常用方法

```java
Thread.currentThread();            // 当前线程
t.setName("biz-pool");             // 命名（排查必备！）
t.setDaemon(true);                 // 守护线程：JVM 退出不等它（GC 线程即守护）
t.join();                          // 等待 t 执行完（本质是本线程 wait）
Thread.sleep(1000);                // 静态：当前线程暂停（不释放锁）
t.interrupt();                     // 中断：协作式信号，不是强杀！
Thread.interrupted();              // 检查并清除中断标志
t.isAlive(); t.getState();
Thread.yield();                    // 让出 CPU（提示性，不可依赖）
```

### 正确响应中断（模板）

```java
// 中断只是"请求停止"标志，线程自己决定如何响应
Runnable task = () -> {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            Thread.sleep(1000);
            doWork();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();   // 捕获后恢复中断标志（sleep 会清除）
            break;                                 // 然后退出
        }
    }
};
```

**铁律**：捕获 InterruptedException 后要么重抛要么恢复中断标志，**绝不静默吞掉**——否则取消机制失效。

## 五、线程安全问题初识

```java
// 经典竞态
class Counter {
    private int count = 0;
    void increment() { count++; }    // count++ 是「读-改-写」三步，非原子！
    int get() { return count; }
}
// 两线程并发 increment → 丢失更新
```

问题根源：可见性、原子性、有序性（21 篇 JMM 统一解决）。

## 六、守护线程与线程组

```java
// 守护线程：为其他线程服务（如心跳/清理）。注意：主线程结束，守护线程立即死，事务/IO 可能中断
Thread monitor = new Thread(() -> { while (true) { } });
monitor.setDaemon(true);            // 必须在 start() 前设置
monitor.start();
```

## 七、线程未捕获异常

```java
// 线程里的异常不会传播给调用方 → 静默丢失
Thread t = new Thread(() -> { throw new RuntimeException("boom"); });
t.setUncaughtExceptionHandler((thread, e) ->
        log.error("线程 {} 异常", thread.getName(), e));    // 兜底记录
```

线程池场景对应 AsyncUncaughtExceptionHandler / Future.get 捕获（23 篇）。

## 常见坑

- **坑 1**：`new Thread(r).run()` → 单线程执行；必须 start()
- **坑 2**：吞 InterruptedException → 取消机制失效
- **坑 3**：不设置线程名 → jstack 一堆 Thread-1 无从排查
- **坑 4**：Future.get() 无超时 → 永久阻塞
- **坑 5**：daemon 线程写文件/事务 → JVM 退出被硬断
- **坑 6**：以为 interrupt 能强停线程 → 协作式，需线程自己配合

## Checklist

- [ ] 任务用 Runnable/Callable，执行用线程池
- [ ] 线程必命名
- [ ] 中断处理模板记牢（恢复标志 + 退出）
- [ ] Future.get 带 timeout
