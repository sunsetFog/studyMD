# 66 · 面试高频速查

## 1. 基础

**Q：== 和 equals 的区别？**
`==` 比较基本类型的值或引用地址；`equals` 是 Object 方法，默认也是 `==`，重写后比较内容。包装类比较必须用 `equals`（缓存 -128~127）。

**Q：String 为什么不可变？**
`final class` + `private final byte[]`，所有修改返回新对象。好处：字符串池共享、线程安全、hashCode 可缓存、安全。

**Q：String、StringBuilder、StringBuffer？**
String 不可变；StringBuilder 可变非线程安全（快）；StringBuffer 可变线程安全（synchronized，慢）。

**Q：重载与重写的区别？**
重载：同名不同参，编译期绑定。重写：子类覆盖父类同签名方法，运行期多态，返回类型协变、异常不能更宽、可见性不能更严。

**Q：接口与抽象类？**
接口：多实现、只能常量、default/static 方法、can-do 语义。抽象类：单继承、可有状态与构造器、模板方法。需要共享代码用抽象类，定义契约用接口。

**Q：Java 只有值传递吗？**
是。基本类型传值副本，对象传引用副本（可修改对象内容，但不能改变调用方的引用指向）。

**Q：final 的作用？**
类不可继承、方法不可重写、字段只能赋值一次（引用不变，内容可变）、局部变量可被 Lambda 捕获、保证安全发布的内存语义。

**Q：equals 与 hashCode 契约？**
equals 相等 → hashCode 必须相等；反之不要求。只重写 equals 会导致 HashMap/HashSet 行为错误。

## 2. 集合

**Q：HashMap 底层结构？**
JDK 8+：数组 + 链表 + 红黑树。链表长度 ≥ 8 且容量 ≥ 64 转红黑树，≤ 6 退化。

**Q：HashMap 为什么容量是 2 的幂？**
`(n-1) & hash` 等价取模但更快，且 n-1 低位全 1，分布均匀。

**Q：HashMap 扩容机制？**
默认 16、负载因子 0.75，超过阈值翻倍。JDK 8 用高低位链表拆分，元素要么原位要么原位+oldCap。

**Q：为什么负载因子 0.75？**
空间与冲突的折中（泊松分布下链表长度 8 的概率极低）。太大冲突多，太小浪费空间。

**Q：HashMap 线程不安全的表现？**
并发 put 丢数据、size 不准；JDK 7 扩容头插法可能成环导致 CPU 100%（JDK 8 改尾插解决，但仍不安全）。

**Q：ConcurrentHashMap 如何保证线程安全？**
JDK 8：空桶 CAS 写入，非空桶 synchronized 锁首节点；多线程协助扩容；size 用 CounterCell 分散计数。不允许 null。

**Q：ArrayList 与 LinkedList？**
ArrayList 数组，随机访问 O(1)，中间插入 O(n)，内存紧凑；LinkedList 双向链表，头尾 O(1)，随机访问 O(n)，每个节点额外开销。实际几乎总用 ArrayList / ArrayDeque。

**Q：HashMap、LinkedHashMap、TreeMap？**
无序哈希 / 插入或访问序（LRU 基础）/ 红黑树排序 O(log n)。

**Q：如何实现 LRU？**
`LinkedHashMap(16, 0.75f, true)` + 重写 `removeEldestEntry`。生产用 Caffeine。

**Q：fail-fast 与 fail-safe？**
fail-fast 通过 modCount 检测并发修改抛 CME（ArrayList/HashMap）；fail-safe 遍历快照或弱一致（CopyOnWriteArrayList/ConcurrentHashMap）。

## 3. 泛型

**Q：什么是类型擦除？**
泛型只在编译期存在，擦除到边界（无界 → Object），运行时无泛型信息；必要时插入强转与桥方法。

**Q：为什么不能 new T[]？**
运行时数组需要知道元素类型做类型检查，擦除后类型未知。

**Q：PECS？**
Producer Extends, Consumer Super。读取用 `? extends T`，写入用 `? super T`。

**Q：`List<Object>` 与 `List<?>` 区别？**
`List<Object>` 只能接收 Object 列表；`List<?>` 可接收任意类型列表但只能读（Object）不能写。

