# 15 · 异常体系

## 一、Throwable 家族

```
Throwable
├── Error          —— 严重系统错误，不该捕获（OOM、StackOverflowError）
└── Exception
    ├── RuntimeException（非受检 unchecked）
    │     NullPointerException / IndexOutOfBounds / IllegalArgumentException
    │     ArithmeticException / ClassCastException / IllegalStateException
    └── 其他（受检 checked）
          IOException / SQLException / InterruptedException / ParseException
```

| 类别 | 处理义务 | 语义 |
|---|---|---|
| 受检 Exception | 编译器强制 try/catch 或 throws | 可预期、可恢复的外部故障 |
| 非受检 RuntimeException | 自愿处理 | 编程错误/业务异常 |
| Error | 不处理 | JVM 级不可恢复 |

## 二、try-catch-finally

```java
try {
    int r = 10 / 0;
} catch (ArithmeticException e) {
    // 处理（或包装重抛）
} catch (NullPointerException | IllegalArgumentException e) {   // 多类型合并（Java 7+）
    log.error("参数问题", e);
} catch (Exception e) {
    // 兜底：catch 子类型放前面！父类在前编译错误
} finally {
    // 一定执行（除 JVM 退出）；释放资源用
}
```

### try-with-resources（资源释放唯一正确姿势）

```java
// AutoCloseable：文件流/连接/socket 全实现它
try (var reader = new BufferedReader(new FileReader("a.txt"));
     var writer = new BufferedWriter(new FileWriter("b.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) writer.write(line);
} catch (IOException e) {
    log.error("copy failed", e);
}
// 自动逆序 close()，异常抑制机制自动处理，胜过手写 finally
```

## 三、throw / throws / 自定义异常

```java
public void withdraw(BigDecimal amount) {
    if (amount.signum() <= 0) {
        throw new IllegalArgumentException("金额必须为正: " + amount);
    }
}

// 声明受检异常
public String read(String path) throws IOException { ... }

// 自定义业务异常（继承 RuntimeException，业务代码首选）
public class BizException extends RuntimeException {
    private final int code;
    public BizException(int code, String message) {
        super(message);
        this.code = code;
    }
    public int getCode() { return code; }
}
```

## 四、异常链与包装

```java
// 捕获底层异常，包装成业务异常抛出——保留 cause！
try {
    Files.readString(path);
} catch (IOException e) {
    throw new BizException(500, "配置读取失败", e);   // 第三参数 cause
}
// 打印时 getCause() 链完整，排查不丢根因
```

## 五、异常设计最佳实践

| 规则 | 说明 |
|---|---|
| 能用条件判断就别用异常 | 可预见的业务分支（用户不存在）返回 Optional，异常留给异常情况 |
| catch 必须处理或重抛 | 空 catch = 吞异常 = 事故黑洞 |
| 打日志带堆栈 | `log.error("ctx", e)` 不是 `log.error(e.getMessage())` |
| 不用异常做流程控制 | 性能 + 可读性双输 |
| 业务异常继承 RuntimeException | 免去处处 throws 的污染 |
| 对外 API 收敛异常 | 框架层统一兜底转响应（衔接 SpringBoot 系列 07 篇） |
| finally 里不 return | 吞掉 try/catch 的异常 |

### 受检 vs 非受检的现代共识

《Effective Java》：可恢复用受检、程序错误用非受检。**工程实践**：业务异常一律非受检（RuntimeException），受检异常在边界（IO/网络）出现时捕获并包装——避免 `throws Exception` 层层传染。

## 六、性能常识

- 异常构造会**填充栈轨迹**（开销大）；热路径高频抛异常是性能杀手
- 抑制栈轨迹：`new Exception(msg, null, false, false)`（极端优化场景）
- 正常路径零异常 = 零成本；异常路径慢百倍也无所谓（低频）

## 常见坑

- **坑 1**：空 catch 吞异常 → 静默失败
- **坑 2**：`catch (Exception e)` 一把抓掩盖真实类型 → 精确 catch 或分层兜底
- **坑 3**：包装时丢 cause → `throw new BizException(msg)` 丢堆栈链；必传 e
- **坑 4**：finally 与 return/异常交互 → finally 里 return 吞异常
- **坑 5**：try-with-resources 资源变量在 try 外声明 → 用增强 try 语法或提前判空
- **坑 6**：多 catch 顺序（父类在前）→ 编译错误「已捕获」

## Checklist

- [ ] 资源一律 try-with-resources
- [ ] 业务异常统一 BizException(code, msg)
- [ ] 异常链保留 cause
- [ ] 无空 catch；error 日志带堆栈
