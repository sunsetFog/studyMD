# 13 · 日志体系：Logback / Log4j2 / MDC

## 1. 门面 + 实现

```
SLF4J（门面） → Logback（默认实现）
              → Log4j2（可选）
              → JUL / JCL（桥接）
```

Boot 默认 **SLF4J + Logback**，通过 `spring-boot-starter-logging` 引入，并把 JUL、JCL、Log4j(1) 全部桥接到 SLF4J。

> **铁律**：代码里只用 `org.slf4j.Logger`，不要直接依赖 Logback/Log4j2 的类。

```java
@Slf4j                       // Lombok，等价于 private static final Logger log = LoggerFactory.getLogger(Xxx.class);
public class Demo {
    void m() {
        log.trace("..."); log.debug("..."); log.info("..."); log.warn("..."); log.error("...");
        // ★ 占位符，避免字符串拼接开销
        log.info("user={} orderId={}", userId, orderId);
        log.error("支付失败, orderId={}", orderId, e);   // 最后一个参数是异常
    }
}
```

## 2. 基础配置

```yaml
logging:
  level:
    root: info
    com.example: debug
    org.springframework.web: warn
    org.hibernate.SQL: debug
  file:
    name: logs/app.log          # 或 logging.file.path: logs/
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n"
    file: "%d %-5level [%thread] %logger{36} - %msg%n"
  logback:
    rollingpolicy:
      max-file-size: 100MB
      max-history: 30
      total-size-cap: 5GB
```

## 3. Logback 配置详解（logback-spring.xml）

放在 `src/main/resources/logback-spring.xml`（**推荐用 `-spring` 后缀**，可享受 Boot 的 profile 支持）。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty scope="context" name="APP_NAME" source="spring.application.name"/>
    <property name="LOG_HOME" value="logs"/>
    <property name="PATTERN"
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread][%X{traceId}] %logger{40} - %msg%n"/>

    <!-- 控制台 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder><pattern>${PATTERN}</pattern></encoder>
    </appender>

    <!-- 滚动文件 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_HOME}/${APP_NAME}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_HOME}/${APP_NAME}.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>5GB</totalSizeCap>
        </rollingPolicy>
        <encoder><pattern>${PATTERN}</pattern></encoder>
    </appender>

    <!-- ERROR 单独文件 -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
        <!-- ... -->
    </appender>

    <springProfile name="dev">
        <root level="INFO"><appender-ref ref="CONSOLE"/></root>
        <logger name="com.example" level="DEBUG"/>
    </springProfile>

    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="FILE"/><appender-ref ref="ERROR_FILE"/>
        </root>
        <logger name="com.example" level="INFO"/>
    </springProfile>
</configuration>
```

> `<springProfile>` 与 `<springProperty>` 是 Boot 扩展，**必须**用 `logback-spring.xml` 文件名才生效。

## 4. 切换到 Log4j2

```xml
<dependency>
    <groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId>
    <exclusions><exclusion><groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-logging</artifactId></exclusion></exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

配置 `log4j2-spring.xml`。Log4j2 优势：**异步日志（LMAX Disruptor）性能极高**，适合高 QPS。

```xml
<Configuration>
  <Appenders>
    <Async name="Async">
      <AppenderRef ref="RollingFile"/>
    </Async>
  </Appenders>
</Configuration>
```

> 需要 `com.lmax:disruptor` 依赖才能开启真正的异步（AsyncAppender 与 AsyncLogger 不同）。

## 5. MDC：链路追踪的关键

MDC（Mapped Diagnostic Context）是 Logback/Log4j2 提供的**线程级** Map，在 pattern 中用 `%X{key}` 输出。