## 4. 并发

**Q：线程的几种状态？**
NEW、RUNNABLE、BLOCKED、WAITING、TIMED_WAITING、TERMINATED。

**Q：start 与 run？**
`start()` 创建新线程执行 run；`run()` 就是普通方法调用。

**Q：如何安全停止线程？**
`interrupt()` + 检查中断标志；阻塞方法捕获 InterruptedException 后恢复标志并退出。禁用 `stop()`。

**Q：synchronized 原理？**
字节码 `monitorenter/monitorexit`，对象头 Mark Word 记录锁状态；锁升级：无锁 → 偏向锁（已废弃）→ 轻量级（CAS 自旋）→ 重量级（Monitor 阻塞）。

**Q：synchronized 与 ReentrantLock？**
前者 JVM 内置、自动释放、非公平、单条件；后者 API 层、手动 unlock、可公平/超时/中断/多 Condition。

**Q：volatile 的作用？**
可见性 + 禁止重排序，不保证原子性。适合状态标志、DCL 单例。

**Q：JMM 的 happens-before 规则？**
程序顺序、锁、volatile、线程启动/终止、中断、传递性。

**Q：AQS 原理？**
volatile int state + CLH 双向队列 + LockSupport.park/unpark；子类实现 tryAcquire/tryRelease。

**Q：线程池核心参数？**
corePoolSize、maximumPoolSize、keepAliveTime、unit、workQueue、threadFactory、handler。

**Q：线程池执行流程？**
核心线程 → 队列 → 扩容到 max → 拒绝策略。

**Q：为什么不建议用 Executors？**
`newFixedThreadPool`/`newSingleThreadExecutor` 用无界队列（OOM）；`newCachedThreadPool` 最大线程数无上限（线程爆炸）。

**Q：线程数怎么定？**
CPU 密集 N+1；IO 密集 N×(1+W/C)。必须压测并结合下游承受能力。

**Q：死锁的四个条件与预防？**
互斥、持有并等待、不可剥夺、循环等待。预防：统一加锁顺序、tryLock 超时、减少嵌套。

**Q：ThreadLocal 内存泄漏？**
ThreadLocalMap 的 key 是弱引用、value 强引用；线程池线程不销毁 → value 泄漏。用完必须 `remove()`。

**Q：CountDownLatch 与 CyclicBarrier？**
前者一次性、一个/多个线程等 N 个事件；后者可循环、N 个线程互相等待、支持 barrierAction。

**Q：CompletableFuture 的作用？**
异步编排：链式转换、组合、异常处理、超时。必须传自定义线程池，避免 commonPool 饥饿。

**Q：虚拟线程？**
JDK 21 正式，JVM 调度的轻量线程（M:N），阻塞时卸载，适合 IO 密集；不要池化；注意 synchronized pinning（JDK 24 前）与 ThreadLocal 内存。

## 5. JVM

**Q：JVM 内存区域？**
堆（共享）、方法区/Metaspace（共享）、虚拟机栈/本地方法栈/程序计数器（私有）、直接内存。

**Q：对象创建过程？**
类加载检查 → 分配内存（指针碰撞/空闲列表，TLAB）→ 零值初始化 → 设置对象头 → 执行构造器。

**Q：对象内存布局？**
对象头（Mark Word + Klass Pointer）+ 实例数据 + 对齐填充。

**Q：如何判断对象可回收？**
可达性分析（GC Roots：栈引用、静态变量、常量、JNI、活跃线程）。

**Q：四种引用？**
强、软（内存不足回收）、弱（下次 GC）、虚（配合引用队列做清理）。

**Q：GC 算法？**
标记-清除（碎片）、复制（新生代）、标记-整理（老年代）、分代收集。

**Q：GC 收集器？**
Serial、Parallel、CMS（已移除）、G1（JDK 9+ 默认，Region 化、可预测停顿）、ZGC（< 1ms，染色指针 + 读屏障）。

**Q：什么时候触发 Full GC？**
老年代不足、Metaspace 不足、担保失败、`System.gc()`。

