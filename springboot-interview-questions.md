# Spring Boot 面试题深度清单

> 一份覆盖从基础到架构的 Spring Boot 面试题全集，每题附详细答案要点与考察意图分析。

---

## 📋 目录

- [一、Spring 基础（必问）](#一spring-基础必问)
- [二、Spring Boot 核心原理（高频）](#二spring-boot-核心原理高频)
- [三、自动配置与 Starter（高频）](#三自动配置与-starter高频)
- [四、Web 开发（高频）](#四web-开发高频)
- [五、数据访问与事务（必问）](#五数据访问与事务必问)
- [六、MyBatis / MyBatis-Plus（国内必问）](#六mybatis--mybatis-plus国内必问)
- [七、JPA（外企/大型项目常问）](#七jpa外企大型项目常问)
- [八、Redis 缓存（高频）](#八redis-缓存高频)
- [九、消息队列（中高频率）](#九消息队列中高频率)
- [十、安全与认证（高频）](#十安全与认证高频)
- [十一、微服务与 Spring Cloud（高级岗位必问）](#十一微服务与-spring-cloud高级岗位必问)
- [十二、性能调优（高级岗位必问）](#十二性能调优高级岗位必问)
- [十三、设计模式与架构（架构师方向）](#十三设计模式与架构架构师方向)
- [十四、场景设计题](#十四场景设计题)
- [十五、源码理解题](#十五源码理解题)
- [十六、综合高难度题](#十六综合高难度题)
- [十七、面试软技能题](#十七面试软技能题)
- [十八、面试准备清单](#十八面试准备清单)

---

## 一、Spring 基础（必问）

### 1.1 IoC 与 DI

**Q1：什么是 IoC？Spring 是如何实现 IoC 的？**

> 考察意图：理解控制反转的本质，是否只停留在"把 new 交给 Spring"的表面认知。

**答案要点**：
- IoC（控制反转）是一种设计思想：将对象的创建、依赖管理、生命周期控制权从程序代码转移给外部容器
- DI（依赖注入）是 IoC 的具体实现方式
- Spring IoC 容器通过 `BeanFactory` / `ApplicationContext` 管理所有 Bean
- 容器启动时扫描配置、创建 BeanDefinition → 实例化 Bean → 注入依赖 → 初始化
- 底层用 `ConcurrentHashMap` 存储单例 Bean（`singletonObjects`）
- IoC 带来的好处：解耦、可测试、声明式事务、AOP 支持

---

**Q2：BeanFactory 和 ApplicationContext 的区别？**

> 考察意图：对 Spring 容器继承体系的理解深度。

**答案要点**：

| 维度 | BeanFactory | ApplicationContext |
|------|-------------|-------------------|
| 层级 | 顶层接口，基础设施 | 继承 BeanFactory，功能更全 |
| 实例化时机 | 延迟加载（第一次 getBean 才创建） | 预加载（容器启动时创建所有单例 Bean） |
| 功能 | 仅提供基本 DI | + AOP、事件发布、国际化、资源加载 |
| 常用实现 | XmlBeanFactory（已废弃） | ClassPathXmlApplicationContext / AnnotationConfigApplicationContext |

---

**Q3：描述 Bean 的完整生命周期？**

> 考察意图：对 Spring 内部运作机制的深入理解，高级岗位高频出现。

**答案要点**（按顺序背出）：

```
1. 实例化（Instantiation）：反射调用构造方法创建对象
2. 属性赋值（Populate Properties）：@Autowired / @Value 注入
3. Aware 接口回调：
   - BeanNameAware.setBeanName()
   - BeanFactoryAware.setBeanFactory()
   - ApplicationContextAware.setApplicationContext()
4. BeanPostProcessor.postProcessBeforeInitialization()
5. @PostConstruct 注解方法
6. InitializingBean.afterPropertiesSet()
7. 自定义 init-method（@Bean(initMethod = "...") 或 XML init-method）
8. BeanPostProcessor.postProcessAfterInitialization()
   → 此处可能产生 AOP 代理对象，替换原始 Bean
9. Bean 就绪，可供使用
10. 容器关闭时销毁：
    - @PreDestroy 注解方法
    - DisposableBean.destroy()
    - 自定义 destroy-method
```

> 加分亮点：补充 AOP 代理对象在 `postProcessAfterInitialization` 阶段生成，解释为何同类方法调用 AOP 失效。

---

**Q4：Spring 中 Bean 的作用域有哪些？**

**答案要点**：
- **singleton**（默认）：整个容器中只有一个实例
- **prototype**：每次获取都创建新实例（注意：容器不管理 prototype Bean 的销毁）
- **request**：每个 HTTP 请求一个实例（Web 环境）
- **session**：每个 HTTP Session 一个实例
- **application**：ServletContext 生命周期对应
- **websocket**：WebSocket 生命周期对应
- 自定义作用域：实现 `Scope` 接口

> 追问：prototype Bean 中的销毁方法为什么不会被调用？怎么处理？  
> 答：Spring 只管理 singleton 的完整生命周期。prototype Bean 创建后交给调用者，容器不再持有引用。需要手动调用销毁逻辑，或使用 `CustomScopes` + 自定义 Scope 管理。

---

**Q5：@Autowired 和 @Resource 的区别？**

**答案要点**：

| 维度 | @Autowired | @Resource |
|------|-----------|-----------|
| 来源 | Spring 注解 | JSR-250（JDK 内置） |
| 注入方式 | 默认 byType | 默认 byName |
| 搭配使用 | @Qualifier 指定名称 | name 属性指定名称 |
| required | 默认 true（找不到抛异常） | 无此属性 |
| 支持位置 | 构造器、方法、字段、参数 | 字段、setter 方法 |

> 追问：如果一个接口有多个实现类，@Autowired 如何选择？  
> 答：1) @Primary 标注首选；2) @Qualifier 指定 Bean 名称；3) 字段名匹配 Bean 名称（byType 失败后降级 byName）。

---

### 1.2 AOP

**Q6：什么是 AOP？Spring AOP 的底层实现是什么？**

> 考察意图：AOP 是声明式事务、日志、权限控制的基础，高级开发必须掌握。

**答案要点**：
- AOP（面向切面编程）：将横切关注点（日志、事务、权限等）从业务代码中分离
- 核心概念：Aspect（切面）、JoinPoint（连接点）、Advice（通知）、Pointcut（切入点）、Weaving（织入）
- Spring AOP 底层使用**动态代理**：
  - 目标类实现了接口 → **JDK 动态代理**（`Proxy.newProxyInstance`，基于接口）
  - 目标类没有接口 → **CGLIB 代理**（字节码增强，基于继承生成子类）
- Spring Boot 2.x 开始默认使用 CGLIB（`spring.aop.proxy-target-class=true`）
- 与 AspectJ 的区别：Spring AOP 是运行时织入、只支持方法级别；AspectJ 编译时/类加载时织入、更强大

---

**Q7：JDK 动态代理和 CGLIB 代理的详细区别？**

**答案要点**：

| 维度 | JDK 动态代理 | CGLIB |
|------|-------------|-------|
| 原理 | 基于接口反射生成代理类 | 基于继承，通过 ASM 字节码生成子类 |
| 要求 | 目标类必须实现接口 | 目标类不能是 final（final 类无法继承） |
| 方法限制 | final 方法可代理（接口中的方法） | final/static 方法无法代理 |
| 性能 | 创建快，运行稍慢（反射调用） | 创建慢（字节码生成），运行快（直接调用） |
| 内存 | 轻量 | 生成子类字节码，占用稍大 |

---

**Q8：AOP 有哪些通知类型？执行顺序是怎样的？**

**答案要点**：
```java
@Around     // 环绕通知（最强大，手动控制执行）
  → @Before   // 前置通知
    → 目标方法执行
  → @AfterReturning  // 返回通知（正常返回后）
  → @After           // 后置通知（类似 finally，无论是否异常都执行）
→ @AfterThrowing   // 异常通知（抛出异常时）
```

> 追问：@Around 和 @Before/@After 组合使用时的执行顺序？  
> 答：如果有同一切面的 @Around，则 @Before/@After 等不会单独执行，而是由 @Around 中 `proceed()` 控制。多个切面的顺序由 @Order 或 Ordered 接口控制。

---

**Q9：Spring AOP 什么时候会失效？列举常见场景。**

> 考察意图：实际项目经验，能否排查 AOP 相关问题。

**答案要点**：
1. **同类方法调用**（最常见）：`this.methodB()` 调用的是原始对象而非代理对象 → 解决方案：注入自己 / `AopContext.currentProxy()` / 拆到不同类
2. **方法非 public**：默认 JDK/CGLIB 代理只能代理 public 方法
3. **final 方法**：CGLIB 基于继承，final 方法无法重写
4. **Bean 未被 Spring 管理**：自己 new 的对象
5. **Aspect 类没有被 @Component/@Configuration 管理**
6. **@Pointcut 表达式写错**，目标方法没被匹配到
7. **内部调用者被 final 修饰**：CGLIB 无法创建子类

---

### 1.3 事务

> 事务部分详见 [五、数据访问与事务](#五数据访问与事务必问)。

---

## 二、Spring Boot 核心原理（高频）

**Q10：@SpringBootApplication 注解背后做了什么？**

> 考察意图：这是面试必问题，考察能否透过注解看到本质。

**答案要点**：它是一个组合注解，包含：

1. `@SpringBootConfiguration`（≈ `@Configuration`）
   - 标注当前类是配置类，可以注册 Bean
2. `@EnableAutoConfiguration`
   - 核心是 `@Import(AutoConfigurationImportSelector.class)`
   - 通过 `SpringFactoriesLoader` 读取 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（Spring Boot 3.x）或 `spring.factories`（2.x）
   - 加载所有自动配置类，通过条件注解过滤后注册
3. `@ComponentScan`
   - 扫描当前包及子包下的 @Component/@Service/@Repository/@Controller
   - 默认扫描范围是主类所在包及子包

---

**Q11：Spring Boot 的启动流程是怎样的？**

> 考察意图：高频题，源码理解深度的分水岭。

**答案要点**（核心 8 步）：

```java
// 入口
SpringApplication.run(Application.class, args);

// 内部流程：
1. new SpringApplication(primarySources)
   - 推断应用类型（SERVLET / REACTIVE / NONE）
   - 设置 Initializers 和 Listeners（从 spring.factories 加载）
   - 推断主类

2. run(args)
   a. 创建 StopWatch 计时
   b. 准备 Environment（加载配置）
   c. 打印 Banner
   d. 创建 ApplicationContext（根据类型：AnnotationConfigServletWebServerApplicationContext）
   e. prepareContext：执行 Initializers、加载主类为 BeanDefinition
   f. refreshContext：调用 AbstractApplicationContext.refresh()
      - prepareRefresh
      - obtainFreshBeanFactory
      - prepareBeanFactory
      - postProcessBeanFactory
      - invokeBeanFactoryPostProcessors  ← 加载自动配置类
      - registerBeanPostProcessors
      - initMessageSource
      - initApplicationEventMulticaster
      - onRefresh  ← 启动嵌入式 Web 服务器（Tomcat/Jetty）
      - registerListeners
      - finishBeanFactoryInitialization  ← 实例化所有单例 Bean
      - finishRefresh
   g. afterRefresh（空方法，扩展点）
   h. 启动完成，触发 ApplicationReadyEvent
```

> 加分亮点：说出 `refresh()` 是 Spring 框架最核心的方法，一切功能都在这里组装完成。

---

**Q12：Spring Boot 相比 Spring Framework 解决了什么问题？**

**答案要点**：
1. **依赖管理**：starter 机制一键引入全套依赖，版本由 parent 统一管控
2. **自动配置**：按约定自动配置 DataSource、Redis、MQ、Web 容器等，零 XML 配置
3. **嵌入式容器**：内置 Tomcat/Jetty/Undertow，打 jar 包直接 `java -jar` 运行
4. **生产就绪**：Actuator 提供健康检查、指标、监控端点
5. **外部化配置**：17 级配置优先级 + 类型安全的 @ConfigurationProperties
6. **开发体验**：devtools 热部署、自动重启

---

**Q13：Spring Boot 如何做到"约定优于配置"？**

**答案要点**：
- 自动配置类提供默认行为（如自动配置 DataSource、自动配置 Jackson）
- 用户只需在配置文件或创建自己的 Bean 来覆盖默认行为
- `@ConditionalOnMissingBean` 使得"你有我不用，你没有我顶上"成为可能
- starter 依赖决定了类路径下有什么，自动配置据此决定做什么

---

## 三、自动配置与 Starter（高频）

**Q14：Spring Boot 自动配置的原理是什么？画一下流程图。**

> 考察意图：能否把自动配置加载链路完整讲清楚。

**答案要点**：

```
@SpringBootApplication
  └─ @EnableAutoConfiguration
       └─ @Import(AutoConfigurationImportSelector.class)
            └─ selectImports()
                 └─ getAutoConfigurationEntry()
                      └─ getCandidateConfigurations()
                           └─ SpringFactoriesLoader.loadFactoryNames()
                                └─ 读取 META-INF/spring/org.springframework.boot.
                                   autoconfigure.AutoConfiguration.imports
                                     (Spring Boot 3.x 新位置)
                                或 META-INF/spring.factories (2.x 旧位置)
                                      ↓
                              拿到约 150+ 个自动配置类全限定名
                                      ↓
                      └─ removeDuplicates() + filter(条件注解)
                           @ConditionalOnClass     - 类存在才生效
                           @ConditionalOnBean      - Bean 存在才生效
                           @ConditionalOnMissingBean - Bean 不存在才生效
                           @ConditionalOnProperty  - 配置属性满足才生效
                           @ConditionalOnWebApplication - Web 环境才生效
                                      ↓
                              过滤后得到最终要注册的配置类
                                      ↓
                              注册到容器 → 自动配置生效
```

> 追问：为什么你的项目启动后自动配了 DataSource？  
> 答：因为 pom.xml 中有 `spring-boot-starter-jdbc` 或 `mybatis-spring-boot-starter`，引入了 DataSource 相关类 → `DataSourceAutoConfiguration` 的 `@ConditionalOnClass({DataSource.class, EmbeddedDatabaseType.class})` 条件满足 → 自动配置生效 → 读取 `spring.datasource.*` 配置 → 创建 DataSource Bean。

---

**Q15：如何手写一个自定义 Starter？**

> 考察意图：中高级岗位常见动手题，考察对自动配置机制的实际掌握。

**答案要点**（完整步骤）：

```
1. 创建两个模块：
   xxx-spring-boot-autoconfigure  （自动配置模块）
   xxx-spring-boot-starter        （starter 模块，空项目引入 autoconfigure）

2. autoconfigure 模块：
   a. 创建配置属性类：
      @ConfigurationProperties(prefix = "xxx.client")
      public class XxxProperties {
          private String host = "localhost";
          private int port = 8080;
          // getter/setter
      }
   
   b. 创建核心服务类（你要提供的功能）
   
   c. 创建自动配置类：
      @Configuration
      @EnableConfigurationProperties(XxxProperties.class)
      @ConditionalOnClass(XxxService.class)
      @ConditionalOnProperty(prefix = "xxx.client", name = "enabled", 
                             havingValue = "true", matchIfMissing = true)
      public class XxxAutoConfiguration {
          @Bean
          @ConditionalOnMissingBean
          public XxxService xxxService(XxxProperties properties) {
              return new XxxService(properties);
          }
      }
   
   d. 在 src/main/resources/META-INF/spring/ 下创建文件：
      org.springframework.boot.autoconfigure.AutoConfiguration.imports
      内容：
      com.example.xxx.autoconfigure.XxxAutoConfiguration
   
   e. 在 pom.xml 中添加：
      <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-configuration-processor</artifactId>
          <optional>true</optional>
      </dependency>

3. starter 模块 pom.xml 引入 autoconfigure 模块即可
```

---

**Q16：条件注解有哪些？@ConditionalOnBean 和 @ConditionalOnMissingBean 的使用场景？**

**答案要点**：
- `@ConditionalOnClass`：类路径存在指定类
- `@ConditionalOnMissingClass`：类路径不存在指定类
- `@ConditionalOnBean`：容器中存在指定 Bean
- `@ConditionalOnMissingBean`：容器中不存在指定 Bean（**最常用**）
- `@ConditionalOnProperty`：配置属性满足条件
- `@ConditionalOnResource`：指定资源文件存在
- `@ConditionalOnWebApplication`：是 Web 应用
- `@ConditionalOnExpression`：SpEL 表达式成立
- `@ConditionalOnJava`：Java 版本满足条件
- `@ConditionalOnSingleCandidate`：容器中只有一个候选 Bean

使用场景举例：
- `@ConditionalOnMissingBean`：用户自定义了则用用户的，否则用默认配置
- `@ConditionalOnClass`：引入 Redis 依赖才启用 Redis 配置，否则不报错

---

## 四、Web 开发（高频）

**Q17：Spring MVC 处理请求的完整流程是怎样的？**

> 考察意图：高频必考题，能否把 DispatcherServlet 的请求处理链路说清楚。

**答案要点**：

```
HTTP 请求
  ↓
Filter Chain（过滤器链）
  ↓
DispatcherServlet
  ↓
1. HandlerMapping 查找处理器
   - RequestMappingHandlerMapping（处理 @RequestMapping）
   - 根据 URL 找到 HandlerMethod + 拦截器链
  ↓
2. HandlerAdapter 调用处理器
   - RequestMappingHandlerAdapter
   - 参数解析（HandlerMethodArgumentResolver）：@PathVariable、@RequestParam、@RequestBody...
   - 数据绑定与校验（@Validated）
   - 调用目标 Controller 方法
   - 返回值处理（HandlerMethodReturnValueHandler）：@ResponseBody → HttpMessageConverter 转 JSON
  ↓
3. HandlerInterceptor 执行
   - preHandle → Controller → postHandle → afterCompletion
  ↓
4. 异常处理（如果有异常）
   - @ExceptionHandler → @ControllerAdvice 全局异常处理
  ↓
5. 视图解析（如果是 @Controller 返回视图名）
   - ViewResolver 解析
  ↓
6. HTTP 响应
```

> 加分亮点：能说出 `HttpMessageConverter` 根据请求头 `Accept` / `Content-Type` 自动选择合适的转换器（如 Jackson → JSON）。

---

**Q18：Spring Boot 如何处理跨域（CORS）？**

**答案要点**：
1. `@CrossOrigin` 注解：粒度最细，可放类/方法上
2. 全局 CORS 配置：
```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://www.example.com")  // !禁止用 *
                .allowedMethods("GET","POST","PUT","DELETE")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```
3. CorsFilter（更底层、优先级最高）
4. Spring Security 中还需要额外配置 `.cors()`

---

**Q19：Filter 和 Interceptor 的区别？执行顺序？**

**答案要点**：

| 维度 | Filter | Interceptor |
|------|--------|-------------|
| 规范 | Servlet 规范 | Spring MVC 框架 |
| 依赖注入 | 无法直接注入 Bean（需 delegate） | 天然支持 @Autowired |
| 拦截范围 | 所有进入 Servlet 的请求 | 只拦截进入 DispatcherServlet 后的请求 |
| 颗粒度 | 只能拿到 Request/Response | 能拿到 HandlerMethod 对象 |
| 执行顺序 | Filter → Interceptor → Aspect |

```
请求 → Filter1 → Filter2 → DispatcherServlet → Interceptor1.preHandle
     → Interceptor2.preHandle → Controller → Interceptor2.postHandle
     → Interceptor1.postHandle → 视图渲染
     → Interceptor2.afterCompletion → Interceptor1.afterCompletion → Filter2 → Filter1 → 响应
```

> 为什么 Filter 可以拦截静态资源而 Interceptor 不行？  
> 因为静态资源请求不一定经过 DispatcherServlet（取决于配置）。

---

**Q20：Spring Boot 统一异常处理怎么做？**

**答案要点**：
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    // 处理参数校验异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Result<?> handleValidation(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors()
                       .stream()
                       .map(err -> err.getField() + ": " + err.getDefaultMessage())
                       .collect(Collectors.joining("; "));
        return Result.error(400, msg);
    }
    
    // 处理业务异常
    @ExceptionHandler(BusinessException.class)
    public Result<?> handleBusiness(BusinessException e) {
        return Result.error(e.getCode(), e.getMessage());
    }
    
    // 兜底处理
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Result<?> handleUnknown(Exception e) {
        log.error("未捕获异常", e);
        return Result.error(500, "服务器内部错误");
    }
}
```

> 追问：多个 @ExceptionHandler 匹配时的优先级？  
> 答：精确匹配优先（子类优于父类），同级别按声明顺序。通常把精确异常放前面，`Exception.class` 放最后兜底。

---

**Q21：@RestController 和 @Controller 的区别？**

**答案要点**：
- `@RestController` = `@Controller` + `@ResponseBody`
- `@Controller` 方法返回字符串默认解析为视图名（走 ViewResolver）
- `@RestController` 所有方法返回值自动序列化为 JSON（走 HttpMessageConverter）
- RESTful API 开发统一用 @RestController

---

**Q22：@RequestBody 和 @RequestParam 的区别？什么时候用哪个？**

**答案要点**：

| 维度 | @RequestBody | @RequestParam |
|------|-------------|---------------|
| 数据来源 | HTTP 请求体（Body） | URL 查询参数（Query String）或表单 |
| 内容类型 | application/json 为主 | application/x-www-form-urlencoded |
| 数据格式 | 自动反序列化为对象 | 单个字段 |
| 使用场景 | POST/PUT 提交复杂对象 | GET 请求过滤、分页参数 |

> 追问：@RequestParam 的 required 默认是什么？  
> 答：默认 `required=true`，参数不存在抛 MissingServletRequestParameterException。

---

**Q23：Spring Boot 有哪些常用的 HttpMessageConverter？**

**答案要点**：
- `MappingJackson2HttpMessageConverter`：JSON 转换（ObjectMapper）
- `StringHttpMessageConverter`：String 类型
- `ByteArrayHttpMessageConverter`：字节数组
- `FormHttpMessageConverter`：表单数据
- `ResourceHttpMessageConverter`：资源文件
- 选择逻辑：根据请求 `Content-Type` 和响应 `Accept` 头匹配

---

## 五、数据访问与事务（必问）

**Q24：@Transactional 失效的场景有哪些？**

> 考察意图：必问题，每个场景都是实际开发中的坑。

**答案要点**（8 大失效场景）：

| 场景 | 原因 | 解决方案 |
|------|------|----------|
| 1. 非 public 方法 | Spring AOP 默认只代理 public 方法 | 改为 public |
| 2. 同类方法调用 | `this.method()` 绕过了代理对象 | 注入自身 / `AopContext.currentProxy()` / 拆分到另一个类 |
| 3. 异常被 catch 吞掉 | 代理感知不到异常 | catch 中重新抛出或手动回滚 |
| 4. rollbackFor 设置错误 | 默认只回滚 RuntimeException 和 Error | 指定 `rollbackFor = Exception.class` |
| 5. 数据库引擎不支持 | MyISAM 不支持事务 | 使用 InnoDB |
| 6. 多线程调用 | 子线程不受当前事务管理 | 每个子线程独立开事务 |
| 7. @Transactional 加在接口默认方法上 | CGLIB 代理无法拦截接口默认方法 | 写在实现类上 |
| 8. propagation 设置不当 | 如 NOT_SUPPORTED / NEVER | 检查传播行为是否合理 |

---

**Q25：Spring 事务的传播行为有哪些？分别什么场景用？**

> 考察意图：中高级岗位高频题。

**答案要点**：

| 传播行为 | 说明 | 典型场景 |
|----------|------|----------|
| **REQUIRED**（默认） | 有事务则加入，无则新建 | 大多数业务方法（增删改） |
| **REQUIRES_NEW** | 始终新建独立事务，挂起当前 | 日志记录（失败不应影响主业务） |
| **SUPPORTS** | 有事务则加入，无则以非事务运行 | 查询方法（有事务也行，没有也可以） |
| **NOT_SUPPORTED** | 以非事务运行，挂起当前事务 | 不需要事务的耗时操作 |
| **MANDATORY** | 必须在事务中，否则抛异常 | 强制要求调用者开启事务 |
| **NEVER** | 必须非事务，否则抛异常 | 明确禁止在事务中执行 |
| **NESTED** | 嵌套事务（保存点机制） | 部分回滚场景 |

> 追问：REQUIRED 和 REQUIRES_NEW 的详细区别？  
> 答：REQUIRED 加入外层事务，外层回滚则内层也回滚；REQUIRES_NEW 是独立事务，外层回滚不影响内层提交。

> 追问：NESTED 和 REQUIRES_NEW 的区别？  
> 答：NESTED 是嵌套事务，内层回滚只回滚到保存点，外层提交时一起提交；REQUIRES_NEW 是完全独立的两个物理事务。

---

**Q26：事务的隔离级别有哪些？MySQL 默认是什么？**

**答案要点**：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|----------|------|-----------|------|----------|
| READ_UNCOMMITTED | ✓ | ✓ | ✓ | 几乎不用 |
| READ_COMMITTED | ✗ | ✓ | ✓ | 每次读快照 |
| **REPEATABLE_READ** | ✗ | ✗ | ✓（InnoDB 通过 Next-Key Lock 解决了大部分） | MVCC + 间隙锁 |
| SERIALIZABLE | ✗ | ✗ | ✗ | 所有读加共享锁 |

- MySQL InnoDB 默认：**REPEATABLE_READ**
- Oracle/PostgreSQL 默认：**READ_COMMITTED**
- Spring 默认隔离级别：**DEFAULT**（跟随数据库默认）

> 追问：MVCC 是什么？MySQL 如何通过 MVCC 实现 REPEATABLE_READ？  
> 答：MVCC（多版本并发控制），每行数据有隐藏的 `trx_id`（事务ID）和 `roll_pointer`（回滚指针）。ReadView 决定当前事务能看到哪个版本的数据。REPEATABLE_READ 级别下，事务开启时生成 ReadView，整个事务期间不变。

---

**Q27：什么是事务的 ACID 特性？**

**答案要点**：
- **A（Atomicity 原子性）**：事务要么全部成功，要么全部失败回滚（undo log 实现）
- **C（Consistency 一致性）**：事务前后数据满足完整性约束
- **I（Isolation 隔离性）**：并发事务之间互不影响（MVCC + 锁实现）
- **D（Durability 持久性）**：已提交事务的数据永久保存（redo log 实现）

---

**Q28：Spring 事务底层是怎么实现的？**

> 考察意图：源码理解深度。

**答案要点**：
1. 声明式事务基于 **AOP 代理** 实现
2. `@EnableTransactionManagement` 导入 `TransactionManagementConfigurationSelector`
3. 默认使用 **PROXY** 模式，创建 `BeanFactoryTransactionAttributeSourceAdvisor`（一个 Advisor）
4. 拦截器是 `TransactionInterceptor`，在 `invoke()` 方法中：
   - 获取事务属性（传播行为、隔离级别等）
   - `TransactionManager.getTransaction()` 创建/获取事务
   - `invocation.proceed()` 执行业务方法
   - 根据异常类型决定提交或回滚
5. 底层调用 `DataSourceTransactionManager` → `Connection.setAutoCommit(false)` → 业务 SQL → commit/rollback

---

## 六、MyBatis / MyBatis-Plus（国内必问）

**Q29：MyBatis 中 #{} 和 ${} 的区别？为什么 #{} 可以防 SQL 注入？**

> 考察意图：国内面试几乎必问。

**答案要点**：

| 维度 | #{} | ${} |
|------|-----|-----|
| 处理方式 | 预编译占位符 `?` | 直接字符串拼接 |
| SQL 注入 | **安全**（参数化查询） | 不安全 |
| 类型处理 | 自动加引号（字符串类型） | 原样输出 |
| 使用场景 | 绝大多数情况 | 动态表名、动态列名、ORDER BY 字段 |

```sql
-- #{name} 生成：
SELECT * FROM user WHERE name = ?  -- '张三' 作为参数传入

-- ${name} 生成（危险！）：
SELECT * FROM user WHERE name = 张三  -- 直接拼接，可注入
```

> 如果必须用 `${}`（如动态排序），怎么防注入？  
> 答：在代码中对 `${}` 的值做**白名单校验**，只允许预定义的列名和排序方向。

---

**Q30：MyBatis 中 resultType 和 resultMap 的区别？**

**答案要点**：
- `resultType`：直接指定 Java 类型，自动映射（要求数据库字段名和实体属性名一致，或开启驼峰映射）
- `resultMap`：自定义映射规则，适用于：
  - 字段名与属性名不一致
  - 复杂关联查询（一对一 `association`、一对多 `collection`）
  - 需要类型转换器

---

**Q31：MyBatis-Plus 中 LambdaQueryWrapper 比 QueryWrapper 好在哪里？**

**答案要点**：
```java
// QueryWrapper：字符串硬编码字段名，容易写错，重构不安全
wrapper.eq("user_name", "张三");

// LambdaQueryWrapper：方法引用，类型安全，IDE 可提示和重构
wrapper.eq(User::getUserName, "张三");
```
- Lambda 方式利用 **序列化 Lambda** 解析方法引用，运行时获取属性名
- 重构时自动更新，编译期就能发现错误

---

**Q32：MyBatis-Plus 的分页插件是怎么配置的？原理是什么？**

**答案要点**：
```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

原理：
1. 拦截器拦截 SQL 执行前的 `prepare` 阶段
2. 判断是否为分页查询（参数中有 Page 对象）
3. 修改 SQL：在原 SQL 基础上追加 `LIMIT offset, size`
4. 额外执行 `SELECT COUNT(*)` 查询总记录数
5. 将结果封装到 `Page` 对象中

---

**Q33：MyBatis 一级缓存和二级缓存？**

**答案要点**：

| 维度 | 一级缓存 | 二级缓存 |
|------|---------|---------|
| 作用范围 | SqlSession 级别 | Mapper（Namespace）级别 |
| 默认开启 | 是 | 否 |
| 生命周期 | SqlSession 关闭即清空 | 整个应用生命周期 |
| 数据共享 | 同一 SqlSession 内 | 跨 SqlSession（同一 Namespace） |
| 注意事项 | 和 Spring 整合后每次查询新建 SqlSession，一级缓存几乎无效 | 需要实体类实现 Serializable；容易出现脏数据；**生产环境很少启用** |

---

## 七、JPA（外企/大型项目常问）

**Q34：JPA 中 N+1 查询问题是什么？如何解决？**

> 考察意图：ORM 框架的常见性能陷阱，体现优化意识。

**答案要点**：
- N+1 问题：查询 1 个主实体 → 执行 1 条 SQL；然后懒加载每个关联实体 → 额外执行 N 条 SQL
```java
// 假设有 10 个 User，每个 User 有多个 Order
List<User> users = userRepository.findAll();  // 1 条 SQL
for (User user : users) {
    // 每次访问 user.getOrders() 都触发 1 条新 SQL
    user.getOrders().size();  // N 条 SQL → 总共 N+1 条 SQL
}
```

**解决方案**：
1. `@EntityGraph`：声明式指定关联抓取
```java
@EntityGraph(attributePaths = {"orders"})
List<User> findAll();
```
2. `JOIN FETCH`：JPQL 中显式写 JOIN
```java
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```
3. `@BatchSize`：批量加载（Hibernate 扩展）

---

**Q35：JPA 中 FetchType.LAZY 和 EAGER 的区别？**

**答案要点**：

| 维度 | LAZY（懒加载） | EAGER（急加载） |
|------|---------------|----------------|
| 加载时机 | 首次访问关联属性时 | 立即加载 |
| 实现方式 | 返回代理对象 | JOIN 或多次查询 |
| 默认值 | @OneToMany / @ManyToMany 默认为 LAZY | @OneToOne / @ManyToOne 默认为 EAGER |
| 风险 | 事务外访问可能抛 LazyInitializationException | 可能加载不需要的数据 |
| 推荐 | **几乎总是推荐 LAZY** | 仅在确定需要关联数据且数据量小时使用 |

---

**Q36：JPA Entity 的生命周期有哪些状态？**

**答案要点**：

```
New（瞬时态）→ new User()
  ↓ persist() / cascade
Managed（持久态）→ EntityManager 管理，自动脏检查
  ↓ detach() / clear() / close()
Detached（游离态）→ 有 ID 但不再被管理
  ↓ merge()
Managed → 重新管理
  ↓ remove()
Removed（删除态）→ 事务提交后从数据库删除
```

---

## 八、Redis 缓存（高频）

**Q37：缓存穿透、缓存击穿、缓存雪崩分别是什么？怎么解决？**

> 考察意图：Redis 面试最经典的三个问题，必问。

**答案要点**：

| 问题 | 描述 | 解决方案 |
|------|------|----------|
| **缓存穿透** | 查询不存在的数据，缓存和数据库都没有，每个请求打穿缓存直达 DB | ① 布隆过滤器（Bloom Filter）提前拦截<br>② 缓存空值（短过期时间）|
| **缓存击穿** | 热点 Key 过期瞬间，大量请求同时打到 DB | ① 互斥锁（SETNX），只让一个线程重建缓存<br>② 逻辑过期（永不过期，后台异步刷新） |
| **缓存雪崩** | 大量 Key 同时过期，或 Redis 宕机，DB 瞬间压力过大 | ① 过期时间加随机值（避免同时过期）<br>② 多级缓存（本地 Caffeine + Redis）<br>③ Redis 集群/哨兵保证高可用<br>④ 限流降级保底 |

**布隆过滤器原理简述**：
- 用多个 Hash 函数将数据映射到位数组
- 判断存在：所有 Hash 位都是 1 → "可能存在"（有误判率）
- 判断不存在：任意 Hash 位是 0 → "一定不存在"
- 适合缓存穿透场景：先查布隆过滤器，不在则直接返回

---

**Q38：Redis 有哪些数据结构？分别适合什么场景？**

**答案要点**：

| 数据结构 | 常用命令 | 典型场景 |
|----------|----------|----------|
| **String** | SET/GET/SETEX/INCR | 缓存、计数器、分布式锁（SETNX） |
| **Hash** | HSET/HGET/HGETALL | 存储对象（用户信息、商品详情） |
| **List** | LPUSH/RPOP/BRPOP | 消息队列、最新消息列表 |
| **Set** | SADD/SMEMBERS/SINTER | 标签、共同好友、去重 |
| **ZSet** | ZADD/ZRANGE/ZREVRANK | 排行榜、优先级队列、延迟队列 |
| **Bitmap** | SETBIT/GETBIT/BITCOUNT | 签到统计、用户在线状态 |
| **HyperLogLog** | PFADD/PFCOUNT | UV 统计、去重计数（有 0.81% 误差） |
| **GEO** | GEOADD/GEORADIUS | 附近的人、地理位置计算 |
| **Stream** | XADD/XREAD/XGROUP | 消息队列（支持消费者组） |

---

**Q39：Redis 的过期策略和内存淘汰策略？**

**答案要点**：

**过期策略**（如何删除过期 Key）：
- **惰性删除**：访问 Key 时检查是否过期，过期则删除（CPU 友好，内存不友好）
- **定期删除**：每隔一段时间随机抽取一批 Key 检查并删除
- Redis 使用 **惰性删除 + 定期删除** 组合策略

**内存淘汰策略**（内存满后的处理方式）：

| 策略 | 说明 |
|------|------|
| noeviction | 不淘汰，写入报错（默认） |
| allkeys-lru | 所有 Key 中淘汰最近最少使用的 |
| volatile-lru | 有过期时间的 Key 中淘汰 LRU |
| allkeys-lfu | 所有 Key 中淘汰最不频繁使用的（Redis 4.0+） |
| volatile-lfu | 有过期时间的 Key 中淘汰 LFU |
| allkeys-random | 所有 Key 中随机淘汰 |
| volatile-random | 有过期时间的 Key 中随机淘汰 |
| volatile-ttl | 有过期时间的 Key 中淘汰 TTL 最短的 |

> 生产环境推荐：`allkeys-lru`（缓存场景）

---

**Q40：Redis 如何实现分布式锁？有什么注意事项？**

**答案要点**：
```java
// 基本实现
String lockKey = "order:" + orderId;
String lockValue = UUID.randomUUID().toString();  // 唯一标识
boolean locked = redisTemplate.opsForValue()
    .setIfAbsent(lockKey, lockValue, 10, TimeUnit.SECONDS);

// 释放锁（Lua 脚本保证原子性：判断 + 删除）
String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                "return redis.call('del', KEYS[1]) else return 0 end";
redisTemplate.execute(new DefaultRedisScript<>(script, Long.class),
    Collections.singletonList(lockKey), lockValue);
```

**注意事项**：
1. **加锁要设过期时间**：防止死锁。推荐用 `SET key value NX EX seconds` 原子命令
2. **释放锁要校验 value**：防止释放别人的锁（用 Lua 脚本保证原子性）
3. **锁续期（看门狗）**：Redisson 的 `RLock` 自动续期，业务没执行完不会过期
4. **RedLock 算法**：多节点 Redis 场景保证高可用（但业界有争议）

---

**Q41：Redis 和数据库双写一致性如何保证？**

> 考察意图：高级岗位常问，考察对分布式一致性问题的理解。

**答案要点**：

几种策略对比：

| 策略 | 描述 | 一致性 |
|------|------|--------|
| 先删缓存再更新 DB | 并发读写可能读到旧数据 | 弱 |
| 先更新 DB 再删缓存 | 最常用，异常情况需要重试 | 较强 |
| 先更新 DB 再更新缓存 | 并发写可能顺序错乱 | 弱 |
| 延迟双删 | 删缓存 → 更新 DB → 延时再删缓存 | 较强 |
| 订阅 binlog + 异步更新 | Canal 监听 binlog → 更新 Redis | 最终一致 |

**推荐方案**：先更新 DB → 删除缓存（失败则 MQ 重试）+ 监听 binlog 兜底

---

## 九、消息队列（中高频率）

**Q42：消息队列在项目中怎么用的？解决了什么问题？**

> 考察意图：是否真正在生产中使用过 MQ，能否说清楚业务价值。

**答案要点**：
1. **异步处理**：发邮件/短信、生成报表等耗时操作不阻塞主流程
2. **解耦**：订单系统创建订单后，通知库存、物流、积分系统，通过 MQ 解耦
3. **削峰填谷**：秒杀场景，请求先入 MQ，后端按能力消费
4. **数据同步**：多系统间数据最终一致性同步

---

**Q43：如何保证消息不丢失？（生产端 → MQ → 消费端）**

> 考察意图：消息可靠性的全链路思考。

**答案要点**（以 RabbitMQ 为例）：

```
生产者 → Broker → 消费者
```

1. **生产端**：
   - 开启 Publisher Confirm（confirmCallback）
   - 消息持久化到磁盘（deliveryMode=2 / persistent）

2. **Broker 端**：
   - Queue、Exchange 设置为 durable（持久化）
   - 镜像队列（集群模式）
   
3. **消费端**：
   - 关闭自动 ACK，改为**手动确认**（manual ack）
   - 业务处理成功后再 `basicAck`
   - 消费失败 → `basicNack`（requeue 或投递到死信队列）

---

**Q44：如何保证消息不被重复消费？（幂等性）**

**答案要点**：
1. **数据库唯一索引**：插入时用消息 ID 做唯一约束，重复插入自动失败
2. **Redis SETNX**：`SET msg:{id} consumed EX 3600 NX`，成功返回表示首次消费
3. **业务状态机**：如订单状态从"待支付"→"已支付"，重复设置幂等
4. **数据库去重表**：单独的消费记录表，消费前检查

> 为什么会有重复消息？  
> - 生产端：confirmCallback 超时重试，但消息已到 Broker
> - 消费端：消费完但 ACK 网络超时，Broker 重新投递

---

**Q45：消息积压了怎么处理？**

**答案要点**：

**临时紧急处理**（止损优先）：
1. 快速扩容消费者实例数
2. 跳过非关键消息，先处理核心业务
3. 临时提高消费并发度

**预防措施**：
1. 监控队列长度，提前告警
2. 消费者处理能力 > 生产者发送速率
3. 设计降级开关，高峰期可关闭非核心消息

**事后处理**：
1. 新建消费者批量消费积压消息
2. 将积压消息转储到数据库，批量处理
3. 修复消费者性能瓶颈

---

## 十、安全与认证（高频）

**Q46：JWT 的认证流程是怎样的？**

> 考察意图：前后端分离项目的标配，几乎必问。

**答案要点**：

```
客户端                    服务端
  │                        │
  │── ① 登录（用户名+密码）──→│
  │                        │ ② 验证凭据
  │                        │ ③ 生成 JWT（Header.Payload.Signature）
  │←── ④ 返回 JWT ────────│
  │                        │
  │── ⑤ 请求（Header: Authorization: Bearer <JWT>）──→│
  │                        │ ⑥ 过滤器拦截，提取 Token
  │                        │ ⑦ 验证签名 + 有效期
  │                        │ ⑧ 解析用户信息 → SecurityContext
  │←── ⑨ 响应数据 ────────│
```

JWT 结构：
```
Header:  {"alg": "HS256", "typ": "JWT"}
Payload: {"sub": "123", "name": "张三", "iat": 1516239022, "exp": 1516242622}
Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

---

**Q47：Access Token 过期了怎么办？Refresh Token 怎么设计？**

**答案要点**：
```
Access Token：  短期（15-30 分钟），每次请求携带
Refresh Token： 长期（7-30 天），仅用于刷新 Access Token

流程：
1. Access Token 过期 → 返回 401
2. 客户端用 Refresh Token 请求 /refresh 接口
3. 服务端验证 Refresh Token（数据库中对比/Redis 中检查）
4. 生成新的 Access Token 返回
5. 如果 Refresh Token 也过期 → 重新登录
```

**安全考虑**：
- Refresh Token 存储在 httpOnly Cookie 或安全存储
- Refresh Token 可设置单次使用（每次刷新换一个新的）
- 支持 Refresh Token 吊销（用户修改密码后使旧 Token 失效）

---

**Q48：Spring Security 的过滤器链是什么？常用的过滤器有哪些？**

**答案要点**：

核心过滤器链（按顺序）：
```
SecurityContextPersistenceFilter      — 从 Session 恢复 SecurityContext
  ↓
UsernamePasswordAuthenticationFilter  — 处理表单登录
  ↓
BasicAuthenticationFilter             — 处理 HTTP Basic 认证
  ↓
(自定义 JWT 过滤器)                    — 从 Header 提取 Token
  ↓
ExceptionTranslationFilter            — 转换认证/授权异常
  ↓
FilterSecurityInterceptor             — 授权决策（最后一道防线）
```

---

**Q49：RBAC 权限模型是什么？怎么设计数据库表？**

**答案要点**：

```
RBAC：基于角色的访问控制（Role-Based Access Control）

核心表设计：
┌─────────┐     ┌───────────────┐     ┌─────────┐
│  User   │────→│  User_Role   │←────│  Role   │
└─────────┘     └───────────────┘     └─────────┘
                                              │
                                              ↓
              ┌───────────────┐     ┌────────────┐
              │ Role_Permission│←────│ Permission │
              └───────────────┘     └────────────┘
```

五张表（最小模型）：
- `user`：用户表
- `role`：角色表
- `permission`：权限表（菜单、按钮、接口 URL）
- `user_role`：用户-角色关联
- `role_permission`：角色-权限关联

RBAC 的好处：不用为每个用户单独分配权限，将用户归类为角色统一管理。

---

**Q50：CSRF 攻击是什么？Spring Boot 如何防护？**

**答案要点**：
- CSRF（跨站请求伪造）：攻击者诱导用户在已登录的网站执行非预期操作
- **前后端分离项目**（JWT + Stateless）：天然免疫 CSRF，因为没有 Cookie-Session 机制
- **传统 Cookie-Session 项目**：
  - Spring Security 默认开启 CSRF 保护
  - 生成 CSRF Token 嵌入表单
  - `CsrfFilter` 验证请求中的 `_csrf` 参数
- 前后端分离一般 `.csrf().disable()` 关闭

---

## 十一、微服务与 Spring Cloud（高级岗位必问）

**Q51：Spring Cloud 全家桶有哪些组件？各自作用是什么？**

**答案要点**：

| 组件 | 作用 | 替代品 |
|------|------|--------|
| **Nacos / Eureka** | 服务注册与发现 | Consul、Zookeeper |
| **OpenFeign** | 声明式 HTTP 远程调用 | RestTemplate、gRPC |
| **Spring Cloud LoadBalancer** | 客户端负载均衡 | Ribbon（已停更） |
| **Spring Cloud Gateway** | API 网关（路由、过滤、限流） | Zuul（已停更）、Kong |
| **Sentinel** | 服务熔断、限流、降级 | Hystrix（已停更）、Resilience4j |
| **Nacos Config / Apollo** | 配置中心，动态刷新配置 | Spring Cloud Config |
| **Sleuth + Zipkin** | 分布式链路追踪 | SkyWalking、Jaeger |
| **Seata** | 分布式事务 | RocketMQ 事务消息 |

---

**Q52：Spring Cloud Gateway 的路由和过滤器机制？**

**答案要点**：

路由（Route）三要素：
- **id**：路由唯一标识
- **uri**：目标服务地址（`lb://service-name` 走负载均衡）
- **predicates**：断言（匹配规则）
- **filters**：过滤器（修改请求/响应）

```yaml
spring.cloud.gateway.routes:
  - id: user-service
    uri: lb://user-service
    predicates:
      - Path=/api/users/**
      - Method=GET,POST
      - Header=X-Request-Source, mobile
    filters:
      - StripPrefix=1
      - AddRequestHeader=X-Gateway, true
      - name: RequestRateLimiter
        args:
          redis-rate-limiter.replenishRate: 100
          redis-rate-limiter.burstCapacity: 200
```

Gateway 基于 **WebFlux + Netty** 实现，非阻塞 I/O，性能优于 Zuul 1.x。

---

**Q53：Sentinel 的熔断降级策略有哪些？**

**答案要点**：

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| **慢调用比例** | 响应时间超过阈值的调用比例超过设定值 → 熔断 | 依赖服务变慢 |
| **异常比例** | 异常数占总调用数的比例超过阈值 → 熔断 | 依赖服务异常 |
| **异常数** | 异常数量超过阈值 → 熔断 | 依赖服务不可用 |

熔断状态机：
```
CLOSED（正常）→ 失败达阈值 → OPEN（熔断，拒绝请求）
                              ↓ 等待时间窗口过后
                       HALF-OPEN（尝试放行少量请求）
                         ↙           ↘
                    成功 → CLOSED    失败 → OPEN
```

---

**Q54：什么是 CAP 定理？在微服务中如何权衡？**

> 考察意图：分布式系统基础理论，高级岗位几乎必问。

**答案要点**：
- **C（Consistency 一致性）**：所有节点在同一时间看到相同数据
- **A（Availability 可用性）**：每次请求都能得到响应（不保证数据最新）
- **P（Partition Tolerance 分区容错）**：网络分区时系统仍能工作
- **CAP 不能三者兼得**，网络分区不可避免，必须在 C 与 A 之间权衡

微服务中的取舍：
- **注册中心**：Eureka 选 AP，Nacos 可切换 CP/AP（服务发现选 AP，配置中心选 CP）
- **分布式事务**：强一致性选 CP（Seata AT 模式），可用性优先选 AP（最终一致性 + 补偿）
- 互联网业务通常是 **AP + 最终一致性**（BASE 理论）

---

**Q55：服务间的调用怎么保证数据一致性？**

> 考察意图：分布式事务方案选型能力。

**答案要点**：

**方案对比**：

| 方案 | 原理 | 一致性 | 性能 | 复杂度 |
|------|------|--------|------|--------|
| **Seata AT** | 自动生成回滚 SQL | 强一致 | 较好 | 中 |
| **TCC** | Try-Confirm-Cancel，需手动实现 | 强一致 | 好 | 高 |
| **Saga** | 正向操作 + 补偿操作链 | 最终一致 | 好 | 中 |
| **可靠消息** | 本地事务表 + MQ 异步 | 最终一致 | 最好 | 中 |

**推荐**：大多数场景用**可靠消息最终一致性**（性能最好）；对一致性要求高的用 Seata AT。

---

## 十二、性能调优（高级岗位必问）

**Q56：你做过哪些性能优化？思路是什么？**

> 考察意图：考察性能优化的系统化方法论，而非零散技巧。

**答案要点**（自上而下的优化思路）：

```
业务层优化：
  1. 减少不必要的请求（前端去重、批量接口）
  
架构层优化：
  2. 缓存（浏览器 → CDN → 本地缓存 → Redis → DB）
  3. 异步化（非核心流程用消息队列解耦）
  4. 读写分离 + 分库分表
  
应用层优化：
  5. 减少数据库查询（批量查询代替 N+1、合理 JOIN）
  6. 索引优化（慢 SQL → EXPLAIN → 加索引/改写 SQL）
  7. 连接池调优（HikariCP 配置）
  8. 序列化优化（Protobuf 替代 JSON）
  
JVM 层优化：
  9. GC 调优（选择合适 GC、调整堆大小）
  10. 内存泄漏排查（Arthas / MAT 分析）
```

**具体案例话术**：
"之前项目中有一个商品列表接口响应时间 2s+，排查发现：
1. 每个商品循环查库存（N+1 问题）→ 改为批量查询，减少 95% SQL
2. 图片信息每次查询外部服务 → 加 Redis 缓存，TTL 1 小时
3. 最终接口从 2s 降到 50ms 以内。"

---

**Q57：SQL 慢查询如何排查和优化？**

**答案要点**：

**排查流程**：
```sql
-- 1. 确认慢 SQL
SHOW FULL PROCESSLIST;
-- 或：开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;

-- 2. 分析执行计划
EXPLAIN SELECT * FROM order WHERE user_id = 123 AND status = 1;
-- 重点看：type（ALL → index → range → ref → eq_ref → const）
--         rows（扫描行数）
--         Extra（Using filesort / Using temporary 是坏信号）

-- 3. 分析索引使用
SHOW INDEX FROM order;
```

**常见优化手段**：
- 添加合适的索引（最左前缀原则）
- 避免 SELECT *，只查需要的字段
- 避免索引失效：WHERE 条件不用函数、不进行隐式类型转换
- 大数据分页优化：使用游标分页替代 OFFSET 大偏移量
- 大表 JOIN 优化：先过滤再 JOIN

---

**Q58：接口 QPS 从 100 提升到 1000 需要关注哪些方面？**

**答案要点**：
1. **代码层面**：消除 N+1、减少锁粒度、对象复用
2. **缓存层面**：热点数据提前预热到 Redis、本地缓存
3. **数据库层面**：连接池扩容、读写分离、必要时分库分表
4. **JVM 层面**：调大堆内存、选择合适 GC（G1GC/ZGC）
5. **容器层面**：Tomcat 线程数调优（`server.tomcat.threads.max=500`）
6. **架构层面**：水平扩展实例数、加消息队列削峰

---

## 十三、设计模式与架构（架构师方向）

**Q59：Spring 中使用了哪些设计模式？各举一个例子。**

**答案要点**：

| 设计模式 | Spring 中的应用 |
|----------|----------------|
| **单例模式** | Bean 默认 Scope 为 Singleton，通过 ConcurrentHashMap 实现 |
| **工厂方法** | BeanFactory / ApplicationContext 创建 Bean |
| **代理模式** | AOP 动态代理（JDK/CGLIB） |
| **模板方法** | JdbcTemplate / RestTemplate / RedisTemplate（固定流程+可变步骤） |
| **观察者模式** | ApplicationEvent + @EventListener |
| **适配器模式** | HandlerAdapter（将不同类型 Handler 适配给 DispatcherServlet） |
| **装饰器模式** | ServerHttpRequestDecorator（包装 Request 增加功能） |
| **策略模式** | Resource 接口多种实现（ClassPathResource / UrlResource） |
| **责任链模式** | Filter Chain / Interceptor Chain |
| **建造者模式** | SpringApplicationBuilder / MockMvcBuilder |

---

**Q60：如何设计一个秒杀系统？**

> 考察意图：经典系统设计题，考察高并发下的全链路思考。

**答案要点**：

```
整体架构：
客户端 → CDN（静态资源） → Nginx 限流 → Gateway 
       → 秒杀服务 → Redis（库存预减） → MQ（异步下单） → DB

核心链路：
1. 活动开始前：秒杀商品库存预热到 Redis
2. 用户请求 → 限流（令牌桶/漏桶），拦截大部分无效流量
3. Redis DECR 预减库存：
   - 库存 >0 → 生成秒杀 Token，放入 MQ
   - 库存 =0 → 直接返回"已售罄"
4. 消费者从 MQ 拉取 → 校验 Token → 创建订单（唯一索引防重）
5. 前端：按钮防抖、验证码、答题防刷
```

**关键技术点**：
- Redis 原子操作 DECR 保证不超卖
- MQ 削峰，避免 DB 压力
- 唯一索引防重复下单
- 静态资源 CDN + 页面静态化

---

## 十四、场景设计题

**Q61：如何实现一个接口的幂等性？**

**答案要点**：
1. **Token 机制**：请求前先获取 Token，提交时携带，Token 使用后删除（Redis）
2. **数据库唯一索引**：关键字段组合唯一索引（如 userId + orderId）
3. **状态机**：状态流转保证幂等（待支付 → 已支付，不可逆向）
4. **乐观锁**：`UPDATE ... SET status=1, version=version+1 WHERE id=1 AND version=0`

---

**Q62：一个 10GB 的大文件，如何导出而不 OOM？**

**答案要点**：
1. **流式查询**：MyBatis Cursor / JPA Stream，逐条从数据库读取不全部加载
2. **流式写入**：EasyExcel 流式写入 / 分多个 Sheet/文件
3. **异步导出**：接收请求后返回"导出中"，后台异步处理完成通知用户下载
4. **进度反馈**：WebSocket 或轮询推送导出进度

---

**Q63：如何设计一个短链接系统？**

**答案要点**：
1. **算法选型**：
   - Hash 算法（MD5 → 取前 N 位） + 哈希冲突解决（追加盐值）
   - 分布式 ID 生成器（Snowflake） → Base62 编码
2. **存储**：MySQL（长链-短链映射表）+ Redis（热点缓存）
3. **高并发**：预生成一批短码放入 Redis 队列，用完再补充
4. **301 vs 302**：临时场景用 302（方便统计），永久用 301

---

## 十五、源码理解题

**Q64：Spring 容器启动时 refresh() 方法做了什么？**

> 考察意图：源码阅读深度的分水岭。

**答案要点**（12 个关键步骤）：
```
1. prepareRefresh()          — 准备：设置启动时间、验证必要属性
2. obtainFreshBeanFactory()  — 获取 BeanFactory
3. prepareBeanFactory()      — 配置 BeanFactory（类加载器、后置处理器）
4. postProcessBeanFactory()  — 子类扩展点（空方法）
5. invokeBeanFactoryPostProcessors() — ★ 执行 BeanFactory 后置处理器
                                 （包括 ConfigurationClassPostProcessor，
                                   解析 @Configuration 类、扫描 Bean）
6. registerBeanPostProcessors() — 注册 Bean 后置处理器
7. initMessageSource()          — 国际化
8. initApplicationEventMulticaster() — 事件广播器
9. onRefresh()                  — ★ 子类扩展（Spring Boot 在此启动 Tomcat）
10. registerListeners()         — 注册监听器
11. finishBeanFactoryInitialization() — ★★ 实例化所有非懒加载的单例 Bean
12. finishRefresh()             — 完成：发布 ContextRefreshedEvent
```

---

**Q65：Spring Boot 如何判断当前是 Web 应用？**

**答案要点**：
- `SpringApplication` 构造时调用 `deduceWebApplicationType()`
```java
static WebApplicationType deduceFromClasspath() {
    if (ClassUtils.isPresent(DISPATCHER_SERVLET_CLASS, null)
        && !ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)) {
        return WebApplicationType.SERVLET;  // Servlet Web
    }
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)) {
        return WebApplicationType.REACTIVE; // WebFlux
    }
    return WebApplicationType.NONE;  // 非 Web 应用
}
```
- 通过检查类路径中是否存在 `DispatcherServlet`、`DispatcherHandler` 等类来判断

---

## 十六、综合高难度题

**Q66：Service A 调用 Service B，B 调用 C，C 抛异常，A 加了 @Transactional，B 的事务传播是 REQUIRES_NEW，描述回滚情况。**

**答案要点**：
```
A（REQUIRED） → B（REQUIRES_NEW） → C（默认 REQUIRED）
                                       ↓
                                    抛异常

结果：
- C 在 B 的事务中，B 是独立事务
- C 抛异常 → B 的事务回滚
- A 的事务：取决于 A 如何捕获异常
  - 如果 A 也抛异常 → A 回滚
  - 如果 A try-catch 了 → A 正常提交（B 已独立回滚）
```

---

**Q67：Spring Boot 项目启动慢，如何排查？**

**答案要点**：
1. `StopWatch` 定位各阶段耗时（SpringApplication 自带）
2. `@Bean` 初始化耗时：检查是否有重量级初始化逻辑
3. MyBatis XML 过多：配置 `mybatis-plus.configuration.lazy-initialization=true`
4. 连接池初始化超时：检查 `spring.datasource.hikari.connection-timeout`
5. 排除不需要的自动配置类：`@SpringBootApplication(exclude = {...})`
6. 启动时懒加载：`spring.main.lazy-initialization=true`（开发环境）

---

**Q68：生产环境 CPU 100%，怎么排查？**

**答案要点**：
```bash
# 1. 找到 Java 进程 PID
top -c

# 2. 找到 CPU 最高的线程
top -Hp <pid>

# 3. 将线程 ID 转 16 进制
printf "%x\n" <tid>

# 4. 用 jstack 导出线程栈，搜索对应线程
jstack <pid> | grep -A 30 <hex_tid>

# 或用 Arthas 一键定位：
thread -n 3  # 显示 CPU 最高的 3 个线程
```

常见原因：死循环、频繁 GC、正则回溯、锁竞争严重

---

## 十七、面试软技能题

**Q69：你在项目中使用 Spring Boot 遇到的最大挑战是什么？如何解决的？**

> 答题思路：STAR 法则（Situation-Task-Action-Result）

**参考话术框架**：
- Situation："项目中出现了一个线上问题..."
- Task："我需要快速定位并修复，同时不能影响线上业务"
- Action："我通过 Arthas 在线诊断定位到...，然后做了...优化"
- Result："最终问题解决，性能提升了 X%，并沉淀了排查 SOP"

---

**Q70：Spring Boot 和 Spring Cloud Alibaba 在你的项目中是怎么配合使用的？**

> 考察意图：是否有实际的微服务项目经验。

**参考话术**：
- "项目使用了 Nacos 做注册中心和配置中心"
- "服务间调用使用 OpenFeign，配置了超时和重试策略"
- "网关层使用 Spring Cloud Gateway，做了统一鉴权和限流"
- "用 Sentinel 做了关键接口的熔断降级"
- "分布式事务选用了 Seata AT 模式处理订单和库存的一致性"

---

## 十八、面试准备清单

### 面试前一天
- [ ] 复习 Bean 生命周期和 Spring Boot 启动流程（必问）
- [ ] 复习事务传播行为与失效场景（必问）
- [ ] 复习缓存穿透/击穿/雪崩（必问）
- [ ] 准备 2-3 个自己项目的技术亮点（带数据）
- [ ] 准备 1-2 个生产故障排查案例（体现解决问题的能力）

### 面试前 1 小时
- [ ] 过一遍自己的简历，确保写上去的都能讲清楚
- [ ] 准备一段 1 分钟自我介绍（突出 Spring Boot 相关经验）
- [ ] 准备 3 个要问面试官的问题（体现你的思考深度）

### 面试中注意事项
- [ ] 不会的问题：诚实说"这块了解不深，但我可以谈一下我的理解..."
- [ ] 场景题：先确认需求，再分析，最后给出方案
- [ ] 源码题：不需要一字不差，说清楚核心流程和设计思想即可
- [ ] 数据说话：用具体数字说明优化效果（QPS 从 X 提升到 Y，响应时间降低 Z%）

### 面试后复盘
- [ ] 记录没答上的问题，针对性补充学习
- [ ] 分析面试官关注的重点（技术深度/项目经验/架构能力/团队管理）
- [ ] 更新自己的知识体系

---

### 📊 难度分级速查

| 级别 | 题号 | 适合年限 |
|------|------|----------|
| **初级** | Q1-Q7, Q10, Q17, Q21, Q22, Q29, Q30, Q31, Q47 | 1-3 年 |
| **中级** | Q8, Q9, Q11-Q16, Q18-Q20, Q24-Q28, Q32-Q36, Q46, Q48, Q49 | 3-5 年 |
| **高级** | Q37-Q45, Q50-Q58, Q61-Q63, Q66-Q68 | 5-8 年 |
| **架构** | Q53-Q55, Q59, Q60, Q64, Q65 | 8+ 年 |

---

> **最后提醒**：面试不是背答案，而是展示你的技术思维和解决问题的能力。每道题都要能关联到实际项目经验："这个我在 XXX 项目中用到过，当时..." 这样的回答比干巴巴的理论陈述有力 10 倍。

---

*最后更新：2026年7月19日*
