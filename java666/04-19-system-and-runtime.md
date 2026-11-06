# 19 · System、Runtime 与进程管理

## 1. System 类

`java.lang.System` 是 final 类，全静态方法，禁止实例化。

### 标准流

```java
System.out.println("普通输出");    // PrintStream，带缓冲
System.err.println("错误输出");    // 无缓冲，立即输出
int b = System.in.read();          // InputStream
```

重定向（测试/日志拦截常用）：

```java
PrintStream original = System.out;
System.setOut(new PrintStream(new ByteArrayOutputStream()));
try {
    // 被测代码
} finally {
    System.setOut(original);
}
```

### 系统属性

```java
String v = System.getProperty("java.version");
String home = System.getProperty("user.home");
String line = System.lineSeparator();

System.setProperty("app.env", "prod");           // 运行期设置
Properties props = System.getProperties();       // 全部属性（可修改，慎用）

// 启动时通过 -D 传入
// java -Dapp.env=prod -Dserver.port=8080 -jar app.jar
```

常用属性：

| 属性 | 说明 |
|---|---|
| `java.version` | JDK 版本 |
| `java.home` | JRE 目录 |
| `java.class.path` | 类路径 |
| `java.library.path` | native 库路径 |
| `user.dir` | 工作目录 |
| `user.home` | 用户主目录 |
| `file.separator` | 路径分隔符（`/` 或 `\`） |
| `line.separator` | 换行符 |
| `os.name` / `os.arch` | 操作系统 / 架构 |
| `file.encoding` | 默认字符集（Java 18+ 固定 UTF-8） |

### 环境变量

```java
String path = System.getenv("PATH");
Map<String, String> env = System.getenv();       // 只读

// 启动时：ENV=prod java -jar app.jar
```

> 环境变量不可在运行期修改（JVM 层面），需要修改只能重启或用反射 hack（不推荐）。

### 数组复制

```java
System.arraycopy(src, srcPos, dest, destPos, length);   // native，最快
int[] a = {1,2,3,4,5};
int[] b = new int[5];
System.arraycopy(a, 1, b, 0, 3);    // b = [2,3,4,0,0]
```

- 支持重叠复制（内部处理方向）。
- 类型不匹配抛 `ArrayStoreException`；越界抛 `IndexOutOfBoundsException`。

### 时间与退出

```java
long ms = System.currentTimeMillis();      // 墙钟时间，可被 NTP 调整
long ns = System.nanoTime();               // 单调时钟，仅用于测耗时

System.exit(0);        // 正常退出，执行 shutdown hook
System.exit(1);        // 异常退出
Runtime.getRuntime().halt(1);   // 立即退出，不执行 hook
```

**耗时测量必须用 `nanoTime`**，`currentTimeMillis` 会因系统时间调整出现负值。

### 其他

```java
System.gc();              // 建议 GC（不保证执行，生产禁用）
System.runFinalization(); // 已废弃
System.identityHashCode(obj);  // 与对象默认 hashCode 一致（忽略重写）
```

## 2. Runtime

每个 JVM 一个实例：`Runtime.getRuntime()`。

```java
Runtime rt = Runtime.getRuntime();

rt.availableProcessors();     // CPU 核数（容器内可能不准，见下）
rt.totalMemory();             // 当前堆总容量（字节）
rt.freeMemory();              // 堆空闲
rt.maxMemory();               // 最大可用堆（-Xmx）
rt.usedMemory = total - free;

rt.version();                 // Runtime.Version
rt.addShutdownHook(new Thread(() -> System.out.println("bye")));
rt.removeShutdownHook(thread);
rt.halt(1);
rt.exec("ls");                // 已过时，用 ProcessBuilder
```

### 内存监控片段

```java
public static String heapInfo() {
    Runtime rt = Runtime.getRuntime();
    long total = rt.totalMemory() / 1024 / 1024;
    long free = rt.freeMemory() / 1024 / 1024;
    long max = rt.maxMemory() / 1024 / 1024;
    return "used=%dMB total=%dMB max=%dMB".formatted(total - free, total, max);
}
```

### 容器内 CPU 核数问题

`availableProcessors()` 读的是宿主机核数（JDK 10 之前），在容器里会导致线程池过大。

```java
// JDK 10+ 已支持 cgroup 限制（UseContainerSupport 默认开启）
// 显式指定：
// -XX:ActiveProcessorCount=4
```

## 3. ProcessBuilder

```java
ProcessBuilder pb = new ProcessBuilder("ls", "-l", "/tmp");
pb.directory(new File("/Users/ivory"));
pb.redirectErrorStream(true);                     // 合并 stderr 到 stdout
Map<String, String> env = pb.environment();
env.put("LANG", "en_US.UTF-8");

Process p = pb.start();