**Q：类加载过程？**
加载 → 验证 → 准备（零值）→ 解析 → 初始化（`<clinit>`）。

**Q：双亲委派？**
先委派父加载器，父加载不了再自己加载。保证核心类安全与唯一性。打破场景：SPI（上下文类加载器）、Tomcat、OSGi。

**Q：OOM 排查思路？**
看错误类型 → `jmap -dump` + MAT 找支配树与 GC Root 引用链 → 定位代码 → 修复泄漏或调堆。

**Q：CPU 100% 排查？**
`top -Hp` 找线程 → `printf %x` 转十六进制 → `jstack` 搜索 nid → 定位代码。

## 6. 框架/工程

**Q：Spring Bean 生命周期？**
实例化 → 属性注入 → Aware 回调 → BeanPostProcessor 前置 → 初始化（@PostConstruct / afterPropertiesSet / init-method）→ BeanPostProcessor 后置（AOP 代理）→ 使用 → 销毁。

**Q：Spring 循环依赖？**
单例 + setter/字段注入通过三级缓存解决；构造器注入无法解决（需 @Lazy 或重构）。

**Q：Spring AOP 原理？**
有接口用 JDK 动态代理，无接口用 CGLIB；BeanPostProcessor 在初始化后创建代理。自调用失效（this 不是代理）。

**Q：@Transactional 失效场景？**
自调用、非 public 方法、异常被吞、异常类型不匹配（默认只回滚 RuntimeException）、多线程、类未被 Spring 管理、数据库引擎不支持事务。

**Q：MyBatis Mapper 原理？**
JDK 动态代理，方法 → MappedStatement → 执行 SQL → 结果映射。

**Q：如何设计一个秒杀系统？**
限流（网关/令牌桶）→ 缓存预热 → Redis 预扣库存（Lua 原子）→ MQ 异步下单 → 数据库最终一致 → 防超卖/防重。

## 7. 编码实战题思路

**Q：手写单例**
枚举 / 静态内部类 / DCL + volatile。

**Q：手写生产者消费者**
`BlockingQueue` + 消费者循环 + `poll(timeout)` + 异常捕获 + 优雅关闭。

**Q：手写线程池（简化）**
核心线程 + 阻塞队列 + 拒绝策略 + 工作线程循环取任务。

**Q：两个线程交替打印 1-100**
`ReentrantLock` + 两个 `Condition`，或 `Semaphore`，或 `AtomicInteger` + 自旋。

**Q：统计大文件词频**
`BufferedReader` 逐行 → 正则/分词 → `HashMap` 计数 → 按 value 排序取 Top N（或 `PriorityQueue`）。

**Q：实现 LRU**
`LinkedHashMap(accessOrder=true)` 或 `HashMap + 双向链表`。

**Q：判断链表是否有环**
快慢指针。

**Q：反转链表**
三指针迭代或递归。

## 8. 答题技巧

1. **先说结论，再展开**（面试官时间有限）。
2. **分层回答**：是什么 → 怎么做 → 为什么 → 边界/坑。
3. **主动补充**：提到「JDK 8 用 CAS + synchronized 锁桶」比只说「分段锁」加分。
4. **不会就说不会**，但给出思路（「我会先看 GC 日志确认是哪种 OOM」）。
5. **结合项目**：讲你实际遇到的 OOM/死锁/性能问题及解决过程。
6. **画图**：HashMap 结构、AQS 队列、线程池流程。

## 9. 高频「坑」清单（能说出即加分）

- `Integer` 缓存 127 陷阱
- `Arrays.asList` 不可增删
- `subList` 是视图
- 增强 for 中删除元素
- `BigDecimal.equals` 比 scale
- `SimpleDateFormat` 非线程安全
- DCL 单例必须 volatile
- HashMap key 必须不可变
- `Collectors.toMap` 重复 key 抛异常
- 线程池 `submit` 异常被吞
- `ThreadLocal` 未 remove
- AOP 自调用失效
- `@Transactional` 只回滚 RuntimeException
- 连接池 `maxLifetime` 大于 DB `wait_timeout`
- MySQL `LIMIT` 深分页
- 索引失效（函数、隐式转换、最左前缀）
