# 24 · Spring Boot 常见坑汇总

> 按「现象 → 根因 → 解决」组织，覆盖启动、配置、Bean、Web、数据、事务、性能七大类。

## 一、启动类

| 现象 | 根因 | 解决 |
|---|---|---|
| `NoSuchBeanDefinitionException` | 启动类不在根包，子包没扫到 | 启动类移到根包或 `scanBasePackages` |
| `Port 8080 was already in use` | 端口占用 | 改端口 / `lsof -i:8080` 杀进程 / `server.port=0` 随机 |
| `Failed to configure a DataSource` | 引入了 JPA/MyBatis 但没配数据源 | 配数据源或排除 `DataSourceAutoConfiguration` |
| `ClassNotFoundException: javax.servlet.*` | Boot 3 迁移 | 全局 `javax.*` → `jakarta.*` |
| `Unsupported class file major version` | JDK 版本不匹配 | Boot 3 需 Java 17+ |
| 启动卡住无日志 | 配置里连接超时（如 Redis 不可达） | 检查外部依赖、设置合理超时 |
| `Consider defining a bean` | 缺 `@Service`/`@MapperScan` | 补注解或扫描 |
| Bean 循环依赖 | 构造器互相依赖 | 见 [04](04-dependency-injection.md) |

## 二、配置类

| 现象 | 根因 | 解决 |
|---|---|---|
| 配置改了不生效 | 命令行/环境变量优先级更高 | 查 `/actuator/env` |
| `@Value` 报 `Could not resolve placeholder` | 无默认值且配置缺失 | `:default` |
| yml 解析失败 | Tab 缩进 / 冒号后无空格 | 用空格缩进 |
| `@ConfigurationProperties` 绑定为 null | prefix 错、无 setter、未 `@EnableConfigurationProperties` | 检查三项 |
| 本机好使服务器不行 | 环境变量覆盖 | 打印 `/actuator/env` 对比 |
| Profile 没生效 | 激活方式不对 | 命令行 > 环境变量 > 配置文件 |

## 三、Bean 与注入

| 现象 | 根因 | 解决 |
|---|---|---|
| 字段注入 NPE（单测） | 手动 `new` 对象 | 构造器注入 |
| `NoUniqueBeanDefinitionException` | 多实现 | `@Qualifier` / `@Primary` |
| 注入实现类失败 | JDK 代理类型不匹配 | 注入接口，或保持 CGLIB 默认 |
| `@Autowired` static 为 null | Spring 不注入静态字段 | setter 赋值或 `@PostConstruct` |
| `A component required a bean of type X` | 缺依赖 / 未扫描 | 补 Bean |
| `@Bean` 方法间调用不生效 | 误以为走容器 | 配置类需完整模式（默认代理），非 `@Configuration` 的 `@Component` 类里 `@Bean` 调用是普通方法 |

## 四、Web 类

| 现象 | 根因 | 解决 |
|---|---|---|
| 404 | 路径/context-path/未扫描 | 检查 `@RequestMapping` 与 `server.servlet.context-path` |
| 400 | 参数类型/日期格式/缺参 | 加 `@DateTimeFormat`、默认值 |
| 415 | 未带 `Content-Type: application/json` | 前端加请求头 |
| CORS 报错 | 未配置或 Security 拦截预检 | `addCorsMappings` + `http.cors()` |
| 中文乱码 | 编码 | `server.servlet.encoding.charset=UTF-8` |
| 上传超限 | 默认 1MB | `spring.servlet.multipart.max-file-size` |
| 日期差 8 小时 | 时区 | `spring.jackson.time-zone=Asia/Shanghai` + MySQL `serverTimezone` |
| 全局异常不生效 | 未扫描到 `@RestControllerAdvice` | 检查包路径 |
| Filter 里异常不走 Advice | Filter 早于 DispatcherServlet | Filter 内自行写响应 |

## 五、数据访问

