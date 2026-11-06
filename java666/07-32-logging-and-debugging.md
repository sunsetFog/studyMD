# 32 · 日志与调试

## 1. 日志门面与实现

```
你的代码
   ↓
SLF4J（门面/API）
   ↓
Logback / Log4j2 / JUL（实现）
```

| 组件 | 角色 |
|---|---|
| SLF4J | 门面，代码只依赖它 |
| Logback | SLF4J 原生实现，Spring Boot 默认 |
| Log4j2 | 性能更强（异步日志），功能丰富 |
| JUL (java.util.logging) | JDK 自带，功能弱 |
| JCL (commons-logging) | 老门面，Spring 内部用 |

**永远面向 SLF4J 编程**，不要直接依赖实现类。

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
</dependency>
<!-- 实现只在运行时引入 -->
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <scope>runtime</scope>
</dependency>
```

## 2. SLF4J 用法

```java
private static final Logger log = LoggerFactory.getLogger(OrderService.class);

log.trace("trace");           // 最细，几乎不用
log.debug("debug: id={}", id);
log.info("订单创建成功 orderId={}", orderId);
log.warn("库存不足 sku={}, need={}, left={}", sku, need, left);
log.error("支付回调失败 orderId={}", orderId, e);     // 最后一个参数是异常
```

### 参数化日志（重点）

```java
// ✔ 正确：占位符，只有需要输出时才拼接
log.debug("user={}", user);

// ✘ 错误：无论级别是否开启都会执行字符串拼接
log.debug("user=" + user);

// ✘ 错误：异常当占位符参数
log.error("failed: {}", e);            // 只打印 e.toString()，无堆栈
log.error("failed", e);                // ✔ 保留堆栈

// 多参数 + 异常
log.error("failed id={} name={}", id, name, e);
```

### 条件日志

```java
if (log.isDebugEnabled()) {
    log.debug("expensive: {}", expensiveToString());
}

// 或 Supplier（Log4j2 / Logback 1.3+ 支持）
log.debug("expensive: {}", () -> expensiveToString());
```

## 3. 日志级别

| 级别 | 用途 | 生产建议 |
|---|---|---|
| TRACE | 详细流程追踪 | 关闭 |
| DEBUG | 调试信息（入参、中间值） | 关闭 / 按包开启 |
| INFO | 关键业务节点（启动、订单创建） | 开启 |
| WARN | 可恢复的异常（重试、降级） | 开启 |
| ERROR | 需要人工介入的错误 | 开启 + 告警 |

选择原则：

- ERROR：影响用户、需要处理（DB 挂、第三方超时）。
- WARN：不影响主流程但异常（缓存失效、重试成功）。
- INFO：业务里程碑（每分钟几条，不是每条请求都有）。
- DEBUG：排查用，含参数与分支。

## 4. Logback 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property name="LOG_DIR" value="./logs"/>
    <property name="PATTERN"
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{traceId}] - %msg%n"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_DIR}/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_DIR}/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>${PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 错误单独文件 -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_DIR}/error.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_DIR}/error.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder><pattern>${PATTERN}</pattern></encoder>
    </appender>

    <!-- 异步提升性能 -->
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>1024</queueSize>
        <discardingThreshold>0</discardingThreshold>   <!-- 0 = 不丢弃 INFO 及以下 -->
        <neverBlock>true</neverBlock>                   <!-- 队列满时不阻塞业务 -->
        <appender-ref ref="FILE"/>
    </appender>

    <logger name="com.acme" level="DEBUG"/>
    <logger name="org.springframework" level="WARN"/>
    <logger name="com.zaxxer.hikari" level="INFO"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="ASYNC"/>
        <appender-ref ref="ERROR_FILE"/>
    </root>
</configuration>
```

### 异步日志注意

- `neverBlock=true` 会在队列满时**丢弃日志**（性能优先）。
- 进程退出前需 flush（Logback 有 shutdown hook）。
- Log4j2 的 AsyncLogger（Disruptor）性能更好。

## 5. Spring Boot 日志配置

```yaml
logging:
  level:
    root: info
    com.acme: debug
  file:
    name: logs/app.log
  logback:
    rollingpolicy:
      max-file-size: 100MB
      max-history: 30
  pattern:
    console: "%d{HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n"
```

## 6. MDC 与链路追踪

```java
// 请求入口设置
MDC.put("traceId", UUID.randomUUID().toString().replace("-", ""));
MDC.put("userId", String.valueOf(userId));
try {
    chain.doFilter(req, resp);
} finally {
    MDC.clear();            // 必须清理，线程池复用会串数据
}
```

```xml
<pattern>%d{HH:mm:ss.SSS} [%thread] %-5level [%X{traceId}] %logger{36} - %msg%n</pattern>
```

异步任务中 MDC 会丢失：

```java
Map<String, String> ctx = MDC.getCopyOfContextMap();
executor.submit(() -> {
    if (ctx != null) MDC.setContextMap(ctx);
    try { task.run(); } finally { MDC.clear(); }
});
```

