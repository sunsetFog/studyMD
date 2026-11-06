# 46 · 并发陷阱与设计模式

## 1. 死锁

### 四个必要条件

1. 互斥（资源不可共享）
2. 持有并等待
3. 不可剥夺
4. 循环等待

### 典型代码

```java
// 线程 1：先锁 A 再锁 B
// 线程 2：先锁 B 再锁 A
Object A = new Object(), B = new Object();

new Thread(() -> { synchronized (A) { sleep(100); synchronized (B) { } } }).start();
new Thread(() -> { synchronized (B) { sleep(100); synchronized (A) { } } }).start();
```

### 检测

```java
ThreadMXBean mx = ManagementFactory.getThreadMXBean();
long[] ids = mx.findDeadlockedThreads();
if (ids != null) {
    for (ThreadInfo ti : mx.getThreadInfo(ids)) {
        System.out.println(ti);
    }
}
```

```bash
jstack <pid> | grep -A 30 "Found one Java-level deadlock"
```

### 预防

```java
// 1. 统一加锁顺序（最简单有效）
// 约定所有代码按 id 升序加锁

// 2. 使用 tryLock 超时
if (lockA.tryLock(1, SECONDS)) {
    try {
        if (lockB.tryLock(1, SECONDS)) {
            try { /* ... */ } finally { lockB.unlock(); }
        } else { /* 回退，释放 A */ }
    } finally { lockA.unlock(); }
}

// 3. 减少锁嵌套 / 用单一锁

// 4. 用无锁结构（ConcurrentHashMap、原子类）

// 5. 用超时兜底（数据库锁、分布式锁）
```

## 2. 活锁与饥饿

### 活锁（Livelock）

线程不断重试但无法推进（互相礼让）。

```java
// 两个线程都检测到冲突就退让，结果同时退让又同时重试
while (conflict) {
    releaseLock();
    sleep(random);       // 需要随机化打破对称
    retry();
}
```

解决：引入随机退避、优先级、仲裁者。

### 饥饿（Starvation）

某线程长期拿不到资源：

- 非公平锁 + 高竞争。
- 线程优先级差异。
- 任务队列中总有更高优先级任务。

解决：公平锁、限制优先级使用、任务分片。

## 3. 可见性与原子性问题

```java
// 1. 忘记 volatile
private boolean running = true;      // 改为 volatile
public void stop() { running = false; }

// 2. 复合操作非原子
private int count;
public void inc() { count++; }       // 改 AtomicInteger

// 3. check-then-act
if (map.get(k) == null) map.put(k, v);      // 改 putIfAbsent

// 4. 延迟初始化
private Config config;
public Config get() {
    if (config == null) config = load();    // 改 DCL + volatile 或静态内部类
    return config;
}

// 5. 发布了未完成初始化的对象
static User user;
user = new User();      // 其他线程可能看到字段为默认值的对象
// 改：final 字段 / volatile / 安全初始化
```

## 4. 线程安全的三要素实现

| 手段 | 适用 |
|---|---|
| 不可变对象 | 最优（无需同步） |
| 线程封闭（栈封闭、ThreadLocal） | 单线程使用 |
| 同步（synchronized/Lock） | 临界区 |
| 原子类 | 单变量 CAS |
| 并发容器 | 数据结构级 |
| 消息传递（队列） | 线程间解耦 |

**优先顺序**：不可变 > 线程封闭 > 消息传递 > 并发容器 > 加锁。

## 5. ThreadLocal 正确使用

```java
private static final ThreadLocal<SimpleDateFormat> FMT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

// 使用后必须 remove（线程池复用会残留）
try {
    String s = FMT.get().format(date);
} finally {
    FMT.remove();
}
```

问题：

- 线程池中线程复用 → 脏数据/内存泄漏。
- 异步任务传递丢失（需 `InheritableThreadLocal` 或手动传递）。
- 虚拟线程中百万实例 → 内存压力。

### 跨线程传递

```java
// InheritableThreadLocal：只在创建子线程时继承（线程池无效）
// 解决：TransmittableThreadLocal（阿里 TTL）或手动包装
```

```java
// 手动传递（推荐，显式清晰）
String ctx = MDC.get("traceId");
executor.submit(() -> {
    MDC.put("traceId", ctx);
    try { doWork(); } finally { MDC.clear(); }
});
```

### 内存泄漏原理

```
Thread → ThreadLocalMap → Entry(WeakReference<ThreadLocal>, value)
```

- key 是弱引用（ThreadLocal 可被回收）。
- value 是强引用（若不 remove，线程不死则 value 不释放）。
- 线程池线程长期存活 → 泄漏。

## 6. 生产者-消费者模式

### 基于 BlockingQueue（推荐）

```java
public class ProducerConsumer {
    private final BlockingQueue<Task> queue = new ArrayBlockingQueue<>(1000);
    private final ExecutorService consumers = Executors.newFixedThreadPool(4);

    public void start() {
        for (int i = 0; i < 4; i++) {
            consumers.submit(() -> {
                while (!Thread.currentThread().isInterrupted()) {
                    try {
                        Task t = queue.poll(500, TimeUnit.MILLISECONDS);
                        if (t != null) process(t);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                }
            });
        }
    }

    public boolean submit(Task t) {
        return queue.offer(t);      // 满了返回 false（背压）
    }
}
```

关键设计：

- 有界队列（防 OOM）。
- 消费者捕获异常（单任务失败不退出）。
- 优雅关闭（中断 + 排空）。
- 监控队列长度（背压信号）。

## 7. 其他并发模式

### 不可变对象（Immutable）

