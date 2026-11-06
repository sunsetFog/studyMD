# 40 · Lock 家族与 AQS

## 1. 为什么需要 Lock

`synchronized` 的局限：

- 不能尝试非阻塞获取（`tryLock`）。
- 不能带超时。
- 不能响应中断（阻塞在锁上时）。
- 不支持公平锁。
- 不能按条件分离等待队列（只有一个 wait set）。

## 2. Lock 接口

```java
public interface Lock {
    void lock();                                     // 阻塞获取
    void lockInterruptibly() throws InterruptedException;  // 可中断
    boolean tryLock();                               // 立即尝试
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```

**铁律**：`unlock()` 必须放在 `finally`。

```java
lock.lock();
try {
    // 临界区
} finally {
    lock.unlock();
}
```

## 3. ReentrantLock

```java
private final ReentrantLock lock = new ReentrantLock();          // 非公平（默认）
private final ReentrantLock fairLock = new ReentrantLock(true);  // 公平

lock.lock();
try {
    // 可重入：同一线程可再次 lock()
} finally {
    lock.unlock();     // 每次 lock 必须对应一次 unlock
}
```

### 公平 vs 非公平

| 类型 | 行为 | 吞吐 | 饥饿 |
|---|---|---|---|
| 公平 | 按等待顺序获取 | 低（频繁上下文切换） | 无 |
| 非公平 | 允许插队 | 高 | 理论上可能 |

默认非公平，因为「刚释放锁的线程大概率还能拿到」→ 减少唤醒开销。

### 可中断与超时

```java
try {
    if (lock.tryLock(3, TimeUnit.SECONDS)) {
        try { /* ... */ } finally { lock.unlock(); }
    } else {
        // 获取超时，降级处理
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### 与 synchronized 对比

| 维度 | synchronized | ReentrantLock |
|---|---|---|
| 实现 | JVM 内置（monitor） | AQS（Java 层） |
| 释放 | 自动 | 手动（必须 finally） |
| 公平 | 非公平 | 可选 |
| 超时 | ✘ | ✔ |
| 可中断 | ✘ | ✔ |
| 条件变量 | 1 个 | 多个 Condition |
| 性能 | JDK 6 后优化良好 | 相近，高竞争略优 |

**结论**：优先 `synchronized`（简单、不易错），需要高级功能时用 `ReentrantLock`。

## 4. AQS（AbstractQueuedSynchronizer）

### 核心结构

```java
public abstract class AbstractQueuedSynchronizer {
    private volatile int state;               // 同步状态
    private transient volatile Node head;     // 等待队列头（哨兵）
    private transient volatile Node tail;     // 尾

    static final class Node {
        volatile int waitStatus;              // CANCELLED/SIGNAL/CONDITION/PROPAGATE
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
        Node nextWaiter;                      // Condition 队列 / 共享模式标记
    }
}
```

- `state` 表示锁状态：0 = 未持有，>0 = 重入次数。
- 未获取到锁的线程包装成 Node 加入 **CLH 双向队列**（FIFO）。
- 通过 `LockSupport.park/unpark` 阻塞/唤醒。

### 模板方法

```java
// 子类实现
protected boolean tryAcquire(int arg);        // 独占获取
protected boolean tryRelease(int arg);        // 独占释放
protected int tryAcquireShared(int arg);      // 共享获取（返回负数失败）
protected boolean tryReleaseShared(int arg);  // 共享释放
protected boolean isHeldExclusively();

// AQS 提供
public final void acquire(int arg);           // 失败则入队阻塞
public final void release(int arg);
public final void acquireShared(int arg);
public final void releaseShared(int arg);
```

### acquire 流程

```
acquire(arg)
  ├── tryAcquire(arg) 成功 → 返回
  └── 失败 → addWaiter(EXCLUSIVE) 入队
             └── acquireQueued(node)
                   ├── 前驱是 head → tryAcquire 再试
                   ├── 失败 → shouldParkAfterFailedAcquire → park()
                   └── 被唤醒后继续循环
```

### 自定义同步器

```java
public class Mutex implements Lock {
    private final Sync sync = new Sync();

    private static class Sync extends AbstractQueuedSynchronizer {
        @Override protected boolean tryAcquire(int arg) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        @Override protected boolean tryRelease(int arg) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        @Override protected boolean isHeldExclusively() {
            return getState() == 1;
        }
        Condition newCondition() { return new ConditionObject(); }
    }

    @Override public void lock() { sync.acquire(1); }
    @Override public void unlock() { sync.release(1); }
    @Override public boolean tryLock() { return sync.tryAcquire(1); }
    @Override public boolean tryLock(long t, TimeUnit u) throws InterruptedException {
        return sync.tryAcquireNanos(1, u.toNanos(t));
    }
    @Override public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    @Override public Condition newCondition() { return sync.newCondition(); }
}
```

## 5. Condition

```java
private final ReentrantLock lock = new ReentrantLock();
private final Condition notEmpty = lock.newCondition();
private final Condition notFull = lock.newCondition();
private final Deque<Task> queue = new ArrayDeque<>();
private final int capacity = 100;

