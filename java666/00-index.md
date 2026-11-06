# Java 全栈知识体系（从 0 开始）

> 目标：一套可顺序阅读、也可按需查阅的 Java 专家级笔记。
> 约定：所有示例基于 **Java 17 LTS**（部分特性标注 21+），JDK 21 亦可直接运行。
> 结构：入门 → 语法 → 面向对象 → 核心 API → 集合/泛型 → 异常/IO → 并发 → JVM → 反射/函数式 → 工程实践。

---

## 目录

### 一、入门与环境
| 编号 | 文件 | 内容 |
|---|---|---|
| 01 | [01-01-java-introduction.md](01-01-java-introduction.md) | Java 是什么、JVM/JRE/JDK、字节码与跨平台、版本演进 |
| 02 | [01-02-jdk-setup-hello-world.md](01-02-jdk-setup-hello-world.md) | 安装 JDK、JAVA_HOME、javac/java/jar/jshell、第一个程序 |
| 03 | [01-03-program-structure-naming.md](01-03-program-structure-naming.md) | 包、import、类结构、main 方法、命名规范、Javadoc |
| 04 | [01-04-idea-maven-gradle.md](01-04-idea-maven-gradle.md) | IDEA 使用、Maven 生命周期与依赖、Gradle 基础 |

### 二、基础语法
| 编号 | 文件 | 内容 |
|---|---|---|
| 05 | [02-05-variables-data-types.md](02-05-variables-data-types.md) | 8 种基本类型、包装类、装箱拆箱、类型转换、var |
| 06 | [02-06-operators.md](02-06-operators.md) | 算术/比较/逻辑/位运算、短路、优先级、陷阱 |
| 07 | [02-07-control-flow.md](02-07-control-flow.md) | if/switch（含箭头与 yield）、四种循环、break/continue/label |
| 08 | [02-08-arrays.md](02-08-arrays.md) | 数组声明、内存布局、二维数组、Arrays 工具、可变参数 |

### 三、面向对象
| 编号 | 文件 | 内容 |
|---|---|---|
| 09 | [03-09-class-and-object.md](03-09-class-and-object.md) | 类与对象、构造器、this、static、初始化顺序 |
| 10 | [03-10-encapsulation-inheritance-polymorphism.md](03-10-encapsulation-inheritance-polymorphism.md) | 封装、继承、多态、重载 vs 重写、super、final |
| 11 | [03-11-abstract-class-interface.md](03-11-abstract-class-interface.md) | 抽象类、接口、默认方法、函数式接口、选型 |
| 12 | [03-12-modifiers-inner-class.md](03-12-modifiers-inner-class.md) | 访问修饰符、四种内部类、匿名类、闭包限制 |
| 13 | [03-13-object-class-methods.md](03-13-object-class-methods.md) | equals/hashCode 契约、toString、clone、finalize |
| 14 | [03-14-enum.md](03-14-enum.md) | 枚举本质、带字段方法、EnumMap/EnumSet、单例 |
| 15 | [03-15-record-sealed.md](03-15-record-sealed.md) | record、sealed 类、模式匹配 instanceof |

### 四、核心 API
| 编号 | 文件 | 内容 |
|---|---|---|
| 16 | [04-16-string.md](04-16-string.md) | String 不可变、字符串池、intern、StringBuilder/Buffer |
| 17 | [04-17-java-time.md](04-17-java-time.md) | java.time 全家桶、时区、Duration/Period、格式化 |
| 18 | [04-18-math-bigdecimal-random.md](04-18-math-bigdecimal-random.md) | Math、BigDecimal 精度与舍入、BigInteger、随机数 |
| 19 | [04-19-system-runtime-process.md](04-19-system-and-runtime.md) | System、Runtime、ProcessBuilder、环境变量 |
| 20 | [04-20-regex.md](04-20-regex.md) | 正则语法、Pattern/Matcher、分组、常用校验 |
| 21 | [04-21-formatting-i18n.md](04-21-formatting-i18n.md) | printf、DecimalFormat、DateTimeFormatter、Locale |