```java
public final class Config {
    private final Map<String, String> values;

    public Config(Map<String, String> values) {
        this.values = Map.copyOf(values);      // 防御性拷贝
    }

    public String get(String k) { return values.get(k); }
}
```

### 线程封闭（Thread Confinement）

```java
// 栈封闭：局部变量天然线程安全
public int compute() {
    int local = 0;      // 只在本线程栈
    return local;
}

// ThreadLocal 封闭
```

### 写时复制（Copy-On-Write）

```java
// 读无锁，写复制
private volatile List<Listener> listeners = List.of();

public void add(Listener l) {
    List<Listener> copy = new ArrayList<>(listeners);
    copy.add(l);
    listeners = List.copyOf(copy);
}
```

### 未来模式（Future/Promise）

```java
Future<Result> f = pool.submit(task);
// 稍后获取
Result r = f.get(1, SECONDS);
```

### Actor 模式

每个 Actor 单线程处理消息队列，避免共享状态（Akka）。

```java
// 简化版
class Counter {
    private final BlockingQueue<Runnable> mailbox = new LinkedBlockingQueue<>();
    private long count;       // 只被 actor 线程访问

    Counter() {
        Thread t = new Thread(() -> {
            while (true) {
                try { mailbox.take().run(); }
                catch (InterruptedException e) { return; }
            }
        });
        t.setDaemon(true);
        t.start();
    }

    void increment() { mailbox.offer(() -> count++); }
}
```

## 8. 双重检查锁与安全发布

```java
// 正确版本
public class SafeLazy {
    private static volatile SafeLazy instance;

    public static SafeLazy get() {
        SafeLazy local = instance;
        if (local == null) {
            synchronized (SafeLazy.class) {
                local = instance;
                if (local == null) {
                    instance = local = new SafeLazy();
                }
            }
        }
        return local;
    }
}
```

## 9. 性能陷阱

| 陷阱 | 后果 | 解决 |
|---|---|---|
| 锁粒度过大 | 串行化 | 缩小临界区 |
| 锁粒度过小 | 无法保护复合操作 | 合理设计 |
| 频繁上下文切换 | CPU 浪费 | 减少线程数、批量处理 |
| 伪共享（False Sharing） | 缓存行失效 | `@Contended` / 填充 |
| 过度同步 | 单线程也慢 | 无可变状态时不同步 |
| 忙等待（自旋） | CPU 100% | `park`/`BlockingQueue` |
| 大对象锁 | GC 压力 | 用专用锁对象 |

### 伪共享示例

```java
// 两个 volatile 变量在同一缓存行（64B），互相干扰
class Bad {
    volatile long a;
    volatile long b;
}

// 缓解：填充（JDK 8+ 用 @Contended，需 -XX:-RestrictContended）
@jdk.internal.vm.annotation.Contended
class Good {
    volatile long a;
}
```

## 10. 并发调试手段

```java
// 1. 线程转储
jstack <pid> > dump.txt
kill -3 <pid>

// 2. 找 BLOCKED 线程
grep -A 5 "java.lang.Thread.State: BLOCKED" dump.txt

// 3. 找死锁
jstack <pid> | grep -i deadlock

// 4. 找 CPU 高的线程
top -Hp <pid>
printf "%x\n" <tid>
jstack <pid> | grep -A 20 <hex>

// 5. 压力测试
// JMH（微基准）、JMeter、wrk

// 6. 并发正确性测试
// jcstress：专门测 JMM 的竞态
```

## 11. 并发代码审查清单

- [ ] 共享可变状态是否有保护？
- [ ] 是否可以用不可变对象替代？
- [ ] 复合操作是否原子（check-then-act）？
- [ ] 锁的获取顺序是否一致？
- [ ] 是否在锁内做 IO/远程调用？
- [ ] `ThreadLocal` 是否 `remove`？
- [ ] 线程池参数是否合理（有界队列、拒绝策略）？
- [ ] 任务异常是否被捕获？
- [ ] 是否有超时（`get`、`await`、`orTimeout`）？
- [ ] 中断标志是否正确处理？
- [ ] 计数器是否用原子类/`LongAdder`？
- [ ] 是否依赖 `Thread.sleep` 做同步（不可靠）？

## 12. 常见"看似正确"的错误代码

```java
// 1. 用 sleep 等待结果
thread.start();
Thread.sleep(1000);              // 不可靠，结果可能未就绪
// 改：join / Future / CountDownLatch

// 2. 用 boolean 标志"停止"线程但没有 volatile
// 3. 用 HashMap 做缓存（并发下丢数据/死循环历史问题）
// 4. 认为 i++ 在多线程下安全
// 5. 在 synchronized 中调用外部回调（可能死锁）
// 6. 用 String 常量做锁（不同模块共享同一锁）
// 7. 用 Integer 做锁（缓存范围内相同对象）
// 8. 双重检查锁忘记 volatile
// 9. 用 notify 而非 notifyAll（唤醒错误线程）
// 10. 在 finally 中 return（吞掉异常）
```

## 13. 黄金法则

1. **能不共享就不共享**（无状态设计、线程封闭）。
2. **能不可变就不可变**（`final` + 不可变集合）。
3. **必须共享时用现成并发容器**，别自己造轮子。
4. **锁的范围尽可能小**，且不要在锁内做慢操作。
5. **所有等待都有超时**。
6. **所有异步都有异常处理**。
7. **优先用高层抽象**（`ExecutorService`、`CompletableFuture`、`BlockingQueue`），而非 `wait/notify`。
8. **压测验证**，不要靠猜。
