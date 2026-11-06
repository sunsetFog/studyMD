# 39 · JMM、synchronized 与 volatile

## 1. Java 内存模型（JMM）

JMM 定义了多线程下**共享变量的可见性、有序性、原子性**规则。

```
        ┌─────────────┐
线程 A  │ 工作内存     │ ←→ 主内存（堆中的共享变量）
        └─────────────┘
线程 B  │ 工作内存     │
        └─────────────┘
```

- 每个线程有私有的工作内存（CPU 缓存 + 寄存器的抽象）。
- 共享变量存于主内存。
- 线程读写要经过工作内存 → **可能读到旧值**。

JMM 不保证「一个线程的写立刻对另一个线程可见」，需要**同步机制**建立 happens-before 关系。

## 2. 三大问题

### 可见性

```java
class Visibility {
    static boolean flag = false;          // 无 volatile

    public static void main(String[] args) throws Exception {
        new Thread(() -> {
            while (!flag) { }             // 可能永远看不到 flag 变 true
            System.out.println("exit");
        }).start();

        Thread.sleep(1000);
        flag = true;
    }
}
```

原因：读线程把 `flag` 缓存在寄存器/CPU cache 中，JIT 甚至可能把循环优化成 `while(true)`。

### 原子性

```java
static int count = 0;
// 多线程 count++ 结果小于预期
```

`count++` 是「读-改-写」三步，非原子。

### 有序性

```java
int a = 0;
boolean ready = false;

// 线程 1
a = 1;
ready = true;

// 线程 2
if (ready) System.out.println(a);   // 可能输出 0
```

编译器和 CPU 可能**重排序**（在不改变单线程语义的前提下）。

## 3. happens-before 规则

如果 A happens-before B，则 A 的结果对 B 可见，且 A 在 B 之前执行（逻辑上）。

| 规则 | 说明 |
|---|---|
| 程序顺序 | 同一线程内，前面的操作 hb 后面的操作 |
| 监视器锁 | 解锁 hb 后续对同一锁的加锁 |
| volatile | 对 volatile 变量的写 hb 后续对该变量的读 |
| 线程启动 | `Thread.start()` hb 新线程中的所有操作 |
| 线程终止 | 线程中所有操作 hb 其他线程检测到它终止（`join` 返回） |
| 中断 | `interrupt()` hb 被中断线程检测到中断 |
| 终结器 | 对象构造完成 hb `finalize()` 开始 |
| 传递性 | A hb B, B hb C → A hb C |

## 4. synchronized 的语义

### 三种用法

```java
// 1. 实例方法：锁 this
public synchronized void m() { }

// 2. 静态方法：锁 Class 对象
public static synchronized void s() { }

// 3. 同步块：锁指定对象
public void b() {
    synchronized (lock) { }
}
```

### 三大保证

1. **原子性**：临界区代码互斥执行。
2. **可见性**：解锁前把工作内存写回主内存；加锁时清空工作内存并从主内存读取。
3. **有序性**：临界区内代码不会被重排序到锁外（通过内存屏障）。

### 实现原理

```java
// 字节码
monitorenter
...临界区...
monitorexit
```

对象头 Mark Word 记录锁状态：

| 状态 | Mark Word 内容 |
|---|---|
| 无锁 | hashCode + 分代年龄 + 01 |
| 偏向锁 | 线程 ID + epoch + 01 |
| 轻量级锁 | 指向栈中锁记录的指针 + 00 |
| 重量级锁 | 指向 Monitor 的指针 + 10 |
| GC 标记 | 11 |

> JDK 15 起偏向锁默认关闭（JDK 18 移除），因为维护成本高、收益有限。

### 锁升级

```
无锁 → 偏向锁（同一线程反复进入）→ 轻量级锁（CAS 自旋）→ 重量级锁（OS 互斥量，阻塞）
```

只能升级不能降级（简化实现）。

### Monitor 结构

```
ObjectMonitor
├── _owner       当前持有线程
├── _EntryList   等待锁的线程
├── _WaitSet     调用 wait() 的线程
├── _recursions  重入次数
└── _count
```

### 可重入

```java
synchronized (lock) {
    synchronized (lock) { }     // 同一线程可重复进入，_recursions++
}
```

## 5. volatile

### 两个语义

1. **可见性**：写 volatile 变量立即刷回主内存，读时从主内存读。
2. **禁止重排序**：通过内存屏障（LoadLoad/StoreStore/LoadStore/StoreLoad）禁止指令重排。

### 不保证原子性

```java
volatile int count = 0;
count++;            // 仍然不是原子的！
```

正确做法：

```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
```

### 适用场景

| 场景 | 是否适用 |
|---|---|
| 状态标志（`volatile boolean running`） | ✔ |
| 一次性安全发布（双重检查锁） | ✔ |
| 独立观察（多个线程读，一个线程写） | ✔ |
| 计数器（`i++`） | ✘ |
| 复合操作 | ✘ |
| 需要互斥 | ✘ |

### 双重检查锁（DCL）