### 五、集合框架
| 编号 | 文件 | 内容 |
|---|---|---|
| 22 | [05-22-collections-overview.md](05-22-collections-overview.md) | 集合体系、数组 vs 集合、选型速查 |
| 23 | [05-23-list.md](05-23-list.md) | ArrayList 扩容、LinkedList、CopyOnWriteArrayList |
| 24 | [05-24-map.md](05-24-map.md) | HashMap 红黑树与扩容、TreeMap、LinkedHashMap、ConcurrentHashMap |
| 25 | [05-25-set.md](05-25-set.md) | HashSet/TreeSet/LinkedHashSet、去重原理 |
| 26 | [05-26-queue-deque.md](05-26-queue-deque.md) | PriorityQueue、ArrayDeque、阻塞队列族 |
| 27 | [05-27-collections-utils-comparator.md](05-27-collections-utils-comparator.md) | Collections 工具、Comparable/Comparator、不可变集合 |
| 28 | [05-28-iterator-failfast.md](05-28-iterator-failfast.md) | 迭代器、fail-fast/fail-safe、遍历删除正确姿势 |

### 六、泛型
| 编号 | 文件 | 内容 |
|---|---|---|
| 29 | [06-29-generics-basics.md](06-29-generics-basics.md) | 泛型类/接口/方法、类型推断、泛型边界 |
| 30 | [06-30-generics-wildcards-erasure.md](06-30-generics-wildcards-erasure.md) | 通配符 PECS、类型擦除、桥方法、避坑 |

### 七、异常与日志
| 编号 | 文件 | 内容 |
|---|---|---|
| 31 | [07-31-exceptions.md](07-31-exceptions.md) | 异常体系、try-with-resources、自定义异常、最佳实践 |
| 32 | [07-32-logging-and-debugging.md](07-32-logging-and-debugging.md) | SLF4J/Logback/Log4j2、日志规范、远程调试 |

### 八、IO 与 NIO
| 编号 | 文件 | 内容 |
|---|---|---|
| 33 | [08-33-file-path-files.md](08-33-file-path-files.md) | File、Path、Files、目录遍历、临时文件 |
| 34 | [08-34-io-streams.md](08-34-io-streams.md) | 字节流/字符流、缓冲、转换流、对象流、装饰器体系 |
| 35 | [08-35-serialization.md](08-35-serialization.md) | 序列化机制、serialVersionUID、安全与替代方案 |
| 36 | [08-36-nio-buffer-channel-selector.md](08-36-nio-buffer-channel-selector.md) | Buffer 三要素、Channel、Selector、零拷贝 |
| 37 | [08-37-nio2-mmap-filelock-watchservice.md](08-37-nio2-mmap-filelock-watchservice.md) | 内存映射、文件锁、WatchService、异步通道 |

### 九、并发编程
| 编号 | 文件 | 内容 |
|---|---|---|
| 38 | [09-38-thread-basics.md](09-38-thread-basics.md) | 线程创建、生命周期、Thread API、线程通信 |
| 39 | [09-39-jmm-synchronized-volatile.md](09-39-jmm-synchronized-volatile.md) | JMM、happens-before、synchronized 原理、volatile |
| 40 | [09-40-locks-aqs.md](09-40-locks-aqs.md) | Lock 家族、AQS 原理、ReentrantLock、读写锁、StampedLock |
| 41 | [09-41-juc-collections-atomics.md](09-41-juc-collections-atomics.md) | 并发容器、CAS、原子类、LongAdder |
| 42 | [09-42-threadpool-executors.md](09-42-threadpool-executors.md) | 线程池原理、参数、拒绝策略、ForkJoin、虚拟线程池 |
| 43 | [09-43-completablefuture.md](09-43-completablefuture.md) | 异步编排、组合、异常处理、超时 |
| 44 | [09-44-sync-utils.md](09-44-sync-utils.md) | CountDownLatch、CyclicBarrier、Semaphore、Phaser、Exchanger |
| 45 | [09-45-virtual-threads.md](09-45-virtual-threads.md) | 虚拟线程原理、用法、局限、结构化并发 |
| 46 | [09-46-concurrency-pitfalls-patterns.md](09-46-concurrency-pitfalls-patterns.md) | 死锁、活锁、可见性 bug、生产者消费者、最佳实践 |