// 读取输出（必须消费，否则缓冲区满导致子进程阻塞）
try (BufferedReader r = new BufferedReader(
        new InputStreamReader(p.getInputStream(), StandardCharsets.UTF_8))) {
    r.lines().forEach(System.out::println);
}

boolean finished = p.waitFor(30, TimeUnit.SECONDS);
if (!finished) {
    p.destroyForcibly();
}
int code = p.exitValue();
```

### 超时 + 输出捕获工具

```java
public static String exec(String... cmd) throws Exception {
    Process p = new ProcessBuilder(cmd)
            .redirectErrorStream(true)
            .start();
    StringBuilder out = new StringBuilder();
    Thread reader = new Thread(() -> {
        try (var r = new BufferedReader(new InputStreamReader(p.getInputStream()))) {
            r.lines().forEach(l -> out.append(l).append('\n'));
        } catch (IOException ignored) { }
    });
    reader.start();
    if (!p.waitFor(10, TimeUnit.SECONDS)) {
        p.destroyForcibly();
        throw new TimeoutException("command timeout: " + String.join(" ", cmd));
    }
    reader.join(1000);
    return out.toString();
}
```

### 安全提示

- **禁止**把用户输入直接拼进 shell 命令（命令注入）。
- 用数组形式传参（不经过 shell 解析），不要用 `sh -c`。
- 限制执行的命令白名单。

```java
// 危险
Runtime.getRuntime().exec("ping " + userInput);   // userInput = "1.1.1.1; rm -rf /"

// 安全
new ProcessBuilder("ping", "-c", "1", validatedIp).start();
```

## 4. shutdown hook

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // 优雅关闭：停止接收请求、刷新缓冲、关闭连接池
    server.stop();
    dataSource.close();
}, "shutdown-hook"));
```

触发时机：

- `System.exit()`
- 收到 SIGTERM / SIGINT（`kill -15` / Ctrl+C）
- 最后一个非守护线程结束

不触发：

- `Runtime.halt()`
- `kill -9`
- JVM 崩溃 / OOM 后无法恢复时
- native 崩溃

> Spring Boot 的优雅停机本质是注册 `ContextClosedEvent` 或 shutdown hook，等待 `server.shutdown=graceful` 配置的宽限期。

## 5. 桌面与剪贴板（了解）

```java
java.awt.Desktop.getDesktop().browse(URI.create("https://example.com"));
Toolkit.getDefaultToolkit().beep();
```

服务器环境（headless）会抛 `HeadlessException`，需 `-Djava.awt.headless=true`（服务端默认）。

## 6. 获取 PID

```java
// Java 9+
long pid = ProcessHandle.current().pid();
ProcessHandle.current().info().commandLine().ifPresent(System.out::println);
ProcessHandle.current().parent().ifPresent(ph -> System.out.println(ph.pid()));

// 列出所有进程
ProcessHandle.allProcesses()
    .filter(ph -> ph.info().command().orElse("").contains("java"))
    .forEach(ph -> System.out.println(ph.pid() + " " + ph.info().commandLine().orElse("")));

// 监控退出
ProcessHandle.current().onExit().thenRun(() -> System.out.println("exited"));
```

## 7. 常用诊断片段

```java
// 打印所有 JVM 参数相关属性
System.getProperties().entrySet().stream()
      .filter(e -> e.getKey().toString().startsWith("java."))
      .sorted(Map.Entry.comparingByKey())
      .forEach(e -> System.out.println(e.getKey() + "=" + e.getValue()));

// 内存快照
MemoryMXBean memory = ManagementFactory.getMemoryMXBean();
MemoryUsage heap = memory.getHeapMemoryUsage();
System.out.printf("init=%d used=%d committed=%d max=%d%n",
    heap.getInit(), heap.getUsed(), heap.getCommitted(), heap.getMax());

// 线程数
ThreadMXBean threads = ManagementFactory.getThreadMXBean();
System.out.println("threads=" + threads.getThreadCount()
    + " peak=" + threads.getPeakThreadCount());

// GC 统计
ManagementFactory.getGarbageCollectorMXBeans().forEach(gc ->
    System.out.println(gc.getName() + " count=" + gc.getCollectionCount()
        + " time=" + gc.getCollectionTime() + "ms"));
```

## 8. 常见坑

1. `System.currentTimeMillis()` 测耗时（应用 `nanoTime`）。
2. `System.gc()` 当万能药（触发 Full GC，伤性能）。
3. 子进程输出不消费 → 缓冲区满 → 死锁。
4. `Runtime.exec(String)` 按空格切分，路径带空格会出错。
5. 容器中 `availableProcessors` 读到宿主机核数（老 JDK）。
6. 依赖 `System.getenv` 可写（实际只读）。
7. 把密码通过 `-D` 传参（`ps` 可见，用环境变量或配置中心）。
8. shutdown hook 中做耗时操作（JVM 等待超时后强杀）。
