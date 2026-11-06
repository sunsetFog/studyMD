# 33 · JVM 调优与容器化

## 一、调优原则

```
1. 先测量后调优：无监控数据不动参数
2. 目标优先级：延迟敏感（接口 RT）→ 低停顿收集器；吞吐敏感（批处理）→ 大堆+并行
3. 最小干预：JVM 自适应机制（G1/ZGC）已覆盖 90% 场景
4. 每次只改一个变量
```

## 二、标准参数模板

### 通用服务（4C8G 容器）

```bash
java -jar app.jar \
  -Xms4g -Xmx4g \                          # 堆占容器内存 50-75%，Xms=Xmx 防抖
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/dump \
  -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=50m \
  -XX:ActiveProcessorCount=4               # 容器配额感知（或依赖自动探测）
```

### 延迟敏感（大堆 P99 要求 <10ms）

```bash
-XX:+UseZGC -XX:+ZGenerational            # JDK 21 分代 ZGC
-Xms16g -Xmx16g
```

### 批处理/吞吐优先

```bash
-XX:+UseParallelGC -Xms8g -Xmx8g
```

## 三、容器化 JVM 专项（必读）

```
Java 10+ 默认容器感知：自动读取 cgroup 的 CPU/内存配额
JVM 内存推导：MaxRAMPercentage（默认 25%）→ 生产显式设置
```

```bash
-XX:MaxRAMPercentage=70.0        # 堆占容器 limit 的 70%
# 而不是手写 -Xmx 与 limit 冲突 → OOMKilled

-XX:ActiveProcessorCount=4       # CPU limit 2 核但宿主 64 核时强制声明
                                 # （影响 GC 线程数/线程池默认大小）
```

**容器 OOMKilled 排查三步**：limit 与堆比例 → 元空间+线程栈+直接内存总账 → 是否有非堆泄漏。

## 四、分层调优清单

### 1. 堆大小

```
经验：堆 = 容器内存 × 60-70%（留元空间/栈/直接内存/OS）
过大堆：GC 一次成本高、故障恢复慢；过小：频繁 GC
依据：GC 日志中老年代峰值 × 安全系数(1.5)
```

### 2. 新生代比例

```
Minor GC 频繁但很快 → 正常（看晋升率）
晋升率高 → 增大新生代 / 排查中周期对象
大对象多 → 调大 G1 Region（-XX:G1HeapRegionSize=16m）
```

### 3. 线程与栈

```
-Xss 默认 1M：千级线程占 GB 级 → IO 密集可调 512k
线程数失控排查：unable to create native thread（35 篇）
```

## 五、JDK 21 虚拟线程时代的参数变化

```
传统：线程数 200 → -Xss 大栈内存被线程吃掉
新：虚拟线程（25 篇）→ 平台线程极少 → -Xss 占用骤降
   并发上限由下游资源控制（连接池/信号量）
→ 调优重心从「线程池」转移到「资源池」
```

## 六、启动速度优化（衔接 SpringBoot 36 篇）

```bash
-XX:TieredStopAtLevel=1          # 只用 C1，启动快但峰值低（开发/Serverless）
-Xshare:on -XX:SharedArchiveFile=app.jsa    # CDS 类加载缓存
-XX:MaxRAMPercentage=75.0
spring.main.lazy-initialization  # 应用层手段
```

## 七、调优案例模板

```
现象：接口 P99 抖动，GC 日志 Young GC 25ms 偶发 Full GC
分析：老年代斜率持续上升 → 排查发现本地缓存无上限（35 篇 dump 分析）
处置：缓存加上限与过期（Caffeine）→ Full GC 归零
教训：GC 抖动九成是代码问题，参数是最后手段
```

## 常见坑

- **坑 1**：Xmx > 容器 limit → OOMKilled 反复重启
- **坑 2**：Xms != Xmx → 启动/运行期堆伸缩抖动
- **坑 3**：调参不看 GC 日志 → 玄学调优
- **坑 4**：ActiveProcessorCount 未感知 → GC/GC 线程数与真实配额不匹配
- **坑 5**：把业务慢归咎 GC → 先做链路耗时分解（35 篇）

## Checklist

- [ ] 生产参数模板（G1/dump/GC 日志）标准化
- [ ] 容器感知参数（Percentage/ProcessorCount）就位
- [ ] GC 日志接入监控告警
- [ ] 调优遵循先测量后动参数
