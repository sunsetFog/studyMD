# 47 · JVM 运行时内存

## 1. 运行时数据区

```
┌────────────────────────────────────────────┐
│                 JVM                        │
│  ┌──────────────┐  ┌────────────────────┐  │
│  │ 方法区/元空间 │  │      堆 Heap        │  │
│  │（共享）       │  │  （共享，GC 主战场） │  │
│  └──────────────┘  └────────────────────┘  │
│  ┌──────────────────────────────────────┐  │
│  │ 线程私有：程序计数器 / 虚拟机栈 / 本地方法栈│
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

| 区域 | 线程 | 内容 | 异常 |
|---|---|---|---|
| 程序计数器（PC） | 私有 | 当前字节码行号 | 无 OOM |
| 虚拟机栈 | 私有 | 栈帧（局部变量、操作数栈、动态链接、返回地址） | `StackOverflowError` / OOM |
| 本地方法栈 | 私有 | native 方法调用 | 同上 |
| 堆 | 共享 | 对象实例、数组 | `OutOfMemoryError: Java heap space` |
| 方法区（元空间） | 共享 | 类元信息、常量、静态变量、JIT 代码 | `OutOfMemoryError: Metaspace` |
| 运行时常量池 | 共享 | 字面量、符号引用（JDK 7+ 字符串常量池在堆） | OOM |
| 直接内存 | — | NIO DirectByteBuffer | `OutOfMemoryError: Direct buffer memory` |

## 2. 堆结构

### 分代（G1 之前）

```
Young（1/3）
├── Eden（8/10）
├── Survivor 0（1/10）
└── Survivor 1（1/10）
Old（2/3）
```

新对象在 Eden 分配；Minor GC 后存活对象进 Survivor（复制算法），年龄 +1；达到阈值（默认 15）晋升 Old。

### G1（JDK 9+ 默认）

堆划分为大小相等的 Region（1-32MB），逻辑上分 Eden/Survivor/Old/Humongous。

### 参数

```bash
-Xms2g                  # 初始堆
-Xmx2g                  # 最大堆（建议与 Xms 相同，避免动态扩容）
-Xmn1g                  # 新生代大小
-XX:NewRatio=2          # 老年代/新生代比例
-XX:SurvivorRatio=8     # Eden/Survivor 比例
-XX:MaxTenuringThreshold=15   # 晋升年龄
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
-XX:MaxDirectMemorySize=256m
-Xss1m                  # 每线程栈大小
```

## 3. 栈帧结构

```
栈帧
├── 局部变量表（Local Variables）    slot 为单位，long/double 占 2 个
├── 操作数栈（Operand Stack）        字节码运算的工作区
├── 动态链接（指向常量池的方法引用）
├── 方法返回地址
└── 附加信息
```

```java
public int add(int a, int b) {
    int c = a + b;
    return c;
}
```

字节码：

```
iload_1        // 压入 a
iload_2        // 压入 b
iadd           // 弹出两个相加，压入结果
istore_3       // 存入局部变量 3（c）
iload_3        // 压入 c
ireturn        // 返回
```

### StackOverflowError

```java
static int f(int n) { return f(n + 1); }   // 无限递归
// java.lang.StackOverflowError
```

默认栈深约 1 万层（取决于 `-Xss` 与栈帧大小）。增大 `-Xss` 只缓解，根本要改算法（迭代/尾递归改写）。

## 4. 对象内存布局

```
| Mark Word (8B) | Klass Pointer (4B，压缩) | 实例数据 | Padding |
```

### Mark Word（64 位）

| 状态 | 内容 |
|---|---|
| 未锁定 | hash(31) + age(4) + biased(1) + lock(2=01) |
| 偏向锁 | threadId(54) + epoch(2) + age(4) + 1 + 01 |
| 轻量级锁 | 指向栈中 Lock Record 的指针 + 00 |
| 重量级锁 | 指向 ObjectMonitor 的指针 + 10 |
| GC 标记 | 空 + 11 |

### 对象大小计算

```java
// 例：class A { int a; boolean b; }
// 对象头 12B（8 + 4 压缩指针）
// 实例数据：int 4 + boolean 1 = 5B
// 对齐到 8 的倍数 → 12 + 5 = 17 → 24B

