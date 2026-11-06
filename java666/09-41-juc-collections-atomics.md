# 41 · 并发容器、CAS 与原子类

## 1. CAS 原理

```java
// 伪代码
boolean cas(int expected, int newValue) {
    if (memory == expected) { memory = newValue; return true; }
    return false;
}
```

CPU 指令：`cmpxchg`（x86）、`ldrex/strex`（ARM）。

Java 暴露：`sun.misc.Unsafe.compareAndSwapInt` → `AtomicInteger` 封装。

### 三大问题

| 问题 | 说明 | 解决 |
|---|---|---|
| ABA | 值 A→B→A，CAS 误认为未变 | `AtomicStampedReference` / `AtomicMarkableReference` |
| 自旋开销 | 高竞争时大量空转 | 退避、`LongAdder` 分段 |
| 只能保证单变量 | 多变量原子性 | `AtomicReference` 封装对象、锁 |

```java
// ABA 演示
AtomicInteger v = new AtomicInteger(1);
v.compareAndSet(1, 2);
v.compareAndSet(2, 1);      // 回到 1，其他线程的 CAS(1, 3) 仍成功

// 解决
AtomicStampedReference<Integer> sr = new AtomicStampedReference<>(1, 0);
int stamp = sr.getStamp();
sr.compareAndSet(1, 2, stamp, stamp + 1);
```

## 2. 原子类家族

### 基本类型

```java
AtomicInteger i = new AtomicInteger(0);
i.incrementAndGet();          // ++i
i.getAndIncrement();          // i++
i.addAndGet(5);
i.getAndSet(10);
i.compareAndSet(10, 20);
i.updateAndGet(x -> x * 2);   // Java 8
i.accumulateAndGet(3, Integer::sum);

AtomicLong / AtomicBoolean / AtomicReference<T>
```

### 数组

```java
AtomicIntegerArray arr = new AtomicIntegerArray(10);
arr.incrementAndGet(0);
arr.compareAndSet(0, 1, 2);
AtomicLongArray / AtomicReferenceArray
```

### 字段更新器（反射 + CAS）

```java
public class Counter {
    volatile int value;                    // 必须 volatile
    private static final AtomicIntegerFieldUpdater<Counter> UPDATER =
        AtomicIntegerFieldUpdater.newUpdater(Counter.class, "value");

    void inc() { UPDATER.incrementAndGet(this); }
}
```

优点：不用为每个字段创建 Atomic 对象（省内存）。
限制：字段必须 `volatile`、不能是 `private`（除非同类）、不能是 `final`/`static`。

### 引用与标记

```java
AtomicReference<Config> ref = new AtomicReference<>(old);
ref.updateAndGet(c -> c.withTimeout(3000));

AtomicStampedReference<V>   // 版本号，解决 ABA
AtomicMarkableReference<V>  // 布尔标记
```

### 累加器（Java 8）

```java
LongAdder adder = new LongAdder();
adder.increment();
adder.add(10);
long sum = adder.sum();

LongAccumulator acc = new LongAccumulator(Long::max, Long.MIN_VALUE);
acc.accumulate(5);
long max = acc.get();

DoubleAdder / DoubleAccumulator
```

**LongAdder vs AtomicLong**：

| 维度 | AtomicLong | LongAdder |
|---|---|---|
| 原理 | 单变量 CAS | 分段 Cell 数组，最后求和 |
| 高并发写 | 竞争激烈，性能下降 | 分散竞争，快数倍 |
| 读 | O(1) | O(n) 求和 |
| 适用 | 读写均衡 | 写多读少（统计计数） |

```java
// 统计 QPS：写极多，读偶尔
private final LongAdder requestCount = new LongAdder();
requestCount.increment();
```

## 3. 并发容器全景

