# 35 · 线上问题排查手册

## 一、排查总方法论

```
1. 现象定性：慢 / 挂 / 错 / 泄漏
2. 证据收集：监控 → GC 日志 → 线程 dump → 堆 dump → 应用日志
3. 由外到内：网关 → 容器资源 → JVM → 代码
4. 一次只验证一个假设
```

**工具箱分层**：
```
JDK 自带：jps / jstack / jmap / jstat / jcmd
可视化：Arthas（在线诊断神器）/ async-profiler（火焰图）/ MAT（堆分析）
```

## 二、CPU 100% 排查（标准四步）

```bash
# 1. 定位进程
top                      # 找到 java 进程 PID

# 2. 定位线程
top -Hp <pid>            # 线程级 CPU，记下高耗线程号（如 12345）

# 3. 转十六进制查栈
printf '%x\n' 12345      # → 3039
jstack <pid> | grep -A 20 '0x3039'
# → 看到具体业务栈：死循环 / 正则回溯 / 大量序列化

# 4. 火焰图看全貌
async-profiler: ./profiler.sh -d 30 -f cpu.html <pid>
```

**CPU 高三大常见**：死循环（for 条件错）、正则灾难回溯（`(.*)*`）、频繁 GC（先看 jstat）。

## 三、内存泄漏排查

```bash
# 1. 先判断是否真泄漏：jstat -gc <pid> 5000
#    老年代 after-FG 回收不掉 + 持续增长 → 泄漏嫌疑

# 2. 抓堆快照（主动）
jmap -dump:live,format=b,file=heap.hprof <pid>

# 3. MAT 分析
#    - Dominator Tree：谁占内存最多
#    - Leak Suspects：泄漏嫌疑报告
#    - 查 GC Roots 引用链：为什么没被回收

# 4. 常见泄漏源：
- 静态集合缓存无上限（Map 只加不减）
- ThreadLocal 未 remove（线程池复用）
- 连接/流未关
- 内部类隐式持有外部（监听器注册未注销）
- 类加载器泄漏（热部署）
```

**Arthas 快速定位**：

```bash
heapdump /tmp/heap.hprof          # dump
vmtool --action getInstances --className java.util.HashMap --limit 5
```

## 四、Full GC 频繁

```
排查链：
1. GC 日志：Full GC 原因（Promotion failed / Metadata / System.gc()）
2. 老年代增长斜率：平稳锯齿 vs 单向上移
3. dump 分析晋升对象类型
对策：
- 晋升压力 → 新生代加大 / 大对象优化
- 缓存无上限 → Caffeine 限容
- System.gc() → 移除
- 元空间 → 动态类排查
```

## 五、接口超时/抖动

```
1. traceId 拉全链路日志，定位慢在哪层（外部 IO / GC / 锁 / CPU）
2. GC 暂停：jstat -gcutil 看暂停占比
3. 锁竞争：jstack 多次抓取，看 BLOCKED 线程
4. 线程池打满：jstack 看 idle 线程是否耗尽 + 队列积压
5. Arthas trace / watch 实时观测：
   trace com.demo.OrderService create '#cost > 100ms'    # 方法内耗时分解
   watch com.demo.XxxService doWork '{params, returnObj}' -x 2
```

## 六、线程问题

```bash
jstack <pid> > t1.txt; sleep 5; jstack <pid> > t2.txt   # 抓两次对比

关键模式：
- 大量 BLOCKED（同一把锁）        → 锁竞争热点
- Found one Java-level deadlock   → 死锁（jstack 自动检测）
- 大量 WAITING (parking)          → 线程池任务堆积/连接等待
- TIMED_WAITING sleep             → 正常业务等待
```

## 七、OOM 应急与预防

```bash
# 预防（必须配置）
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dump
-XX:OnOutOfMemoryError="kill -15 %p"        # OOM 后触发动作（通知/重启）

# 应急预案
1. dump 自动生成 → MAT 事后分析（勿现场分析拖垮服务）
2. 立即重启恢复服务（dump 已落盘）
3. 事后复盘 → 泄漏修复 → 压测回归
```

## 八、Arthas 常用命令速查

```bash
dashboard                    # 总览：线程/内存/GC
thread -n 3                  # CPU 最高 3 线程
thread -b                    # 找死锁
sc -d com.demo.User          # 类信息（查依赖冲突：谁加载的）
watch / trace / tt           # 方法观测/耗时/时间隧道回放
profiler start/stop          # 火焰图
```

## 九、排查速查表

| 症状 | 首查 | 深查 |
|---|---|---|
| CPU 100% | top -Hp + jstack | 火焰图 |
| 内存持续涨 | jstat -gcutil | MAT dump |
| 接口偶发超时 | GC 日志 + traceId 日志 | Arthas trace |
| 服务假死（无响应） | jstack（锁/线程池） | 线程 dump 对比 |
| OOM 重启 | dump + MAT | Leak Suspects |
| 启动后不久挂 | 启动日志 + OOM 报错类型 | 对应 30 篇 OOM 分类 |

## Checklist

- [ ] CPU 四步法熟到肌肉记忆
- [ ] dump 参数永远预先配置
- [ ] Arthas 安全上线（非生产高频使用）
- [ ] 每次事故有复盘文档沉淀模式
