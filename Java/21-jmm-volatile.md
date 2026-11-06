# 21 · JMM 与 volatile：可见性、有序性、原子性

## 一、并发三要素

| 特性 | 定义 | 保障手段 |
|---|---|---|
| **原子性** | 操作不可分割 | synchronized / Lock / Atomic 类 |
| **可见性** | 一线程的修改另一线程立刻可见 | volatile / synchronized / final |
| **有序性** | 程序按编写顺序执行 | volatile / happens-before / synchronized |

## 二、问题的根源：CPU 缓存与指令重排

```
线程工作内存（CPU 缓存/寄存器副本）  ←→  主内存（共享堆）
- 写操作可能滞留缓存 → 其他线程看不到（可见性）
- 编译器/CPU 为优化会重排无依赖指令（有序性）
- i++ 是三条指令（读-改-写）交错（原子性）
```

## 三、JMM（Java Memory Model）

JMM 定义了「主内存 + 工作内存」的抽象与交互规则：所有共享变量存主内存，线程操作自己的工作内存副本，**同步规则靠 happens-before 保证**。

### happens-before 规则（核心八条）

```
1. 程序顺序：单线程内，前面的操作先于后面
2. 锁：解锁先于后续加锁（同一把锁）
3. volatile：写先于后续读
4. 线程启动：start() 前的操作先于线程内任意操作
5. 线程终止：线程内操作先于 join() 返回 / isAlive()=false
6. 中断：interrupt 调用先于被中断线程检测到
7. 对象创建：构造完成先于 finalize/引用发布
8. 传递性：A hb B 且 B hb C → A hb C
```

**实战意义**：判断「两线程访问共享数据是否安全」就是找它们之间有没有 happens-before 边——没有就存在数据竞争。

## 四、volatile

```java
private volatile boolean running = true;

// 线程 A
running = false;

// 线程 B
while (running) { }        // 没有 volatile → B 可能永远看不到 false（JIT 把读提出循环）
```

volatile 两个语义：
1. **可见性**：写立即刷主存 + 使其他线程缓存行失效（MESI 协议）；读强制取最新
2. **有序性**：写读之间插入内存屏障，禁止重排穿越

### volatile 不保证原子性（高频陷阱）

```java
private volatile int count = 0;
count++;      // 读-改-写三步，volatile 管不了交错执行！

// 解决：
AtomicInteger count = new AtomicInteger();   // CAS 原子操作
count.incrementAndGet();
// 或 synchronized 包裹
```

**volatile 适用**：状态标志、一次性发布（DCL 单例的实例字段）。**不适用**：复合操作计数。

## 五、volatile 的禁止重排细节

```
volatile 写之前：普通写不能重排到 volatile 写之后
volatile 读之后：普通读不能提前到 volatile 读之前
→ 这正是双重检查锁（DCL）需要 volatile 的原因：
```

```java
public class Singleton {
    private static volatile Singleton instance;   // volatile 必需！

    public static Singleton getInstance() {
        if (instance == null) {                          // 1
            synchronized (Singleton.class) {
                if (instance == null) {                  // 2
                    instance = new Singleton();          // 3：分配内存→初始化→赋引用
                }                                        // 无 volatile：2/3 步可重排
            }                                            // 其他线程拿到「未初始化完」的引用
        }
        return instance;
    }
}
// 现代替代：静态内部类 Holder（类加载天然线程安全）或枚举
```

## 六、Atomic 原子类族（CAS）

```java
AtomicInteger ai = new AtomicInteger(0);
ai.incrementAndGet();           // ++i
ai.getAndAdd(5);
ai.compareAndSet(expect, update);   // CAS 核心
ai.updateAndGet(x -> x * 2);
ai.accumulateAndGet(5, Math::max);

AtomicLong al;                  // 高竞争下 CAS 自旋失败多 →
LongAdder adder = new LongAdder();   // 分段累加，统计场景首选
adder.increment(); adder.sum();

AtomicReference<T> ref;          // 对象引用原子更新
AtomicBoolean flag;
AtomicIntegerFieldUpdater<MyObj> upd = ...;   // 反射式字段更新（少用）
```

**CAS 原理**：`compareAndSet(期望值, 新值)` → CPU 原子指令（cmpxchg）。失败自旋重试。
**ABA 问题**：值从 A→B→A，CAS 感知不到中间变化 → `AtomicStampedReference` 版本号解决。

## 七、final 与安全发布

- final 字段：构造器中赋值后无需同步即可被其他线程安全看到（JMM 特殊保证）
- **不可变对象天然线程安全**：String、record、全 final 字段类

## 八、综合选型

```
状态标志/一次性发布        → volatile
计数器                     → Atomic / LongAdder
复合操作（多变量）         → synchronized / Lock
读多写少状态               → volatile 或 StampedLock
容器共享                   → 并发容器（24 篇）
```

## 常见坑

- **坑 1**：volatile 用在 count++ → 非原子，照样丢更新
- **坑 2**：DCL 单例漏 volatile → 偶发拿到半初始化对象
- **坑 3**：以为 synchronized 只保证互斥 → 同时保证可见性与有序性
- **坑 4**：局部变量误当线程安全 → 共享的可变静态/实例字段才危险
- **坑 5**：AtomicLong 高竞争吞吐下降 → LongAdder
- **坑 6**：对象「this 逸出」（构造中发布 this 给其他线程）→ 未安全发布

## Checklist

- [ ] 三特性与对应手段对应清晰
- [ ] happens-before 能解释 volatile/锁/线程启动终止的可见性
- [ ] volatile 只用于标志与发布，不用于计数
- [ ] DCL 单例会手写并解释 volatile 必要性
