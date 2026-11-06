# 08 · Filter / Interceptor / AOP 三层拦截体系

## 一、全景对比

| 维度 | Filter | Interceptor | AOP |
|---|---|---|---|
| 层级 | Servlet 容器层 | Spring MVC 层 | Bean 方法层 |
| 依赖 | 不依赖 Spring | 依赖 Spring | 依赖 Spring 容器 |
| 能否拿到 Handler | 否 | 是（HandlerMethod） | 是（方法签名/注解） |
| 典型场景 | 编码、XSS、CORS、认证 token 初筛 | 登录鉴权、日志、接口幂等 | 事务、缓存、审计、限流 |
| 执行顺序 | 最先 | 中间 | 最内层 |

执行顺序：`Filter → DispatcherServlet → Interceptor.preHandle → AOP → Controller 方法 → AOP 出 → postHandle → afterCompletion → Filter`

## 二、Filter

```java
@Component
public class AuthFilter extends OncePerRequestFilter {   // 保证一次请求只执行一次

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        String token = request.getHeader("Authorization");
        if (token == null && isProtected(request)) {
            response.setStatus(401);
            response.setContentType("application/json;charset=UTF-8");
            response.getWriter().write("{\"code\":401,\"message\":\"未登录\"}");
            return;   // 不放行
        }
        chain.doFilter(request, response);
    }
}
```

注册方式：`@Component` 自动注册到所有路径，或精细注册：

```java
@Bean
public FilterRegistrationBean<AuthFilter> authFilter() {
    FilterRegistrationBean<AuthFilter> reg = new FilterRegistrationBean<>(new AuthFilter());
    reg.addUrlPatterns("/api/*");
    reg.setOrder(1);
    return reg;
}
```

## 三、Interceptor（拦截器）

```java
@Component
@RequiredArgsConstructor
public class LoginInterceptor implements HandlerInterceptor {

    private final TokenService tokenService;

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response, Object handler) {
        // 放行跨域预检
        if (HttpMethod.OPTIONS.matches(request.getMethod())) return true;

        if (!(handler instanceof HandlerMethod hm)) return true;  // 静态资源放行

        // 方法/类级注解判断：@RequireLogin
        if (!hm.hasMethodAnnotation(RequireLogin.class)
                && !hm.getBeanType().isAnnotationPresent(RequireLogin.class)) {
            return true;
        }

        String token = request.getHeader("Authorization");
        LoginUser user = tokenService.parse(token);
        if (user == null) {
            response.setStatus(401);
            return false;
        }
        UserContext.set(user);            // 存入 ThreadLocal（见下）
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse resp,
                                Object handler, Exception ex) {
        UserContext.clear();              // 必须清理，防内存泄漏（线程池复用）
    }
}
```

注册（注意排除路径）：

```java
@Configuration
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {

    private final LoginInterceptor loginInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/auth/login", "/api/auth/register")
                .order(0);
    }
}
```

**ThreadLocal 用户上下文**：

```java
public class UserContext {
    private static final ThreadLocal<LoginUser> HOLDER = new ThreadLocal<>();
    public static void set(LoginUser u) { HOLDER.set(u); }
    public static LoginUser get() { return HOLDER.get(); }
    public static void clear() { HOLDER.remove(); }
}
```

## 四、AOP 层拦截（详见 17 篇）

```java
@Aspect
@Component
public class ApiLogAspect {
    @Around("@annotation(com.example.demo.common.annotation.ApiLog)")
    public Object around(ProceedingJoinPoint pjp) throws Throwable { ... }
}
```

## 五、如何选层

```
需要处理"所有 HTTP 请求"（含静态资源、跨域）        → Filter
需要知道"哪个 Controller 方法"（读注解做鉴权/日志）  → Interceptor
需要知道"方法参数/返回值"并横切多个 Service        → AOP
```

实操建议：认证 token 解析放 Filter/Interceptor，业务增强（审计、缓存、分布式锁）放 AOP。

## 常见坑

- **坑 1**：拦截器忘了 `afterCompletion` 里清理 ThreadLocal → 线程池复用导致串用户数据（严重安全事故）
- **坑 2**：OPTIONS 预检请求被拦截器拦截 → 前端 CORS 失败，需放行 OPTIONS
- **坑 3**：Filter 里抛异常进不了 @RestControllerAdvice → 全局异常处理器只覆盖 MVC 层
- **坑 4**：Interceptor 中注入 Bean 为 null → 拦截器被 `new` 出来而非交给 Spring 管理
- **坑 5**：异步请求返回 DeferredResult 时 preHandle/postHandle 语义不同 → 用 AsyncHandlerInterceptor

## Checklist

- [ ] ThreadLocal 上下文在 afterCompletion 清理
- [ ] 登录/公开路径通过 excludePathPatterns 统一管理
- [ ] 注解式鉴权（@RequireLogin/@RequirePermission）优先于硬编码路径判断
- [ ] 多个 Filter/Interceptor 顺序明确（order 值）
