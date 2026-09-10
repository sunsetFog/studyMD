# Spring Boot 深度学习清单

> 一份从入门到精通的 Spring Boot 系统学习指南，涵盖核心原理、生态整合、微服务架构、性能调优等全方位内容。

---

## 📋 目录

- [一、Spring 基础回顾](#一spring-基础回顾)
- [二、Spring Boot 入门](#二spring-boot-入门)
- [三、核心原理深度解析](#三核心原理深度解析)
- [四、Web 开发全栈](#四web-开发全栈)
- [五、数据访问与持久化](#五数据访问与持久化)
- [六、缓存系统](#六缓存系统)
- [七、消息队列](#七消息队列)
- [八、安全与认证授权](#八安全与认证授权)
- [九、任务调度与异步处理](#九任务调度与异步处理)
- [十、日志与监控](#十日志与监控)
- [十一、配置管理与多环境](#十一配置管理与多环境)
- [十二、测试体系](#十二测试体系)
- [十三、微服务架构](#十三微服务架构)
- [十四、云原生与容器化](#十四云原生与容器化)
- [十五、性能调优](#十五性能调优)
- [十六、常见实战场景](#十六常见实战场景)
- [十七、设计模式在 Spring Boot 中的实践](#十七设计模式在-spring-boot-中的实践)
- [十八、常见陷阱与解决方案](#十八常见陷阱与解决方案)
- [十九、源码阅读指南](#十九源码阅读指南)
- [二十、持续进阶路线图](#二十持续进阶路线图)

---

## 一、Spring 基础回顾

### 1.1 IoC 容器
- [ ] 理解 IoC（控制反转）与 DI（依赖注入）的核心思想
- [ ] BeanFactory 与 ApplicationContext 的区别与层级关系
- [ ] Bean 的生命周期（实例化 → 属性赋值 → 初始化 → 销毁）
  ```
  实例化 Instantiation
    ↓
  属性赋值 Populate Properties
    ↓
  BeanNameAware.setBeanName()
    ↓
  BeanFactoryAware.setBeanFactory()
    ↓
  ApplicationContextAware.setApplicationContext()
    ↓
  BeanPostProcessor.postProcessBeforeInitialization()
    ↓
  InitializingBean.afterPropertiesSet()
    ↓
  自定义 init-method
    ↓
  BeanPostProcessor.postProcessAfterInitialization()
    ↓
  Bean 就绪
    ↓
  DisposableBean.destroy()
    ↓
  自定义 destroy-method
  ```
- [ ] Bean 作用域：singleton、prototype、request、session、application、websocket
- [ ] Bean 的装配方式：XML、注解（@Component/@Service/@Repository/@Controller）、JavaConfig（@Configuration/@Bean）
- [ ] @Autowired vs @Resource vs @Inject 的区别
- [ ] @Primary 与 @Qualifier 的使用场景
- [ ] @Scope 与 @Lazy 的实际应用时机

### 1.2 AOP 面向切面编程
- [ ] AOP 核心概念：切面（Aspect）、连接点（JoinPoint）、通知（Advice）、切入点（Pointcut）、织入（Weaving）
- [ ] 通知类型详解
  - @Before：前置通知
  - @After：后置通知（无论是否异常都执行）
  - @AfterReturning：返回通知
  - @AfterThrowing：异常通知
  - @Around：环绕通知（最强大，手动控制执行）
- [ ] 切入点表达式语法
  ```java
  execution(* com.example.service.*.*(..))  // 最常用
  within(com.example.service.*)              // 按类匹配
  @annotation(com.example.anno.Log)          // 按注解匹配
  args(java.lang.String)                     // 按参数类型匹配
  ```
- [ ] JDK 动态代理 vs CGLIB 代理的底层区别
- [ ] 理解代理失效的场景（同类方法调用、final 方法等）

### 1.3 Spring 事件机制
- [ ] ApplicationEvent 与 ApplicationListener
- [ ] @EventListener 注解使用
- [ ] @TransactionalEventListener（事务事件监听）
- [ ] 异步事件处理
- [ ] 自定义事件发布与消费

### 1.4 SpEL（Spring 表达式语言）
- [ ] 基本语法与运算符
- [ ] @Value 中的 SpEL 使用
- [ ] 集合过滤与投影
- [ ] 方法调用与类型操作
- [ ] 安全导航操作符（?.）

---

## 二、Spring Boot 入门

### 2.1 第一个 Spring Boot 项目
- [ ] 使用 Spring Initializr 创建项目
  - 在线：https://start.spring.io
  - IDEA 内置创建器
  - CLI：`spring init`
- [ ] 理解 pom.xml 核心依赖
  ```xml
  <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
  </parent>
  ```
- [ ] 理解 starter 机制
- [ ] @SpringBootApplication 注解拆解
  - @SpringBootConfiguration（≈@Configuration）
  - @EnableAutoConfiguration（自动配置核心）
  - @ComponentScan（组件扫描）
- [ ] main 方法启动流程
  ```java
  SpringApplication.run(Application.class, args);
  ```

### 2.2 项目结构规范
- [ ] 推荐的项目包结构
  ```
  com.example.demo
  ├── config          // 配置类
  ├── controller      // 控制器
  ├── service         // 业务接口
  │   └── impl        // 业务实现
  ├── repository      // 数据访问层（DAO）
  ├── model           // 数据模型
  │   ├── entity      // 实体类
  │   ├── dto         // 数据传输对象
  │   └── vo          // 视图对象
  ├── enums           // 枚举类
  ├── exception       // 异常定义
  ├── interceptor     // 拦截器
  ├── util            // 工具类
  └── constant        // 常量
  ```
- [ ] 分层职责明确：Controller → Service → Repository
- [ ] 数据传输对象的使用规范（DTO 与 Entity 分离）

### 2.3 配置文件
- [ ] application.properties vs application.yml 的选择
- [ ] YAML 语法（缩进、列表、多文档）
- [ ] 核心配置项速查
  ```yaml
  # 服务器
  server.port: 8080
  server.servlet.context-path: /api
  # 数据源
  spring.datasource.url: jdbc:mysql://localhost:3306/db
  spring.datasource.username: root
  spring.datasource.password: ${DB_PASSWORD}
  # 日志
  logging.level.root: INFO
  logging.level.com.example: DEBUG
  logging.file.name: app.log
  ```
- [ ] 随机值生成
  ```properties
  app.secret=${random.uuid}
  app.port=${random.int(1024,65535)}
  ```

### 2.4 Profile 多环境配置
- [ ] application-{profile}.yml 命名规则
- [ ] spring.profiles.active 激活方式
  ```yaml
  # 1. 配置文件指定
  spring.profiles.active: dev
  # 2. 命令行参数
  java -jar app.jar --spring.profiles.active=prod
  # 3. 环境变量
  export SPRING_PROFILES_ACTIVE=prod
  # 4. JVM 参数
  -Dspring.profiles.active=prod
  ```
- [ ] @Profile 注解条件化注册 Bean
- [ ] 多文档块 YAML（`---` 分隔多个 profile 配置）
- [ ] Profile 包含（include）机制

---

## 三、核心原理深度解析

### 3.1 自动配置原理（重中之重）
- [ ] Spring Boot 启动流程源码跟踪
  1. `SpringApplication` 构造
  2. `run()` 方法执行
  3. 准备 Environment
  4. 创建 ApplicationContext
  5. 执行 `AutoConfigurationImportSelector`
  6. 加载 `spring.factories` / `spring-autoconfigure-metadata.properties`
  7. 条件注解过滤（@ConditionalOnXxx）
  8. 注册自动配置类
- [ ] `@EnableAutoConfiguration` 工作机制
- [ ] `spring.factories` 文件的作用与配置
  ```
  org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyAutoConfiguration
  ```
- [ ] 手写一个 Starter（自定义自动配置）
  1. 创建 autoconfigure 模块
  2. 编写配置属性类 @ConfigurationProperties
  3. 编写自动配置类 @Configuration + @ConditionalOnXxx
  4. 在 spring.factories 中注册
  5. 创建 starter 模块（空模块，引入 autoconfigure）

### 3.2 条件注解全家桶
- [ ] `@ConditionalOnClass` — 类路径存在某类时生效
- [ ] `@ConditionalOnMissingClass` — 类路径缺失某类时生效
- [ ] `@ConditionalOnBean` — 容器中存在某 Bean 时生效
- [ ] `@ConditionalOnMissingBean` — 容器中不存在某 Bean 时生效（最常用）
- [ ] `@ConditionalOnProperty` — 配置属性满足条件时生效
  ```java
  @ConditionalOnProperty(
      name = "my.feature.enabled",
      havingValue = "true",
      matchIfMissing = false  // 属性不存在时不匹配
  )
  ```
- [ ] `@ConditionalOnResource` — 资源文件存在时生效
- [ ] `@ConditionalOnWebApplication` — 是 Web 应用时生效
- [ ] `@ConditionalOnExpression` — SpEL 表达式满足时生效
- [ ] 自定义 Condition 实现（实现 Condition 接口）

### 3.3 @ConfigurationProperties
- [ ] 类型安全的配置绑定
  ```java
  @ConfigurationProperties(prefix = "app.mail")
  @Component
  public class MailProperties {
      private String host = "localhost";  // 默认值
      private int port = 25;
      private String username;
      // getter/setter 必须
  }
  ```
- [ ] @EnableConfigurationProperties 激活方式
- [ ] 属性校验（@Validated + JSR-303 注解）
  ```java
  @NotNull
  @Min(1) @Max(65535)
  private Integer port;
  ```
- [ ] 复杂类型绑定（Map、List、嵌套对象）
- [ ] 松散绑定（camelCase、kebab-case、underscore 自动映射）
- [ ] IDEA 中配置提示（spring-boot-configuration-processor）

### 3.4 ApplicationContext 与 Bean
- [ ] ApplicationContext 继承体系
  ```
  BeanFactory → ApplicationContext → ConfigurableApplicationContext
                                   → WebApplicationContext
  ```
- [ ] Bean 注册的 5 种方式
  1. @Component/@Service/@Repository/@Controller 扫描
  2. @Configuration + @Bean 方法
  3. @Import 导入
  4. ImportBeanDefinitionRegistrar（编程式）
  5. BeanDefinitionRegistryPostProcessor
- [ ] BeanFactoryPostProcessor vs BeanPostProcessor
- [ ] FactoryBean 模式（MyBatis Mapper 代理原理）
- [ ] Aware 接口家族：获取容器资源
  - ApplicationContextAware
  - BeanFactoryAware
  - EnvironmentAware
  - ResourceLoaderAware

### 3.5 Spring Boot 内置 Web 容器
- [ ] 嵌入式 Tomcat 的启动与配置
  ```yaml
  server.tomcat.max-threads: 200
  server.tomcat.min-spare-threads: 10
  server.tomcat.accept-count: 100
  server.tomcat.max-connections: 10000
  server.tomcat.connection-timeout: 30000
  ```
- [ ] 切换容器：Tomcat → Jetty → Undertow
- [ ] 容器性能对比与选择场景
- [ ] 容器自定义（TomcatServletWebServerFactory）
- [ ] 容器的优雅关闭（Graceful Shutdown）

---

## 四、Web 开发全栈

### 4.1 Spring MVC 核心
- [ ] DispatcherServlet 请求处理流程（画出完整流程图）
  ```
  请求 → Filter Chain → DispatcherServlet
    → HandlerMapping（找到 Controller）
    → HandlerAdapter（调用方法）
    → HandlerInterceptor（拦截器链）
    → Controller Method
    → ReturnValueHandler（处理返回值）
    → ViewResolver（解析视图）/ HttpMessageConverter（JSON）
    → 响应
  ```
- [ ] @Controller vs @RestController
- [ ] 请求映射注解：@RequestMapping / @GetMapping / @PostMapping / @PutMapping / @DeleteMapping / @PatchMapping
- [ ] 参数绑定方式
  ```java
  // 路径参数
  @GetMapping("/users/{id}")
  public User getUser(@PathVariable Long id)
  
  // 查询参数
  @GetMapping("/users")
  public List<User> list(@RequestParam(required = false) String name)
  
  // 请求体 JSON
  @PostMapping("/users")
  public User create(@RequestBody @Validated UserDTO dto)
  
  // 表单数据（或 JSON + 文件混合）
  @PostMapping("/users/avatar")
  public String upload(@RequestPart("user") UserDTO dto,
                       @RequestPart("file") MultipartFile file)
  
  // 请求头
  @RequestHeader("Authorization") String token
  
  // Cookie
  @CookieValue("JSESSIONID") String sessionId
  ```
- [ ] 参数校验：@Valid / @Validated + JSR-303
  ```java
  public class UserDTO {
      @NotBlank(message = "用户名不能为空")
      @Size(min = 2, max = 20)
      private String username;
      
      @Email
      @NotBlank
      private String email;
      
      @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
      private String phone;
  }
  ```

### 4.2 拦截器与过滤器
- [ ] Filter（Servlet 规范）的使用
  ```java
  @WebFilter(urlPatterns = "/*")
  public class LogFilter implements Filter {
      @Override
      public void doFilter(ServletRequest req, ServletResponse resp,
                           FilterChain chain) {
          // 前置处理
          chain.doFilter(req, resp);
          // 后置处理
      }
  }
  ```
- [ ] HandlerInterceptor（Spring MVC）
  ```java
  public class AuthInterceptor implements HandlerInterceptor {
      @Override
      public boolean preHandle(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler) {
          // 返回 true 放行，false 拦截
          return true;
      }
      @Override
      public void postHandle(...) { }  // Controller 执行后，视图渲染前
      @Override
      public void afterCompletion(...) { }  // 视图渲染后
  }
  ```
- [ ] Filter vs Interceptor 的区别
  | 特性 | Filter | Interceptor |
  |------|--------|-------------|
  | 规范 | Servlet 规范 | Spring 框架 |
  | 执行顺序 | Filter → Interceptor |
  | 颗粒度 | 只能拿到 Request/Response | 能拿到 Handler 信息 |
  | 依赖注入 | 需要额外配置 | 天然支持 Bean 注入 |
- [ ] 拦截器注册与路径匹配规则
- [ ] 执行链顺序控制（@Order / Ordered 接口）

### 4.3 全局异常处理
- [ ] @ControllerAdvice + @ExceptionHandler
  ```java
  @RestControllerAdvice
  public class GlobalExceptionHandler {
      
      @ExceptionHandler(MethodArgumentNotValidException.class)
      public Result handleValidation(MethodArgumentNotValidException e) {
          String msg = e.getBindingResult()
                        .getFieldErrors().stream()
                        .map(FieldError::getDefaultMessage)
                        .collect(Collectors.joining(", "));
          return Result.error(400, msg);
      }
      
      @ExceptionHandler(BusinessException.class)
      public Result handleBusiness(BusinessException e) {
          return Result.error(e.getCode(), e.getMessage());
      }
      
      @ExceptionHandler(Exception.class)
      public Result handleUnknown(Exception e) {
          log.error("未知异常", e);
          return Result.error(500, "服务器内部错误");
      }
  }
  ```
- [ ] 自定义业务异常体系
- [ ] 异常处理优先级（精确匹配 > 父类匹配）
- [ ] 统一返回格式设计（Result / R 包装类）
- [ ] ErrorController 自定义错误页面

### 4.4 RESTful API 设计
- [ ] URI 设计规范（资源命名、层级关系、版本号）
  ```
  GET    /api/v1/users          # 列表
  GET    /api/v1/users/{id}     # 详情
  POST   /api/v1/users          # 创建
  PUT    /api/v1/users/{id}     # 全量更新
  PATCH  /api/v1/users/{id}     # 部分更新
  DELETE /api/v1/users/{id}     # 删除
  ```
- [ ] HTTP 状态码正确使用
  - 200 OK / 201 Created / 204 No Content
  - 400 Bad Request / 401 Unauthorized / 403 Forbidden / 404 Not Found
  - 409 Conflict / 422 Unprocessable Entity
  - 500 Internal Server Error / 502 Bad Gateway / 503 Service Unavailable
- [ ] 分页、排序、筛选 API 设计
  ```
  GET /api/v1/users?page=0&size=20&sort=name,asc&name=张三&status=active
  ```
- [ ] HATEOAS 超媒体驱动（可选进阶）
- [ ] API 版本管理策略（URL / Header / 参数）

### 4.5 跨域处理（CORS）
- [ ] @CrossOrigin 注解（单个 Controller）
- [ ] 全局 CORS 配置
  ```java
  @Configuration
  public class CorsConfig implements WebMvcConfigurer {
      @Override
      public void addCorsMappings(CorsRegistry registry) {
          registry.addMapping("/**")
                  .allowedOrigins("https://example.com")
                  .allowedMethods("GET", "POST", "PUT", "DELETE")
                  .allowedHeaders("*")
                  .allowCredentials(true)
                  .maxAge(3600);
      }
  }
  ```
- [ ] CorsFilter 方式（更灵活）
- [ ] 预检请求（OPTIONS）的理解与处理

### 4.6 文件上传与下载
- [ ] 单文件 / 多文件上传
  ```java
  @PostMapping("/upload")
  public String upload(@RequestParam("files") MultipartFile[] files) {
      for (MultipartFile file : files) {
          // 存储到本地或 OSS
      }
      return "success";
  }
  ```
- [ ] 文件大小限制配置
  ```yaml
  spring.servlet.multipart.max-file-size: 10MB
  spring.servlet.multipart.max-request-size: 50MB
  ```
- [ ] 文件类型校验（Magic Number + 扩展名双重校验）
- [ ] 文件下载（流式输出、断点续传）
- [ ] 文件存储方案：本地 vs OSS（阿里云/腾讯云/MinIO）

### 4.7 响应式编程（WebFlux）
- [ ] 响应式编程核心概念：Reactive Streams 规范
- [ ] Reactor 核心类型：Mono（0-1）与 Flux（0-N）
- [ ] Reactive API 开发
  ```java
  @RestController
  public class UserController {
      @GetMapping("/users/{id}")
      public Mono<User> getUser(@PathVariable String id) {
          return userRepository.findById(id);
      }
      @GetMapping("/users")
      public Flux<User> list() {
          return userRepository.findAll();
      }
  }
  ```
- [ ] Spring WebFlux vs Spring MVC 选择场景
- [ ] WebClient（取代 RestTemplate 的响应式 HTTP 客户端）
- [ ] R2DBC 响应式数据库访问

---

## 五、数据访问与持久化

### 5.1 JDBC 与 数据源
- [ ] DataSource 配置与连接池
  ```yaml
  spring.datasource.hikari:
    minimum-idle: 5
    maximum-pool-size: 20
    idle-timeout: 300000
    max-lifetime: 1200000
    connection-timeout: 30000
    connection-test-query: SELECT 1
  ```
- [ ] HikariCP（默认连接池）vs Druid 对比
- [ ] Druid 监控配置（StatViewServlet + WallFilter）
- [ ] JdbcTemplate 基本使用
- [ ] NamedParameterJdbcTemplate 具名参数
- [ ] RowMapper 自定义映射

### 5.2 MyBatis（国内主流）
- [ ] MyBatis-Spring-Boot-Starter 集成
- [ ] 两种开发模式对比
  - XML 映射：复杂 SQL、动态 SQL、存储过程
  - 注解方式：简单 CRUD，@Select/@Insert/@Update/@Delete
- [ ] Mapper 接口 + XML 绑定规范（namespace 对应）
- [ ] 动态 SQL
  ```xml
  <select id="findByCondition" resultType="User">
      SELECT * FROM user
      <where>
          <if test="name != null and name != ''">
              AND name LIKE CONCAT('%', #{name}, '%')
          </if>
          <if test="status != null">
              AND status = #{status}
          </if>
          <if test="ids != null and ids.size() > 0">
              AND id IN
              <foreach item="id" collection="ids" open="(" separator="," close=")">
                  #{id}
              </foreach>
          </if>
      </where>
  </select>
  ```
- [ ] `<choose>/<when>/<otherwise>` 条件选择
- [ ] `<trim>/<where>/<set>` 智能标签
- [ ] MyBatis-Plus 增强（国内主流选择）
  - BaseMapper 通用 CRUD
  - 条件构造器：QueryWrapper / LambdaQueryWrapper
  ```java
  List<User> list = userMapper.selectList(
      new LambdaQueryWrapper<User>()
          .eq(User::getStatus, 1)
          .like(User::getName, "张")
          .orderByDesc(User::getCreateTime)
  );
  ```
  - 分页插件配置
  - 逻辑删除、乐观锁、自动填充
  - 代码生成器（AutoGenerator）
- [ ] 分页查询
  - MyBatis PageHelper 分页插件
  - MyBatis-Plus 内置分页
  - Spring Data 分页（Pageable）
- [ ] 多数据源配置（dynamic-datasource）
- [ ] 读写分离方案
- [ ] 慢 SQL 监控与拦截

### 5.3 JPA / Spring Data JPA
- [ ] JPA 核心注解
  ```java
  @Entity
  @Table(name = "t_user")
  public class User {
      @Id
      @GeneratedValue(strategy = GenerationType.IDENTITY)
      private Long id;
      
      @Column(name = "user_name", length = 50, nullable = false)
      private String userName;
      
      @Enumerated(EnumType.STRING)  // 枚举映射
      private UserStatus status;
      
      @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
      private List<Order> orders;
  }
  ```
- [ ] JPA 关联关系
  - @OneToOne（一对一）
  - @OneToMany / @ManyToOne（一对多/多对一）
  - @ManyToMany（多对多）
  - FetchType.LAZY vs EAGER（懒加载陷阱与解决）
  - N+1 问题识别与 @EntityGraph 解决
- [ ] Spring Data JPA 方法命名查询
  ```java
  // 方法名即语义，框架自动解析
  List<User> findByNameAndStatusOrderByCreateTimeDesc(String name, Integer status);
  Page<User> findByAgeBetween(Integer min, Integer max, Pageable pageable);
  ```
- [ ] @Query 自定义 JPQL / SQL
- [ ] Specification 动态查询（JpaSpecificationExecutor）
- [ ] JPA 审计功能（@CreatedDate / @LastModifiedDate）
- [ ] MyBatis vs JPA 选择决策
  | 场景 | 推荐 |
  |------|------|
  | 简单 CRUD | JPA |
  | 复杂 SQL、报表 | MyBatis |
  | ORM 深度优化 | MyBatis |
  | 快速原型 | JPA |

### 5.4 事务管理
- [ ] @Transactional 注解详解
  ```java
  @Transactional(
      propagation = Propagation.REQUIRED,
      isolation = Isolation.REPEATABLE_READ,
      timeout = 30,
      readOnly = false,
      rollbackFor = Exception.class,
      noRollbackFor = IllegalArgumentException.class
  )
  ```
- [ ] 事务传播行为（7 种）
  | 传播行为 | 说明 |
  |----------|------|
  | REQUIRED（默认） | 存在事务则加入，不存在则新建 |
  | REQUIRES_NEW | 始终新建事务，挂起当前 |
  | SUPPORTS | 存在事务则加入，不存在则以非事务运行 |
  | NOT_SUPPORTED | 以非事务运行，挂起现有事务 |
  | MANDATORY | 必须在事务中运行，否则抛异常 |
  | NEVER | 必须非事务运行，否则抛异常 |
  | NESTED | 嵌套事务（保存点机制） |
- [ ] 事务隔离级别
  - READ_UNCOMMITTED（读未提交）— 脏读
  - READ_COMMITTED（读已提交）— 不可重复读
  - REPEATABLE_READ（可重复读，MySQL 默认）— 幻读
  - SERIALIZABLE（序列化）— 性能最低
- [ ] 事务失效的 8 大场景
  1. 方法非 public
  2. 同类方法调用（this.xxx() 绕过代理）
  3. 异常被 catch 吞掉
  4. rollbackFor 设置不正确
  5. 数据库引擎不支持（如 MyISAM）
  6. @Transactional 注解在非 Bean 方法上
  7. 多线程调用
  8. propagation 设置不当
- [ ] 编程式事务（TransactionTemplate）使用场景
- [ ] 分布式事务简介（Seata / TCC / Saga / 可靠消息）

### 5.5 数据库版本管理（Flyway / Liquibase）
- [ ] Flyway 迁移脚本命名规范
  ```
  V1__init.sql
  V2__add_user_table.sql
  V2.1__alter_user_add_email.sql
  R__repeatable_view.sql  # 可重复执行
  ```
- [ ] Flyway 配置与自动执行
- [ ] 回退策略与注意事项
- [ ] Flyway vs Liquibase 对比

---

## 六、缓存系统

### 6.1 Spring Cache 抽象
- [ ] @EnableCaching 启用缓存
- [ ] 核心注解
  ```java
  @Cacheable(value = "user", key = "#id", unless = "#result == null")
  public User getUser(Long id);
  
  @CachePut(value = "user", key = "#user.id")  // 更新缓存
  public User updateUser(User user);
  
  @CacheEvict(value = "user", key = "#id")  // 清除缓存
  public void deleteUser(Long id);
  
  @Caching(  // 组合操作
      evict = { @CacheEvict("user"), @CacheEvict("role") }
  )
  ```
- [ ] SpEL 表达式生成 Key
  ```java
  key = "'user_' + #id"
  key = "#page.pageNum + '_' + #page.pageSize"
  key = "T(com.example.UserContext).getCurrentUserId()"  // 静态方法
  ```
- [ ] 缓存管理器选择：Caffeine（本地）/ Redis（分布式）

### 6.2 Redis 深度使用
- [ ] Spring Data Redis 集成
- [ ] RedisTemplate vs StringRedisTemplate
- [ ] Redis 序列化方案
  ```java
  @Configuration
  public class RedisConfig {
      @Bean
      public RedisTemplate<String, Object> redisTemplate(
              RedisConnectionFactory factory) {
          RedisTemplate<String, Object> template = new RedisTemplate<>();
          template.setConnectionFactory(factory);
          // Key 使用 String 序列化
          template.setKeySerializer(new StringRedisSerializer());
          // Value 使用 JSON 序列化
          template.setValueSerializer(
              new GenericJackson2JsonRedisSerializer());
          return template;
      }
  }
  ```
- [ ] Redis 数据结构操作
  ```java
  // String
  redisTemplate.opsForValue().set("key", "value", 10, TimeUnit.MINUTES);
  // Hash
  redisTemplate.opsForHash().put("user:1001", "name", "张三");
  // List（消息队列）
  redisTemplate.opsForList().leftPush("queue", message);
  redisTemplate.opsForList().rightPop("queue");
  // Set（去重集合）
  redisTemplate.opsForSet().add("tags", "java", "spring");
  // ZSet（排行榜）
  redisTemplate.opsForZSet().add("rank", "user1", 99.5);
  // BitMap（签到）
  redisTemplate.opsForValue().setBit("sign:202401", userId, true);
  // HyperLogLog（UV 统计）
  redisTemplate.opsForHyperLogLog().add("uv:page1", userId);
  ```
- [ ] Redis 发布/订阅（Pub/Sub）
- [ ] Redis Pipeline 批量操作
- [ ] Redis Lua 脚本（原子操作）
- [ ] 缓存穿透 / 缓存击穿 / 缓存雪崩 解决方案
  - 穿透：布隆过滤器（Redisson RBloomFilter）
  - 击穿：互斥锁 / 逻辑过期
  - 雪崩：随机过期时间 / 多级缓存

### 6.3 Redisson 分布式能力
- [ ] 分布式锁
  ```java
  RLock lock = redissonClient.getLock("order:" + orderId);
  try {
      if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
          // 业务逻辑
      }
  } finally {
      if (lock.isHeldByCurrentThread()) {
          lock.unlock();
      }
  }
  ```
  - [ ] 看门狗（Watch Dog）机制理解
  - [ ] 可重入锁 / 公平锁 / 联锁 / 红锁
- [ ] 分布式集合
  - RMap（分布式 Map）
  - RQueue（分布式队列）
  - RCountDownLatch（分布式闭锁）
  - RSemaphore（分布式信号量）

---

## 七、消息队列

### 7.1 消息队列基础
- [ ] 理解消息队列的核心作用
  - 异步处理（提高响应速度）
  - 解耦（发送方与消费方独立演进）
  - 削峰填谷（平滑流量压力）
  - 数据同步（多系统数据一致性）
- [ ] 常见 MQ 对比
  | MQ | 吞吐量 | 延迟 | 可靠性 | 复杂度 |
  |----|--------|------|--------|--------|
  | RabbitMQ | 万级 | 微秒 | 高 | 中 |
  | RocketMQ | 十万级 | 毫秒 | 极高 | 高 |
  | Kafka | 百万级 | 毫秒 | 高 | 中 |

### 7.2 RabbitMQ 集成
- [ ] Spring AMQP 集成
- [ ] 核心组件理解：ConnectionFactory → Connection → Channel → Exchange → Queue
- [ ] 交换机类型
  - Direct（直连，精确路由）
  - Topic（主题，模式匹配路由）
  - Fanout（广播，忽略 routing key）
  - Headers（头匹配路由）
- [ ] 消息发送
  ```java
  rabbitTemplate.convertAndSend("exchange", "order.create", orderMsg);
  // 带过期时间的消息
  rabbitTemplate.convertAndSend("exchange", "key", msg, m -> {
      m.getMessageProperties().setExpiration("60000");
      return m;
  });
  ```
- [ ] 消息消费
  ```java
  @RabbitListener(queues = "order.queue")
  public void handleOrder(@Payload OrderMessage message, 
                          @Headers Map<String, Object> headers) {
      // 处理消息
  }
  ```
- [ ] 手动确认模式（ACK / NACK / Reject）
- [ ] 死信队列（DLX）配置
- [ ] 延迟队列实现（TTL + DLX / 插件）
- [ ] 消息可靠性保障
  1. 生产者确认（Publisher Confirm）
  2. 消息持久化
  3. 消费者手动确认
  4. 幂等性处理

### 7.3 Kafka 集成
- [ ] Spring Kafka 集成
- [ ] 核心概念：Broker / Topic / Partition / Consumer Group / Offset
- [ ] KafkaTemplate 消息发送
- [ ] @KafkaListener 消息消费
- [ ] 顺序消费保障（同 Key 写入同一 Partition）
- [ ] 消息重复消费与幂等性
- [ ] 消息积压处理策略

### 7.4 消息可靠性模式
- [ ] 本地事务消息表（保证业务与消息一致）
- [ ] 事务消息（RocketMQ 支持，Kafka 需幂等+重试补偿）
- [ ] 最终一致性模式
- [ ] 死信处理策略

---

## 八、安全与认证授权

### 8.1 Spring Security 基础
- [ ] Spring Security 过滤器链理解
  ```
  请求 → SecurityContextPersistenceFilter
      → UsernamePasswordAuthenticationFilter
      → BasicAuthenticationFilter
      → ExceptionTranslationFilter
      → FilterSecurityInterceptor
      → 业务处理
  ```
- [ ] 核心配置（SecurityFilterChain）
  ```java
  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http) {
      return http
          .csrf(CsrfConfigurer::disable)  // 前后端分离一般关闭
          .authorizeHttpRequests(auth -> auth
              .requestMatchers("/api/public/**").permitAll()
              .requestMatchers("/api/admin/**").hasRole("ADMIN")
              .anyRequest().authenticated()
          )
          .sessionManagement(s -> s
              .sessionCreationPolicy(SessionCreationPolicy.STATELESS)  // 无状态
          )
          .build();
  }
  ```

### 8.2 JWT 认证方案
- [ ] JWT 结构解析：Header.Payload.Signature
- [ ] Access Token + Refresh Token 双 Token 设计
  ```
  1. 用户登录 → 服务端验证 → 返回 AccessToken(15min) + RefreshToken(7天)
  2. 客户端携带 AccessToken 请求
  3. AccessToken 过期 → 用 RefreshToken 换取新 AccessToken
  4. RefreshToken 过期 → 重新登录
  ```
- [ ] JWT 过滤器实现
  ```java
  public class JwtAuthFilter extends OncePerRequestFilter {
      @Override
      protected void doFilterInternal(HttpServletRequest request,
                                      HttpServletResponse response,
                                      FilterChain chain) {
          String token = extractToken(request);
          if (token != null && jwtUtil.validateToken(token)) {
              Authentication auth = jwtUtil.getAuthentication(token);
              SecurityContextHolder.getContext().setAuthentication(auth);
          }
          chain.doFilter(request, response);
      }
  }
  ```
- [ ] Token 刷新机制
- [ ] 黑名单与强制下线（Redis 存储）
- [ ] 并发登录控制

### 8.3 OAuth 2.0 / OIDC
- [ ] 授权码模式（Authorization Code）— 最安全
- [ ] 密码模式（Password）— 内部系统
- [ ] 客户端模式（Client Credentials）— 服务间调用
- [ ] 简化模式（Implicit）— 已废弃
- [ ] Spring Security OAuth2 配置
- [ ] 集成微信/企业微信/飞书 第三方登录
- [ ] OIDC（OpenID Connect）身份认证

### 8.4 权限模型
- [ ] RBAC（基于角色的访问控制）模型设计
  ```
  User ↔ Role ↔ Permission（用户-角色-权限）
  ```
- [ ] @PreAuthorize / @PostAuthorize 方法级权限
  ```java
  @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
  public User getOwnProfile(Long id);
  
  @PreAuthorize("@permissionChecker.hasPermission(#id, 'delete')")
  public void deleteProject(Long id);
  ```
- [ ] 动态权限控制（数据库存储权限规则）
- [ ] 数据权限（行级别隔离）
- [ ] ABAC（基于属性的访问控制）简介

### 8.5 安全防护
- [ ] CSRF 防护（前后端分离时的策略）
- [ ] XSS 防护（输入过滤、输出编码）
- [ ] SQL 注入防护（MyBatis `#{}` vs `${}`）
- [ ] CORS 安全配置（精确域名白名单）
- [ ] 敏感数据脱敏（日志脱敏、序列化脱敏）
- [ ] 接口限流（Rate Limiting）
  - Guava RateLimiter
  - 令牌桶 vs 漏桶算法
- [ ] 接口幂等性（Token 机制 / 数据库唯一索引）

---

## 九、任务调度与异步处理

### 9.1 Spring Task（@Scheduled）
- [ ] 启用定时任务
  ```java
  @Configuration
  @EnableScheduling
  public class ScheduleConfig { }
  ```
- [ ] 定时任务编写
  ```java
  @Scheduled(fixedRate = 5000)       // 每 5 秒执行（从上一次开始计时）
  @Scheduled(fixedDelay = 5000)      // 上一次结束后 5 秒执行
  @Scheduled(cron = "0 0 2 * * ?")   // 每天凌晨 2 点
  @Scheduled(initialDelay = 10000, fixedRate = 5000)  // 首次推迟 10 秒
  ```
- [ ] Cron 表达式语法精通
  ```
  ┌─── 秒 (0-59)
  │ ┌─── 分钟 (0-59)
  │ │ ┌─── 小时 (0-23)
  │ │ │ ┌─── 日期 (1-31)
  │ │ │ │ ┌─── 月份 (1-12)
  │ │ │ │ │ ┌─── 星期 (0-7, 0/7=周日)
  │ │ │ │ │ │
  * * * * * *
  
  0 0/30 9-17 * * MON-FRI   # 工作日 9:00-17:30 每半小时
  0 0 2 L * ?               # 每月最后一天凌晨 2 点
  0 15 10 ? * 6L             # 每月最后一个周六 10:15
  ```
- [ ] 定时任务线程池配置
  ```yaml
  spring.task.scheduling.pool.size: 10
  spring.task.scheduling.thread-name-prefix: sched-
  ```
- [ ] 分布式环境下的重复执行问题（配合分布式锁）

### 9.2 异步处理
- [ ] @EnableAsync + @Async
  ```java
  @Service
  public class EmailService {
      @Async
      public CompletableFuture<String> sendEmail(String to) {
          // 发送邮件，此处异步执行
          return CompletableFuture.completedFuture("sent");
      }
  }
  ```
- [ ] 异步线程池自定义配置
  ```java
  @Bean("taskExecutor")
  public Executor taskExecutor() {
      ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
      executor.setCorePoolSize(10);
      executor.setMaxPoolSize(50);
      executor.setQueueCapacity(200);
      executor.setRejectedExecutionHandler(
          new ThreadPoolExecutor.CallerRunsPolicy());
      executor.setThreadNamePrefix("async-");
      executor.initialize();
      return executor;
  }
  ```
- [ ] @Async 失效场景（同类调用、私有方法、没有 @EnableAsync）
- [ ] CompletableFuture 编排
  ```java
  CompletableFuture<String> f1 = service1.asyncOp();
  CompletableFuture<String> f2 = service2.asyncOp();
  CompletableFuture.allOf(f1, f2)
      .thenAccept(v -> {
          String result1 = f1.join();
          String result2 = f2.join();
      })
      .exceptionally(e -> { log.error("error", e); return null; });
  ```

### 9.3 分布式任务调度（XXL-JOB / Quartz）
- [ ] XXL-JOB 架构理解（调度中心 + 执行器）
- [ ] XXL-JOB 集成配置
- [ ] 任务编写与注册
- [ ] 分片广播（大数据量并行处理）
- [ ] 失败重试与告警
- [ ] Quartz 简介（适合需要动态增删任务的场景）

---

## 十、日志与监控

### 10.1 日志框架
- [ ] SLF4J + Logback（默认）体系
- [ ] logback-spring.xml 配置
  ```xml
  <configuration>
      <!-- 控制台输出 -->
      <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
          <encoder>
              <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level 
                       %logger{36} - %msg%n</pattern>
          </encoder>
      </appender>
      <!-- 文件滚动输出 -->
      <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
          <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
              <fileNamePattern>logs/app-%d{yyyy-MM-dd}.log</fileNamePattern>
              <maxHistory>30</maxHistory>
          </rollingPolicy>
      </appender>
      <root level="INFO">
          <appender-ref ref="CONSOLE"/>
          <appender-ref ref="FILE"/>
      </root>
  </configuration>
  ```
- [ ] 日志级别动态调整（Spring Boot Actuator）
- [ ] 结构化日志（JSON 格式，方便 ELK 采集）
- [ ] MDC（Mapped Diagnostic Context）追踪链路
  ```java
  MDC.put("traceId", traceId);  // 全链路追踪 ID
  // 日志 pattern 中自动注入: %X{traceId}
  ```

### 10.2 Spring Boot Actuator
- [ ] Actuator 端点大全
  ```yaml
  management:
    endpoints:
      web:
        exposure:
          include: health,info,metrics,prometheus,loggers,env,mappings
    endpoint:
      health:
        show-details: always
  ```
- [ ] `/health` 健康检查（数据库、Redis、MQ 等组件状态）
- [ ] `/info` 应用信息
- [ ] `/metrics` 指标（JVM 内存、GC、线程、HTTP 请求统计）
- [ ] `/loggers` 动态调整日志级别
- [ ] `/env` 环境变量与配置
- [ ] `/mappings` 所有请求映射
- [ ] `/heapdump` 堆转储
- [ ] `/threaddump` 线程转储
- [ ] Actuator 安全保护（结合 Spring Security）

### 10.3 Micrometer + Prometheus + Grafana
- [ ] Micrometer 指标库集成
- [ ] Prometheus 指标暴露格式
- [ ] 自定义指标
  ```java
  @Bean
  public MeterBinder customMetrics() {
      Counter counter = Counter.builder("orders.total")
          .description("总订单数")
          .tag("status", "created")
          .register(MeterRegistry);
      return null;
  }
  ```
- [ ] Grafana 仪表盘配置（JVM 监控面板）
- [ ] 接口调用耗时分布（Histogram + p50/p95/p99）

### 10.4 链路追踪（Sleuth + Zipkin / SkyWalking）
- [ ] 分布式链路追踪概念
- [ ] Spring Cloud Sleuth 自动注入 TraceId / SpanId
- [ ] Zipkin 数据上报与查询
- [ ] SkyWalking 集成（无侵入 Java Agent）
- [ ] 跨服务调用链路分析

### 10.5 告警体系
- [ ] 关键指标定义（错误率、响应时间、可用率）
- [ ] 告警规则配置（Prometheus AlertManager）
- [ ] 告警通知渠道（钉钉/飞书/企业微信 Webhook）
- [ ] 告警降噪与值班轮转

---

## 十一、配置管理与多环境

### 11.1 配置优先级
- [ ] Spring Boot 配置加载顺序（17 级优先级）
  ```
  高优先级
  1. 命令行参数（--server.port=9090）
  2. JNDI 属性
  3. 系统环境变量
  4. application-{profile}.yml（jar 外部）
  5. application-{profile}.yml（jar 内部）
  6. application.yml（jar 外部）
  7. application.yml（jar 内部）
  ...
  低优先级
  ```
- [ ] 外部化配置原则（12-Factor App）

### 11.2 配置加密
- [ ] Jasypt 敏感配置加密
  ```properties
  spring.datasource.password=ENC(加密后的密码)
  jasypt.encryptor.password=${JASYPT_KEY}  # 密钥通过环境变量注入
  ```

### 11.3 配置中心（Nacos / Apollo / Consul）
- [ ] 配置中心 vs 本地配置的优劣
- [ ] Nacos 集成
  ```yaml
  spring.cloud.nacos.config.server-addr: localhost:8848
  spring.cloud.nacos.config.file-extension: yaml
  ```
- [ ] 配置动态刷新（@RefreshScope）
- [ ] 配置版本管理与回滚
- [ ] 灰度发布与标签路由

---

## 十二、测试体系

### 12.1 单元测试
- [ ] JUnit 5 核心注解
  - @Test / @BeforeEach / @AfterEach
  - @BeforeAll / @AfterAll（静态方法）
  - @DisplayName / @Nested / @Tag
  - @ParameterizedTest + @ValueSource/@CsvSource
- [ ] Mockito Mock 框架
  ```java
  @ExtendWith(MockitoExtension.class)
  class UserServiceTest {
      @Mock
      private UserRepository userRepository;
      
      @InjectMocks
      private UserService userService;
      
      @Test
      void shouldReturnUser() {
          // Arrange
          when(userRepository.findById(1L))
              .thenReturn(Optional.of(new User(1L, "张三")));
          // Act
          User user = userService.getUser(1L);
          // Assert
          assertEquals("张三", user.getName());
          verify(userRepository, times(1)).findById(1L);
      }
  }
  ```
- [ ] Mock 行为定义技巧
  - when().thenReturn() / thenThrow()
  - doNothing().when()（void 方法）
  - ArgumentCaptor 捕获参数
  - spy() 部分 Mock
- [ ] 覆盖率工具（JaCoCo）

### 12.2 Spring Boot 集成测试
- [ ] @SpringBootTest 启动完整上下文
- [ ] @WebMvcTest（仅测试 Controller 层）
  ```java
  @WebMvcTest(UserController.class)
  class UserControllerTest {
      @Autowired
      private MockMvc mockMvc;
      
      @MockBean
      private UserService userService;
      
      @Test
      void shouldReturnUserJson() throws Exception {
          when(userService.getUser(1L)).thenReturn(new User(1L, "张三"));
          
          mockMvc.perform(get("/users/1"))
              .andExpect(status().isOk())
              .andExpect(jsonPath("$.name").value("张三"));
      }
  }
  ```
- [ ] @DataJpaTest（仅测试 Repository 层，使用 H2 内存数据库）
- [ ] @RestClientTest（测试 RestTemplate）
- [ ] TestRestTemplate vs MockMvc
- [ ] 测试数据库（H2 / TestContainers）

### 12.3 测试最佳实践
- [ ] 测试金字塔：单元 > 集成 > E2E
- [ ] AAA 模式（Arrange-Act-Assert）
- [ ] 测试命名规范：`should_xxx_when_xxx`
- [ ] 测试数据管理（@Sql 初始化脚本）
- [ ] 不在测试中依赖外部服务（用 WireMock / MockServer）
- [ ] TDD（测试驱动开发）实践

---

## 十三、微服务架构

### 13.1 Spring Cloud 核心组件
- [ ] 服务注册与发现（Nacos / Eureka / Consul）
  ```yaml
  spring.cloud.nacos.discovery.server-addr: localhost:8848
  ```
- [ ] 远程调用（OpenFeign）
  ```java
  @FeignClient(name = "user-service", path = "/api/users")
  public interface UserFeignClient {
      @GetMapping("/{id}")
      UserDTO getUser(@PathVariable("id") Long id);
  }
  ```
  - [ ] Feign 拦截器（传递请求头）
  - [ ] Feign 超时与重试配置
  - [ ] Feign 日志配置
- [ ] 负载均衡（Spring Cloud LoadBalancer）
- [ ] 网关（Spring Cloud Gateway）
  ```yaml
  spring.cloud.gateway.routes:
    - id: user-service
      uri: lb://user-service
      predicates:
        - Path=/api/users/**
      filters:
        - StripPrefix=1
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20
  ```
  - [ ] 路由规则：Path / Host / Header / Method / Query
  - [ ] 过滤器：鉴权、限流、日志、跨域
  - [ ] 网关聚合文档
- [ ] 服务熔断降级（Sentinel / Resilience4j）
  ```java
  @SentinelResource(value = "getUser", blockHandler = "getUserFallback")
  public User getUser(Long id) {
      // 可能失败的业务逻辑
  }
  public User getUserFallback(Long id, BlockException e) {
      return User.DEFAULT;  // 降级返回
  }
  ```
  - [ ] 熔断策略：慢调用比例 / 异常比例 / 异常数
  - [ ] 热点参数限流
  - [ ] 系统自适应保护

### 13.2 微服务设计模式
- [ ] 服务拆分原则（DDD 限界上下文 / 单一职责）
- [ ] API 网关模式（BFF：Backend for Frontend）
- [ ] 服务发现模式（客户端发现 vs 服务端发现）
- [ ] 配置中心模式
- [ ] 分布式配置管理

### 13.3 分布式数据一致性
- [ ] CAP 理论在微服务中的权衡
- [ ] 最终一致性实现方式
  - 可靠消息最终一致性（RocketMQ 事务消息）
  - TCC（Try-Confirm-Cancel）— Seata
  - Saga 模式（长事务补偿）
- [ ] 分布式事务选型决策
- [ ] 幂等性设计（数据库唯一约束 + 状态机）

### 13.4 微服务测试
- [ ] 消费者驱动契约测试（Spring Cloud Contract / Pact）
- [ ] 服务间 Mock（WireMock / MockServer）
- [ ] 端到端测试策略
- [ ] 金丝雀发布与蓝绿部署

---

## 十四、云原生与容器化

### 14.1 Docker 化
- [ ] 编写高效 Dockerfile（多阶段构建）
  ```dockerfile
  # 构建阶段
  FROM maven:3.8-openjdk-17 AS build
  WORKDIR /app
  COPY pom.xml .
  RUN mvn dependency:go-offline
  COPY src ./src
  RUN mvn package -DskipTests
  
  # 运行阶段
  FROM openjdk:17-slim
  RUN addgroup --system spring && adduser --system spring -g spring
  USER spring:spring
  COPY --from=build /app/target/*.jar app.jar
  ENTRYPOINT ["java", "-jar", "app.jar"]
  ```
- [ ] Docker Compose 编排
  ```yaml
  services:
    app:
      build: .
      ports: ["8080:8080"]
      depends_on:
        mysql:
          condition: service_healthy
    mysql:
      image: mysql:8.0
      environment:
        MYSQL_ROOT_PASSWORD: root
      healthcheck:
        test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  ```
- [ ] JVM 参数调优
- [ ] 容器内存限制与 JVM 堆配置

### 14.2 Kubernetes
- [ ] Deployment + Service 基本概念
- [ ] Spring Boot 应用的 K8s 资源配置
- [ ] 健康检查：livenessProbe + readinessProbe
  ```yaml
  livenessProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
  readinessProbe:
    httpGet:
      path: /actuator/health/readiness
      port: 8080
    initialDelaySeconds: 10
    periodSeconds: 5
  ```
- [ ] ConfigMap / Secret 配置注入
- [ ] 优雅上下线（PreStop Hook + Graceful Shutdown）
- [ ] 滚动更新策略

### 14.3 云原生 Spring
- [ ] Spring Cloud Kubernetes（服务发现基于 K8s Service）
- [ ] Spring Native / GraalVM 原生镜像
- [ ] 构建与部署流水线

---

## 十五、性能调优

### 15.1 JVM 调优
- [ ] GC 日志参数
  ```
  -Xlog:gc*:file=gc.log:time,uptime,level,tags
  ```
- [ ] 常用 GC 选择
  - G1GC（JDK 11+ 默认，低延迟大堆）
  - ZGC（JDK 17+，亚毫秒级停顿）
  - Shenandoah（低停顿）
- [ ] 堆内存调优
  ```
  -Xms2g -Xmx2g              # 初始堆 = 最大堆，避免动态扩展
  -XX:MaxMetaspaceSize=256m   # 元空间限制
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/tmp/heapdump.hprof
  ```
- [ ] 内存泄漏排查（MAT / JProfiler / Arthas）

### 15.2 数据库调优
- [ ] 慢 SQL 定位
  ```yaml
  mybatis-plus.configuration.log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
  # 或使用 Druid 的慢 SQL 监控
  ```
- [ ] SQL 执行计划分析（EXPLAIN）
- [ ] 索引优化策略
  - 最左前缀原则
  - 覆盖索引
  - 避免索引失效（函数、计算、类型转换、NULL 判断、前导模糊查询）
- [ ] 连接池配置优化（HikariCP）
- [ ] 读写分离 + 分库分表（ShardingSphere）

### 15.3 接口性能优化
- [ ] 性能压测工具（JMeter / wrk / ab）
- [ ] 接口响应时间优化清单
  1. [ ] 减少数据库查询次数（批量查询代替循环查询）
  2. [ ] 添加缓存（本地 Caffeine / 分布式 Redis）
  3. [ ] 异步化非核心逻辑（@Async / MQ）
  4. [ ] 数据压缩（Gzip）
  5. [ ] 连接复用（HTTP 连接池）
  6. [ ] 代码级优化（避免锁竞争、减少对象创建）
- [ ] 高并发场景设计
  - 秒杀：Redis 预减库存 + MQ 异步下单
  - 排行榜：Redis ZSet
  - 计数：Redis INCR + 定时同步

### 15.4 Arthas 在线诊断
- [ ] Arthas 安装与启动
  ```bash
  curl -O https://arthas.aliyun.com/arthas-boot.jar
  java -jar arthas-boot.jar
  ```
- [ ] 常用命令
  - `dashboard` — 实时面板
  - `thread` — 线程分析（`thread -b` 找死锁）
  - `watch` — 方法耗时观察
  - `trace` — 方法调用链路追踪
  - `jad` — 反编译类
  - `logger` — 动态调整日志级别
  - `heapdump` — 堆转储

---

## 十六、常见实战场景

### 16.1 场景一：构建一个完整的 RESTful 后端服务
1. [ ] 项目初始化（Spring Initializr + 依赖选择）
2. [ ] 数据库设计（ER 图 + DDL）
3. [ ] Entity + Mapper 生成（MyBatis-Plus 代码生成器）
4. [ ] Service 层实现（业务逻辑 + 事务管理）
5. [ ] Controller 层实现（参数校验 + 统一返回）
6. [ ] 全局异常处理
7. [ ] 分页查询 + 条件筛选
8. [ ] Swagger/Knife4j API 文档
9. [ ] 单元测试 + 集成测试

### 16.2 场景二：用户认证与授权系统
1. [ ] 数据库表设计（user / role / permission / user_role / role_permission）
2. [ ] 注册接口（密码 BCrypt 加密）
3. [ ] 登录接口（返回 JWT Token）
4. [ ] JWT 过滤器（请求拦截认证）
5. [ ] 权限注解（@PreAuthorize 集成）
6. [ ] 动态权限菜单加载
7. [ ] Refresh Token 刷新
8. [ ] 强制登出（Redis 黑名单）

### 16.3 场景三：文件管理服务
1. [ ] 文件上传（本地存储 / OSS）
2. [ ] 文件名校验（防止路径穿越）
3. [ ] 文件类型校验（Magic Number + 白名单）
4. [ ] 分片上传（大文件 >100MB）
5. [ ] 秒传（文件 MD5 去重）
6. [ ] 下载（流式 + 断点续传）
7. [ ] 文件生命周期管理（过期清理）

### 16.4 场景四：数据导入导出
- [ ] EasyExcel 集成
  ```java
  // 导出 - 流式写入，支持百万数据
  EasyExcel.write(outputStream, UserExportVO.class)
      .sheet("用户数据")
      .doWrite(dataList);
  
  // 导入 - 逐行读取，内存友好
  EasyExcel.read(inputStream, UserImportDTO.class,
      new ReadListener<UserImportDTO>() {
          @Override
          public void invoke(UserImportDTO data, AnalysisContext ctx) {
              // 逐行处理
          }
          @Override
          public void doAfterAllAnalysed(AnalysisContext ctx) {
              // 导入完成
          }
      }).sheet().doRead();
  ```
- [ ] 大数据量分批处理（生产者-消费者模式）
- [ ] 导入进度追踪（WebSocket 推送进度）
- [ ] 数据校验与错误行反馈

### 16.5 场景五：WebSocket 实时通信
- [ ] Spring WebSocket 集成
- [ ] STOMP 协议（消息代理模式）
  ```java
  @Configuration
  @EnableWebSocketMessageBroker
  public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
      @Override
      public void configureMessageBroker(MessageBrokerRegistry config) {
          config.enableSimpleBroker("/topic", "/queue");
          config.setApplicationDestinationPrefixes("/app");
      }
  }
  ```
- [ ] 点对点消息 / 广播消息
- [ ] WebSocket 连接管理（心跳、重连）
- [ ] 结合 Redis Pub/Sub 实现分布式 WebSocket

### 16.6 场景六：接口 API 文档（Knife4j / Swagger）
- [ ] 基本配置
  ```java
  @Bean
  public OpenAPI customOpenAPI() {
      return new OpenAPI()
          .info(new Info().title("XXX API 文档").version("1.0"))
          .addSecurityItem(new SecurityRequirement().addList("Bearer"))
          .components(new Components().addSecuritySchemes("Bearer",
              new SecurityScheme()
                  .type(SecurityScheme.Type.HTTP)
                  .scheme("bearer")
                  .bearerFormat("JWT")));
  }
  ```
- [ ] 实体注解
  ```java
  @Schema(description = "用户信息")
  public class UserDTO {
      @Schema(description = "用户名", example = "张三")
      private String name;
  }
  ```
- [ ] Controller 注解
  ```java
  @Operation(summary = "创建用户", description = "传入用户信息创建")
  @Parameters({
      @Parameter(name = "version", description = "API版本")
  })
  ```
- [ ] Knife4j 增强（离线文档、全局参数、排序）

---

## 十七、设计模式在 Spring Boot 中的实践

### 17.1 创建型模式
- [ ] **工厂方法**：各种 FactoryBean（SqlSessionFactoryBean）
- [ ] **抽象工厂**：BeanFactory → ApplicationContext（Spring 容器本身就是抽象工厂）
- [ ] **单例模式**：Spring Bean 默认 Scope Singleton，通过 ConcurrentHashMap 实现的单例注册表
- [ ] **建造者模式**：SpringApplicationBuilder / RestTemplateBuilder / MockMvcBuilder
- [ ] **原型模式**：@Scope("prototype") 每次创建新实例

### 17.2 结构型模式
- [ ] **适配器模式**：HandlerAdapter（将不同 Handler 适配给 DispatcherServlet）
- [ ] **装饰器模式**：BeanDefinitionDecorator / ServerHttpRequestDecorator（包装 Request 增加功能）
- [ ] **代理模式**：AOP 动态代理（JDK / CGLIB）、@Transactional 事务代理
- [ ] **外观模式**：JdbcTemplate（简化 JDBC 操作）
- [ ] **组合模式**：CompositeFilter（多个 Filter 组合）

### 17.3 行为型模式
- [ ] **模板方法**：JdbcTemplate / RestTemplate / RedisTemplate（固定流程 + 可变步骤）
- [ ] **策略模式**：Resource 接口不同实现（ClassPathResource / UrlResource / FileUrlResource）
- [ ] **观察者模式**：ApplicationEvent + @EventListener
- [ ] **责任链模式**：Filter Chain / Interceptor Chain
- [ ] **命令模式**：Runnable / Callable 配合 @Async 使用
- [ ] **状态模式**：状态机（Spring State Machine）

---

## 十八、常见陷阱与解决方案

### 18.1 事务相关
| 陷阱 | 原因 | 解决方案 |
|------|------|----------|
| 事务不生效 | 同类方法调用绕过代理 | 注入自己 / AopContext.currentProxy() / 拆分到另一个 Service |
| 事务不生效 | 方法非 public | 改为 public 或使用 AspectJ 模式 |
| 事务不回滚 | catch 吞掉异常 | catch 中手动 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` |
| 长事务超时 | 事务中做了耗时操作 | 拆分事务 / 非必要逻辑移出事务 |

### 18.2 依赖注入
| 陷阱 | 原因 | 解决方案 |
|------|------|----------|
| 循环依赖 | A → B → A | 重构设计 / @Lazy / Setter 注入 |
| @Autowired 注入 null | Bean 未被 Spring 管理 | 确保类上有 @Component 或通过 @Bean 注册 |
| 多实现类注入冲突 | 多个 Bean 实现同一接口 | @Qualifier / @Primary 指定 |

### 18.3 MyBatis 常见问题
- [ ] `#{}` vs `${}`：`#{}` 预编译防注入，`${}` 直接拼接（仅在动态表名/排序字段时使用并严格校验）
- [ ] Mapper XML 找不到：检查 namespace 路径、检查 build 是否有 xml 被过滤
  ```xml
  <!-- pom.xml 确保 resources 目录的 xml 被复制 -->
  <resource>
      <directory>src/main/java</directory>
      <includes>
          <include>**/*.xml</include>
      </includes>
  </resource>
  ```
- [ ] 字段映射问题：开启驼峰映射（`map-underscore-to-camel-case: true`）

### 18.4 性能陷阱
- [ ] 循环中查询数据库（N+1）：改为批量查询或 JOIN 查询
- [ ] 大事务：RPC 调用、文件 IO 不应放在事务中
- [ ] 不合理的锁粒度：能用行锁不用表锁
- [ ] Redis 大 Key：单 Key 不宜超过 10KB，集合不超过 5000 个元素

### 18.5 并发陷阱
- [ ] SimpleDateFormat 线程不安全：用 DateTimeFormatter / ThreadLocal 包装
- [ ] HashMap 并发死循环：用 ConcurrentHashMap
- [ ] @Autowired 异步线程：子线程无法获取 Request 作用域 Bean

---

## 十九、源码阅读指南

### 19.1 必读源码清单
- [ ] SpringApplication.run() 启动流程
- [ ] AutoConfigurationImportSelector.selectImports()（自动配置加载）
- [ ] AbstractApplicationContext.refresh()（容器刷新核心）
- [ ] DispatcherServlet.doDispatch()（请求分发）
- [ ] AbstractAutowireCapableBeanFactory.doCreateBean()（Bean 创建）
- [ ] CglibAopProxy.getProxy()（AOP 代理创建）
- [ ] TransactionInterceptor.invoke()（事务拦截器）
- [ ] MyBatis MapperProxy.invoke()（Mapper 代理）

### 19.2 源码阅读技巧
- [ ] 带着问题看源码（不要漫无目的）
- [ ] 画时序图辅助理解调用链路
- [ ] Debug 断点跟踪关键流程
- [ ] 先看核心流程，忽略分支细节
- [ ] 关注 Spring 中大量使用的设计模式
- [ ] 做好笔记，定期回顾

### 19.3 阅读源码的 IDE 技巧
- [ ] 查看类继承体系（Ctrl+H / Cmd+U）
- [ ] 查找实现类（Ctrl+Alt+B）
- [ ] 断点条件设置
- [ ] 表达式求值（Alt+F8）
- [ ] 调用栈分析

---

## 二十、持续进阶路线图

### 第一阶段：入门（第 1-2 周）
- [ ] 完成第一个 Spring Boot RESTful API 项目
- [ ] 理解 IoC 容器与 Bean 生命周期
- [ ] 掌握 @SpringBootApplication 三合一注解
- [ ] 学会 MyBatis-Plus 基本 CRUD
- [ ] 配置多环境（dev / test / prod）

### 第二阶段：夯实（第 3-6 周）
- [ ] 深入理解自动配置原理
- [ ] 掌握事务传播与隔离级别
- [ ] 集成 Redis 实现缓存
- [ ] 实现 JWT 认证与权限控制
- [ ] 统一异常处理与返回格式
- [ ] 手写一个自定义 Starter

### 第三阶段：进阶（第 7-12 周）
- [ ] Spring Cloud 微服务全家桶实战
- [ ] 分布式事务方案设计与实现
- [ ] 消息队列完整集成（RabbitMQ / Kafka）
- [ ] 性能压测与 JVM 调优
- [ ] 容器化与 K8s 部署
- [ ] API 网关 + 限流 + 熔断

### 第四阶段：专家（持续）
- [ ] Spring / Spring Boot 源码深度阅读
- [ ] 复杂分布式系统架构设计
- [ ] 中间件选型与治理
- [ ] 生产故障排查经验积累
- [ ] 性能优化体系化方法论
- [ ] 团队技术规范制定与推广
- [ ] 开源贡献与技术布道

---

## 📚 推荐学习资源

### 官方文档（必读）
- [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)
- [Spring Framework Documentation](https://docs.spring.io/spring-framework/reference/)
- [MyBatis-Plus 官方文档](https://baomidou.com/)

### 经典书籍
- 《Spring 实战》(Spring in Action) — Craig Walls
- 《Spring 揭秘》— 王福强（Spring 原理层面讲得最好之一）
- 《高性能 MySQL》— 数据库调优必读
- 《深入理解 Java 虚拟机》(JVM) — 周志明

### 优质博客与社区
- [Spring 官方博客](https://spring.io/blog)
- [MacroZheng 的 Spring Cloud 实战](https://github.com/macrozheng/mall) — GitHub 高星项目
- [JavaGuide](https://javaguide.cn/) — Java 知识体系
- [美团技术团队博客](https://tech.meituan.com/) — 性能优化案例
- [阿里中间件博客](http://jm.taobao.org/) — 分布式架构实践

### 实战项目推荐
- [ ] mall（电商系统）：Spring Boot + MyBatis + Redis + RabbitMQ + Spring Security
- [ ] RuoYi-Vue（后台管理系统）：Spring Boot + Vue3
- [ ] JeecgBoot（低代码平台）：Spring Boot + MyBatis-Plus + Spring Cloud
- [ ] pig（微服务架构）：Spring Cloud 全家桶 + OAuth2

---

## 🏆 学习检验标准

在每个阶段结束时，用以下 checklist 检验掌握程度：

### 入门检验
- [ ] 不查资料，独立搭建 Spring Boot + MyBatis + MySQL 项目
- [ ] 完成 User 表的 CRUD + 分页查询 API
- [ ] 能说出 @SpringBootApplication 包含哪三个注解

### 夯实检验
- [ ] 能画出 Spring Boot 自动配置流程图
- [ ] 能手写一个自定义 Starter
- [ ] 解决过一次事务失效问题
- [ ] 完整实现 JWT 认证方案

### 进阶检验
- [ ] 独立设计一个微服务拆分方案
- [ ] 解决过分布式事务场景
- [ ] 完成至少一次性能压测与优化
- [ ] 成功将一个项目部署到 K8s

### 专家检验
- [ ] 读过 Spring 核心源码的 50%+ 关键路径
- [ ] 能指导团队解决复杂生产问题
- [ ] 有过技术分享输出（博客/演讲/内训）
- [ ] 对 3 个以上中间件源码有深度理解

---

> **最后提醒**：Spring Boot 是工具，不是目标。真正有价值的是你用它解决了什么业务问题、构建了什么系统。动手实践 > 死记硬背，理解原理 > 记忆配置。

---

*最后更新：2026年7月18日*
