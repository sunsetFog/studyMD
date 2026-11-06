# 48 · 垃圾回收

## 1. 如何判断对象可回收

### 引用计数（Java 未采用）

每个对象维护引用计数，为 0 则回收。问题：**循环引用**无法回收。

### 可达性分析（Java 采用）

从 **GC Roots** 出发遍历引用链，不可达的对象可回收。

GC Roots 包括：

- 虚拟机栈中局部变量引用的对象。
- 方法区中静态变量引用的对象。
- 方法区中常量引用的对象。
- 本地方法栈 JNI 引用的对象。
- 活跃线程。
- 同步锁持有的对象。
- JVM 内部引用（Class 对象、异常、类加载器）。

```java
// 即使互相引用，只要从 GC Root 不可达，都会被回收
A a = new A();
B b = new B();
a.b = b;
b.a = a;
a = null;
b = null;      // 两者都不可达 → 回收
```

## 2. 引用类型

| 类型 | 类 | 回收时机 | 场景 |
|---|---|---|---|
| 强引用 | 默认 | 永不（除非不可达） | 普通对象 |
| 软引用 | `SoftReference` | 内存不足时 | 缓存 |
| 弱引用 | `WeakReference` | 下次 GC | `WeakHashMap`、ThreadLocal key |
| 虚引用 | `PhantomReference` | 随时，需配合 `ReferenceQueue` | 资源清理（DirectByteBuffer） |

```java
// 软引用缓存
Map<String, SoftReference<byte[]>> cache = new HashMap<>();
cache.put(k, new SoftReference<>(data));
byte[] v = cache.get(k).get();     // 可能为 null（已被回收）

// 弱引用
WeakReference<User> wr = new WeakReference<>(user);
// 下次 GC 后 wr.get() 可能为 null
```

## 3. 回收算法

### 标记-清除（Mark-Sweep）

- 标记可达对象 → 清除未标记对象。
- 问题：**内存碎片**、效率随对象数增长。

### 复制（Copying）

- 内存分两块，存活对象复制到另一块，整块清空。
- 优点：无碎片、分配快（指针碰撞）。
- 缺点：浪费一半空间。
- 用于：新生代（Eden → Survivor）。

### 标记-整理（Mark-Compact）

- 标记 → 存活对象向一端移动 → 清理边界外内存。
- 优点：无碎片。
- 缺点：移动成本高（需更新引用）。
- 用于：老年代。

### 分代收集

- 新生代：大量对象朝生夕死 → 复制算法。
- 老年代：对象存活率高 → 标记-清除/整理。

**弱分代假说**：绝大多数对象都是朝生夕灭。

## 4. 新生代回收（Minor GC）

```
Eden + S0 → 存活对象复制到 S1，年龄+1，清空 Eden + S0
         → 下次 S1 + Eden → S0
```

细节：

- Eden:Survivor = 8:1:1，可用空间 90%（避免浪费）。
- 大对象直接进老年代（`-XX:PretenureSizeThreshold`）。
- 动态年龄判定：Survivor 中同龄对象总和 > 一半 → 直接晋升。
- **空间分配担保**：Minor GC 前检查老年代剩余空间是否够放所有新生代对象（否则直接 Full GC）。

## 5. 老年代回收

- **Major GC**：只回收老年代。
- **Full GC**：回收整个堆 + 方法区，STW 长。
- 触发条件：老年代空间不足、Metaspace 不足、`System.gc()`、担保失败。

## 6. 垃圾收集器

| 收集器 | 区域 | 算法 | 特点 | 状态 |
|---|---|---|---|---|
| Serial | 新生代 | 复制 | 单线程，STW | 客户端/小堆 |
| ParNew | 新生代 | 复制 | Serial 多线程版 | 配合 CMS |
| Parallel Scavenge | 新生代 | 复制 | 吞吐优先 | JDK 8 默认 |
| Serial Old | 老年代 | 标记-整理 | 单线程 | 备用 |
| Parallel Old | 老年代 | 标记-整理 | 吞吐优先 | JDK 8 默认 |
| CMS | 老年代 | 标记-清除 | 低延迟，并发 | JDK 9 废弃，14 移除 |
| **G1** | 全堆 | 复制+整理 | 可预测停顿 | **JDK 9+ 默认** |
| ZGC | 全堆 | 染色指针 | 停顿 < 1ms | JDK 15+ 生产可用 |
| Shenandoah | 全堆 | Brooks 指针 | 低停顿 | RedHat 主导 |
| Epsilon | — | 不回收 | 性能测试 | 实验 |

### Serial / Parallel

```bash
-XX:+UseSerialGC
-XX:+UseParallelGC          # 吞吐优先（批处理）
-XX:ParallelGCThreads=8
-XX:MaxGCPauseMillis=200
-XX:GCTimeRatio=99          # 吞吐量目标
```

### G1

