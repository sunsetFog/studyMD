# Spring Boot 知识点总结

## 1. 简介

Spring Boot 是基于 Spring 的快速开发框架，通过「约定优于配置」和自动装配，大幅简化 Spring 应用的搭建和配置。

特点：
- 内嵌 Tomcat/Jetty 等容器，可独立运行
- 自动装配，减少 XML/Java 配置
- 提供 starter 依赖，简化 Maven/Gradle 配置
- 内置监控、健康检查等功能

## 2. 启动类

Spring Boot 应用入口，`@SpringBootApplication` 是核心注解。

```
// 启动类，放在根包下，以便扫描所有子包
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        // 启动 Spring Boot 应用
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

`@SpringBootApplication` 等价于 `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`。

## 3. 配置文件

Spring Boot 默认读取 `application.yml` 或 `application.properties`。

```
# application.yml
server:
  port: 8080                 # 服务端口
  servlet:
    context-path: /demo      # 上下文路径

spring:
  application:
    name: demo-service       # 应用名
  datasource:                # 数据源
    url: jdbc:mysql://localhost:3306/demo
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
```

## 4. 多环境配置

通过 profile 区分开发、测试、生产环境。

```
# application.yml（主配置，指定激活环境）
spring:
  profiles:
    active: dev              # 激活 dev 环境

# application-dev.yml（开发环境）
server:
  port: 8080

# application-prod.yml（生产环境）
server:
  port: 80
```

启动时可命令行覆盖：`java -jar app.jar --spring.profiles.active=prod`。

## 5. 依赖注入 IOC / DI

Spring 容器管理对象的创建和依赖关系，通过注解注入。

```
// 方式一：构造器注入（推荐）
@RestController
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;   // 构造器注入，final 保证不可变
    }
}

// 方式二：字段注入（简单，但不推荐用于测试）
@Autowired
private UserService userService;

// 方式三：setter 注入
@Autowired
public void setUserService(UserService userService) {
    this.userService = userService;
}
```

## 6. Bean 注解

用于把类交给 Spring 管理。

```
@Component               // 通用组件
@Service                 // 业务层
@Repository              // 数据访问层
@Controller              // 控制层（MVC）
@RestController          // 控制层（返回 JSON）
```

```
@Service
public class UserService {
    public String getName() {
        return "tom";
    }
}
```

## 7. @Configuration 与 @Bean

用 Java 类替代 XML 配置，`@Bean` 方法返回的对象交给 Spring 管理。

```
@Configuration
public class AppConfig {

    // 返回的对象会被注册为 Bean，方法名即 Bean 名
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    // @Bean 可指定名称和初始化/销毁方法
    @Bean(name = "customBean", initMethod = "init", destroyMethod = "destroy")
    public CustomBean customBean() {
        return new CustomBean();
    }
}
```

## 8. 属性注入

将配置文件中的值注入到字段或类。

```
# application.yml
app:
  name: demo
  version: 1.0
  owner: tom
```

```
// 方式一：@Value 单个注入
@Component
public class AppConfig {
    @Value("${app.name}")
    private String name;

    @Value("${app.owner:默认值}")   // 可指定默认值
    private String owner;
}

// 方式二：@ConfigurationProperties 批量注入
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private String version;
    private String owner;
    // 省略 getter/setter
}
```

## 9. RESTful 接口

`@RestController` + 请求映射注解返回 JSON。

```
@RestController
@RequestMapping("/users")
public class UserController {

    // GET /users         查询列表
    @GetMapping
    public List<User> list() {
        return userService.list();
    }

    // GET /users/1       查询单个
    @GetMapping("/{id}")
    public User getById(@PathVariable Long id) {
        return userService.getById(id);
    }

    // POST /users        新增
    @PostMapping
    public User create(@RequestBody User user) {
        return userService.create(user);
    }

    // PUT /users/1       更新
    @PutMapping("/{id}")
    public User update(@PathVariable Long id, @RequestBody User user) {
        user.setId(id);
        return userService.update(user);
    }

