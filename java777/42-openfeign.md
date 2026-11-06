# 42 · OpenFeign 声明式调用

## 1. 依赖与启用

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableFeignClients(basePackages = "com.example.client")
public class OrderApp { }
```

## 2. 定义接口

```java
@FeignClient(name = "user-service", path = "/api/users",
             fallbackFactory = UserClientFallbackFactory.class)
public interface UserClient {

    @GetMapping("/{id}")
    Result<UserVO> getById(@PathVariable("id") Long id);

    @GetMapping
    Result<List<UserVO>> listByIds(@RequestParam("ids") List<Long> ids);

    @PostMapping
    Result<Long> create(@RequestBody UserDTO dto);       // ★ 必须有 @RequestBody

    @GetMapping("/search")
    Result<PageResult<UserVO>> search(@SpringQueryMap UserQuery query);   // GET 传对象
}
```

> **必须有 `@PathVariable` 的 value**：`@PathVariable("id")`，否则报错（Feign 要求显式命名）。

## 3. 超时配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:                    # 全局
            connect-timeout: 3000     # 建连超时
            read-timeout: 10000       # ★ 读取超时（默认 60s，太长）
            logger-level: basic
          user-service:               # 单服务覆盖
            connect-timeout: 2000
            read-timeout: 5000
      compression:
        request:
          enabled: true
          mime-types: application/json
          min-request-size: 2048
```

> 生产**必须**配超时，否则下游卡死会拖垮上游线程池。

## 4. 日志（排查利器）

```yaml
logging:
  level:
    com.example.client.UserClient: debug    # ★ 接口全限定名
```
```java
@Bean
public Logger.Level feignLoggerLevel() { return Logger.Level.FULL; }
```
级别：`NONE`（默认）/ `BASIC`（方法与 URL、耗时）/ `HEADERS` / `FULL`（含请求响应体）。

## 5. 拦截器：传递 token / traceId

```java
@Component
public class FeignAuthInterceptor implements RequestInterceptor {
    @Override public void apply(RequestTemplate template) {
        String token = TokenContext.get();
        if (token != null) template.header(HttpHeaders.AUTHORIZATION, "Bearer " + token);
        template.header("X-Trace-Id", MDC.get("traceId"));
    }
}
// 全局生效（在 @FeignClient 的 configuration 中指定可限定单个 client）
```

## 6. 降级 Fallback

```java
@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {
    @Override public UserClient create(Throwable cause) {
        return new UserClient() {
            @Override public Result<UserVO> getById(Long id) {
                log.error("用户服务降级", cause);
                return Result.ok(UserVO.anonymous());      // 返回兜底数据
            }
            // ...
        };
    }
}
```

```yaml
# 需要熔断支持（Resilience4j / Sentinel）
feign:
  circuitbreaker:
    enabled: true
```

> 单纯 `fallback` 会在**任何异常**（含业务异常）时降级；用 `fallbackFactory` 可拿到异常原因区分处理。

## 7. 重试

```java
@Bean
public Retryer feignRetryer() {
    return new Retryer.Default(100, 1000, 3);   // 间隔 100ms，最大 1s，重试 3 次
}
```

> 缺点：Feign 自带重试比较粗暴，且**不区分幂等**。生产推荐用 **Resilience4j 的 Retry**（见 44）或直接关掉重试 + 幂等设计。

## 8. 编码器/解码器

默认 `SpringEncoder`/`SpringDecoder`（Jackson）。可自定义：

```java
@Bean
public Decoder feignDecoder(ObjectFactory<HttpMessageConverters> converters) {
    return new ResponseEntityDecoder(new SpringDecoder(converters));
}
```

## 9. 异常处理

```java
@Bean
public ErrorDecoder feignErrorDecoder() {
    return (methodKey, response) -> {
        int status = response.status();
        if (status >= 500) return new BizException("下游服务异常: " + methodKey);
        // 解析 body 中的 Result
        try (Reader r = response.body().asReader(StandardCharsets.UTF_8)) {
            Result<?> res = JsonUtil.read(r, Result.class);
            return new BizException(res.getMsg());
        } catch (Exception e) {
            return new BizException("调用失败 status=" + status);
        }
    };
}
```

异常类型：`FeignException`（4xx/5xx）、`RetryableException`（超时/IO，可重试）。

## 10. HTTP 客户端替换

默认 `HttpURLConnection`（无连接池）。推荐换成 **Apache HttpClient5** 或 **OkHttp**：

```xml
<dependency><groupId>io.github.openfeign</groupId>
           <artifactId>feign-hc5</artifactId></dependency>
```
```yaml
spring:
  cloud:
    openfeign:
      httpclient:
        hc5:
          enabled: true
          max-connections: 200
          max-connections-per-route: 50
      okhttp:
        enabled: false
```
或用 OkHttp：`feign-okhttp` + `spring.cloud.openfeign.okhttp.enabled=true`。

> 不换连接池在高并发下性能很差（每次新建连接）。

## 11. 常见问题与坑

| 坑 | 现象 | 解决 |
|---|---|---|
| `PathVariable annotation was empty` | 启动失败 | 显式写 `@PathVariable("id")` |
| 参数丢失 | 下游收到 null | GET 多参数用 `@RequestParam`；对象用 `@SpringQueryMap`；POST 必须 `@RequestBody` |
| 超时不生效 | 一直等 60s | 配置 read-timeout（注意 `default` 段） |
| 404 | 路径错 | `path` + 方法路径拼接 |
| 降级不生效 | 无 fallback | `feign.circuitbreaker.enabled=true` + 引入 resilience4j |
| 服务名大小写 | `UnknownHost` | 服务名小写、与注册中心一致 |
| 文件上传 | 失败 | 需要 `feign-form` 扩展 + `SpringFormEncoder` |
| 循环依赖 | 启动失败 | 两个服务互相 Feign 调用要避免，或用事件解耦 |
| 未使用连接池 | 性能差 | 启用 hc5/okhttp |

## 12. 面试速答

- **Feign 原理**：基于 JDK 动态代理，解析接口注解生成 `RequestTemplate` → 编码器 → LoadBalancer 选实例 → HTTP 客户端执行 → 解码器。
- **Feign 与 RestTemplate**：Feign 声明式、面向接口、与 LoadBalancer/熔断天然集成；RestTemplate 命令式、灵活。
- **如何做降级**：`fallback` / `fallbackFactory` + `feign.circuitbreaker.enabled=true` + Resilience4j。
- **超时怎么配**：`spring.cloud.openfeign.client.config.default.connect-timeout/read-timeout`，非幂等接口建议关闭重试。
