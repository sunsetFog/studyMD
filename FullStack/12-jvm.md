# 12 - JVM 考点

## 1. 运行时内存区域（必考⭐⭐）

```
线程私有：程序计数器、虚拟机栈（栈帧：局部变量表/操作数栈/动态链接/返回地址）、本地方法栈
线程共享：堆（新生代 Eden+2*Survivor / 老年代）、方法区（JDK8+ 元空间 Metaspace，用本地内存）
直接内存：NIO Buffer
```

- 常见 OOM：堆（对象堆积）、元空间（动态生成类）、栈溢出 StackOverflowError（递归深）
- 对象分配流程：Eden → Survivor 交替 → 年龄阈值(默认15)晋升老年代；大对象直入老年代；动态年龄判断

## 2. GC 基础

- 判活：**可达性分析**（GC Roots：栈引用/静态变量/JNI/常量）；引用计数（循环引用问题，JVM 不用）
- 四种引用：强/软（内存不足回收，缓存）/弱（下次 GC 回收，ThreadLocalMap key）/虚
- 回收算法：标记-清除（碎片）、复制（新生代，浪费空间换无碎片）、标记-整理（老年代）
- 安全点 STW：GC 需要所有线程停在安全点

## 3. 垃圾收集器演进

| 收集器 | 特点 | 场景 |
|---|---|---|
| Parallel | 吞吐优先 | 批处理 |
| **CMS（已移除）** | 并发标记清除 | 历史考点：浮动垃圾/并发失败退化 |
| **G1（9+ 默认）** | Region 分区、可预测停顿（-XX:MaxGCPauseMillis） | 大堆通用 |
| **ZGC（21 生产）** | 染色指针、读屏障、停顿 <1ms，TB 级堆 | 低延迟 |

G1 考点：按 Region 回收价值最高的（Garbage First 得名）、Mixed GC、写屏障维护 remembered set。
ZGC 考点：并发整理的实现（染色指针标记对象状态）。

## 4. 类加载（必考⭐）

- 过程：加载 → 验证 → 准备（静态变量默认值）→ 解析 → 初始化（clinit，静态赋值）
- **双亲委派**：App → Platform(扩展) → Bootstrap；加载请求逐级上抛，父加载器无法加载才自己加载
  - 作用：安全（核心类不被篡改）+ 唯一性
  - 打破场景：SPI（Thread.contextClassLoader）、热部署（Tomcat 隔离类加载器）、OSGi
- 触发初始化：new/静态方法/反射/main 类等主动引用

## 5. JIT 与性能

- 解释执行 → C1（快速编译）→ C2（深度优化）
- 热点探测：方法调用计数器/回边计数器，触发阈值（默认 10000）
- 关键优化：方法内联（小方法快的原因）、逃逸分析（栈上分配/标量替换/锁消除）
- 兆结论：**热点代码要预热**（JMH 测试需 warmup）

## 6. 参数与排查（实战）

```bash
-Xms4g -Xmx4g              # 堆（初始=最大，避免动态抖动）
-Xmn / -XX:NewRatio        # 新生代
-XX:MetaspaceSize=256m
-XX:+UseG1GC -XX:MaxGCPauseMillis=200
-Xlog:gc*                  # 8+ GC 日志
```

排查命令：
```
jps / jstat -gcutil <pid> 1000      # GC 频率与各代占用
jmap -dump:live,format=b,file=heap.hprof <pid>
jstack <pid>                        # 线程栈（死锁/CPU 高）
arthas：dashboard / thread -n 3 / heapdump / watch
```

CPU 100% 四步：top 找进程 → top -Hp 找线程 → printf %x 转十六进制 → jstack 匹配 nid。

## 常见追问

1. 对象一定分配在堆上吗？——逃逸分析后可能栈上分配/标量替换
2. Full GC 触发条件：老年代满/元空间满/System.gc/空间分配担保失败
3. 为什么元空间取代永久代？——字符串池移走 + 本地内存上限可控 + 与 HotSpot 内部结构解耦

## Checklist

- [ ] 内存区域图能默画
- [ ] G1 的 Region 模型能讲
- [ ] 双亲委派 + 打破场景是必答题
