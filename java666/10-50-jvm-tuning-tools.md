# 50 · JVM 调优与诊断工具

## 1. 常用参数总览

### 内存

```bash
-Xms4g -Xmx4g                      # 堆初始/最大（建议相等）
-Xmn2g                             # 新生代
-Xss512k                           # 线程栈
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
-XX:MaxDirectMemorySize=512m
-XX:SurvivorRatio=8
-XX:MaxTenuringThreshold=15
```

### GC

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
-XX:+ParallelRefProcEnabled
-XX:+UseStringDeduplication         # G1 字符串去重
```

### 日志与诊断

```bash
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=50m
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump
-XX:ErrorFile=/data/logs/hs_err_pid%p.log
-XX:+ShowCodeDetailsInExceptionMessages    # NPE 详细信息（15+ 默认）
```

### 容器环境

```bash
-XX:+UseContainerSupport            # 默认开启（JDK 10+）
-XX:MaxRAMPercentage=75.0           # 按容器内存百分比设置堆
-XX:ActiveProcessorCount=4          # 覆盖 CPU 核数
```

## 2. 基础工具

### jps —— 进程列表

```bash
jps -l                # 全限定主类名
jps -v                # JVM 参数
jps -m                # main 参数
```

### jinfo —— 查看/修改参数

```bash
jinfo -flags <pid>
jinfo -flag MaxHeapSize <pid>
jinfo -flag +PrintGCDetails <pid>      # 动态开启（部分参数支持）
```

### jstat —— 运行时统计

```bash
jstat -gc <pid> 1000 10          # 每 1s 输出 10 次
jstat -gcutil <pid> 1000         # 百分比形式
jstat -gcmetacapacity <pid>
jstat -class <pid>
jstat -compiler <pid>
```

`-gcutil` 输出列：

| 列 | 含义 |
|---|---|
| S0/S1 | Survivor 使用率 |
| E | Eden 使用率 |
| O | 老年代使用率 |
| M | Metaspace 使用率 |
| CCS | 压缩类空间 |
| YGC/YGCT | Young GC 次数/耗时 |
| FGC/FGCT | Full GC 次数/耗时 |
| GCT | 总 GC 耗时 |

判断：

- `O` 持续接近 100% 且 FGC 频繁 → 内存不足或泄漏。
- `YGCT` 增长快 → 新生代太小或分配率高。

### jmap —— 堆分析

```bash
jmap -heap <pid>                     # 堆配置与使用（JDK 9+ 部分废弃）
jmap -histo <pid>                    # 对象直方图
jmap -histo:live <pid>               # 触发 Full GC 后统计（谨慎）
jmap -dump:live,format=b,file=heap.hprof <pid>
jmap -clstats <pid>                  # 类加载器统计
jmap -finalizerinfo <pid>
```

替代（更安全）：

```bash
jcmd <pid> GC.heap_dump /tmp/heap.hprof
jcmd <pid> GC.class_histogram
jcmd <pid> GC.heap_info
```

### jstack —— 线程栈

```bash
jstack <pid> > stack.txt
jstack -l <pid>          # 附加锁信息
jstack -F <pid>          # 强制（进程无响应时）
```

分析：

```bash
grep -c "java.lang.Thread.State" stack.txt          # 线程总数
grep "java.lang.Thread.State: BLOCKED" stack.txt
grep -A 30 "Found one Java-level deadlock" stack.txt
```

### jcmd —— 万能命令

```bash
jcmd -l
jcmd <pid> help
jcmd <pid> VM.flags
jcmd <pid> VM.system_properties
jcmd <pid> Thread.print
jcmd <pid> GC.heap_info
jcmd <pid> GC.run
jcmd <pid> VM.native_memory summary          # 需 -XX:NativeMemoryTracking=summary
jcmd <pid> JFR.start duration=60s filename=/tmp/rec.jfr
```

### jhsdb / jdb

底层调试（一般不用）。

## 3. 可视化工具

| 工具 | 用途 |
|---|---|
| **JConsole** | JMX 监控（内存、线程、MBean） |
| **VisualVM** | 综合（CPU/内存/线程/采样） |
| **Java Flight Recorder (JFR)** | 低开销事件记录（生产可用） |
| **JMC** | JFR 分析 |
| **Eclipse MAT** | 堆转储分析 |
| **JProfiler / YourKit** | 商业，功能强 |
| **Arthas** | 在线诊断（阿里开源） |

### JFR（推荐生产使用）

```bash
-XX:StartFlightRecording=duration=120s,filename=/tmp/app.jfr,settings=profile
```

开销 < 1%，可长期开启。分析 CPU 热点、GC、锁竞争、IO、异常。

## 4. Arthas 常用命令

```bash
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```

| 命令 | 用途 |
|---|---|
| `dashboard` | 总览（线程/内存/GC） |
| `thread` | 线程列表 |
| `thread -n 5` | CPU 最高的 5 个线程 |
| `thread -b` | 找出阻塞其他线程的线程 |
| `thread <id>` | 指定线程栈 |
| `jvm` | JVM 信息 |
| `memory` | 内存信息 |
| `sysprop` / `sysenv` | 系统属性/环境变量 |
| `sc -d <class>` | 查看类信息 |
| `sm <class> <method>` | 查看方法 |
| `jad <class>` | 反编译 |
| `watch <class> <method> '{params,returnObj}'` | 观察方法出入参 |
| `trace <class> <method>` | 方法内部调用耗时 |
| `monitor -c 5 <class> <method>` | 统计调用次数/RT |
| `stack <class> <method>` | 查看调用栈 |
| `tt` | 记录调用（TimeTunnel） |
| `ognl` | 执行表达式 |
| `heapdump` | 导出堆 |
| `profiler start/stop` | 生成火焰图 |

示例：

```bash
# 找出耗时 > 100ms 的调用
trace com.acme.OrderService create '#cost > 100'

