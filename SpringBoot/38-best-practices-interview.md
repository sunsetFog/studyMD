# 38 · 最佳实践清单 + 高频面试题

## 一、最佳实践 Checklist

### 架构与分层
- [ ] Controller 不写业务；Service 不碰 Servlet API；Entity 不出 Service
- [ ] 统一 Result<T> 响应体 + GlobalExceptionHandler（07 篇）
- [ ] 错误码枚举集中管理
- [ ] 分层依赖单向：controller → service → mapper

### 配置与环境
- [ ] @ConfigurationProperties 优先于散点 @Value
- [ ] 生产敏感配置走环境变量/配置中心，不进 Git
- [ ] jar 环境无关，差异外置

### 数据与事务
- [ ] @Transactional(rollbackFor = Exception.class)，无自调用
- [ ] 事务内无 RPC/IO，大事务拆小
- [ ] 批量操作用 batch 能力
- [ ] 逻辑删除 + 唯一索引冲突方案明确
- [ ] Flyway 管理库表变更

### 并发与资源
- [ ] 线程池手动创建，拒绝策略明确（19 篇）
- [ ] @Async/@Scheduled 多实例防重
- [ ] Hikari 连接池与超时配置齐全
- [ ] 缓存三问题（穿透/雪崩/击穿）有对策

### 安全
- [ ] BCrypt 存密码，JWT 短过期 + refresh
- [ ] ThreadLocal 用户上下文 afterCompletion 清理
- [ ] Actuator 不裸奔公网
- [ ] 文件上传白名单 + 重命名

### 工程与运维
- [ ] 全局 traceId 贯穿日志
- [ ] 单测覆盖核心 Service，Testcontainers 集成测试
- [ ] 优雅停机 + K8s 探针
- [ ] Prometheus 指标 + 错误率告警
- [ ] Docker 非 root + 多阶段构建

---

## 二、高频面试题 30 道

### 基础与自动配置（1-8）

**1. @SpringBootApplication 包含哪些注解？**
`@SpringBootConfiguration`（配置类）+ `@EnableAutoConfiguration`（自动配置入口）+ `@ComponentScan`（包扫描）。追问：自动配置如何触发——@Import AutoConfigurationImportSelector 读 AutoConfiguration.imports 文件 → 条件注解过滤。

**2. 自动配置原理？**（见 35 篇）
读候选配置类 → @ConditionalOnClass 等条件过滤 → @ConditionalOnMissingBean 保证用户优先。加分项：说出条件注解评估顺序与 debug 报告。

**3. Bean 生命周期？**
实例化 → 属性填充 → Aware → BPP before → 初始化（@PostConstruct）→ BPP after（AOP 代理生成）→ 使用 → 销毁（@PreDestroy）。追问：@Autowired 在 @PostConstruct 前完成。

**4. Bean 作用域有哪些？单例如何保证线程安全？**
singleton/prototype/request/session。单例无状态（可变成员变量放方法栈/ThreadLocal）。

**5. 循环依赖如何解决？三级缓存是什么？**
一级成品、二级半成品早期引用、三级 ObjectFactory（生成代理引用）。构造器注入/prototype 无法解决。追问：为什么需要三级不能二级——AOP 代理要在需要时才生成，工厂延迟决定。

**6. IOC 和 AOP 的理解？**
IoC：控制反转，对象创建装配交给容器，DI 是实现；AOP：横切关注点（事务/日志）与业务解耦，Spring AOP 运行时代理（JDK/CGLIB）。

**7. @Autowired 和 @Resource 区别？**
@Autowired Spring 按类型（配 @Qualifier 按名）；@Resource JSR-250 先名后类型。

**8. SpringBoot 2→3 主要变化？**
javax→jakarta、Java 17 起步、Spring Security API 重写、spring.factories→AutoConfiguration.imports。

### Web 与事务（9-16）

**9. 事务传播机制？REQUIRED 与 REQUIRES_NEW 区别？**
REQUIRED 加入当前事务；REQUIRES_NEW 挂起新开独立事务（日志记录场景：主事务回滚日志保留）。追问：嵌套 REQUIRES_NEW 有连接池耗尽风险。

**10. @Transactional 失效场景？（必考）**
自调用、非 public、异常被 catch、checked 异常默认不回滚、类未受管、MyISAM、多线程。能举出修复方案是加分项（14 篇）。

**11. #{} 和 ${} 区别？**
#{} 预编译占位符防注入；${} 字符串拼接，仅白名单动态列名使用。

