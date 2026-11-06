# 67 · Spring 性能调优

## 1. 调优顺序（重要）

```
① 定位瓶颈（监控/压测/剖析） → ② 架构层（缓存/异步/削峰）
③ 应用层（SQL/算法/池）     → ④ JVM（GC/内存）→ ⑤ 系统/容器
```

> **不要一上来调 JVM 参数**。90% 的性能问题在 SQL、缓存、IO 与不合理设计。

## 2. 定位工具

| 工具 | 用途 |
|---|---|
| Arthas | 线上诊断：`dashboard`、`trace`、`watch`、`thread -b`（死锁）、`jad`、`heapdump` |
| JProfiler / YourKit | 本地/远程剖析（CPU、内存、线程） |
| async-profiler | 低开销火焰图 |
| JMH | 微基准测试（避免错误 benchmark） |
| JMeter / Gatling / wrk | 压测 |
| Prometheus + Grafana | 指标 |
| SkyWalking / Zipkin | 链路 |
| MySQL slow log / `EXPLAIN` | SQL |

```bash
# Arthas 常用
trace com.example.OrderService create '#cost > 100'    # 抓耗时 >100ms 的调用
watch com.example.OrderService create '{params, returnObj}' -x 3
thread -n 5                                            # 最忙的 5 个线程
thread -b                                              # 找死锁
```

## 3. 应用层优化

### ① SQL（最常见瓶颈）

| 措施 | 说明 |
|---|---|
| 加索引 | 按 ESR（等值→排序→范围）；避免冗余索引 |
| 避免 `SELECT *` | 只取需要的列，利于覆盖索引 |
| 深分页 | 用游标（`id > lastId`） |
| 批量 | `batchInsert`、`IN` 分批（1000 以内） |
| 避免隐式转换 | 字段类型与参数一致 |
| 避免函数/前导通配符 | 会导致索引失效 |
| 减少事务粒度 | 事务内不放远程调用 |
| 慢查询日志 | 定期分析 Top SQL |

### ② 缓存

- 热点数据放 Redis；极热点放本地缓存（Caffeine）。
- 缓存 key 设计、TTL 随机化防雪崩。
- 避免缓存大对象与频繁序列化。
- 防穿透/击穿/雪崩（见 [27](27-redis-patterns.md)）。

### ③ 连接池与线程池

| 项 | 建议 |
|---|---|
| HikariCP | 10~30，`pending > 0` 才扩容 |
| Tomcat | `max-threads 200~500`，`accept-count 100` |
| 业务线程池 | 有界队列、合理拒绝策略 |
| 连接超时 | 2~5s，快速失败 |

### ④ 序列化

- 使用 Jackson 单例 `ObjectMapper`（线程安全），不要每次 new。
- 响应体不要返回巨大 JSON（分页、字段裁剪）。
- 开启 Gzip（`server.compression.enabled=true`，>1KB）。
- 内部服务可考虑 Protobuf。

### ⑤ Web 层

```yaml
server:
  tomcat:
    threads: {max: 300, min-spare: 30}
    max-connections: 10000
    accept-count: 100
    connection-timeout: 20000
    keep-alive-timeout: 60000
  compression: {enabled: true, min-response-size: 1024}
spring:
  mvc:
    async:
      request-timeout: 30s
```

- 大文件上传走对象存储直传。
- 长耗时接口异步化（返回任务 ID + 轮询/推送）。

## 4. JVM 调优

### 内存设置

```
-Xms=Xmx（避免动态扩容抖动）
-XX:MaxRAMPercentage=70（容器内）
-XX:MaxMetaspaceSize=256m
-Xss512k（线程多时）
-XX:MaxDirectMemorySize（Netty/NIO）
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
-XX:+ExitOnOutOfMemoryError（容器编排可自动重启）
```

### GC 选择

| 场景 | 建议 |
|---|---|
| 默认 | **G1**（JDK 9+ 默认），`MaxGCPauseMillis=200` |
| 大堆低延迟 | ZGC（JDK 17+，亚毫秒停顿） |
| 高吞吐 | Parallel GC |
| 小堆/微服务 | SerialGC 也可（资源受限） |

