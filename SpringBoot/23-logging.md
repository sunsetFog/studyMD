# 23 · 日志体系

## 一、默认栈

SpringBoot 默认 **SLF4J 门面 + Logback 实现**（starter-web 自动引入）。禁止直接用 System.out / log4j1.x / 各框架原生 API。

## 二、基础用法

```java
// 推荐 Lombok
@Slf4j
@Service
public class OrderService { }

// 或传统声明
private static final Logger log = LoggerFactory.getLogger(OrderService.class);
```

**占位符而非字符串拼接**（不满足级别时不执行拼接）：

```java
log.debug("查询用户 id={}, 状态={}", id, status);   // 好
log.debug("查询用户 id=" + id);                     // 差：级别不满足也拼接
log.error("下单失败 orderId=" + id, e);             // error 必须带异常堆栈
```

## 三、日志级别与配置

```yaml
logging:
  level:
    root: info
    com.example.demo: debug        # 业务代码详细
    org.springframework: warn      # 框架降噪
    com.zaxxer.hikari: warn
  file:
    name: /var/log/app/app.log
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%-5level) [%thread] [%X{traceId}] %logger{36} - %msg%n"
```

级别：`TRACE < DEBUG < INFO < WARN < ERROR`。

## 四、logback-spring.xml 生产配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty scope="context" name="APP_NAME" source="spring.application.name"/>

    <!-- 控制台 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %highlight(%-5level) [%X{traceId}] %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 滚动文件 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/app/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>/var/log/app/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>          <!-- 保留 30 天 -->
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] [%X{traceId}] %logger - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 错误日志单独收集（便于告警对接） -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/app/error.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>/var/log/app/error.%d{yyyy-MM-dd}.log.gz</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
        <appender-ref ref="ERROR_FILE"/>
    </root>
</configuration>
```

## 五、MDC 链路追踪（日志关联的钥匙）

```java
// 1. 过滤器/切面在请求入口设置
@Component
public class TraceIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse resp,
                                    FilterChain chain) throws IOException, ServletException {
        String traceId = req.getHeader("X-Trace-Id");
        if (traceId == null || traceId.isBlank()) {
            traceId = UUID.randomUUID().toString().replace("-", "").substring(0, 16);
        }
        MDC.put("traceId", traceId);
        resp.setHeader("X-Trace-Id", traceId);        // 回传给前端/上游
        try {
            chain.doFilter(req, resp);
        } finally {
            MDC.clear();                               // 必须清理
        }
    }
}
```

```java
// 2. pattern 中 %X{traceId} 输出，grep traceId 即可串起一次请求所有日志
// 3. 异步线程传递：TaskDecorator
public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> context = MDC.getCopyOfContextMap();
        return () -> {
            if (context != null) MDC.setContextMap(context);
            try { runnable.run(); }
            finally { MDC.clear(); }
        };
    }
}
executor.setTaskDecorator(new MdcTaskDecorator());
```

> 大型项目直接用 **SkyWalking / OpenTelemetry**，traceId/spanId 自动注入 MDC。

## 六、生产日志规范

| 规则 | 说明 |
|---|---|
| 生产不打 debug | level=info 起步 |
| 打日志必带业务上下文 | orderId/userId，不要只打「操作失败」 |
| 不打敏感信息 | 密码/手机号脱敏，token 全不打 |
| error 必须带异常对象 | `log.error("msg", e)` |
| 循环里禁止逐条 log | 聚合后打一条 |
| JSON 日志（可选） | logstash-logback-encoder 输出 JSON，对接 ELK |

## 常见坑

- **坑 1**：classpath 混入多个日志实现（log4j-over-slf4j 与 log4j 共存）→ 启动警告/冲突，用 `mvn dependency:tree` 排除
- **坑 2**：MDC 用完不清理 → 线程池复用串 traceId
- **坑 3**：日志文件无限增长 → 配 rollingPolicy + maxHistory + totalSizeCap
- **坑 4**：异常日志 `log.error(e.getMessage())` 无堆栈 → `log.error("ctx", e)`
- **坑 5**：@Slf4j 类里 log 不可用 → Lombok 注解处理未开启

## Checklist

- [ ] 统一 Lombok @Slf4j + 占位符
- [ ] traceId 贯穿：入口设置、异步传递、MDC 清理
- [ ] logback 滚动策略配置完整
- [ ] error 日志单独文件，可对接告警
