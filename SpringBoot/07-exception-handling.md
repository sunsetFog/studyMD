# 07 · 全局异常处理与统一响应

## 一、设计目标

```
任何异常 → 统一 JSON 响应 → 前端只需处理 code 字段
日志完整记录 → 不丢堆栈 → 便于排查
```

## 二、统一响应体（见 05 篇）+ 业务异常

```java
@Getter
public class BusinessException extends RuntimeException {
    private final int code;

    public BusinessException(int code, String message) {
        super(message);
        this.code = code;
    }

    public static BusinessException of(String message) {
        return new BusinessException(500, message);
    }
}

// 使用：Service 里直接抛
throw new BusinessException(10001, "用户名已存在");
```

## 三、全局异常处理器

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /** 业务异常：不打 error 日志（预期内） */
    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusiness(BusinessException e) {
        log.warn("业务异常: code={}, msg={}", e.getCode(), e.getMessage());
        return Result.fail(e.getCode(), e.getMessage());
    }

    /** 参数校验（@RequestBody + @Valid） */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + " " + f.getDefaultMessage())
                .collect(Collectors.joining("; "));
        return Result.fail(400, msg);
    }

    /** 查询参数校验 */
    @ExceptionHandler(ConstraintViolationException.class)
    public Result<Void> handleConstraint(ConstraintViolationException e) {
        String msg = e.getConstraintViolations().stream()
                .map(ConstraintViolation::getMessage)
                .collect(Collectors.joining("; "));
        return Result.fail(400, msg);
    }

    /** 请求体 JSON 解析失败 */
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public Result<Void> handleNotReadable(HttpMessageNotReadableException e) {
        return Result.fail(400, "请求体格式错误");
    }

    /** 404 */
    @ExceptionHandler(NoResourceFoundException.class)
    public Result<Void> handle404(NoResourceFoundException e) {
        return Result.fail(404, "资源不存在");
    }

    /** 兜底：未知异常，打全堆栈 */
    @ExceptionHandler(Exception.class)
    public Result<Void> handleUnknown(Exception e) {
        log.error("系统异常", e);
        return Result.fail(500, "系统繁忙，请稍后重试");  // 不暴露内部信息
    }
}
```

## 四、错误码规划

| 分段 | 含义 |
|---|---|
| 0 | 成功 |
| 400/401/403/404/500 | 通用 HTTP 语义 |
| 10xxx | 用户模块 |
| 20xxx | 订单模块 |
| 9xxxx | 三方/系统级 |

错误码集中定义为常量类或枚举，禁止魔法数字散落各处：

```java
public enum ErrorCode {
    USERNAME_EXISTS(10001, "用户名已存在"),
    ORDER_NOT_FOUND(20001, "订单不存在");

    private final int code;
    private final String message;
}
```

## 五、@ControllerAdvice 三种能力

1. `@ExceptionHandler` — 异常处理（本篇）
2. `@ModelAttribute` — 全局模型属性（少用）
3. `@InitBinder` — 全局参数预处理（如日期转换、XSS 过滤）

```java
@RestControllerAdvice
public class GlobalBinder {
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.registerCustomEditor(LocalDate.class,
                new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), true));
    }
}
```

## 六、事务回滚与异常的关系（衔接 14 篇）

- 默认只对 RuntimeException/Error 回滚
- `BusinessException extends RuntimeException` → 天然支持回滚
- 若自定义 checked 异常需 `@Transactional(rollbackFor = Exception.class)`

## 七、异常处理反模式

| 反模式 | 后果 |
|---|---|
| catch 后吞掉不打日志 | 问题消失无踪 |
| catch (Exception e) 后 throw new RuntimeException(e.getMessage()) | 丢堆栈 |
| 返回 null 表示失败 | 前端无法区分错误类型 |
| 在 Controller 里 try-catch 每个方法 | 重复代码爆炸 |

正确姿势：**Service 只管抛，Advice 统一收**。

## 常见坑

- **坑 1**：@RestControllerAdvice 失效 → 异常在 Filter 里抛出（Filter 在 MVC 之外），需 ErrorController 或在 Filter 内自行处理
- **坑 2**：异步线程（@Async / 线程池）里抛异常不会被全局处理器捕获 → 见 19 篇 AsyncUncaughtExceptionHandler
- **坑 3**：handler 内部又抛异常 → 递归 500；兜底 handler 里别再抛
- **坑 4**：日志打了 message 没打堆栈 → `log.error("xx", e)` 才有堆栈

## Checklist

- [ ] GlobalExceptionHandler 覆盖：业务/校验/JSON 解析/404/兜底
- [ ] 错误码枚举统一管理
- [ ] 业务异常 warn 级别、系统异常 error + 堆栈
- [ ] 响应不暴露堆栈和 SQL 等内部信息