**12. MyBatis 一级二级缓存？**
一级 SqlSession 级默认开；二级 namespace 级生产慎用（关联表脏读/分布式失效），用 Redis 替代。

**13. 乐观锁和悲观锁？如何实现？**
乐观：version 字段 + WHERE version=旧值（MP @Version）；悲观：SELECT FOR UPDATE。低冲突乐观高冲突悲观。

**14. 分布式 ID 方案？**
雪花算法（MP ASSIGN_ID）：时间戳+机器 ID+序列，趋势递增，注意前端 Long 精度转 String。

**15. Redis 持久化与过期策略？**
RDB 快照 + AOF 日志；惰性删除 + 定期抽样删除；内存满按 maxmemory-policy 淘汰。

**16. 缓存穿透/雪崩/击穿及对策？**
穿透：缓存空值/布隆过滤器；雪崩：TTL 抖动/多级缓存；击穿：互斥锁重建/逻辑过期。

### 进阶与中间件（17-24）

**17. Spring AOP 实现原理？JDK 动态代理 vs CGLIB？**
有接口 JDK 代理（反射实现接口），无接口 CGLIB 子类字节码。Boot 3 默认 CGLIB。追问：内部调用绕过代理导致事务/切面失效。

**18. Spring 事件机制？**
publisher.publishEvent → @EventListener。默认同步在事务内；@TransactionalEventListener(AFTER_COMMIT) 事务后执行；@EnableAsync 异步。

**19. 线程池核心参数与拒绝策略？**
core/max/queue/keepAlive/rejectedHandler。CallerRuns 让调用线程执行防丢任务；Abort 抛异常走降级。CPU 密集=核+1，IO 密集=2×核。

**20. 消息丢失如何防止？（MQ 三环节）**
生产者 confirm+本地消息表、broker 持久化、消费者手动 ack。追问：重复消费靠幂等（唯一索引/去重表/状态机）。

**21. JWT 认证流程与优缺点？**
登录发签名 token → 请求带 Bearer → 服务端验签。优点无状态水平扩展；缺点无法主动作废（黑名单补救）、payload 可解码。

**22. Spring Security 过滤器链如何工作？**
FilterChainProxy 按配置链匹配请求 → SecurityContextHolder 存认证信息 → 方法级 @PreAuthorize 授权。

**23. 如何排查线上 CPU 100%？**
top -Hp 定位线程 → jstack 转十六进制 nid 定位代码 → 火焰图看热点 → 常见：死循环/正则回溯/频繁 GC（先看 GC 日志）。

**24. OOM 排查思路？**
heapdump（-XX:+HeapDumpOnOutOfMemoryError）→ MAT 分析支配树 → 大对象来源（大查询不分页/缓存无上限/ThreadLocal 泄漏）。

### 架构与综合（25-30）

**25. SpringBoot 启动流程概述？**
run() → Environment 准备 → 容器创建 → refresh（BFPP 解析配置类/自动配置 → 注册 BPP → onRefresh 创建内嵌容器 → 实例化单例）→ 内嵌 Tomcat start → Runners → ReadyEvent。

**26. 接口幂等性如何设计？**
唯一索引/防重表、分布式锁、状态机校验、前端幂等 token。按「提交类接口必须幂等」执行。

**27. 接口性能优化思路？**
链路分析定位瓶颈（APM/日志耗时）：SQL 索引与慢查询 → 缓存 → 异步化非关键路径 → 并行调用 → 批量合并 → 分页与数据裁剪。先测量后优化。

**28. 如何保证接口的最终一致性？**
本地消息表 + MQ 重试 + 对账补偿；或事务消息；强一致才用分布式事务（Seata AT/TCC），能不用就不用。

**29. 线上接口偶发超时，怎么排查？**
traceId 拉全链路日志 → 分层耗时（网关/应用/DB/下游）→ GC 与线程 dump → 连接池 pending → 下游 RT 抖动。加分：压测复现。

**30. 如果让你设计一个秒杀系统，谈整体方案？**
分层削峰：前端限流防刷 → 网关限流 → Redis 预扣库存（Lua 原子）→ MQ 异步落单 → DB 唯一索引兜底 → 库存对账。追问预热、防超卖、热点 key 分片。

---

## 三、答题方法论

1. **先定义再展开**：概念 → 原理 → 场景 → 陷阱
2. **带数字**：连接池 10-20、线程 200、TTL 抖动 5%——体现实战
3. **主动关联**：答事务失效顺带讲代理原理，展示体系化
4. **承认边界**：没用过的中间件直说，讲思路比硬编强
