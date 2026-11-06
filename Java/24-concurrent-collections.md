# 24 · 并发容器

## 一、ConcurrentHashMap（重点中的重点）

### 基本用法

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
map.get("a");

// 原子计算方法（替代 get-check-put 竞态）
map.computeIfAbsent("k", k -> expensiveLoad(k));
map.compute("k", (k, v) -> v == null ? 1 : v + 1);
map.merge("k", 1, Integer::sum);
map.putIfAbsent("k", v);
map.replace("k", 100);

// 多步复合操作仍需同步：ConcurrentHashMap 只保证单方法原子
synchronized (map) {          // 或用外部锁
    if (!map.containsKey("a")) { map.put("a", load()); }
}
// 更好：merge/compute 覆盖绝大多数场景

// 弱一致性迭代器：不抛 CME，但可能读到旧数据（体现一致性 vs 性能取舍）
for (var e : map.entrySet()) { }
```

### 底层结构（面试高频）

```
JDK 7：Segment 分段锁（默认 16 段，锁粒度=段）
JDK 8+：Node 数组 + 链表/红黑树
- put：CAS 初始化 + 首节点直接 CAS；哈希冲突 → synchronized 锁桶头节点（锁粒度=单桶）
- size：CounterCell 分片计数（类似 LongAdder），弱一致
- 扩容：多线程协同迁移（helpTransfer）
- key/value 不允许 null（并发下 get 返回 null 歧义：不存在 vs 值为 null）
```

**进化线**：Hashtable（全表锁）→ Collections.synchronizedMap（包装锁）→ Segment 分段 → JDK 8 CAS+synchronized 单桶 → 现代高性能。

## 二、CopyOnWriteArrayList

```java
List<Listener> listeners = new CopyOnWriteArrayList<>();
listeners.add(l => ...);

// 读：无锁，直接读快照（get/遍历零同步）
// 写：复制整个数组 → 替换引用 + ReentrantLock

// 适用：读极多写极少（监听器列表、配置缓存）
// 代价：写 O(n) 复制 + 内存翻倍 + 迭代器是旧快照（不反映后续修改）
```

**反模式**：高频写列表用它 → 写放大灾难。

## 三、阻塞队列（生产者-消费者标配）

| 队列 | 容量 | 特点 |
|---|---|---|
| ArrayBlockingQueue | 有界 | 数组，公平可选，最常用 |
| LinkedBlockingQueue | 默认**无界**（危险！） | 链表，两个锁（读写分离） |
| SynchronousQueue | 0 | 不存储，直接手递手（CachedThreadPool 用它） |
| PriorityBlockingQueue | 无界 | 优先级出队 |
| DelayQueue | 无界 | 延迟出队（定时任务/缓存过期） |

```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(1000);

// 生产者
queue.put(task);              // 满则阻塞
queue.offer(task, 1, TimeUnit.SECONDS);   // 限时，返回 false

// 消费者
Task t = queue.take();        // 空则阻塞
Task t2 = queue.poll(1, TimeUnit.SECONDS);   // 限时取 null

// 生产者-消费者模板
void producer() {
    while (running) queue.put(produce());
}
void consumer() {
    while (running) {
        Task t = queue.poll(1, TimeUnit.SECONDS);    // 限时轮询可响应停止
        if (t != null) process(t);
    }
}
```

## 四、ConcurrentLinkedQueue / Deque

```java
// 无锁 CAS 队列：非阻塞，无界
ConcurrentLinkedQueue<Task> clq = new ConcurrentLinkedQueue<>();
clq.offer(t); clq.poll();     // CAS 实现 Micheal-Scott 算法
// 适用：非阻塞任务调度、事件收集
// size() 是 O(n) 遍历！别在循环里调
```

## 五、并发工具容器速览

```java
// ConcurrentSkipListMap：并发有序 Map（跳表），替代 TreeMap
ConcurrentNavigableMap<Integer, String> skip = new ConcurrentSkipListMap<>();
skip.firstEntry(); skip.tailMap(10);

// LinkedBlockingDeque 双端阻塞队列
```

## 六、Collections.synchronizedXxx vs 并发容器

```java
Map<String, Integer> m = Collections.synchronizedMap(new HashMap<>());   // 全表单锁
// 遍历仍需手动同步：
synchronized (m) { for (var e : m.entrySet()) { } }      // 否则 CME

// 性能与功能全面落后于 ConcurrentHashMap → 只在遗留代码兼容时用
```

## 七、选择决策表

| 场景 | 选择 |
|---|---|
| 并发键值缓存/计数 | ConcurrentHashMap + compute/merge |
| 高频读低频写列表 | CopyOnWriteArrayList |
| 读多写少 Map + 有序 | ConcurrentSkipListMap |
| 生产者消费者（背压） | ArrayBlockingQueue |
| 任务直接交接 | SynchronousQueue |
| 延迟处理 | DelayQueue |
| 高吞吐无阻塞收集 | ConcurrentLinkedQueue |

## 常见坑

- **坑 1**：ConcurrentHashMap 允许 null 键值吗 → 不允许（HashMap 可以），NPE 意外
- **坑 2**：以为单方法原子 = 复合操作安全 → check-then-act 仍竞态，用 compute/merge
- **坑 3**：LinkedBlockingQueue 默认无界 → 积压 OOM；显式给容量
- **坑 4**：CopyOnWriteArrayList 高频写 → 内存与 GC 灾难
- **坑 5**：size()/isEmpty() 当精确判断 → 弱一致；业务判断用 containsKey 等具体操作
- **坑 6**：迭代中依赖实时一致性 → COW 迭代器是旧快照

## Checklist

- [ ] ConcurrentHashMap + compute/merge 消灭竞态样板
- [ ] 阻塞队列必有界，put/poll 带超时意识
- [ ] COW 只用于读多写极少
- [ ] 复合操作识别（check-then-act / read-modify-write）