> 生产环境用 SkyWalking / Zipkin / OpenTelemetry 做全链路追踪。

## 7. 日志规范

1. **禁止** `System.out.println`（无级别、无时间、无异步、无法关闭）。
2. 日志内容用**英文**或统一中文，避免中英混杂与乱码。
3. 用占位符 `{}`，不要字符串拼接。
4. ERROR 必须带异常对象。
5. 不要在循环里打 INFO（日志风暴）。
6. 不要打印敏感信息（密码、身份证、银行卡、token），必须脱敏。
7. 日志要有**唯一标识**（orderId、requestId），便于串联。
8. 异常不要重复打印（catch 里打印了又往上抛，上层再打印）。
9. 大对象不要 `toString()` 全量打印（用关键字段）。
10. 日志文件必须配置滚动与清理，避免磁盘打满。

```java
// 脱敏示例
private static String maskPhone(String p) {
    return p == null ? null : p.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2");
}
```

## 8. 调试技巧

### IDE 断点

| 功能 | 用途 |
|---|---|
| 行断点 | 常规暂停 |
| 条件断点 | `orderId == 1001` 时才停 |
| 日志断点 | 不暂停，只打印（性能好于加日志重启） |
| 异常断点 | 任何 NPE 抛出时暂停 |
| 方法断点 | 进入/退出方法时暂停 |
| 字段观察点 | 字段被修改时暂停 |

### 远程调试

```bash
# 启动时加参数
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005 -jar app.jar

# IDEA：Run → Edit Configurations → Remote JVM Debug → localhost:5005
```

安全：生产环境**禁止**开放调试端口（可执行任意代码）。

### 线程转储分析

```bash
jstack <pid> > stack.txt
# 或
kill -3 <pid>          # 输出到应用日志
```

关注：BLOCKED（锁竞争）、WAITING（等待）、RUNNABLE 但 CPU 高（死循环）。

### 常用诊断命令

```bash
jps -l                        # 列出 Java 进程
jstat -gcutil <pid> 1000 10   # GC 统计
jmap -heap <pid>              # 堆信息
jmap -dump:format=b,file=heap.hprof <pid>
jcmd <pid> Thread.print       # 线程栈
jcmd <pid> GC.heap_info
```

### Arthas 常用命令

```bash
# 启动
java -jar arthas-boot.jar

dashboard                     # 总览（线程、内存、GC）
thread -n 3                   # CPU 最高的 3 个线程
thread -b                     # 找出阻塞其他线程的线程（死锁）
trace com.acme.OrderService create   # 方法调用链路耗时
watch com.acme.OrderService create '{params, returnObj, throwExp}' -x 3
jad com.acme.OrderService     # 反编译
sc -d com.acme.*Service       # 查找类
ognl '@java.lang.System@getProperty("java.version")'
```

## 9. 生产排障流程

```
1. 看监控：错误率、RT、CPU、内存、GC
2. 看日志：ERROR + traceId 关联
3. 看线程栈：是否有死锁/阻塞
4. 看堆：是否有内存泄漏（jmap + MAT）
5. 看 GC 日志：是否频繁 Full GC
6. 复现：本地/预发加日志或断点
```

### CPU 100% 排查

```bash
top -Hp <pid>                  # 找出高 CPU 线程
printf "%x\n" <tid>            # 转十六进制
jstack <pid> | grep -A 30 <hex>  # 定位栈
```

### 内存泄漏排查

```bash
jmap -histo:live <pid> | head -20    # 对象直方图
jmap -dump:live,format=b,file=heap.hprof <pid>
# 用 Eclipse MAT / JProfiler 分析支配树、GC Root 引用链
```

## 10. 常见坑

1. `e.printStackTrace()`（输出到 stderr，无法收集、无时间戳）。
2. `log.error(e.getMessage())`（丢堆栈）。
3. 日志里拼接 SQL 参数（性能 + 注入风险）。
4. `log.debug("x=" + x)` 在热点路径（字符串拼接开销）。
5. MDC 未清理导致线程池串号。
6. 异步 Appender 队列满丢日志却不知情。
7. 日志文件未滚动，磁盘打满导致服务挂。
8. 生产开 DEBUG（IO 与磁盘爆炸）。
9. 敏感信息入日志（合规风险）。
10. 重复记录同一个异常（一次请求 5 条相同 ERROR）。

## 11. 速查

| 需求 | 做法 |
|---|---|
| 声明 Logger | `private static final Logger log = LoggerFactory.getLogger(X.class);` |
| 打印参数 | `log.info("id={}", id)` |
| 打印异常 | `log.error("msg", e)` |
| 条件日志 | `if (log.isDebugEnabled())` |
| 链路 id | `MDC.put("traceId", id)` |
| 异步 | `AsyncAppender` |
| 敏感脱敏 | 正则替换 |
| 排查 CPU | `top -Hp` + `jstack` |
| 排查内存 | `jmap -dump` + MAT |