    // DELETE /users/1    删除
    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

## 10. 请求参数绑定

```
@RestController
@RequestMapping("/api")
public class ParamController {

    // 查询参数 ?name=tom&age=18
    @GetMapping("/hello")
    public String hello(@RequestParam String name,
                        @RequestParam(defaultValue = "0") int age) {
        return name + ":" + age;
    }

    // 路径参数 /api/user/1
    @GetMapping("/user/{id}")
    public String path(@PathVariable("id") Long id) {
        return "id=" + id;
    }

    // 对象参数自动绑定 ?name=tom&age=18
    @GetMapping("/obj")
    public String obj(UserQuery query) {
        return query.getName();
    }

    // JSON 请求体
    @PostMapping("/body")
    public User body(@RequestBody User user) {
        return user;
    }
}
```

## 11. 参数校验

使用 Validation 注解校验参数，配合 `@Valid` 生效。

```
@Data
public class User {
    @NotBlank(message = "姓名不能为空")
    private String name;

    @NotNull(message = "年龄不能为空")
    @Min(value = 1, message = "年龄最小为 1")
    @Max(value = 150, message = "年龄最大为 150")
    private Integer age;

    @Email(message = "邮箱格式不正确")
    private String email;
}

@RestController
public class UserController {
    @PostMapping("/user")
    public String create(@Valid @RequestBody User user) {
        // 校验不通过会抛 MethodArgumentNotValidException
        return "success";
    }
}
```

## 12. 统一响应封装

用统一结构返回数据，便于前端处理。

```
@Data
public class Result<T> {
    private int code;       // 状态码
    private String message; // 提示信息
    private T data;         // 数据

    public static <T> Result<T> success(T data) {
        Result<T> r = new Result<>();
        r.setCode(200);
        r.setMessage("成功");
        r.setData(data);
        return r;
    }

    public static <T> Result<T> error(int code, String message) {
        Result<T> r = new Result<>();
        r.setCode(code);
        r.setMessage(message);
        return r;
    }
}

@RestController
public class UserController {
    @GetMapping("/user/{id}")
    public Result<User> get(@PathVariable Long id) {
        return Result.success(userService.getById(id));
    }
}
```

## 13. 全局异常处理

`@ControllerAdvice` + `@ExceptionHandler` 统一捕获异常。

```
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 捕获业务异常
    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusiness(BusinessException e) {
        return Result.error(e.getCode(), e.getMessage());
    }

    // 捕获参数校验异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldError().getDefaultMessage();
        return Result.error(400, msg);
    }

    // 捕获所有异常
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.error(500, "系统异常");
    }
}
```

## 14. AOP 切面

面向切面编程，在不改动原代码的情况下增强功能（日志、权限、事务）。

```
@Aspect
@Component
public class LogAspect {

    // 切入点：匹配 controller 包下所有方法
    @Pointcut("execution(* com.demo.controller.*.*(..))")
    public void controllerPointcut() {
    }

    // 前置通知：方法执行前
    @Before("controllerPointcut()")
    public void before(JoinPoint joinPoint) {
        System.out.println("方法开始：" + joinPoint.getSignature().getName());
    }

    // 后置通知：方法执行后
    @After("controllerPointcut()")
    public void after() {
        System.out.println("方法结束");
    }

    // 环绕通知：可控制方法执行前后逻辑
    @Around("controllerPointcut()")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();          // 执行目标方法
        long cost = System.currentTimeMillis() - start;
        System.out.println("耗时：" + cost + "ms");
        return result;
    }
}
```

## 15. 拦截器 Interceptor

拦截请求，在进入 Controller 前后处理（登录校验、参数处理）。

```
@Component
public class AuthInterceptor implements HandlerInterceptor {

    // 进入 Controller 前
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String token = request.getHeader("token");
        if (token == null || token.isEmpty()) {
            throw new BusinessException("未登录");
        }
        return true;   // 返回 true 放行，false 拦截
    }

    // 方法执行后、视图渲染前
    @Override
    public void postHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler,
                           ModelAndView modelAndView) throws Exception {
    }
}

// 注册拦截器
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/**")        // 拦截所有
                .excludePathPatterns("/login"); // 排除登录
    }
}
```

## 16. 过滤器 Filter

比拦截器更底层，在进入 Servlet 之前处理（编码、跨域）。

```
@Component
public class CorsFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletResponse resp = (HttpServletResponse) response;
        resp.setHeader("Access-Control-Allow-Origin", "*");
        resp.setHeader("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE");
        chain.doFilter(request, response);   // 放行到下一个过滤器
    }
}
```

## 17. 数据访问 MyBatis-Plus

MyBatis-Plus 是 MyBatis 的增强工具，内置通用 CRUD。

```
<!-- pom.xml 依赖 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
</dependency>
```

```
// 实体类
@Data
@TableName("user")            // 指定表名
public class User {
    @TableId(type = IdType.AUTO)   // 主键自增
    private Long id;
    private String name;
    private Integer age;
}

// Mapper 接口，继承 BaseMapper 获得通用 CRUD
@Mapper
public interface UserMapper extends BaseMapper<User> {
    // 自定义 SQL
    @Select("SELECT * FROM user WHERE age > #{age}")
    List<User> selectAgeGreater(@Param("age") int age);
}

// Service 层
@Service
public class UserService {
    @Autowired
    private UserMapper userMapper;

    public List<User> list() {
        return userMapper.selectList(null);   // 查询所有
    }
}
```

## 18. 事务管理

`@Transactional` 声明事务，方法内异常自动回滚。

```
@Service
public class OrderService {

    @Transactional(rollbackFor = Exception.class)   // 任何异常都回滚
    public void createOrder(Order order) {
        orderMapper.insert(order);
        // 扣库存，若抛异常则整个方法回滚
        stockMapper.decrease(order.getProductId());
    }

    // 只读事务
    @Transactional(readOnly = true)
    public Order getById(Long id) {
        return orderMapper.selectById(id);
    }
}
```

注意：`@Transactional` 默认只回滚 RuntimeException，需加 `rollbackFor = Exception.class`。

## 19. Redis 集成

```
<!-- pom.xml 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: 123456
      database: 0
```

```
@Service
public class CacheService {
    @Autowired
    private StringRedisTemplate redisTemplate;

    public void set(String key, String value, long timeout) {
        redisTemplate.opsForValue().set(key, value, timeout, TimeUnit.SECONDS);
    }

    public String get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    public void delete(String key) {
        redisTemplate.delete(key);
    }

    // 分布式锁
    public boolean tryLock(String key, String value, long timeout) {
        Boolean ok = redisTemplate.opsForValue()
                .setIfAbsent(key, value, timeout, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(ok);
    }
}
```

## 20. Spring Cache 缓存注解

通过注解声明式使用缓存。

```
@Service
public class UserService {

    // 查询时优先走缓存，命中则直接返回
    @Cacheable(value = "user", key = "#id")
    public User getById(Long id) {
        return userMapper.selectById(id);
    }

    // 更新时同步更新缓存
    @CachePut(value = "user", key = "#user.id")
    public User update(User user) {
        userMapper.updateById(user);
        return user;
    }

    // 删除时清缓存
    @CacheEvict(value = "user", key = "#id")
    public void delete(Long id) {
        userMapper.deleteById(id);
    }
}
```

## 21. 定时任务

`@Scheduled` 实现定时执行，需在启动类开启 `@EnableScheduling`。

```
@Component
@EnableScheduling
public class ScheduledTask {

    // 固定频率：上次开始后 5 秒执行
    @Scheduled(fixedRate = 5000)
    public void task1() {
        System.out.println("每 5 秒执行");
    }

    // 固定延迟：上次结束后 5 秒执行
    @Scheduled(fixedDelay = 5000)
    public void task2() {
    }

    // Cron 表达式：每天 2 点执行
    @Scheduled(cron = "0 0 2 * * ?")
    public void task3() {
    }
}
```

## 22. 异步任务

`@Async` 异步执行，需在启动类开启 `@EnableAsync`。

```
@Service
public class AsyncService {

    @Async
    public void sendEmail(String to) {
        // 异步执行，不阻塞主线程
        System.out.println("发送邮件给 " + to);
    }

    // 有返回值的异步任务
    @Async
    public CompletableFuture<String> doTask() {
        return CompletableFuture.completedFuture("done");
    }
}

// 启动类加 @EnableAsync
@EnableAsync
@SpringBootApplication
public class DemoApplication {
}
```

## 23. 消息队列 RabbitMQ

```
<!-- pom.xml 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```
# application.yml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
```

```
// 生产者
@Service
public class MessageProducer {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void send(String message) {
        rabbitTemplate.convertAndSend("demo.exchange", "demo.routing", message);
    }
}

// 消费者
@Component
public class MessageConsumer {