public void put(Task t) throws InterruptedException {
    lock.lock();
    try {
        while (queue.size() == capacity) notFull.await();
        queue.addLast(t);
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
}

public Task take() throws InterruptedException {
    lock.lock();
    try {
        while (queue.isEmpty()) notEmpty.await();
        Task t = queue.pollFirst();
        notFull.signal();
        return t;
    } finally {
        lock.unlock();
    }
}
```

| 对比 | Object 监视器 | Condition |
|---|---|---|
| 等待 | `wait()` | `await()` |
| 唤醒一个 | `notify()` | `signal()` |
| 唤醒全部 | `notifyAll()` | `signalAll()` |
| 条件数量 | 1 | 多个（精确唤醒） |
| 超时 | `wait(ms)` | `await(time, unit)` |
| 中断响应 | 抛 InterruptedException | 同 |

**优势**：多个 Condition 可精确唤醒（生产者只唤醒消费者，反之亦然），避免无效唤醒。

## 6. ReentrantReadWriteLock

```java
ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
Lock read = rw.readLock();
Lock write = rw.writeLock();

// 读读并发
read.lock();
try { /* 读 */ } finally { read.unlock(); }

// 写写互斥、读写互斥
write.lock();
try { /* 写 */ } finally { write.unlock(); }
```

实现：AQS 的 `state` 高 16 位记录读锁数，低 16 位记录写锁重入次数。

特性：

- 读锁共享，写锁独占。
- 支持锁降级（写锁 → 读锁，反之不行）。
- 公平模式下，等待时间长的写线程会阻塞后续读（防写饥饿）。

```java
// 锁降级示例
write.lock();
try {
    data = newData;
    read.lock();          // 在持有写锁时获取读锁
} finally {
    write.unlock();
}
try {
    use(data);            // 此时保证数据未被修改
} finally {
    read.unlock();
}
```

适用：**读多写少**（缓存、配置）。写多时性能不如 `ReentrantLock`。

## 7. StampedLock（Java 8+）

```java
StampedLock lock = new StampedLock();

// 1. 乐观读（无锁）
long stamp = lock.tryOptimisticRead();
int cur = data;                       // 读数据
if (!lock.validate(stamp)) {          // 读期间有写 → 重试或升级
    stamp = lock.readLock();
    try { cur = data; } finally { lock.unlockRead(stamp); }
}

// 2. 悲观读
long rs = lock.readLock();
try { ... } finally { lock.unlockRead(rs); }

// 3. 写
long ws = lock.writeLock();
try { ... } finally { lock.unlockWrite(ws); }
```

特点：

- 乐观读完全无锁（只有版本号校验），读性能极高。
- **不可重入**（同一线程重复加锁会死锁）。
- 不支持 Condition。
- 适合读远多于写的场景（如坐标、配置快照）。

```java
// 典型用法：Point 类
class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();

    void move(double dx, double dy) {
        long s = lock.writeLock();
        try { x += dx; y += dy; } finally { lock.unlockWrite(s); }
    }

    double distanceFromOrigin() {
        long s = lock.tryOptimisticRead();
        double cx = x, cy = y;
        if (!lock.validate(s)) {
            s = lock.readLock();
            try { cx = x; cy = y; } finally { lock.unlockRead(s); }
        }
        return Math.hypot(cx, cy);
    }
}
```

## 8. LockSupport

```java
LockSupport.park();                    // 阻塞当前线程
LockSupport.unpark(thread);            // 唤醒指定线程
LockSupport.parkNanos(1_000_000);
LockSupport.parkUntil(deadline);
LockSupport.getBlocker(thread);        // 诊断用
```

与 `wait` 的区别：

- 不需要持有锁。
- `unpark` 可以在 `park` 之前调用（发放许可，不丢失）。
- 响应中断但不抛异常（需自行检查 `isInterrupted`）。

AQS、`CountDownLatch`、`ThreadPoolExecutor` 都基于它。

## 9. 各种锁概念辨析

| 概念 | 含义 |
|---|---|
| 公平锁 / 非公平锁 | 是否按等待顺序获取 |
| 可重入锁 | 同一线程可重复获取 |
| 独占锁 / 共享锁 | 是否允许多个持有者 |
| 乐观锁 / 悲观锁 | CAS 重试 vs 加锁阻塞 |
| 自旋锁 | 循环 CAS，不阻塞 |
| 偏向锁 | 同一线程反复进入（已废弃） |
| 分段锁 | 按 key 分桶加锁（JDK 7 CHM） |
| 读写锁 | 读共享写独占 |
| 互斥锁 | 同一时刻只一个持有者 |

## 10. 常见坑

1. `unlock()` 未放在 finally（异常导致死锁）。
2. `lock()` 次数与 `unlock()` 不一致（AQS state 不为 0，其他线程永远阻塞）。
3. 用 `tryLock` 成功后忘记 unlock。
4. `Condition.await()` 未在 `while` 中（虚假唤醒）。
5. `await()` 前未持有锁（`IllegalMonitorStateException`）。
6. 读写锁中读锁升级为写锁（死锁）。
7. StampedLock 重入（死锁）。
8. 锁对象使用可变引用（`Integer`、`String` 常量池）。
9. 多锁顺序不一致（死锁）。
10. 认为 `ReentrantLock` 一定比 `synchronized` 快（低竞争时相反）。

## 11. 死锁排查

```java
// 代码中主动检测
ThreadMXBean mx = ManagementFactory.getThreadMXBean();
long[] deadlocked = mx.findDeadlockedThreads();
if (deadlocked != null) {
    for (ThreadInfo info : mx.getThreadInfo(deadlocked)) {
        System.out.println(info);
    }
}
```

```bash
jstack <pid> | grep -A 20 "Found one Java-level deadlock"
```

预防：统一加锁顺序、使用 `tryLock` 超时、减少锁嵌套。

## 12. 速查

| 需求 | 选择 |
|---|---|
| 简单互斥 | `synchronized` |
| 需要超时/中断/公平 | `ReentrantLock` |
| 多条件等待 | `ReentrantLock` + 多 `Condition` |
| 读多写少 | `ReentrantReadWriteLock` |
| 读极多写极少 | `StampedLock`（注意不可重入） |
| 生产者消费者 | `BlockingQueue` |
| 计数/一次性 | `CountDownLatch` |
| 循环屏障 | `CyclicBarrier` |
| 限流 | `Semaphore` |