| 容器 | 替代 | 特点 |
|---|---|---|
| `ConcurrentHashMap` | HashMap | 分段 CAS + synchronized 桶锁 |
| `CopyOnWriteArrayList` | ArrayList | 写时复制，读无锁 |
| `CopyOnWriteArraySet` | HashSet | 基于 COW 数组 |
| `ConcurrentSkipListMap` | TreeMap | 跳表，有序，O(log n) |
| `ConcurrentSkipListSet` | TreeSet | 同上 |
| `ConcurrentLinkedQueue` | LinkedList（队列） | 无锁 CAS，无界 |
| `ConcurrentLinkedDeque` | Deque | 无锁双端 |
| `ArrayBlockingQueue` | — | 数组 + 单锁 |
| `LinkedBlockingQueue` | — | 链表 + 双锁 |
| `PriorityBlockingQueue` | — | 堆 + 锁 |
| `SynchronousQueue` | — | 零容量移交 |
| `DelayQueue` | — | 延迟出队 |
| `LinkedTransferQueue` | — | 高吞吐移交 |

## 4. ConcurrentHashMap 深入

### JDK 7 vs JDK 8

| 维度 | JDK 7 | JDK 8+ |
|---|---|---|
| 结构 | Segment[] + HashEntry[] | Node[] + 链表/红黑树 |
| 锁 | 分段锁（ReentrantLock，默认 16 段） | CAS + synchronized 锁单个桶 |
| 并发度 | 段数（固定） | 桶数（随扩容增长） |
| 扩容 | 段内独立 | 多线程协助 transfer |
| size | 遍历段统计 | baseCount + CounterCell[] |

### 关键机制

```java
// put 的核心（简化）
if (casTabAt(tab, i, null, new Node(hash, key, value))) return;   // 空桶：CAS 无锁写入
else {
    synchronized (node) {           // 非空桶：锁首节点
        // 链表/红黑树插入
    }
}
```

1. **空桶 CAS 写入**：无锁。
2. **非空桶 synchronized**：锁粒度是一个桶（不是整个表）。
3. **多线程协助扩容**：`helpTransfer`，通过 `ForwardingNode` 标记已迁移的桶。
4. **size**：`baseCount` + `CounterCell[]` 分散计数（类似 LongAdder）。
5. **不允许 null**：key 和 value 都不能为 null（避免 `get` 返回 null 的二义性）。

### 弱一致性

- 迭代器不抛 `ConcurrentModificationException`。
- `size()` 可能不精确（并发下的近似值）。
- `clear()` 不是原子的。

### 复合操作必须用原子方法

```java
// ✘ 非原子
if (!map.containsKey(k)) map.put(k, v);

// ✔ 原子
map.putIfAbsent(k, v);

// ✘ 非原子
Integer old = map.get(k);
map.put(k, old + 1);

// ✔ 原子
map.merge(k, 1, Integer::sum);
map.compute(k, (key, val) -> val == null ? 1 : val + 1);

// ✔ 原子
map.computeIfAbsent(k, key -> new ArrayList<>()).add(item);
```

### 初始化容量

```java
// 期望存 1000 个元素
ConcurrentHashMap<String, User> map = new ConcurrentHashMap<>(1024);
// 内部会调整为 2048（保证 2 的幂，且考虑负载因子）
```

## 5. ConcurrentSkipListMap

- 基于跳表（多层索引链表），O(log n)。
- 线程安全、有序、支持范围查询。
- 无锁（CAS）实现，并发度高。
- 替代 `Collections.synchronizedSortedMap(new TreeMap<>())`。

```java
ConcurrentSkipListMap<Long, Order> timeline = new ConcurrentSkipListMap<>();
timeline.put(System.currentTimeMillis(), order);
timeline.headMap(cutoff);          // 范围查询
timeline.firstEntry(); timeline.lastEntry();
```

应用：排行榜、时间线、按 key 范围扫描。

## 6. 并发集合选型

```
需要 Map？
├─ 是 → 需要排序？ ConcurrentSkipListMap : ConcurrentHashMap
└─ 否 → 需要队列？
     ├─ 是 → 需要阻塞？ BlockingQueue 家族
     │        不需要阻塞？ ConcurrentLinkedQueue
     └─ 否 → 读多写极少？ CopyOnWriteArrayList
              普通 → 不可变集合 或 线程私有
```

## 7. 无锁编程实战

### 无锁栈

