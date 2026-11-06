# 14 · Spring MVC / Boot Web 开发

## 1. WebMvcAutoConfiguration 给了什么

引入了 `spring-boot-starter-web` 后，自动配置提供：

- `DispatcherServlet`（映射到 `/`）
- 内嵌 Tomcat
- `RequestMappingHandlerMapping/Adapter`
- Jackson（`ObjectMapper`）消息转换器
- 静态资源处理、欢迎页、favicon
- 异常页（`/error`）
- `HttpMessageConverters`
- 字符编码过滤器（`UTF-8`）
- `HiddenHttpMethodFilter`（表单 REST 支持）

## 2. 请求映射注解

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @GetMapping("/{id}")
    public OrderVO get(@PathVariable("id") Long id) { ... }

    @GetMapping
    public PageResult<OrderVO> page(@RequestParam(defaultValue = "1") int pageNo,
                                    @RequestParam(defaultValue = "10") int pageSize) { ... }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Long create(@Valid @RequestBody OrderDTO dto) { ... }

    @PutMapping("/{id}")  public void update(@PathVariable Long id, @RequestBody OrderDTO dto) {}
    @DeleteMapping("/{id}") public void delete(@PathVariable Long id) {}
    @PatchMapping("/{id}")  public void patch(@PathVariable Long id, @RequestBody Map<String,Object> patch) {}
}
```

## 3. 参数绑定注解速查

| 注解 | 来源 | 示例 |
|---|---|---|
| `@PathVariable` | URL 路径 | `/users/{id}` |
| `@RequestParam` | 查询参数 `?a=1` | 支持 `defaultValue`、`required` |
| `@RequestBody` | JSON 请求体 | 需 `Content-Type: application/json` |
| `@RequestHeader` | 请求头 | `@RequestHeader("Authorization")` |
| `@CookieValue` | Cookie | |
| `@MatrixVariable` | 矩阵变量（默认关闭） | |
| `@ModelAttribute` | 表单绑定对象 | |
| `@RequestPart` | multipart 文件+JSON | |
| 无注解 | 表单/查询参数绑定 POJO | |

```java
// 分页 + 多条件
@GetMapping("/search")
public List<OrderVO> search(@ModelAttribute OrderQuery query,
                            @PageableDefault(size = 20, sort = "createTime,desc") Pageable pageable) {
    return service.search(query, pageable);
}
```

### 日期参数

```java
@GetMapping
public List<OrderVO> list(@DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate date) {}
// 全局：spring.mvc.format.date=yyyy-MM-dd
// JSON：@JsonFormat(pattern="yyyy-MM-dd HH:mm:ss", timezone="GMT+8") 或全局 jackson 配置
```

## 4. 返回值处理

| 返回 | 效果 |
|---|---|
| 对象 | JSON（`@RestController`） |
| `ResponseEntity<T>` | 自定义状态码/头 |
| `ResponseEntity<Void>` | 204 |
| `void` | 204（或由 `@ResponseStatus` 指定） |
| `String` | 字符串（`@RestController`）或视图名（`@Controller`） |

```java
@GetMapping("/{id}")
public ResponseEntity<OrderVO> get(@PathVariable Long id) {
    return service.findById(id)
        .map(o -> ResponseEntity.ok()
             .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS)).body(o))
        .orElseGet(() -> ResponseEntity.notFound().build());
}
```

## 5. 拦截器 vs 过滤器

| 对比 | Filter | Interceptor | AOP |
|---|---|---|---|
| 归属 | Servlet 规范 | Spring MVC | Spring AOP |
| 粒度 | 请求级 | 请求级（能拿到 Handler） | 方法级 |
| 依赖 | 不依赖 Spring | 依赖 Spring 容器 | 依赖容器 |
| 顺序 | 早于 DispatcherServlet | 在 Handler 前后 | 业务方法前后 |

```java
@Component
public class LoginInterceptor implements HandlerInterceptor {
    @Override public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        String token = req.getHeader("Authorization");
        if (StringUtils.hasText(token)) return true;
        res.setStatus(401);          // 或抛异常交给全局处理
        return false;
    }
    @Override public void postHandle(...) {}
    @Override public void afterCompletion(...) {}   // 适合清理 ThreadLocal
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired private LoginInterceptor loginInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/auth/login", "/api/health")
                .order(1);
    }

    @Override public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOriginPatterns("*")      // ★ 生产别用 *
                .allowedMethods("GET","POST","PUT","DELETE","OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }

    @Override public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // 自定义静态资源/上传目录映射
        registry.addResourceHandler("/upload/**").addResourceLocations("file:/data/upload/");
    }
}
```

> **执行顺序**：Filter → DispatcherServlet → Interceptor#preHandle → AOP → Controller → Interceptor#postHandle → 视图渲染 → afterCompletion → Filter。

## 6. CORS 三种配置方式

```java
// ① WebMvcConfigurer（全局，推荐）
// ② @CrossOrigin 注解（单个控制器）
@CrossOrigin(origins = "https://example.com", maxAge = 3600)
// ③ Security 中开启（见 20 篇）
http.cors(Customizer.withDefaults());
```

> 跨域预检（OPTIONS）会被 Security 拦截，需在 Security 中放行或配置 `cors()`。

## 7. 消息转换器与自定义

Boot 默认按依赖引入 Jackson / Gson / JSON-B。自定义：

```java
@Bean
public ObjectMapper objectMapper(Jackson2ObjectMapperBuilder builder) {
    return builder
        .modules(new JavaTimeModule(), new ParameterNamesModule())
        .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
        .build();
}
// 或只加 Module
@Bean public Module javaTimeModule() { return new JavaTimeModule(); }
```

常用注解：`@JsonIgnore`、`@JsonFormat`、`@JsonProperty`、`@JsonInclude(NON_NULL)`、`@JsonView`、`@JsonPropertyOrder`、枚举的 `@JsonValue`。

## 8. 文件上传

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 50MB
```

