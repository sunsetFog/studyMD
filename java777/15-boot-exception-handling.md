# 15 · 全局异常处理与统一响应

## 1. 错误码设计

```
A0xxx  通用（参数错误、无权限、限流）
1xxxx  用户域
2xxxx  订单域
3xxxx  支付域
9xxxx  系统级
```

```java
@Getter @AllArgsConstructor
public enum ErrorCode {
    SUCCESS(0, "OK"),
    PARAM_ERROR(40000, "参数错误"),
    UNAUTHORIZED(40100, "未登录"),
    FORBIDDEN(40300, "无权限"),
    NOT_FOUND(40400, "数据不存在"),
    RATE_LIMIT(42900, "请求过于频繁"),
    SYSTEM_ERROR(50000, "系统繁忙");
    private final int code; private final String msg;
}
```

## 2. 自定义业务异常

```java
@Getter
public class BizException extends RuntimeException {
    private final int code;
    public BizException(ErrorCode ec) { super(ec.getMsg()); this.code = ec.getCode(); }
    public BizException(ErrorCode ec, String msg) { super(msg); this.code = ec.getCode(); }
    public BizException(String msg) { super(msg); this.code = ErrorCode.SYSTEM_ERROR.getCode(); }
}
// 不建议继承 Exception（受检），因为事务默认不回滚检查异常
```

## 3. 全局异常处理器

```java
@Slf4j
@RestControllerAdvice       // = @ControllerAdvice + @ResponseBody
@Order(Ordered.HIGHEST_PRECEDENCE)
public class GlobalExceptionHandler {

    /** 业务异常 */
    @ExceptionHandler(BizException.class)
    public Result<Void> handleBiz(BizException e) {
        log.warn("biz error: {}", e.getMessage());
        return Result.fail(e.getCode(), e.getMessage());
    }

    /** 参数校验异常：@RequestBody + @Valid */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ":" + f.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return Result.fail(ErrorCode.PARAM_ERROR.getCode(), msg);
    }

    /** 参数校验异常：@RequestParam / @PathVariable + @Validated */
    @ExceptionHandler(ConstraintViolationException.class)
    public Result<Void> handleConstraint(ConstraintViolationException e) {
        String msg = e.getConstraintViolations().stream()
                .map(v -> v.getPropertyPath() + ":" + v.getMessage())
                .collect(Collectors.joining(", "));
        return Result.fail(ErrorCode.PARAM_ERROR.getCode(), msg);
    }

    /** 缺少必填参数 */
    @ExceptionHandler(MissingServletRequestParameterException.class)
    public Result<Void> handleMissing(MissingServletRequestParameterException e) {
        return Result.fail(ErrorCode.PARAM_ERROR.getCode(), "缺少参数 " + e.getParameterName());
    }

    /** 消息体读取失败（JSON 格式错误/缺失） */
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public Result<Void> handleNotReadable(HttpMessageNotReadableException e) {
        return Result.fail(ErrorCode.PARAM_ERROR.getCode(), "请求体格式错误");
    }

    /** 类型不匹配 */
    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    public Result<Void> handleTypeMismatch(MethodArgumentTypeMismatchException e) {
        return Result.fail(ErrorCode.PARAM_ERROR.getCode(), "参数类型错误: " + e.getName());
    }

    /** 404（需开启 throw-exception-if-no-handler-found + 关闭静态资源映射） */
    @ExceptionHandler(NoResourceFoundException.class)   // Boot 3.2+；Boot 2 用 NoHandlerFoundException
    public Result<Void> handle404(NoResourceFoundException e) {
        return Result.fail(ErrorCode.NOT_FOUND.getCode(), ErrorCode.NOT_FOUND.getMsg());
    }

    /** 兜底 */
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("system error", e);      // ★ 打完整堆栈
        return Result.fail(ErrorCode.SYSTEM_ERROR.getCode(), ErrorCode.SYSTEM_ERROR.getMsg());
    }
}
```

> **顺序**：Spring 会选**最具体**的处理器。兜底 `Exception` 一定要放最后（同类内按方法定义顺序匹配更具体的类型）。

## 4. Boot 3 推荐的 ProblemDetail（RFC 7807）