```java
// 拦截器/过滤器中放入 traceId
public class TraceFilter extends OncePerRequestFilter {
    @Override protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                              FilterChain chain) throws ServletException, IOException {
        String traceId = Optional.ofNullable(req.getHeader("X-Trace-Id"))
                                 .orElseGet(() -> UUID.randomUUID().toString().replace("-", ""));
        MDC.put("traceId", traceId);
        res.setHeader("X-Trace-Id", traceId);
        try { chain.doFilter(req, res); }
        finally { MDC.remove("traceId"); }     // ★ 必须清理，线程复用会串号
    }
}
```

```java
@Bean
public FilterRegistrationBean<TraceFilter> traceFilter() {
    FilterRegistrationBean<TraceFilter> b = new FilterRegistrationBean<>(new TraceFilter());
    b.setOrder(Ordered.HIGHEST_PRECEDENCE);
    return b;
}
```

> 有了 Micrometer Tracing / Sleuth 后，traceId 会自动进 MDC（见 [60](60-distributed-tracing.md)）。

## 6. 异步线程的 MDC 传递

```java
@Bean
public TaskDecorator mdcTaskDecorator() {
    return r -> {
        Map<String, String> ctx = MDC.getCopyOfContextMap();
        return () -> {
            if (ctx != null) MDC.setContextMap(ctx);
            try { r.run(); } finally { MDC.clear(); }
        };
    };
}

@Bean("taskExecutor")
public ThreadPoolTaskExecutor executor(TaskDecorator decorator) {
    ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
    e.setCorePoolSize(8); e.setMaxPoolSize(16); e.setQueueCapacity(500);
    e.setThreadNamePrefix("biz-");
    e.setTaskDecorator(decorator);      // ★
    e.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    e.initialize();
    return e;
}
```

> 同样适用于 `SecurityContextHolder`（Spring Security 的 `DelegatingSecurityContextRunnable`）。

## 7. 日志级别动态修改

```yaml
management:
  endpoints:
    web:
      exposure:
        include: loggers
```

```bash
curl -X POST localhost:8080/actuator/loggers/com.example \
  -H 'Content-Type: application/json' -d '{"configuredLevel":"DEBUG"}'
```

> 排障利器，但**生产务必限制访问权限**。

## 8. 日志规范建议

| 项 | 建议 |
|---|---|
| 级别 | ERROR（需人工介入）/ WARN（可自愈异常）/ INFO（关键业务节点）/ DEBUG（调试，生产关闭） |
| 内容 | 谁、何时、做什么、结果、耗时、关键 ID（orderId/userId/traceId） |
| 禁止 | 打印密码、身份证、银行卡、token（**脱敏**） |
| 异常 | `log.error("xxx失败 id={}", id, e)`，异常放最后参数才能打印堆栈 |
| 性能 | 用占位符；`if (log.isDebugEnabled())` 包裹复杂拼接 |
| 慢查询 | 超过阈值的 SQL 单独 WARN |

```java
// 敏感脱敏
log.info("用户注册 mobile={}", Desensitize.mobile(phone));  // 138****8888
```

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| MDC 串号 | 日志里 traceId 张冠李戴 | `finally` 中 `MDC.remove/clear` |
| 异步丢 traceId | 异步日志无链路 | `TaskDecorator` 传递 |
| 日志文件暴涨 | 磁盘打满 | 配置 `maxHistory` + `totalSizeCap` |
| 依赖冲突 | `Class path contains multiple SLF4J bindings` | 排除多余实现（logback/log4j 二选一） |
| `logback.xml` 中 `<springProfile>` 无效 | profile 不生效 | 文件改名 `logback-spring.xml` |
| 循环打日志 | 日志 Appender 里又打日志 | 避免自定义 Appender 递归 |

## 10. 面试速答

- **为什么用 SLF4J**：门面模式，代码与实现解耦，换实现不改业务代码。
- **Logback 与 Log4j2**：Logback 是默认、够用；Log4j2 异步性能更好（Disruptor），高吞吐场景选它。
- **MDC 是什么**：基于 ThreadLocal 的诊断上下文，用于在同一线程日志中输出 traceId/userId；异步需手动传递。
