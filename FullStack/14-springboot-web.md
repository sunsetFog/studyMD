# 14 - SpringBoot Web 开发

## 1. 接口层规范

```java
@RestController
@RequestMapping("/api/v1/orders")
@Validated
public class OrderController {
    @GetMapping("/{id}")
    public R<OrderVO> detail(@PathVariable Long id) { ... }

    @PostMapping
    public R<Long> create(@RequestBody @Valid OrderCreateDTO dto) { ... }

    @GetMapping("/page")
    public R<Page<OrderVO>> page(OrderQuery query) { ... }
}
```

- 参数接收选型：@PathVariable（路径资源 ID）/ @RequestParam（过滤条件）/ @RequestBody（JSON 主体）
- 统一响应体 R<T>：code/message/data + 全局错误码枚举
- 分页统一：pageNum/pageSize，MyBatis-Plus Page 或 PageHelper

## 2. 参数校验（Jakarta Validation）

```java
public record OrderCreateDTO(
    @NotBlank @Size(max = 64) String title,
    @NotNull @Min(1) @Max(999) Integer quantity,
    @Pattern(regexp = "^1[3-9]\\d{9}$") String phone,
    @Email String email
) {}
```

- 嵌套校验：内部对象加 @Valid
- 分组校验：`@Validated(Create.class)` 新建 vs 更新不同规则
- 自定义校验器：实现 ConstraintValidator<A, T>
- 校验失败 → BindException/MethodArgumentNotValidException → 全局异常统一处理

## 3. 全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public R<Void> handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return R.fail(400, msg);
    }
    @ExceptionHandler(BizException.class)          // 业务异常 → 精确 code
    @ExceptionHandler(Exception.class)              // 兜底 → 500，日志记录，不暴露堆栈
}
```

## 4. 拦截体系（必考对比）

| 层级 | 规范 | 粒度 | 典型用途 |
|---|---|---|---|
| Filter | Servlet | 请求级（包括静态资源） | 编码、CORS、链路 traceId |
| Interceptor | Spring | Handler 级 | 登录态、权限、日志 |
| AOP | 方法 | Bean 方法级 | 事务、缓存、审计 |

```java
@Component
public class AuthInterceptor implements HandlerInterceptor {
    public boolean preHandle(req, resp, handler) {
        if (!(handler instanceof HandlerMethod)) return true;   // 放行静态资源
        // 解析 token → ThreadLocal 存用户
    }
    public void afterCompletion(...) { ThreadLocal.remove(); }  // 线程池防脏数据
}
```

## 5. 常用配置要点

```yaml
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
  servlet:
    multipart:
      max-file-size: 20MB
server:
  shutdown: graceful            # 优雅停机
```

- 跨域两种方式：`WebMvcConfigurer#addCorsMappings` 或 CorsFilter（与拦截器/安全框架共存时注意顺序）
- 定时任务 @Scheduled（单线程默认 → 配 taskScheduler 线程池）、@Async 需 @EnableAsync + 自定义线程池
- 日志 MDC + traceId 链路追踪（Filter 写入 MDC，日志 pattern 引用）

## 6. 启动与运维考点

- 启动流程：new SpringApplication → 推断类型 → 加载 Initializer/Listener → run() → 准备环境 → 创建上下文 → refresh（含自动配置）→ 回调 Runner
- Actuator：/health /metrics /info，生产仅暴露给内网监控
- 优雅停机：SIGTERM → 停接收新请求 → 等待存量完成（timeout）

## 常见追问

1. 请求到 Controller 的完整链路（DispatcherServlet → HandlerMapping → Adapter → Resolver → Controller，见 13.7）
2. @ControllerAdvice 多个时优先级：更具体的异常优先
3. 接口幂等怎么做？——唯一索引/token 机制/乐观锁（见 18）

## Checklist

- [ ] 校验 + 全局异常是接口开发标配
- [ ] Filter/Interceptor/AOP 三层能对比
- [ ] ThreadLocal 在 afterCompletion 清理有意识
