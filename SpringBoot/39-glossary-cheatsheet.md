# 39 · 术语表与速查卡

## 一、核心术语表

| 术语 | 一句话解释 |
|---|---|
| IoC | 对象的创建与依赖装配交给容器管理 |
| DI | 依赖注入，IoC 的实现手段 |
| Bean | 被 Spring 容器管理的对象实例 |
| BeanDefinition | Bean 的"配方"（类名/作用域/依赖描述） |
| BeanFactory | IoC 容器底层接口 |
| ApplicationContext | 功能增强的容器（事件/国际化/资源） |
| BeanPostProcessor | Bean 前后处理钩子（AOP/@Autowired 靠它） |
| BeanFactoryPostProcessor | BeanDefinition 级处理（自动配置绑定靠它） |
| starter | 按场景聚合的依赖包 + 自动配置 |
| AutoConfiguration | 条件化的默认配置类 |
| @ConditionalOnXxx | 装配条件开关 |
| DispatcherServlet | MVC 前端控制器 |
| HandlerMethod | 被映射到的 Controller 方法 |
| ArgumentResolver | 参数解析器（@RequestBody 的幕后） |
| ReturnValueHandler | 返回值处理器（@ResponseBody 的幕后） |
| Interceptor | MVC 层拦截器 |
| Filter | Servlet 容器层过滤器 |
| AOP | 横切关注点的模块化（切面） |
| Advice | 通知（Around/Before/After...） |
| Pointcut | 切点表达式 |
| 事务传播 | 事务方法间的加入/新建语义 |
| Savepoint | NESTED 事务的部分回滚点 |
| 雪花 ID | 时间戳+机器+序列的分布式 ID |
| 乐观锁 | version 校验式并发控制 |
| Cache-Aside | 先库后缓存的旁路缓存模式 |
| 布隆过滤器 | 高效"存在性"判断，防缓存穿透 |
| 死信队列 | 消费失败消息的隔离区 |
| 手动 ack | 消费者确认机制，防消息丢失 |
| 幂等 | 重复执行结果不变 |
| JWT | 签名的自包含令牌（无状态认证） |
| RBAC | 用户-角色-权限模型 |
| Actuator | 生产监控端点 |
| Micrometer | 指标门面（对接 Prometheus） |
| HikariCP | 默认连接池 |
| Flyway | 数据库迁移版本管理 |
| Testcontainers | Docker 化的集成测试 |
| MDC | 日志上下文（traceId 载体） |
| GraalVM Native | AOT 编译的原生镜像 |
| CRaC | JVM 进程快照快速恢复 |

## 二、常用命令速查

```bash
# 构建/运行
mvn clean package -DskipTests
java -jar app.jar --spring.profiles.active=prod

# 依赖分析
mvn dependency:tree | grep log4j
mvn dependency:analyze     # 未使用/缺失依赖

# 测试
mvn test
mvn verify                 # 含覆盖率门槛

# 排查
jstack <pid> > dump.txt          # 线程
jmap -dump:live,format=b,file=heap.hprof <pid>   # 堆
jstat -gc <pid> 1000             # GC 实时
arthas: thread -b / dashboard / watch
```

## 三、配置速查卡

```yaml
server:
  port: 8080
  shutdown: graceful
  tomcat:
    threads: { max: 200, min-spare: 10 }
    max-connections: 8192
spring:
  datasource:
    url: jdbc:mysql://host:3306/db?serverTimezone=Asia/Shanghai
    hikari:
      maximum-pool-size: 20
      max-lifetime: 1800000
  data:
    redis: { host: localhost, port: 6379 }
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    default-property-inclusion: non_null
  task:
    scheduling: { pool: { size: 5 } }
  servlet:
    multipart: { max-file-size: 10MB }
mybatis-plus:
  configuration: { map-underscore-to-camel-case: true }
  global-config:
    db-config: { id-type: assign_id, logic-delete-field: deleted }
management:
  endpoints.web.exposure.include: health,prometheus
  endpoint.health.probes.enabled: true
logging:
  level: { com.example.demo: debug, org.springframework: warn }
```

## 四、注解速查

### Web
`@RestController` `@RequestMapping` `@GetMapping/@PostMapping/@PutMapping/@DeleteMapping`
`@RequestParam` `@PathVariable` `@RequestBody` `@RequestHeader` `@Valid/@Validated`

### 配置
`@Configuration` `@Bean` `@Value` `@ConfigurationProperties` `@Profile` `@ConditionalOnXxx`

### 容器
`@Component/@Service/@Repository` `@Autowired/@Resource/@Qualifier/@Primary`
`@Lazy` `@Scope` `@DependsOn` `@PostConstruct` `@PreDestroy`

### AOP 与事务
`@Aspect` `@Pointcut` `@Around/@Before/@AfterReturning/@AfterThrowing`
`@Transactional(rollbackFor, propagation, readOnly, isolation)`

### 缓存与异步
`@EnableCaching` `@Cacheable` `@CachePut` `@CacheEvict`
`@EnableAsync` `@Async` `@EnableScheduling` `@Scheduled(cron)`

### 安全
`@EnableWebSecurity` `@EnableMethodSecurity` `@PreAuthorize` `@AuthenticationPrincipal`

### 事件与消息
`@EventListener` `@TransactionalEventListener` `@RabbitListener` `@KafkaListener`

## 五、优先级/顺序速记

```
配置加载：命令行 > 环境变量 > 外置文件 > jar 内文件 > 默认值
拦截链：  Filter → DispatcherServlet → Interceptor → AOP → Controller
Bean：    实例化 → 注入 → @PostConstruct → (AOP代理) → 使用 → @PreDestroy
异常：    业务 warn / 系统 error(带堆栈) / 兜底不泄密
停机：    SIGTERM → 拒新 → 处理完(30s) → @PreDestroy
```

## 六、数字速记

| 项 | 经验值 |
|---|---|
| Hikari 连接池 | 10-20（核数×2+磁盘数） |
| Tomcat 线程 | 200 默认；IO 密集 200-500 |
| 自建线程池 | CPU 密集=核+1，IO 密集=2×核 |
| JWT accessToken | 15-30 分钟 |
| 缓存 TTL | 30 分钟基线 + 5% 随机抖动 |
| 批量插入批大小 | 500-1000 |
| BCrypt 强度 | 10-12 |
| MQ prefetch | 10-50 |

## 七、速查：X 出问题 → 看哪篇

| 症状 | 翻 |
|---|---|
| 接口 401/403 | 20/21 |
| 响应字段全 null（映射） | 10 |
| 事务没回滚 | 14 |
| 定时任务重复执行 | 19 |
| 循环依赖启动失败 | 15/36 |
| 缓存读到旧数据 | 13/24 |
| 消息丢了 | 25/26 |
| 启动慢 | 36 |
| 高峰期线程池打满 | 37 |
| Bean 没装配 | 35 |