```java
@PostMapping("/upload")
public Result<String> upload(@RequestPart("file") MultipartFile file) throws IOException {
    String name = UUID.randomUUID() + "." + StringUtils.getFilenameExtension(file.getOriginalFilename());
    Path dest = Paths.get("/data/upload", name);
    Files.copy(file.getInputStream(), dest, StandardCopyOption.REPLACE_EXISTING);
    return Result.ok("/upload/" + name);
}
```

> 生产建议传 OSS（阿里云/腾讯 COS）直传，不走应用服务器。

## 9. 静态资源与模板

默认静态目录：`classpath:/static`、`/public`、`/resources`、`/META-INF/resources`。
模板引擎（Thymeleaf/Freemarker）在现代前后端分离项目中已少用。

## 10. 关键配置

```yaml
spring:
  mvc:
    servlet:
      load-on-startup: 1
    throw-exception-if-no-handler-found: true   # 404 时抛异常便于统一处理
    format:
      date: yyyy-MM-dd
  web:
    resources:
      add-mappings: false       # 纯 API 服务可关闭静态资源
  jackson:
    default-property-inclusion: non_null
    time-zone: Asia/Shanghai
server:
  port: 8080
  tomcat:
    threads:
      max: 200
      min-spare: 20
    max-http-form-post-size: 2MB
    connection-timeout: 20s
    accept-count: 100
  compression:
    enabled: true
    mime-types: application/json,text/html
    min-response-size: 1024
  error:
    include-message: always        # Boot 3 起默认不返回 message
    include-stacktrace: never
```

## 11. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 404 | 接口访问不到 | 路径、`context-path`、启动类扫描包 |
| 400 | 参数绑定失败 | 类型不匹配、日期格式、缺 `@RequestBody` |
| 415 | `Unsupported Media Type` | 请求头缺 `application/json` |
| 中文乱码 | 乱码 | `server.servlet.encoding.charset=UTF-8` |
| CORS 不生效 | 预检 403 | Security 中放行 OPTIONS / 开启 `cors()` |
| 拦截器注入 null | NPE | 拦截器要注册为 Bean（`@Component`） |
| `@RequestParam` 必填 | 400 | 加 `required=false` 或 `defaultValue` |
| 大文件上传超时 | 超时 | 调 `max-file-size` 与网关超时 |

## 12. 面试速答

- **Filter / Interceptor / AOP 区别**：Filter 是 Servlet 规范最外层；Interceptor 是 MVC 层，能拿到 Handler 且能注入 Bean；AOP 是方法级，可精确到业务方法。
- **DispatcherServlet 作用**：前端控制器，统一接收请求，经 HandlerMapping 找处理器、HandlerAdapter 执行、ViewResolver 渲染、异常解析器处理。
- **CORS 怎么处理**：`WebMvcConfigurer#addCorsMappings` 全局配置，Security 中需同时启用 `cors()`。
