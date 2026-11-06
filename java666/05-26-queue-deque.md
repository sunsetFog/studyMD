# 26 · Queue、Deque 与阻塞队列

## 1. Queue 接口

```java
public interface Queue<E> extends Collection<E> {
    boolean offer(E e);     // 入队，失败返回 false（不抛异常）
    E poll();               // 出队，空返回 null
    E peek();               // 查看队首，空返回 null
    boolean add(E e);       // 入队，失败抛异常
    E remove();             // 出队，空抛 NoSuchElementException
    E element();            // 查看队首，空抛异常
}
```

| 操作 | 抛异常 | 返回特殊值 |
|---|---|---|
| 入队 | `add` | `offer`（false） |
| 出队 | `remove` | `poll`（null） |
| 查看 | `element` | `peek`（null） |

**生产代码优先用返回特殊值的方法**（减少异常开销）。

## 2. Deque 接口

双端队列，同时可作栈：

```java
public interface Deque<E> extends Queue<E> {
    void addFirst(E e); void addLast(E e);
    boolean offerFirst(E e); boolean offerLast(E e);
    E removeFirst(); E removeLast();
    E pollFirst(); E pollLast();
    E getFirst(); E getLast();
    E peekFirst(); E peekLast();

    // 栈操作（推荐替代 Stack）
    void push(E e);      // = addFirst
    E pop();             // = removeFirst
    E peek();            // = peekFirst

    boolean removeFirstOccurrence(Object o);
    boolean removeLastOccurrence(Object o);
    Iterator<E> descendingIterator();
}
```

## 3. ArrayDeque

### 原理

循环数组 + 两个指针（head、tail），容量为 2 的幂。

```
[ ][a][b][c][ ]
   ↑head   ↑tail
```

- 头部操作：`head = (head - 1) & (len - 1)`
- 尾部操作：`tail = (tail + 1) & (len - 1)`
- 扩容：翻倍，保持 2 的幂。

### 特性

| 维度 | 说明 |
|---|---|
| 复杂度 | 头尾操作 O(1) 均摊 |
| 随机访问 | 不支持（无索引） |
| null | **不允许**（用 null 表示空） |
| 线程安全 | ✘ |
| 内存 | 比 LinkedList 省（无 Node 对象） |

```java
Deque<String> deque = new ArrayDeque<>();
deque.offerLast("b");
deque.offerFirst("a");
deque.peekFirst();    // a
deque.pollLast();     // b

// 当栈用
deque.push("x");
deque.pop();
```

### 为什么替代 LinkedList / Stack

- 比 `LinkedList` 快（连续内存、无节点分配）。
- 比 `Stack` 安全（Stack 继承 Vector，全同步）。
- 官方推荐：`Deque<Integer> stack = new ArrayDeque<>();`

## 4. PriorityQueue

### 原理

二叉小顶堆，数组存储：

```
索引 i 的父节点: (i-1)/2
左子节点: 2i+1，右子节点: 2i+2
```

### 特性

| 操作 | 复杂度 |
|---|---|
| `offer` | O(log n)（上浮） |
| `poll` | O(log n)（下沉） |
| `peek` | O(1) |
| `remove(Object)` | O(n) |
| `contains` | O(n) |

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.offer(5); pq.offer(1); pq.offer(3);
pq.peek();     // 1
pq.poll();     // 1, 然后 3

// 大顶堆
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());

// 自定义对象
PriorityQueue<Task> tasks = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority).thenComparing(Task::getCreatedAt));
```

**注意**：

- `poll` 顺序有序，但**遍历（for/iterator）无序**。
- 不允许 null。
- 非线程安全，并发用 `PriorityBlockingQueue`。
- 不保证稳定性（相同优先级顺序不确定）。

### 经典应用

```java
// Top K 问题（求最大的 K 个）：维护大小为 K 的小顶堆
public static List<Integer> topK(int[] nums, int k) {
    PriorityQueue<Integer> heap = new PriorityQueue<>(k);
    for (int n : nums) {
        if (heap.size() < k) heap.offer(n);
        else if (n > heap.peek()) { heap.poll(); heap.offer(n); }
    }
    return new ArrayList<>(heap);
}
// 时间 O(n log k)，空间 O(k)

// 合并 K 个有序链表 / 合并 K 个有序文件
```

## 5. 阻塞队列（BlockingQueue）

```java
public interface BlockingQueue<E> extends Queue<E> {
    void put(E e) throws InterruptedException;          // 队满阻塞
    boolean offer(E e, long timeout, TimeUnit unit);    // 队满等待超时
    E take() throws InterruptedException;               // 队空阻塞
    E poll(long timeout, TimeUnit unit);
    int remainingCapacity();
    int drainTo(Collection<? super E> c);
}
```

### 家族对比

| 实现 | 结构 | 有界 | 锁 | 特点 |
|---|---|---|---|---|
| `ArrayBlockingQueue` | 数组 | ✔ 必须指定 | 1 把锁（可公平） | 简单、内存固定 |
| `LinkedBlockingQueue` | 链表 | 可选（默认 `Integer.MAX_VALUE`） | 2 把锁（put/take 分离） | 吞吐高，但默认无界易 OOM |
| `PriorityBlockingQueue` | 堆 | 无界 | 1 把锁 | 按优先级出队 |
| `SynchronousQueue` | 无容量 | 容量 0 | — | 直接移交，必须有消费者等待 |
| `DelayQueue` | 优先队列 | 无界 | 1 把锁 | 延迟到期才能取 |
| `LinkedTransferQueue` | 链表 | 无界 | CAS | 高吞吐，支持 transfer |

### 生产者-消费者模板

```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(1000);