```
# G1 常用
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
-XX:+UseStringDeduplication     # 字符串去重（可省内存）
```

### 判断依据

- Full GC 频繁、单次停顿 > 1s、老年代持续上涨 → 内存泄漏或堆太小。
- GC 总时间占比 > 10% → 需要调。
- `jstat -gcutil pid 1000` 观察。

## 5. 启动优化

| 手段 | 效果 |
|---|---|
| `spring.main.lazy-initialization=true` | 启动快，但首请求慢（生产慎用） |
| 精确 `@ComponentScan` | 减少扫描 |
| 减少自动配置（`exclude`） | 减少 |
| **CDS**（AppCDS） | 共享类数据，启动 -20% |
| **AOT + Native Image** | 启动毫秒级，内存 -50% |
| CRaC（JDK 17+，预览） | 检查点恢复 |
| 减少 Bean 数量 | 直接有效 |

```bash
# AppCDS
java -XX:ArchiveClassesAtExit=app.jsa -jar app.jar
java -XX:SharedArchiveFile=app.jsa -jar app.jar
```

## 6. 常见反模式

| 反模式 | 后果 | 改进 |
|---|---|---|
| 循环里查 DB / 调接口 | N+1 | 批量查询、Map 内存匹配 |
| 事务里调远程 | 长事务、连接占满 | 移到事务外 |
| 大对象入缓存 | 序列化开销大 | 裁剪 |
| 无限队列线程池 | OOM | 设界 |
| 每次 new ObjectMapper | CPU 浪费 | 单例 |
| 日志打印过大对象 | IO 与 CPU | 截断/降级 |
| 同步等待所有依赖 | 延迟累加 | 并行（CompletableFuture） |
| 热点 key 集中 | 单分片打爆 | 拆分/本地缓存 |

## 7. 压测与容量规划

```
目标 QPS = 峰值流量 / 单机能力
单机能力 = min(DB 连接池/(RT), Tomcat 线程/(RT), CPU 上限)
```
- 压测要贴近真实（数据分布、接口比例、缓存命中率）。
- 关注 **P99** 而非平均。
- 逐步加压，观察拐点（吞吐不再增长而延迟飙升）。
- 留 30%~50% 余量。

## 8. 检查清单

- [ ] 慢 SQL 是否有索引、是否走索引（EXPLAIN）
- [ ] 是否有 N+1 查询
- [ ] 事务是否过大
- [ ] 连接池是否够（`pending` 指标）
- [ ] 线程池队列是否有界
- [ ] 缓存命中率多少
- [ ] GC 次数与停顿
- [ ] 线程数是否异常
- [ ] 是否有死锁/阻塞（`thread -b`）
- [ ] 外部依赖耗时（Feign/MQ/Redis）
- [ ] 接口是否可异步/批量

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 盲目调大堆 | GC 停顿更长 | 先定位内存使用 |
| 连接池越大越好 | 反而更慢 | 合理值 |
| 只看平均耗时 | 忽略长尾 | 看 P95/P99 |
| 压测数据太干净 | 结论失真 | 造真实分布数据 |
| 缓存预热缺失 | 启动即雪崩 | 预热 |
| 忽略网络 | 跨机房延迟 | 就近部署 |
| 日志拖慢 | DEBUG 打全量 | 生产关 DEBUG |

## 10. 面试速答

- **调优步骤**：先监控定位瓶颈（指标/链路/剖析），再按 架构 → 应用 → JVM → 系统 顺序优化，每次只改一处并压测验证。
- **常见瓶颈**：慢 SQL/N+1、缓存缺失、连接池不足、线程池无界、事务过大、序列化开销、GC。
- **G1 与 ZGC**：G1 平衡吞吐与停顿（默认）；ZGC 停顿亚毫秒，适合大堆与超低延迟。
- **容器内 JVM**：`UseContainerSupport` + `MaxRAMPercentage`，不要用固定 `-Xmx`。
- **如何排查 CPU 飙高**：`top -Hp` 找线程 → `jstack` 转 16 进制匹配 nid → 或用 Arthas `thread -n 5` / async-profiler 火焰图。