```java
public class Singleton {
    private static volatile Singleton instance;    // volatile 必须

    public static Singleton getInstance() {
        if (instance == null) {                    // 第一次检查（避免每次都加锁）
            synchronized (Singleton.class) {
                if (instance == null) {            // 第二次检查
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

为什么必须 volatile：`new Singleton()` 分三步（分配内存 → 初始化 → 赋值引用），2 和 3 可能重排，导致其他线程拿到**未初始化完成**的对象。

更推荐：**静态内部类**或**枚举单例**（由 JVM 类初始化保证）。

```java
// 静态内部类：懒加载 + 线程安全
public class Singleton {
    private Singleton() { }
    private static class Holder { static final Singleton INSTANCE = new Singleton(); }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}

// 枚举：最简洁
public enum Singleton { INSTANCE; }
```

### volatile 的内存屏障

| 屏障 | 作用 |
|---|---|
| LoadLoad | 禁止读读重排 |
| StoreStore | 禁止写写重排 |
| LoadStore | 禁止读写重排 |
| StoreLoad | 禁止写读重排（开销最大，x86 用 mfence/lock 前缀） |

volatile 写前插 StoreStore，写后插 StoreLoad；volatile 读后插 LoadLoad/LoadStore。

x86 天然保证除 StoreLoad 外的大部分顺序，所以 volatile 读几乎无成本，写有 `lock addl` 开销。

## 6. final 的内存语义

```java
public class User {
    private final int id;
    public User(int id) { this.id = id; }
}
```

正确构造的对象，其 final 字段对其他线程**必然可见**，无需同步——即使对象引用通过数据竞争发布。

这是不可变对象安全发布的基础。

## 7. 有序性与重排序

三类重排序：

1. **编译器重排序**（JIT 优化）。
2. **CPU 指令重排序**（乱序执行）。
3. **内存系统重排序**（缓存/写缓冲）。

单线程语义不变（as-if-serial），多线程可能出问题。

```java
// 经典：指令重排导致的对象逸出
public class UnsafePublication {
    int x;
    UnsafePublication() { x = 42; }

    static UnsafePublication instance;
    static void writer() { instance = new UnsafePublication(); }
    static void reader() {
        if (instance != null) {
            int v = instance.x;     // 可能读到 0
        }
    }
}
```

## 8. synchronized vs volatile vs CAS

| 维度 | synchronized | volatile | CAS（原子类） |
|---|---|---|---|
| 原子性 | ✔ | ✘ | ✔（单变量） |
| 可见性 | ✔ | ✔ | ✔ |
| 有序性 | ✔ | ✔ | ✔ |
| 阻塞 | ✔（竞争时） | ✘ | ✘（自旋） |
| 适用 | 复合操作/临界区 | 状态标志 | 计数器/无锁结构 |
| 性能 | 低竞争时优化好 | 高 | 高（高竞争时空转） |

## 9. 锁优化

| 优化 | 说明 |
|---|---|
| 锁消除 | JIT 发现对象不逃逸，去掉锁 |
| 锁粗化 | 相邻的多次加解锁合并 |
| 自适应自旋 | 根据历史成功率决定自旋次数 |
| 偏向锁 | 同一线程反复进入（已废弃） |

```java
// 锁消除示例（局部 StringBuilder，不逃逸）
public String concat(String a, String b) {
    StringBuffer sb = new StringBuffer();   // 内部 synchronized 被消除
    return sb.append(a).append(b).toString();
}
```

## 10. 实战：线程安全计数器

```java
// 1. synchronized
public class Counter1 {
    private int v;
    public synchronized void inc() { v++; }
    public synchronized int get() { return v; }
}

// 2. AtomicInteger（推荐）
public class Counter2 {
    private final AtomicInteger v = new AtomicInteger();
    public void inc() { v.incrementAndGet(); }
    public int get() { return v.get(); }
}

// 3. LongAdder（高并发写）
public class Counter3 {
    private final LongAdder v = new LongAdder();
    public void inc() { v.increment(); }
    public long get() { return v.sum(); }
}
```

## 11. 常见坑

1. `volatile` 用于 `i++`（非原子）。
2. DCL 单例忘记 volatile。
3. 用 `Boolean`/`Integer` 对象做锁（装箱缓存导致意外共享）。
4. 锁对象被重新赋值（`lock = new Object()` 后不同步）。
5. 锁粒度太大（性能差）或太小（无法保护复合操作）。
6. 用字符串常量做锁（`synchronized("a")`，全局共享）。
7. 认为 `synchronized` 修饰静态方法与实例方法锁的是同一个东西（Class vs this）。
8. 忘记 `wait` 释放锁，用 `sleep` 代替（sleep 不释放锁）。
9. 多把锁的获取顺序不一致（死锁）。
10. 以为 JMM 保证 64 位 long/double 的原子性（非 volatile 时不保证，虽然现代 JVM 多数情况原子）。

## 12. 速查

| 需求 | 用法 |
|---|---|
| 状态标志 | `volatile boolean` |
| 互斥临界区 | `synchronized` / `ReentrantLock` |
| 计数 | `AtomicInteger` / `LongAdder` |
| 安全发布 | `final` + 不可变 / `volatile` / 静态初始化 |
| 单例 | 枚举 / 静态内部类 |
| 等待通知 | `Condition` / `BlockingQueue` |
| 可见性调试 | `jcstress`（JVM 并发测试工具） |