// 生产者
public void produce(Task t) throws InterruptedException {
    queue.put(t);                 // 满则阻塞，天然背压
}

// 消费者
public void consume() throws InterruptedException {
    while (true) {
        Task t = queue.take();    // 空则阻塞
        handle(t);
    }
}

// 优雅关闭
private volatile boolean running = true;
public void shutdown() { running = false; }

// 消费者改进：用 poll + 超时感知关闭信号
while (running || !queue.isEmpty()) {
    Task t = queue.poll(500, TimeUnit.MILLISECONDS);
    if (t != null) handle(t);
}
```

线上更推荐直接使用 `ThreadPoolExecutor`（内部就是 BlockingQueue）。

### 各队列选择

```java
// 固定线程池 + 有界队列（防止 OOM，推荐）
new ThreadPoolExecutor(
    4, 16, 60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(2000),
    new ThreadPoolExecutor.CallerRunsPolicy()
);

// 高吞吐：LinkedBlockingQueue（务必指定容量！）
new LinkedBlockingQueue<>(10000);

// 优先级任务调度
new PriorityBlockingQueue<>(1000, Comparator.comparing(Task::getPriority));

// 定时任务
DelayQueue<DelayedTask> dq = new DelayQueue<>();

// 直接移交（CachedThreadPool 用）
new SynchronousQueue<>();
```

### SynchronousQueue

- 容量 0：`put` 必须等到有线程 `take`。
- `newCachedThreadPool` 的队列实现（配合无上限线程数）。
- 适合「任务立即交给线程处理」的场景。

```java
SynchronousQueue<String> sq = new SynchronousQueue<>();
new Thread(() -> { try { System.out.println(sq.take()); } catch (Exception ignored) {} }).start();
sq.put("hello");    // 阻塞直到被取走
```

### DelayQueue

```java
class DelayedTask implements Delayed {
    private final long executeAt;      // 毫秒时间戳
    private final Runnable task;

    DelayedTask(Runnable task, long delayMs) {
        this.task = task;
        this.executeAt = System.currentTimeMillis() + delayMs;
    }

    @Override public long getDelay(TimeUnit unit) {
        return unit.convert(executeAt - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }

    @Override public int compareTo(Delayed o) {
        return Long.compare(getDelay(TimeUnit.MILLISECONDS), o.getDelay(TimeUnit.MILLISECONDS));
    }

    public void run() { task.run(); }
}
```

> 生产环境定时任务用 Quartz / XXL-JOB / ScheduledExecutorService。

## 6. 栈的实现对比

| 方式 | 推荐度 | 说明 |
|---|---|---|
| `ArrayDeque` | ✔✔✔ | 最快，官方推荐 |
| `LinkedList` | ✔ | 可用但更慢 |
| `Stack` | ✘ | 遗留类，全同步 |
| 数组手写 | △ | 极特殊场景 |

```java
Deque<Character> stack = new ArrayDeque<>();
stack.push('(');
stack.pop();
stack.isEmpty();
stack.peek();
```

### 应用：括号匹配

```java
public static boolean valid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    for (char c : s.toCharArray()) {
        switch (c) {
            case '(' , '[' , '{' -> stack.push(c);
            case ')' -> { if (stack.isEmpty() || stack.pop() != '(') return false; }
            case ']' -> { if (stack.isEmpty() || stack.pop() != '[') return false; }
            case '}' -> { if (stack.isEmpty() || stack.pop() != '{') return false; }
        }
    }
    return stack.isEmpty();
}
```

## 7. 常见坑

1. `ArrayDeque` 不允许 null（`offer(null)` 抛 NPE）。
2. `PriorityQueue` 遍历无序（只有 `poll` 有序）。
3. `LinkedBlockingQueue` 默认容量 `Integer.MAX_VALUE`（等于无界，易 OOM）。
4. 用 `LinkedList` 当队列（应 `ArrayDeque`）。
5. 阻塞队列 `put/take` 未处理 `InterruptedException`（吞掉中断导致无法优雅停机）。
6. 消费者异常退出后生产者永久阻塞（需监控队列长度）。
7. `SynchronousQueue` 无消费者时 `put` 永久阻塞。
8. `PriorityQueue` 元素修改优先级后不重新堆化（需 remove + offer）。

```java
// 正确响应中断
try {
    Task t = queue.take();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // 恢复中断标志
    return;                               // 退出循环
}
```

## 8. 选型决策

```
需要阻塞（生产者-消费者）？
├─ 是 → 有界高吞吐？ LinkedBlockingQueue(容量)
│        固定内存？ ArrayBlockingQueue
│        优先级？ PriorityBlockingQueue
│        延迟？ DelayQueue
│        直接移交？ SynchronousQueue
└─ 否 → 需要优先级？ PriorityQueue
         需要双端/栈？ ArrayDeque
         普通 FIFO → ArrayDeque
```
