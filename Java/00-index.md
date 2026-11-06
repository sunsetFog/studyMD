# Java 完整知识体系 · 总索引

> 版本基准：Java 17 / 21 LTS（长期支持版），标注 21 独有特性处会说明
> 学习路径：语法基础 → OOP → 核心 API → 并发 → IO/网络 → JVM → 进阶实战

## 目录

### 一、基础语法
| 篇目 | 内容 |
|---|---|
| [01-setup](01-setup.md) | 环境搭建：JDK/JRE/JVM、安装、IDE、第一个程序 |
| [02-syntax-basics](02-syntax-basics.md) | 语法基础：变量、数据类型、运算符、类型转换 |
| [03-control-flow](03-control-flow.md) | 流程控制：if/switch/for/while、break/continue |
| [04-arrays-strings](04-arrays-strings.md) | 数组与字符串：多维数组、String/StringBuilder |
| [05-methods](05-methods.md) | 方法：定义、重载、可变参数、传值语义 |

### 二、面向对象
| 篇目 | 内容 |
|---|---|
| [06-class-object](06-class-object.md) | 类与对象：构造器、this、static、代码块 |
| [07-encapsulation-inheritance-polymorphism](07-encapsulation-inheritance-polymorphism.md) | 三大特性：封装、继承、多态 |
| [08-abstract-interface](08-abstract-interface.md) | 抽象类与接口、final |
| [09-object-wrapper](09-object-wrapper.md) | Object 类、equals/hashCode、包装类与自动装箱 |
| [10-inner-enum-record](10-inner-enum-record.md) | 内部类、枚举、Record（Java 16+） |
| [11-lambda-functional](11-lambda-functional.md) | Lambda 与函数式接口 |
| [12-oop-design-principles](12-oop-design-principles.md) | OOP 设计原则：SOLID 与组合优于继承 |

### 三、核心 API
| 篇目 | 内容 |
|---|---|
| [13-common-classes](13-common-classes.md) | 常用类：Math、Random、BigDecimal、System |
| [14-date-time](14-date-time.md) | 日期时间：java.time 全面替代 Date/Calendar |
| [15-exception](15-exception.md) | 异常体系：try-catch、自定义异常、最佳实践 |
| [16-collections](16-collections.md) | 集合框架：List/Set/Map 深度剖析 |
| [17-generics](17-generics.md) | 泛型：类型参数、通配符、擦除 |
| [18-stream](18-stream.md) | Stream 流与 Optional |

### 四、并发编程
| 篇目 | 内容 |
|---|---|
| [19-thread-basics](19-thread-basics.md) | 线程基础：创建方式、生命周期、线程方法 |
| [20-synchronization-locks](20-synchronization-locks.md) | 锁体系：synchronized、Lock、读写锁 |
| [21-jmm-volatile](21-jmm-volatile.md) | JMM 与 volatile：可见性、有序性、happens-before |
| [22-concurrency-tools](22-concurrency-tools.md) | 并发工具：CountDownLatch、Semaphore、AQS |
| [23-thread-pool](23-thread-pool.md) | 线程池：参数、工作流程、拒绝策略 |
| [24-concurrent-collections](24-concurrent-collections.md) | 并发容器：ConcurrentHashMap、阻塞队列 |
| [25-virtual-threads](25-virtual-threads.md) | 虚拟线程（Java 21）与 CompletableFuture |

### 五、IO 与网络
| 篇目 | 内容 |
|---|---|
| [26-io-nio](26-io-nio.md) | IO 与 NIO：字节/字符流、缓冲、Files |
| [27-network](27-network.md) | 网络编程：Socket、HTTP 客户端（Java 11+） |
| [28-serialization](28-serialization.md) | 序列化：Serializable、JSON、安全 |
| [29-reflection-annotation](29-reflection-annotation.md) | 反射与注解：Class 对象、动态代理 |

### 六、JVM
| 篇目 | 内容 |
|---|---|
| [30-jvm-memory](30-jvm-memory.md) | 运行时内存区域：堆/栈/方法区 |
| [31-classloading](31-classloading.md) | 类加载：双亲委派、初始化时机 |
| [32-gc](32-gc.md) | 垃圾回收：算法、分代、G1/ZGC |
| [33-jvm-tuning](33-jvm-tuning.md) | JVM 调优：参数、容器化 |
| [34-jit](34-jit.md) | JIT 即时编译：分层编译、逃逸分析 |
| [35-troubleshooting](35-troubleshooting.md) | 线上问题排查：OOM/CPU/内存泄漏 |

### 七、进阶与实战
| 篇目 | 内容 |
|---|---|
| [36-new-features](36-new-features.md) | Java 8→21 新特性演进 |
| [37-design-patterns](37-design-patterns.md) | 常用设计模式（Java 视角） |
| [38-toolchain](38-toolchain.md) | 工具链：Maven/Gradle、Lombok、单元测试 |
| [39-interview-summary](39-interview-summary.md) | 高频面试题 + 学习路线总结 |

## 全局约定

- 所有代码基于 Java 17 语法基线，21 特性单独标注（虚拟线程、Record Pattern 等）
- 全局规则只写一次：
  - equals/hashCode 契约 → [09](09-object-wrapper.md) / [16](16-collections.md)
  - 线程池手动创建 → [23](23-thread-pool.md)
  - 不可变日期时间 → [14](14-date-time.md)
  - BigDecimal 货币计算 → [13](13-common-classes.md)
  - Stream 陷阱 → [18](18-stream.md)
- 每篇结尾有「常见坑」与「Checklist」