    @RabbitListener(queues = "demo.queue")
    public void receive(String message) {
        System.out.println("收到消息：" + message);
    }
}
```

## 24. 文件上传下载

```
@RestController
@RequestMapping("/file")
public class FileController {

    // 上传
    @PostMapping("/upload")
    public String upload(@RequestParam("file") MultipartFile file) throws IOException {
        String filename = file.getOriginalFilename();   // 原始文件名
        long size = file.getSize();                     // 文件大小
        file.transferTo(new File("/tmp/" + filename));  // 保存到磁盘
        return "上传成功：" + filename;
    }

    // 下载
    @GetMapping("/download")
    public ResponseEntity<Resource> download() {
        File file = new File("/tmp/test.txt");
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION,
                        "attachment; filename=\"" + file.getName() + "\"")
                .body(new FileSystemResource(file));
    }
}
```

## 25. 接口文档 Knife4j / Swagger

```
<!-- pom.xml 依赖 -->
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-openapi3-jakarta-spring-boot-starter</artifactId>
    <version>4.4.0</version>
</dependency>
```

```
@RestController
@Tag(name = "用户管理", description = "用户相关接口")
@RequestMapping("/user")
public class UserController {

    @Operation(summary = "根据 ID 查询用户")
    @GetMapping("/{id}")
    public Result<User> getById(@Parameter(description = "用户 ID") @PathVariable Long id) {
        return Result.success(userService.getById(id));
    }
}
```

启动后访问 `http://localhost:8080/doc.html` 查看接口文档。

