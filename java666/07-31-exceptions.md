# 31 · 异常处理

## 1. 异常体系

```
Throwable
├── Error（不可恢复，不要捕获）
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   ├── NoClassDefFoundError
│   └── AssertionError
└── Exception
    ├── RuntimeException（非受检）
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   ├── IndexOutOfBoundsException
    │   ├── ClassCastException
    │   ├── ArithmeticException
    │   ├── NumberFormatException
    │   ├── ConcurrentModificationException
    │   └── UnsupportedOperationException
    └── 受检异常（Checked）
        ├── IOException
        ├── SQLException
        ├── ClassNotFoundException
        ├── InterruptedException
        └── FileNotFoundException
```

| 类型 | 编译期检查 | 典型场景 |
|---|---|---|
| `Error` | 不检查 | JVM 级故障，程序不该处理 |
| `RuntimeException` | 不检查 | 编程错误（参数、状态、NPE） |
| 受检异常 | **必须**捕获或声明 | 外部可恢复故障（IO、网络、DB） |

## 2. 受检 vs 非受检

```java
// 受检：必须处理
public void read() throws IOException {
    Files.readAllLines(Path.of("a.txt"));
}

// 非受检：可以不处理
public int parse(String s) {
    return Integer.parseInt(s);     // 可能抛 NumberFormatException
}
```

设计取舍：

- 受检异常强制调用方处理，但污染签名、难以在 Lambda 中使用。
- 现代框架（Spring）倾向用非受检异常 + 全局异常处理器。

## 3. try-catch-finally

```java
try {
    risky();
} catch (IOException | SQLException e) {          // 多重捕获（Java 7+）
    log.error("failed", e);
    throw new BizException("操作失败", e);
} finally {
    cleanup();                                      // 总会执行
}
```

执行顺序：

1. try 正常结束 → finally → 继续。
2. try 抛异常 → 匹配 catch → finally → 继续/抛出。
3. catch 中抛异常 → finally → 抛出 catch 的异常。
4. try/catch 中有 `return` → **先算返回值，再执行 finally，最后返回**。
5. finally 中有 `return` → **覆盖** try/catch 的返回值（禁止！）。

```java
static int f() {
    try { return 1; }
    finally { return 2; }        // 返回 2！吞掉异常也靠这个
}
```

### finally 的陷阱

```java
// 异常被吞
try {
    throw new RuntimeException("a");
} finally {
    return;                       // 异常消失
}

// 资源关闭异常覆盖业务异常
try {
    throw new BizException("业务失败");
} finally {
    resource.close();             // 若 close 也抛异常，业务异常丢失
}
```

## 4. try-with-resources（推荐）

```java
try (BufferedReader r = Files.newBufferedReader(path);
     BufferedWriter w = Files.newBufferedWriter(out)) {
    r.lines().forEach(w::write);
}
```

要求资源实现 `AutoCloseable`（或 `Closeable`）。

优势：

- 自动关闭，按**声明逆序**关闭。
- 关闭异常作为 **suppressed** 附加在原异常上，不丢失。
- 代码简洁。

```java
// 等价的手写版本（繁琐且易错）
BufferedReader r = null;
try {
    r = Files.newBufferedReader(path);
    ...
} finally {
    if (r != null) r.close();
}
```

多个资源与异常抑制：

```java
try (A a = new A(); B b = new B()) { ... }
// 关闭顺序：b 先，a 后
// 若 body 抛 e1，b.close() 抛 e2 → e1 为主异常，e2 在 e1.getSuppressed() 中
```

Java 9+ 可用已有变量：

```java
BufferedReader r = ...;
try (r) { ... }        // r 必须是 effectively final
```

## 5. 自定义异常

```java
public class BizException extends RuntimeException {
    private final String code;

    public BizException(String message) { super(message); this.code = "BIZ_ERROR"; }

    public BizException(String code, String message) {
        super(message);
        this.code = code;
    }

    public BizException(String message, Throwable cause) {
        super(message, cause);
        this.code = "BIZ_ERROR";
    }

    public String getCode() { return code; }
}
```

设计原则：

- 业务异常继承 `RuntimeException`（不强制调用方处理）。
- 保留 `cause` 链（`super(message, cause)`）。
- 提供错误码，便于国际化与监控。
- 不要为每个小问题建一个异常类；按**领域/层次**划分。

```java
// 分层异常
// - BizException：业务规则失败（用户可见）
// - SystemException：系统故障（500）
// - ThirdPartyException：外部依赖失败（可重试）
```

## 6. 异常转换与传播

