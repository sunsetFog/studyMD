# 30 · JVM 运行时内存区域

## 一、运行时数据区全景（Java 8+）

```
线程私有：
├── 程序计数器        当前执行字节码地址（唯一不会 OOM 的区域）
├── 虚拟机栈          栈帧集合：局部变量表/操作数栈/动态链接/返回地址
│     └── StackOverflowError（深度超栈）、OutOfMemoryError（栈扩展失败）
└── 本地方法栈        native 方法

线程共享：
├── 堆（Heap）        对象实例；GC 主战场；OOM 高发区
│     ├── 新生代：Eden + 2×Survivor（S0/S1）
│     └── 老年代
└── 方法区（元空间 Metaspace 实现）  类元信息/运行时常量池
      └── OOM（类加载器泄漏，动态生成类过多）
堆外：
└── 直接内存（NIO DirectBuffer）     OOM: Direct buffer memory
```

## 二、栈帧与方法调用

```java
void a() { int x = 1; b(x); }
void b(int n) { c(); }
// 每次调用压入一个栈帧（局部变量表+操作数栈），返回弹出
// 递归过深 → StackOverflowError（默认栈约 512KB-1MB）
```

参数：`-Xss256k`（调小可测递归边界；调大支持深递归但容纳线程数变少）。

## 三、对象内存布局

```
Java 对象 = 对象头(16B：MarkWord 8B + 类型指针/压缩 4-8B) + 实例数据 + 对齐填充(8B 整倍)

new Object() ≈ 16 字节
int 字段 4B，long 8B，引用 4B（压缩指针）
```

**估算工具**：JOL（Java Object Layout）：`ClassLayout.parseInstance(obj).toPrintable()`。

## 四、堆内结构细节

```
Eden:Survivor 默认 8:1:1（-XX:SurvivorRatio=8）
新生代:老年代 默认 1:2（-XX:NewRatio=2）

对象分配流程：
1. TLAB（线程本地分配缓冲）优先 —— 无锁分配
2. Eden 满 → Minor GC（复制存活对象到 S 区，年龄+1）
3. 年龄 ≥ 15（MaxTenuringThreshold）或动态判定 → 晋升老年代
4. 大对象（> 大对象阈值）直接进老年代
5. 老年代满 → Major/Full GC
```

## 五、字符串常量池与 Interned

```java
// Java 7+ 字符串常量池在堆中
String s = new String("ab");   // 堆新对象 + 池中 "ab"（若不存在）
s.intern();                    // 返回池中引用

// JDK 常见笔试：intern 的语义变化（Java 6 永久池 vs 7+ 堆池）
String a = "ab";               // 字面量入池
String b = new String("a") + new String("b");   // 堆上拼接对象
b.intern() == b;               // Java 7+ true（池里没有则记录堆引用）
```

## 六、元空间（Metaspace）

- Java 8 起**永久代被移除**，类元数据挪到本地内存（不占堆）
- 溢出条件：动态生成大量类（CGLIB 代理/反射/JSP）+ 类加载器未释放
- 参数：`-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m`

## 七、OOM 分类速查

| 报错 | 区域 | 常见原因 |
|---|---|---|
| Java heap space | 堆 | 泄漏/大对象/堆太小 |
| GC overhead limit exceeded | 堆 | GC 98% 回收 <2%，近乎泄漏 |
| Metaspace | 元空间 | 动态类泄漏 |
| unable to create new native thread | 线程栈/系统 | 线程数失控 |
| Direct buffer memory | 直接内存 | NIO Buffer 未释放 |
| StackOverflowError | 栈 | 递归/深调用链 |

排查全流程见 35 篇。

## 八、参数速查（详细调优见 33 篇）

```bash
-Xms2g -Xmx2g            # 堆初始/最大（生产设为相同，避免动态扩缩抖动）
-Xss512k                 # 线程栈
-XX:MetaspaceSize=256m
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dump/   # OOM 自动快照
-XX:MaxDirectMemorySize=1g
```

## 常见坑

- **坑 1**：把 StackOverflowError 用 try-catch 吞 → 递归逻辑隐藏
- **坑 2**：容器内未感知内存限制 → JVM 用宿主内存计算；启用容器感知（33 篇）
- **坑 3**：堆外内存泄漏监控缺失 → NIO 场景 set MaxDirectMemorySize + 监控
- **坑 4**：动态代理无限生成 → Metaspace OOM；代理类复用缓存
- **坑 5**：以为局部对象一定栈分配 → 默认堆（逃逸分析优化例外，34 篇）

## Checklist

- [ ] 画出运行时数据区并说明线程私有/共享
- [ ] 说清对象分配与晋升流程
- [ ] OOM 六类报错对应区域与原因
- [ ] OOM 自动 dump 参数标配