## 26. 日志

Spring Boot 默认使用 SLF4J + Logback。

```
# application.yml
logging:
  level:
    root: info                  # 全局日志级别
    com.demo.mapper: debug      # 指定包级别
  file:
    name: logs/app.log          # 输出到文件
```

```
// 使用 Lombok @Slf4j 或手动获取 logger
@Slf4j
@RestController
public class DemoController {
    @GetMapping("/log")
    public String log() {
        log.info("info 日志");
        log.debug("debug 日志");
        log.warn("warn 日志");
        log.error("error 日志");
        return "ok";
    }
}
```

## 27. 单元测试

```
<!-- pom.xml 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```
@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;

    @Test
    void testGetById() {
        User user = userService.getById(1L);
        Assertions.assertNotNull(user);
    }

    // 模拟依赖
    @Test
    void testWithMock() {
        UserMapper mapper = Mockito.mock(UserMapper.class);
        Mockito.when(mapper.selectById(1L)).thenReturn(new User());
        // 断言
    }
}
```

## 28. 打包部署

```
# 打包成可执行 jar
mvn clean package

# 运行
java -jar demo-0.0.1.jar

# 指定端口和配置
java -jar demo-0.0.1.jar --server.port=9090

# 后台运行
nohup java -jar demo-0.0.1.jar > app.log 2>&1 &
```

## 29. Actuator 监控

提供应用健康检查、指标等信息。

```
<!-- pom.xml 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics   # 暴露端点
```

访问地址：
- `/actuator/health`：健康检查
- `/actuator/info`：应用信息
- `/actuator/metrics`：指标数据

## 30. 配置类与条件装配

`@ConditionalOnXxx` 根据条件决定是否装配 Bean。

```
@Configuration
public class ConditionalConfig {

    // 当容器中不存在该 Bean 时才创建
    @Bean
    @ConditionalOnMissingBean(RestTemplate.class)
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    // 当配置项存在时才创建
    @Bean
    @ConditionalOnProperty(name = "app.enabled", havingValue = "true")
    public MyService myService() {
        return new MyService();
    }
}
```

## 31. 读取配置的优先级

Spring Boot 配置优先级（从高到低）：
1. 命令行参数
2. 环境变量
3. `application-{profile}.yml`
4. `application.yml`
5. `bootstrap.yml`（需引入 spring-cloud-context）

## 32. 常用 starter 依赖

| starter | 用途 |
|---------|------|
| spring-boot-starter-web | Web 开发 |
| spring-boot-starter-data-redis | Redis |
| spring-boot-starter-data-jpa | JPA |
| spring-boot-starter-jdbc | JDBC |
| spring-boot-starter-amqp | RabbitMQ |
| spring-boot-starter-security | 安全 |
| spring-boot-starter-test | 测试 |
| spring-boot-starter-actuator | 监控 |
| spring-boot-starter-validation | 参数校验 |
