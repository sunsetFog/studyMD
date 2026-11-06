# 04 · Spring MVC 基础

## 一、请求处理流程

```
请求 → DispatcherServlet（前端控制器）
     → HandlerMapping（找 Handler）
     → HandlerAdapter → Controller 方法执行
     → 返回值处理（@ResponseBody → JSON 序列化）
     → 视图渲染 / 响应写出
```

`@RestController` = `@Controller` + `@ResponseBody`（所有方法返回值直接写响应体）。

## 二、接收请求参数

### 1. @RequestParam — 查询参数

```java
@GetMapping("/users")
public List<UserDTO> list(@RequestParam(defaultValue = "1") int page,
                          @RequestParam(required = false) String keyword) { ... }
// GET /users?page=2&keyword=tom
```

### 2. @PathVariable — 路径变量

```java
@GetMapping("/users/{id}")
public UserDTO detail(@PathVariable Long id) { ... }
```

### 3. @RequestBody — JSON 体（POST/PUT）

```java
@PostMapping("/users")
public UserDTO create(@RequestBody @Valid UserCreateDTO dto) { ... }
```

### 4. 对象绑定 — 查询参数自动装配

```java
@GetMapping("/search")
public List<UserDTO> search(UserQueryDTO query) { ... }
// 无注解，按 setter 名匹配 query 参数；不能加 @RequestBody
```

### 5. @RequestHeader / @CookieValue

```java
public String header(@RequestHeader("User-Agent") String ua,
                     @CookieValue("token") String token) { ... }
```

### 选型铁律

| 场景 | 用法 |
|---|---|
| 查询/分页/筛选 | @RequestParam 或对象绑定 |
| 资源标识 | @PathVariable |
| 创建/更新的数据 | @RequestBody + DTO |
| 复杂多条件搜索 GET | 对象绑定 |

## 三、返回值

- DTO 对象 → 自动由 Jackson 序列化为 JSON
- `ResponseEntity<T>` → 完全控制状态码/头/体（框架级代码用）
- 统一响应体 `Result<T>` → 业务代码标准做法（见 07 篇）

## 四、静态资源与 WebMvcConfigurer

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/files/**")
                .addResourceLocations("file:/data/upload/");
    }

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOriginPatterns("*")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

> 前后端分离必配 CORS；`allowCredentials(true)` 时 origin 不能为 `*`，用 `allowedOriginPatterns`。

## 五、Jackson 常用定制

```java
// application.yml 全局
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    default-property-inclusion: non_null   # null 字段不输出

// 字段级
public class UserDTO {
    @JsonProperty("user_name")     // 重命名
    private String userName;

    @JsonIgnore                     // 不序列化
    private String password;

    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate birthday;
}
```

## 六、重定向与转发

```java
@GetMapping("/old")
public String old() { return "redirect:/new"; }   // 302 重定向
@GetMapping("/forward")
public String forward() { return "forward:/api/new"; }  // 服务端转发
```

## 常见坑

- **坑 1**：@RequestBody 用于 GET → 请求体解析失败。GET 用 query 参数
- **坑 2**：POST 405/415 → 前端没设 `Content-Type: application/json`
- **坑 3**：日期传字符串绑定失败 → 全局配置 Jackson date-format 或字段加 @DateTimeFormat/@JsonFormat
- **坑 4**：Long 精度问题 → 前端 JS 超过 2^53 丢精度，序列化为字符串：`@JsonSerialize(using = ToStringSerializer.class)`
- **坑 5**：CORS 报错但接口 Postman 正常 → 预检 OPTIONS 被拦截器挡了，Interceptor 要放行 OPTIONS

## Checklist

- [ ] 参数接收按场景选对注解
- [ ] 统一 Jackson 日期/时区/空值策略
- [ ] CORS 全局配置且放行 OPTIONS
- [ ] Controller 返回统一 Result<T>