Region 化堆，优先回收垃圾最多的 Region（Garbage First）。

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200            # 目标停顿（软目标）
-XX:G1HeapRegionSize=8m
-XX:InitiatingHeapOccupancyPercent=45   # 触发并发标记的堆占用
-XX:G1NewSizePercent=5
-XX:G1MaxNewSizePercent=60
-XX:G1MixedGCLiveThresholdPercent=85
```

回收过程：

1. 年轻代 GC（Young GC，STW）。
2. 并发标记（Concurrent Marking）。
3. 混合回收（Mixed GC，回收部分老年代 Region）。
4. Full GC（兜底，Serial 单线程，尽量避免）。

优点：可预测停顿、无碎片（Region 间复制）、大堆表现好。
缺点：小堆（< 4G）不如 Parallel；频繁 Full GC 时性能骤降。

### ZGC

```bash
-XX:+UseZGC
-XX:ZCollectionInterval=120
-XX:ZAllocationSpikeTolerance=5
```

原理：

- **染色指针（Colored Pointers）**：在 64 位指针中嵌入标记位。
- **读屏障**：读对象时修正引用（并发重定位）。
- 并发标记、并发转移，STW 只有极短的初始/最终标记。
- 停顿 < 1ms（与堆大小无关），支持 TB 级堆。

代价：吞吐略低（读屏障开销）、内存占用略高。

JDK 21 起支持分代 ZGC（`-XX:+UseZGC -XX:+ZGenerational`）。

## 7. GC 日志

### JDK 9+ 统一日志

```bash
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=50m
-Xlog:gc,gc+heap=info
```

输出示例：

```
[2026-01-01T10:00:00.123+0800][12345ms] GC(42) Pause Young (Normal) (G1 Evacuation Pause)
[12345ms] GC(42) Eden regions: 100->0(100) Survivor: 10->5(10) Old: 200->210(500)
[12345ms] GC(42) 12.3ms
```

关键指标：

- 停顿时间（Pause）。
- 各代回收前后占用。
- 回收频率。
- Full GC 次数（应为 0）。

### 分析工具

- **GCViewer**、**GCeasy**（在线）。
- `jstat -gcutil <pid> 1000`。

## 8. 调优思路

```
1. 明确目标：低延迟（停顿短）还是高吞吐？
2. 选收集器：延迟敏感 → G1/ZGC；吞吐 → Parallel
3. 设堆大小（-Xms = -Xmx）
4. 观察 GC 日志：频率、停顿、晋升速率
5. 调整新生代 / 晋升阈值 / IHOP
6. 压测验证，避免过度调参
```

### 常见问题对照

| 现象 | 可能原因 | 调整 |
|---|---|---|
| Young GC 频繁 | 新生代太小 / 分配率高 | 增大 `-Xmn`，减少临时对象 |
| 对象过早晋升 | Survivor 太小 / 年龄阈值低 | 增大 Survivor，调 `MaxTenuringThreshold` |
| Full GC 频繁 | 内存泄漏 / 老年代不足 / 大对象 | dump 分析，增大堆或修泄漏 |
| 停顿过长 | 堆大 + 老年代碎片 | G1/ZGC，调 `MaxGCPauseMillis` |
| GC 后内存不降 | 内存泄漏 | MAT 分析 GC Root |

## 9. STW（Stop The World）

GC 时暂停所有应用线程。

| 收集器 | 停顿 |
|---|---|
| Serial | 长（与堆成正比） |
| Parallel | 中 |
| CMS | 短（初始/重新标记） |
| G1 | 短（可预测） |
| ZGC | 极短（< 1ms） |

停顿期间请求堆积 → 超时 → 雪崩。低延迟系统必须关注 P99 停顿。

## 10. 内存泄漏 vs 内存溢出

| 概念 | 含义 |
|---|---|
| 内存泄漏 | 对象不再使用但无法回收，逐渐累积 |
| 内存溢出 | 内存不足以分配新对象（OOM） |

泄漏的常见来源：

- 静态集合不断 add（缓存无淘汰）。
- 监听器/回调注册未注销。
- `ThreadLocal` 未 remove。
- 连接/流未关闭。
- 内部类持有外部引用。
- 自定义缓存无上限。

```java
// 泄漏示例
static final Map<String, byte[]> CACHE = new HashMap<>();
public void put(String k, byte[] v) { CACHE.put(k, v); }   // 永不清理
// 改：Caffeine/Guava Cache（带淘汰策略）
```

## 11. 速查

| 需求 | 参数 |
|---|---|
| 默认收集器 | G1（JDK 9+） |
| 高吞吐 | `-XX:+UseParallelGC` |
| 低延迟 | `-XX:+UseG1GC -XX:MaxGCPauseMillis=100` |
| 极低延迟 | `-XX:+UseZGC` |
| 小内存客户端 | `-XX:+UseSerialGC` |
| GC 日志 | `-Xlog:gc*:file=gc.log` |
| OOM dump | `-XX:+HeapDumpOnOutOfMemoryError` |
| 查看 GC | `jstat -gcutil <pid> 1000` |