```java
// 底层异常转业务异常，保留 cause
try {
    userMapper.insert(user);
} catch (DuplicateKeyException e) {
    throw new BizException("USER_EXISTS", "用户已存在", e);
}
```

不要这样做：

```java
// 1. 吞异常
try { ... } catch (Exception e) { }             // 灾难

// 2. 只打印不处理
try { ... } catch (Exception e) { e.printStackTrace(); }

// 3. 丢失 cause
catch (IOException e) { throw new BizException("读取失败"); }   // 堆栈断链

// 4. 捕获过宽
try { ... } catch (Exception e) { ... }         // 把编程错误也吞了

// 5. 用异常控制流程
try { return list.get(i); } catch (IndexOutOfBoundsException e) { return null; }
```

## 7. 异常处理最佳实践

1. **能处理才捕获**，否则向上抛。
2. 捕获**具体**异常，不用 `Exception`/`Throwable`（除非最外层兜底）。
3. 必须记录日志或转换，不要静默吞掉。
4. 记录日志时传异常对象，不要只拼 message：
   ```java
   log.error("下单失败 orderId={}", id, e);     // ✔ 保留堆栈
   log.error("下单失败 " + e.getMessage());      // ✘ 丢堆栈
   ```
5. 不要在大循环里 `try-catch` 包住整个循环（无法定位哪条失败）。
6. 资源必须关闭（try-with-resources）。
7. 不要用异常做正常流程控制。
8. 异常信息包含**上下文**（哪个 id、哪个参数）。
9. 不要 `throw e` 后又 `catch` 自己。
10. 框架层统一异常处理，Controller 不写 try-catch。

## 8. 全局异常处理（Spring 示例）

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BizException.class)
    public ResponseEntity<ApiError> handleBiz(BizException e) {
        return ResponseEntity.badRequest()
                .body(new ApiError(e.getCode(), e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage())
                .collect(Collectors.joining("; "));
        return ResponseEntity.badRequest().body(new ApiError("PARAM_ERROR", msg));
    }

    @ExceptionHandler(Exception.class)          // 兜底
    public ResponseEntity<ApiError> handleOther(Exception e) {
        log.error("unexpected error", e);
        return ResponseEntity.status(500).body(new ApiError("SYSTEM_ERROR", "系统繁忙"));
    }
}
```

要点：

- 兜底异常**不返回堆栈给前端**（安全）。
- 记录 traceId，便于排查。
- 区分 4xx（客户端问题）与 5xx（服务端问题）。

## 9. 异常性能

| 操作 | 成本 |
|---|---|
| `new Exception()` | 填充堆栈，约 1-10 微秒 |
| `fillInStackTrace` | 主要开销 |
| 抛出/捕获 | 相对低（无堆栈时） |

优化：

```java
// 高频控制流异常可关闭堆栈（谨慎使用）
class FastException extends RuntimeException {
    @Override public synchronized Throwable fillInStackTrace() { return this; }
}
```

原则：**异常只用于异常情况**，正常流程用返回值/`Optional`。

## 10. 特殊异常说明

### NullPointerException

Java 14+ 开启 `-XX:+ShowCodeDetailsInExceptionMessages`（15+ 默认）会给出详细原因：

```
Cannot invoke "String.length()" because "user.name" is null
```

### InterruptedException

```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // 必须恢复中断标志
    return;                               // 尽快退出
}
```

吞掉中断标志会导致上层无法感知停止信号。

### OutOfMemoryError

- 不要 `catch (Throwable)` 后继续运行（JVM 状态可能已损坏）。
- 应让进程退出，由编排系统重启 + 保留 heap dump（`-XX:+HeapDumpOnOutOfMemoryError`）。

### StackOverflowError

- 递归过深（无终止条件）。
- 增大 `-Xss` 只能缓解，根本要改算法。

### ConcurrentModificationException

见 05-28 迭代器篇。

## 11. 断言

```java
assert x > 0;                        // 默认关闭
assert x > 0 : "x 必须为正，实际 " + x;
```

- 需 `-ea` 开启，生产环境默认关闭。
- 只用于开发期自检（不变量、前置条件）。
- **不要**用断言校验用户输入。

## 12. 速查

| 场景 | 做法 |
|---|---|
| 读文件 | try-with-resources |
| 参数校验 | 抛 `IllegalArgumentException` |
| 状态非法 | 抛 `IllegalStateException` |
| 业务失败 | 自定义 `BizException` |
| 外部调用失败 | 包装为系统异常，保留 cause |
| 线程中断 | catch 后 `Thread.currentThread().interrupt()` |
| 日志 | `log.error("msg {}", ctx, e)` |
| 兜底 | 最外层 `@RestControllerAdvice` |