# 观察异常
watch com.acme.OrderService create '{params, throwExp}' -e -x 2

# 统计 QPS 与 RT
monitor -c 5 com.acme.OrderController query

# 火焰图
profiler start
# ... 运行 30s
profiler stop --format html
```

## 5. CPU 100% 排查流程

```bash
# 1. 找进程
top

# 2. 找线程（-H 显示线程）
top -Hp <pid>

# 3. 线程 ID 转十六进制
printf "%x\n" <tid>

# 4. 导出线程栈并搜索
jstack <pid> > /tmp/stack.txt
grep -A 30 "nid=0x<hex>" /tmp/stack.txt
```

常见原因：

- 死循环（业务代码 bug）。
- 频繁 GC（GC 线程占 CPU）。
- 正则回溯（ReDoS）。
- 大量线程上下文切换（线程数过多）。
- 自旋锁竞争。

## 6. 内存问题排查

### 堆 OOM

```bash
# 1. 自动 dump（推荐预先配置）
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump

# 2. 手动 dump
jcmd <pid> GC.heap_dump /data/dump/heap.hprof

# 3. MAT 分析
#    - Leak Suspects 报告
#    - Dominator Tree（支配树）找最大对象
#    - Path to GC Roots（排除弱引用）找为什么没被回收
```

### 内存泄漏特征

- Full GC 后老年代占用仍持续上升。
- 对象数量持续增长（`jmap -histo` 对比多次）。
- 常见泄漏点：静态集合、缓存、ThreadLocal、监听器、连接未关闭。

### Metaspace OOM

```bash
jcmd <pid> VM.metaspace
jmap -clstats <pid>
```

原因：动态生成类（CGLIB、Groovy、JSP、热部署）、类加载器泄漏。

### 直接内存 OOM

```bash
-XX:NativeMemoryTracking=detail
jcmd <pid> VM.native_memory detail
```

## 7. 线程问题排查

```bash
# 线程数统计
jstack <pid> | grep "java.lang.Thread.State" | sort | uniq -c

