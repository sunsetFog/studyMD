# 11 - Java 核心考点

## 1. 集合框架（必考⭐⭐）

| 接口 | 实现 | 考点 |
|---|---|---|
| List | ArrayList / LinkedList | 数组 vs 双向链表；ArrayList 扩容 1.5 倍；LinkedList 实际很少用 |
| Set | HashSet / TreeSet / LinkedHashSet | 基于 HashMap；TreeSet 红黑树有序 |
| Map | HashMap / TreeMap / ConcurrentHashMap / LinkedHashMap | 见下 |

**HashMap 原理（最高频）**：
- 数组 + 链表 + 红黑树（链表长度 ≥8 且数组 ≥64 树化，退化阈值 6）
- 容量 2 的幂：`hash = h ^ (h >>> 16)`，`(n-1) & hash` 等价取模
- 扩容：负载因子 0.75，翻倍；JDK8 后迁移用高低位链表拆分（无需重算 hash）
- 为什么容量是 2 的幂：位运算取模 + 扩容时高低位拆分简单
- 线程不安全：JDK7 头插死循环；JDK8 数据覆盖；并发用 ConcurrentHashMap

**ConcurrentHashMap（JDK8+）**：
- CAS + synchronized 锁单个桶头节点，替代分段锁
- size 用 CounterCell 分散计数（LongAdder 思想）
- 不允许 null key/value（并发下二义性）

## 2. 并发编程（必考⭐⭐）

### 线程与线程池

```java
ThreadPoolExecutor(core, max, keepAlive, unit, workQueue, threadFactory, handler);
// 执行流程：core 满 → 入队 → 队满 → 扩到 max → 拒绝策略
// 拒绝策略：Abort(默认抛异常)/CallerRuns/Discard/DiscardOldest
```

- 为什么禁用 Executors：FixedThreadPool 无界队列 OOM；CachedThreadPool max≈Integer.MAX 线程爆炸
- 核心数经验：CPU 密集 N+1，IO 密集 N*(1+等待/计算比)

### 锁与同步

- **synchronized**：对象头 Mark Word，锁升级（无锁→偏向→轻量级 CAS→重量级 monitor）
- **AQS**（抽象队列同步器）：state + CLH 变体队列；ReentrantLock/Semaphore/CountDownLatch 全基于它
- ReentrantLock：可中断、公平锁、多 Condition
- volatile：可见性 + 禁止指令重排（内存屏障），不保证原子性；DCL 单例必须 volatile
- ThreadLocal：Thread 内 ThreadLocalMap，弱引用 key；**必须 remove 防泄漏**；线程池场景脏数据

### JMM

- happens-before 规则（程序顺序/锁/volatile/线程启动终止）
- 原子性/可见性/有序性三性对应手段

## 3. 常用类与语法

- String 不可变 + 字符串池；StringBuilder vs StringBuffer
- equals/hashCode 契约：equals 相等则 hash 必须相等（HashMap 正确性前提）
- Integer 缓存 -128~127（== 陷阱）；拆箱 NPE
- BigDecimal：金额计算禁 double；`divide` 必须指定 scale/rounding
- 接口 default/static 方法、函数式接口、方法引用

## 4. Stream 与 Optional

```java
list.stream().filter().map().collect(Collectors.groupingBy(User::getDept));
```

- 惰性求值（中间操作不执行）、并行流谨慎（共享可变状态）
- Optional 防 NPE，不用于字段/参数
- Collectors：toList/groupingBy/joining/toMap（**toMap key 冲突抛异常**，要写 merge 函数）

## 5. 新版本特性（加分）

- Java 8：Lambda/Stream/Optional
- Java 11+：var、HttpClient
- Java 17：sealed、record（值对象）、switch 模式匹配
- Java 21：虚拟线程（IO 密集场景）、Record Pattern

## 常见追问

1. fail-fast 机制：modCount 校验，遍历中结构修改抛 ConcurrentModificationException；用 removeIf/迭代器删除
2. ArrayList 1.5 倍扩容、初始 10（懒加载，首次 add 才分配）
3. synchronized 修饰静态方法锁什么？——Class 对象

## Checklist

- [ ] HashMap 原理能从结构讲到扩容
- [ ] 线程池七参数与执行顺序能背
- [ ] AQS 能说清 state + 队列模型
