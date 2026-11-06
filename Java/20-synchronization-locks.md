# 20 · 锁体系：synchronized 与 Lock

## 一、synchronized：JVM 内置锁

### 三种用法

```java
// 1. 同步方法：锁 this（实例锁）
public synchronized void add() { count++; }

// 2. 静态同步方法：锁 Class 对象（类锁，全局唯一）
public static synchronized void staticAdd() { }

// 3. 同步代码块（推荐：缩小临界区）
private final Object lock = new Object();     // 专用锁对象
public void process() {
    // 无锁区……
    synchronized (lock) {
        count++;                              // 临界区
    }
    // 无锁区……
}
```

**要点**：
- 锁的对象是「同一把」才互斥：锁 this 的方法之间互斥，与锁 Class 的方法不互斥
- 锁对象用 private final 字段，不锁 String/int 常量池对象、不锁可变引用
- **可重入**：同线程可重复获取同一把锁

### 原理（面试高频）

- 对象头 Mark Word 记录锁状态；monitorenter/monitorexit 字节码
- **锁升级**（JDK 6+ 偏向锁→轻量级锁→重量级锁的自适应优化；Java 15 已废弃偏向锁）：
  - 无锁 → 偏向锁（单线程重复获取，仅记线程 ID）
  - → 轻量级锁（CAS 自旋，竞争少）
  - → 重量级锁（操作系统 mutex，阻塞挂起）
- synchronized 由 JVM 管理，无死锁检测但绝不会「忘记解锁」

## 二、wait / notify（synchronized 配套）

```java
synchronized (lock) {
    while (!condition) {              // 必须 while（防虚假唤醒）！
        lock.wait();                  // 释放锁并等待；被唤醒后重新抢锁
    }
    // 条件满足处理……
    lock.notifyAll();                 // 唤醒所有等待线程（notify 随机一个，易死锁）
}
```

生产代码几乎不用 wait/notify → 用并发工具（22 篇 BlockingQueue/CountDownLatch）。

## 三、Lock 接口（JUC）

```java
private final ReentrantLock lock = new ReentrantLock();

public void transfer() {
    lock.lock();                       // 必须紧跟 try！
    try {
        // 临界区
    } finally {
        lock.unlock();                 // finally 必解锁，否则死锁
    }
}
```

### synchronized vs ReentrantLock

| 维度 | synchronized | ReentrantLock |
|---|---|---|
| 获取失败 | 阻塞死等 | **可 tryLock 带超时/立即失败** |
| 公平锁 | 无 | 可选 fair=true |
| 可中断 | 否 | **lockInterruptibly()** |
| 条件队列 | 单一 wait/notify | 多个 Condition 精准唤醒 |
| 释放 | JVM 保证 | 必须手动 finally |
| 性能 | JDK6 后基本持平 | 持平 |
| 建议 | **默认** | 需要上述高级能力才用 |

## 四、读写锁与 StampedLock

```java
// ReentrantReadWriteLock：读读共享、读写/写写互斥
ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
rw.readLock().lock();       // 多线程并发读（读多写少场景）
rw.writeLock().lock();      // 独占写
// 降级：持写锁 → 获取读锁 → 释放写锁（安全降级）

// StampedLock：乐观读（无锁验证版本）性能更高，但不可重入，API 复杂
StampedLock sl = new StampedLock();
long stamp = sl.tryOptimisticRead();      // 乐观读不加锁
if (!sl.validate(stamp)) {                // 版本变了 → 升级悲观读
    stamp = sl.readLock();
    try { ... } finally { sl.unlockRead(stamp); }
}
```

## 五、死锁

```java
// 四个条件：互斥、持有并等待、不可剥夺、循环等待
Thread1: lock(A) → lock(B)
Thread2: lock(B) → lock(A)      // 死锁！

// 预防策略
1. 统一加锁顺序（按固定序获取）
2. tryLock + 超时，失败放弃回退
3. 减小锁粒度/缩短持锁时间
4. 检测工具：jstack 看到 "Found one Java-level deadlock"
```

## 六、锁的最佳实践

```java
// 1. 缩小临界区：只锁共享状态操作
// 2. 锁对象 private final，禁止锁 this（外部也能锁到）
// 3. 分段锁思想降低竞争（ConcurrentHashMap 就是这么做的）
// 4. 优先无锁：Atomic/不可变对象（21/24 篇）
// 5. 持锁期间不调用外部方法/不做 IO
```

## 常见坑

- **坑 1**：synchronized 修可变引用/Integer/String → 锁不住（对象变了）
- **坑 2**：ReentrantLock 忘 finally unlock → 死锁
- **坑 3**：wait 用 if 不用 while → 虚假唤醒
- **坑 4**：lock.lock() 与 try 之间放代码 → unlock 可能因异常跳过
- **坑 5**：读写锁写饥饿（读密集下写者等不到）→ StampedLock 或公平锁
- **坑 6**：锁 this + 外部代码也锁该对象 → 意外串锁

## Checklist

- [ ] 默认 synchronized + private final 锁对象
- [ ] 临界区最小化
- [ ] ReentrantLock 必 try/finally unlock
- [ ] 需要超时/可中断/多条件才升级 Lock
- [ ] 多锁场景统一获取顺序防死锁