```java
@RestControllerAdvice
public class ProblemHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(BizException.class)
    ProblemDetail handleBiz(BizException e) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, e.getMessage());
        pd.setTitle("业务异常");
        pd.setType(URI.create("https://api.example.com/errors/biz"));
        pd.setProperty("code", e.getCode());
        pd.setProperty("traceId", MDC.get("traceId"));
        return pd;                       // 响应 Content-Type: application/problem+json
    }
}
```

```yaml
spring:
  mvc:
    problemdetails:
      enabled: true      # 启用后内置异常也返回 problem+json
```

响应示例：
```json
{
  "type": "about:blank", "title": "Bad Request", "status": 400,
  "detail": "库存不足", "instance": "/api/orders", "traceId": "a1b2c3"
}
```

## 5. 统一响应包装（三种方案）

| 方案 | 优点 | 缺点 |
|---|---|---|
| Controller 手动 `Result.ok(x)` | 直观，类型安全 | 样板代码 |
| `ResponseBodyAdvice` 自动包装 | 零侵入 | 需处理 String、已包装、排除项 |
| 网关层包装 | 统一多语言服务 | 单体不适用 |

```java
@RestControllerAdvice(basePackages = "com.example")
public class WrapAdvice implements ResponseBodyAdvice<Object> {
    @Override public boolean supports(MethodParameter rt, Class<?> c) {
        return !rt.getParameterType().equals(Result.class)     // 已包装的跳过
            && !rt.hasMethodAnnotation(NoWrap.class);          // 注解排除
    }
    @Override public Object beforeBodyWrite(Object body, MethodParameter rt, MediaType mt,
            Class<?> c, ServerHttpRequest req, ServerHttpResponse res) {
        if (body instanceof String) {                          // ★ String 特殊处理
            return JsonUtil.toJson(Result.ok(body));
        }
        return Result.ok(body);
    }
}
```

## 6. 404 / 500 页面（前后端分离可忽略）

```java
@Controller
public class ErrorPageController implements ErrorController {
    @RequestMapping("/error") public String error() { return "forward:/index.html"; }
}
```

## 7. 事务与异常的关系

```java
@Transactional(rollbackFor = Exception.class)
public void create() {
    try {
        riskyOp();                       // 抛异常
    } catch (Exception e) {
        log.error("...", e);
        throw new BizException("下单失败");   // ★ 必须重新抛出，否则事务提交
    }
}
```

> 吞掉异常 = 事务失效 = 数据不一致。这是最高频的线上事故之一。

## 8. 微服务中的异常传递

```java
// Feign 调用异常解码
@Bean
public ErrorDecoder feignErrorDecoder() {
    return (methodKey, response) -> {
        if (response.status() >= 400) {
            // 解析 body 里的 Result，转成下游 BizException
            return new BizException(parse(response));
        }
        return new ErrorDecoder.Default().decode(methodKey, response);
    };
}
```

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| `@ControllerAdvice` 没生效 | 异常未被处理 | 包扫描范围；或用 `basePackages` |
| 404 走不到全局处理 | 返回默认错误页 | 开 `throw-exception-if-no-handler-found` + 关静态资源 |
| `String` 返回值包装报转换异常 | `Result cannot be cast to String` | 单独处理 String + 加 `StringHttpMessageConverter` |
| 异步线程异常丢失 | 日志无堆栈 | `@Async` 配 `AsyncUncaughtExceptionHandler`；线程池提交要 `future.get()` |
| Filter 中异常无法被 Advice 捕获 | 返回 500 默认页 | Filter 早于 DispatcherServlet，需在 Filter 内自行写响应（如 Gateway） |
| 返回 500 但前端显示英文 | 信息不友好 | 兜底统一中文文案 |
| 敏感堆栈暴露 | 安全风险 | `server.error.include-stacktrace: never` |

## 10. 面试速答

- **全局异常处理怎么实现**：`@RestControllerAdvice` + `@ExceptionHandler`，按异常类型从具体到宽泛匹配。
- **参数校验异常类有哪些**：`MethodArgumentNotValidException`（body）、`ConstraintViolationException`（方法参数）、`MissingServletRequestParameterException`、`MethodArgumentTypeMismatchException`。
- **为什么 catch 后事务不回滚**：事务拦截器靠捕获异常决定回滚，吞掉异常即视为正常提交；解决是重新抛出或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。