// 用 JOL 工具测量
System.out.println(GraphLayout.parseInstance(obj).toFootprint());
```

### 指针压缩

```bash
-XX:+UseCompressedOops        # 默认开启（堆 < 32GB）
-XX:+UseCompressedClassPointers
```

堆 > 32GB 时压缩指针失效（4 字节无法寻址），对象变大，**并不是堆越大越好**。

## 5. 逃逸分析

JIT 分析对象是否逃出方法/线程：

| 优化 | 说明 |
|---|---|
| 栈上分配 | 对象不逃逸 → 分配在栈（随方法结束销毁） |
| 标量替换 | 对象拆成基本类型局部变量 |
| 锁消除 | 对象不逃逸 → 去掉 synchronized |

```java
public int sum() {
    Point p = new Point(1, 2);    // 未逃逸，可能被标量替换，不分配堆内存
    return p.x + p.y;
}
```

```bash
-XX:+DoEscapeAnalysis        # 默认开启
-XX:+EliminateAllocations
-XX:+EliminateLocks
-XX:+PrintEscapeAnalysis
```

## 6. 元空间（Metaspace）

JDK 8 起用**本地内存**的 Metaspace 替代永久代（PermGen）。

存储：

- 类元信息（Klass 结构）。
- 方法字节码。
- 常量池（运行时常量池）。
- 注解、字段描述符。

特点：

- 默认无上限（受物理内存限制），建议设 `-XX:MaxMetaspaceSize`。
- 类卸载条件：类加载器可回收 + 无实例 + 无引用。
- 常见 OOM 原因：动态生成类过多（CGLIB、反射、热部署）。

```bash
jstat -gcmetacapacity <pid>
jcmd <pid> VM.metaspace
```

## 7. 直接内存

```java
ByteBuffer buf = ByteBuffer.allocateDirect(1024 * 1024);
```

- 不受堆限制，受 `-XX:MaxDirectMemorySize`（默认等于 `-Xmx`）。
- 分配/释放成本高（`Unsafe.allocateMemory`）。
- 由 `Cleaner` 在 GC 时释放（不可控）。
- Netty 用池化 + 引用计数管理。

OOM 错误信息：`java.lang.OutOfMemoryError: Direct buffer memory`。

## 8. 字符串常量池

```java
String s = "abc";              // 常量池（JDK 7+ 在堆中）
String t = new String("abc");  // 堆对象
```

- JDK 6：在永久代（易 OOM）。
- JDK 7+：移到堆（可被 GC）。
- `intern()` 把字符串放入池并返回池中引用。

## 9. TLAB（Thread Local Allocation Buffer）

每个线程在 Eden 中有一块私有分配缓冲区，避免分配对象时的锁竞争。

```bash
-XX:+UseTLAB              # 默认开启
-XX:TLABSize=256k
-XX:-ResizeTLAB
```

大对象（超过 TLAB 剩余空间）直接在 Eden 共享区分配，可能直接进老年代（`PretenureSizeThreshold`）。

## 10. 内存溢出类型与排查

| 错误 | 原因 | 排查 |
|---|---|---|
| `Java heap space` | 对象过多/泄漏 | `jmap -dump` + MAT |
| `GC overhead limit exceeded` | GC 占用 >98% 时间却回收 <2% | 同上，检查内存泄漏 |
| `Metaspace` | 类太多 | 检查动态代理/CGLIB/热部署 |
| `Direct buffer memory` | 堆外泄漏 | `jcmd VM.native_memory`、Netty 泄漏检测 |
| `unable to create new native thread` | 线程数超限 | 减少线程、调 ulimit |
| `Requested array size exceeds VM limit` | 数组过大 | 分块处理 |
| `Map failed` | 内存映射失败 | 检查虚拟内存/磁盘 |

### OOM 时自动 dump

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump/
```

### 分析工具

- **Eclipse MAT**：支配树、泄漏 suspects、GC Root 路径。
- **JProfiler / YourKit**：实时分析。
- **jvisualvm**：轻量。
- **Arthas**：`heapdump`、`vmtool`。

## 11. 内存参数模板

```bash
# 4C8G 应用服务器
-Xms4g -Xmx4g
-Xss512k
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m
-XX:MaxDirectMemorySize=512m
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump
-Xlog:gc*:file=/data/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=50m
```

## 12. 常见误区

1. 「堆越大越好」——超过 32GB 压缩指针失效，GC 停顿变长。
2. 「`-Xms` 可以小于 `-Xmx`」——会导致堆动态扩容，引发额外 GC，建议相等。
3. 「对象都在堆上」——逃逸分析后可能栈上分配/标量替换。
4. 「方法区就是永久代」——JDK 8 后是 Metaspace（本地内存）。
5. 「`System.gc()` 能解决 OOM」——只是触发一次 Full GC。
6. 「`finalize` 能释放资源」——不可靠，已废弃。
7. 「字符串常量池在方法区」——JDK 7+ 在堆。
8. 「栈溢出调大 `-Xss` 就行」——递归无终止条件仍会溢出。

## 13. 速查

| 区域 | 参数 | OOM |
|---|---|---|
| 堆 | `-Xms/-Xmx` | Java heap space |
| 新生代 | `-Xmn/-XX:NewRatio` | — |
| 元空间 | `-XX:MaxMetaspaceSize` | Metaspace |
| 栈 | `-Xss` | StackOverflowError |
| 直接内存 | `-XX:MaxDirectMemorySize` | Direct buffer memory |
| 代码缓存 | `-XX:ReservedCodeCacheSize` | CodeCache is full |