```java
public class LockFreeStack<T> {
    private final AtomicReference<Node<T>> top = new AtomicReference<>();

    private static class Node<T> {
        final T value;
        Node<T> next;
        Node(T v, Node<T> n) { value = v; next = n; }
    }

    public void push(T v) {
        Node<T> old;
        Node<T> newNode;
        do {
            old = top.get();
            newNode = new Node<>(v, old);
        } while (!top.compareAndSet(old, newNode));
    }

    public T pop() {
        Node<T> old;
        Node<T> newTop;
        do {
            old = top.get();
            if (old == null) return null;
            newTop = old.next;
        } while (!top.compareAndSet(old, newTop));
        return old.value;
    }
}
```

> 无锁栈的 ABA 问题在实践中较少（节点不会被复用），但存在内存回收问题（需要 GC 或 Hazard Pointer）。

### 无锁计数器

```java
public class StripedCounter {
    private final AtomicLongArray cells;
    private static final int STRIPES = 16;

    public StripedCounter() {
        cells = new AtomicLongArray(STRIPES);
    }

    public void inc() {
        int idx = Thread.currentThread().hashCode() & (STRIPES - 1);
        cells.incrementAndGet(idx);
    }

    public long sum() {
        long s = 0;
        for (int i = 0; i < STRIPES; i++) s += cells.get(i);
        return s;
    }
}
```

## 8. 线程安全集合的陷阱

```java
// 1. 复合操作不安全
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
if (map.get("a") == null) map.put("a", 1);      // 检查-执行 非原子

// 2. 并发容器内的对象仍需线程安全
ConcurrentHashMap<String, List<String>> m = new ConcurrentHashMap<>();
m.get("a").add("x");                            // List 本身不是线程安全的
// 改：computeIfAbsent(k, x -> Collections.synchronizedList(new ArrayList<>()))
// 或 CopyOnWriteArrayList

// 3. 迭代期间修改的语义
for (String k : chm.keySet()) {
    chm.remove(k);      // 不抛异常，但可能漏删
}
// 改：chm.keySet().removeIf(...)

// 4. size 不精确
chm.size();            // 并发下是近似值

// 5. Collections.synchronizedMap 的复合操作
Map<String, Integer> sync = Collections.synchronizedMap(new HashMap<>());
if (!sync.containsKey(k)) sync.put(k, 1);        // 仍需外部 synchronized
synchronized (sync) { ... }
```

## 9. 性能对比参考

| 操作 | HashMap | CHM（1 线程） | CHM（8 线程） |
|---|---|---|---|
| get | 最快 | 略慢 | 扩展性好 |
| put | 最快 | 略慢 | 显著更快 |
| 内存 | 小 | 略大 | 略大 |

| 计数器 | 单线程 | 8 线程竞争 |
|---|---|---|
| `synchronized int` | 快 | 慢 |
| `AtomicLong` | 快 | 中等 |
| `LongAdder` | 略慢 | 最快 |

## 10. 常见坑

1. 用 `Collections.synchronizedMap` 替代 `ConcurrentHashMap`（性能差、复合操作不安全）。
2. 认为 `ConcurrentHashMap` 的 `get` + `put` 是原子的。
3. 存 null 到 CHM（NPE）。
4. 用 `size()` 做业务判断（不精确）。
5. `CopyOnWriteArrayList` 写频繁（性能灾难）。
6. 遍历 CHM 时假设看到所有元素（弱一致）。
7. `AtomicInteger` 高并发计数（用 LongAdder）。
8. 忘记 `AtomicReference` 里对象的不可变性（内部字段仍可变）。
9. 用 `volatile` + 非原子操作替代原子类。
10. 自定义无锁结构未考虑内存回收（节点泄漏）。

## 11. 速查

| 需求 | 类 |
|---|---|
| 并发 Map | `ConcurrentHashMap` |
| 有序并发 Map | `ConcurrentSkipListMap` |
| 并发 List（读多） | `CopyOnWriteArrayList` |
| 并发 Set | `ConcurrentHashMap.newKeySet()` |
| 并发队列 | `ConcurrentLinkedQueue` |
| 阻塞队列 | `ArrayBlockingQueue` 等 |
| 计数器 | `AtomicLong` / `LongAdder` |
| 状态标志 | `AtomicBoolean` / `volatile` |
| 对象引用 | `AtomicReference` |
| 解决 ABA | `AtomicStampedReference` |
| 字段更新 | `AtomicIntegerFieldUpdater` |