| 现象 | 根因 | 解决 |
|---|---|---|
| `Invalid bound statement` | XML namespace/路径错 | 核对 `mapper-locations` |
| N+1 查询 | 懒加载循环查 | `join fetch` / `@EntityGraph` / 批量查 |
| `LazyInitializationException` | Session 已关闭 | 事务内访问 / `join fetch`；关 `open-in-view` |
| 慢查询 | 无索引 / 深分页 | 加索引 / 游标分页 |
| 连接池耗尽 | 大事务 / 慢 SQL | 拆事务、加超时、调池大小 |
| `Too many connections` | 连接泄漏或池过大 | 用连接池、检查未关闭资源 |
| 批量插入慢 | 单条提交 | `rewriteBatchedStatements=true` + 批量 |
| 深分页慢 | `LIMIT 100000,20` | 记录上次最大 ID |
| MyBatis 一级缓存脏读 | SqlSession 复用 | 理解生命周期，必要时 `flushCache` |

## 六、事务与 AOP

| 现象 | 根因 | 解决 |
|---|---|---|
| 事务不回滚 | 吞异常 / 检查异常 / 非 public / 自调用 | 见 [18](18-boot-transaction.md) |
| AOP 不生效 | 同类调用 / 非 public / 切点不对 | 见 [19](19-boot-aop.md) |
| `@Async` 不生效 | 未 `@EnableAsync` / 同类调用 / 非 public | 开启 + 拆类 |
| `@Cacheable` 不生效 | 同类调用 / key 冲突 | 走代理 |
| `@Scheduled` 不执行 | 未 `@EnableScheduling` | 加注解 |
| `@Transactional` + `@Async` 同方法 | 事务失效 | 拆分 |

## 七、性能与资源

| 现象 | 根因 | 解决 |
|---|---|---|
| 启动慢 | 自动配置过多 / 扫描范围大 | `spring.main.lazy-initialization`、精确 `@ComponentScan`、AOT |
| 内存占用高 | 默认堆按物理内存 1/4 | `-Xms/-Xmx`、容器感知（Boot 已默认支持） |
| CPU 飙高 | 死循环 / 频繁 Full GC / 正则回溯 | 抓 `threaddump`、`heapdump` 分析 |
| 线程数暴涨 | 每次 new 线程池 | 池化、统一管理 |
| 日志打满磁盘 | 无滚动策略 | `maxHistory` + `totalSizeCap` |
| fat jar 很大 | 依赖多 | 瘦身（只打包变动模块）/ 分层镜像 |

## 八、其他高频

- **`devtools` 导致类型不同**：devtools 用 `RestartClassLoader` 加载你的类，某些 `instanceof`/序列化场景出现 `ClassCastException`。解决：关闭 devtools，或在 `spring-devtools.properties` 中把相关 jar 加入 `restart.include`。
- **`jackson` 反序列化失败**：无默认构造器、字段无 setter、使用了 Lombok 但注解处理器未开。解决：用 `record`/加 `@JsonCreator`，或配置 `ParameterNamesModule`（Boot 默认已注册）。
- **`BeanCurrentlyInCreationException`**：循环依赖。
- **`MethodArgumentNotValidException` 返回 500**：未做全局处理。
- **Redis 序列化乱码**：默认 `JdkSerializationRedisSerializer`，改 Jackson/String 序列化（见 [26](26-spring-data-redis.md)）。
- **时间字段精度丢失**：`LocalDateTime` 与 MySQL `datetime`（秒级）不一致，用 `datetime(3)`。

## 九、排查清单（速用）

```
1 看启动日志：APPLICATION FAILED TO START 段落会给出建议
2 看 /actuator/conditions：为什么自动配置没生效
3 看 /actuator/beans：Bean 到底注没注册
4 看 /actuator/env：配置最终值是多少
5 开 --debug：打印自动配置报告
6 加 logging.level.org.springframework=DEBUG
7 线程栈：jstack / /actuator/threaddump
8 内存：jmap + MAT / /actuator/heapdump
9 抓包/慢日志：MySQL slow query log、Arthas trace
```

## 十、面试速答

- **遇到 Boot 启动失败怎么排查**：先看 `APPLICATION FAILED TO START` 后的 `Action:` 提示；再看 `/actuator/conditions` 判断自动配置匹配；最后看 `/actuator/beans`、`/actuator/env`。
- **最常被问的三个坑**：事务失效（自调用/吞异常）、循环依赖（构造器注入）、配置优先级（命令行覆盖）。