# 找阻塞
jstack <pid> | grep -B 2 -A 15 "BLOCKED"

# 找等待同一把锁的线程
jstack <pid> | grep "waiting to lock"
```

线程数暴涨原因：

- 线程池创建过多。
- `new Thread` 无节制。
- 阻塞队列消费者过多。
- Tomcat 线程池配置过大。

## 8. GC 调优实战

### 步骤

```bash
# 1. 开启 GC 日志
-Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=10,filesize=50m

# 2. 观察（jstat 或日志）
jstat -gcutil <pid> 1000 60

# 3. 识别问题
#    - Young GC 频率 > 10 次/分钟？
#    - 单次 Young GC > 50ms？
#    - Full GC 次数 > 0？
#    - 老年代使用率持续上升？

# 4. 调整
#    - 增大堆 / 新生代
#    - 换收集器
#    - 减少对象分配（代码优化）
#    - 修复内存泄漏
```

### 案例：频繁 Young GC

现象：YGC 每秒数次，Eden 每次分配满。

原因：请求量大 + 每次创建大量临时对象。

优化：

1. 减少临时对象（复用、StringBuilder、避免装箱）。
2. 增大新生代（`-Xmn` 或 `-XX:G1NewSizePercent`）。
3. 提高晋升阈值（避免过早晋升）。

### 案例：Full GC 频繁

原因：老年代不足或泄漏。

优化：

1. MAT 分析 dump，修复泄漏（根本）。
2. 增大堆。
3. 调整 G1 的 IHOP（提前并发标记）。
4. 避免大对象（缓存分片）。

## 9. 容器化注意事项

```bash
# 旧版本（JDK 8u191 之前）不感知 cgroup 限制，会用宿主机内存算堆
-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap

# 新版本
-XX:MaxRAMPercentage=75.0     # 推荐（75% 给堆，剩余给元空间/栈/直接内存）
-XX:InitialRAMPercentage=75.0
-XX:+UseContainerSupport
```

CPU：

```bash
-XX:ActiveProcessorCount=4     # 覆盖核数（影响 GC 线程、ForkJoinPool 并行度）
```

> K8s 中 limit 与 request 不一致时，JVM 按 limit 计算。

## 10. 生产参数模板

```bash
# 8C16G，延迟敏感服务
java -server \
  -Xms10g -Xmx10g \
  -Xss512k \
  -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
  -XX:MaxDirectMemorySize=1g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=100 \
  -XX:G1HeapRegionSize=8m \
  -XX:InitiatingHeapOccupancyPercent=40 \
  -XX:+ParallelRefProcEnabled \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/data/dump \
  -Xlog:gc*:file=/data/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=50m \
  -jar app.jar
```

## 11. 调优原则

1. **不要为了调优而调优**：先测量，再优化。
2. **优先优化代码**（减少对象分配、避免大对象、修复泄漏），其次调参数。
3. **一次只改一个参数**，对比效果。
4. **压测环境要接近生产**（数据量、并发、硬件）。
5. **关注 P99 而非平均值**。
6. **保留 GC 日志**，便于事后分析。
7. **不要随意 `System.gc()`**。
8. 堆不是越大越好（> 32GB 压缩指针失效）。

## 12. 速查表

| 问题 | 命令 |
|---|---|
| 找进程 | `jps -l` |
| GC 情况 | `jstat -gcutil <pid> 1000` |
| 对象分布 | `jmap -histo <pid>` |
| 堆快照 | `jcmd <pid> GC.heap_dump path` |
| 线程栈 | `jstack <pid>` |
| CPU 高 | `top -Hp` + `printf %x` + `jstack` |
| 死锁 | `jstack` grep deadlock |
| 在线诊断 | Arthas `dashboard/thread/trace/watch` |
| 性能剖析 | JFR / `profiler start` |
| 内存细节 | `jcmd VM.native_memory` |