### 十、JVM
| 编号 | 文件 | 内容 |
|---|---|---|
| 47 | [10-47-jvm-runtime-memory.md](10-47-jvm-runtime-memory.md) | 运行时数据区、栈帧、对象布局、逃逸分析 |
| 48 | [10-48-garbage-collection.md](10-48-garbage-collection.md) | 可达性分析、分代、GC 算法、G1/ZGC |
| 49 | [10-49-class-loading.md](10-49-class-loading.md) | 类加载过程、双亲委派、自定义类加载器 |
| 50 | [10-50-jvm-tuning-tools.md](10-50-jvm-tuning-tools.md) | 常用参数、jps/jstat/jmap/jstack、Arthas、OOM 排查 |

### 十一、反射、注解与字节码
| 编号 | 文件 | 内容 |
|---|---|---|
| 51 | [11-51-reflection.md](11-51-reflection.md) | Class 对象、反射调用、性能、应用场景 |
| 52 | [11-52-annotations.md](11-52-annotations.md) | 元注解、自定义注解、运行时处理、APT |
| 53 | [11-53-dynamic-proxy.md](11-53-dynamic-proxy.md) | JDK 动态代理、CGLIB、AOP 原理 |
| 54 | [11-54-bytecode-classfile.md](11-54-bytecode-classfile.md) | class 文件结构、字节码指令、ASM/Javassist |

### 十二、函数式与新特性
| 编号 | 文件 | 内容 |
|---|---|---|
| 55 | [12-55-lambda-functional-interface.md](12-55-lambda-functional-interface.md) | Lambda 语法、方法引用、四大函数式接口 |
| 56 | [12-56-stream-api.md](12-56-stream-api.md) | Stream 全操作、惰性求值、并行流、Collectors |
| 57 | [12-57-optional.md](12-57-optional.md) | Optional 正确用法与反模式 |
| 58 | [12-58-version-features-8-to-21.md](12-58-version-features-8-to-21.md) | 8/9/10/11/12-17/18-21 特性全景 |
| 59 | [12-59-module-system.md](12-59-module-system.md) | JPMS 模块、module-info、jlink/jpackage |

### 十三、工程实践
| 编号 | 文件 | 内容 |
|---|---|---|
| 60 | [13-60-network-programming.md](13-60-network-programming.md) | Socket、NIO 网络、HTTP Client、RPC 基础 |
| 61 | [13-61-jdbc.md](13-61-jdbc.md) | JDBC 流程、事务、连接池、批处理 |
| 62 | [13-62-json-xml.md](13-62-json-xml.md) | Jackson/Gson、序列化配置、XML 处理 |
| 63 | [13-63-design-patterns.md](13-63-design-patterns.md) | 23 种设计模式 + Java 落地示例 |
| 64 | [13-64-junit5-mockito.md](13-64-junit5-mockito.md) | JUnit 5、Mockito、断言、测试规范 |
| 65 | [13-65-performance-best-practices.md](13-65-performance-best-practices.md) | 性能优化、代码规范、常见坑清单 |
| 66 | [13-66-interview-cheatsheet.md](13-66-interview-cheatsheet.md) | 高频面试题速查与答题要点 |

---

## 学习路径建议

1. **第 1 周**：01–08（环境 + 语法），保证能独立写出命令行程序。
2. **第 2 周**：09–15（OOP），重点吃透多态、equals/hashCode 契约。
3. **第 3 周**：16–21 + 22–28（核心 API + 集合），能手写 HashMap 简化版。
4. **第 4 周**：29–37（泛型 + 异常 + IO），理解类型擦除与装饰器模式。
5. **第 5–6 周**：38–46（并发），必须动手写线程池与生产者消费者。
6. **第 7 周**：47–54（JVM + 反射），会用工具排查 OOM/CPU 飙高。
7. **第 8 周**：55–66（函数式 + 工程实践），产出一个小型完整项目。

## 环境版本速查

| 版本 | 发布时间 | 关键特性 |
|---|---|---|
| Java 8 | 2014 | Lambda、Stream、Optional、新日期 API |
| Java 11 | 2018 | var、HttpClient、单文件运行、ZGC(实验) |
| Java 17 | 2021 | record、sealed、switch 模式匹配、文本块转正 |
| Java 21 | 2023 | 虚拟线程、结构化并发(预览)、switch 模式匹配转正 |
| Java 25 | 2025 | LTS，虚拟线程成熟、模式匹配增强 |

> 生产环境建议：新项目直接 Java 21 LTS；存量系统至少 Java 17。
