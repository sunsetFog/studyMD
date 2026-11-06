# 20 · Spring Security 与 JWT

## 1. 核心概念

```
SecurityFilterChain   过滤器链（由一系列 Filter 组成）
Authentication        认证（你是谁）
Authorization         授权（你能做什么）
UserDetailsService    加载用户
PasswordEncoder       密码加密
SecurityContext       当前用户上下文（ThreadLocal）
```

Boot 3 关键变化：`WebSecurityConfigurerAdapter` **已废弃移除**，改为**声明 `SecurityFilterChain` Bean**。

## 2. 依赖与最小配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId><artifactId>jjwt-api</artifactId><version>0.12.6</version>
</dependency>
<dependency><groupId>io.jsonwebtoken</groupId><artifactId>jjwt-impl</artifactId><version>0.12.6</version><scope>runtime</scope></dependency>
<dependency><groupId>io.jsonwebtoken</groupId><artifactId>jjwt-jackson</artifactId><version>0.12.6</version><scope>runtime</scope></dependency>
```

默认行为：所有请求需认证，提供 `Basic` 认证，随机密码打印在控制台。

## 3. 现代配置（Boot 3 / Security 6）

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity            // 开启 @PreAuthorize
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;
    private final UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
          .csrf(csrf -> csrf.disable())                     // 前后端分离 + JWT，关闭 CSRF
          .cors(Customizer.withDefaults())
          .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
          .authorizeHttpRequests(auth -> auth
              .requestMatchers("/api/auth/**", "/actuator/health", "/doc.html").permitAll()
              .requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
              .requestMatchers("/api/admin/**").hasRole("ADMIN")
              .anyRequest().authenticated())
          .exceptionHandling(e -> e
              .authenticationEntryPoint((req, res, ex) -> write(res, 401, "未登录"))
              .accessDeniedHandler((req, res, ex) -> write(res, 403, "无权限")))
          .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
          .headers(h -> h.frameOptions(f -> f.disable()));
        return http.build();
    }

    @Bean public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }

    @Bean public AuthenticationManager authManager(AuthenticationConfiguration c) throws Exception {
        return c.getAuthenticationManager();
    }

    @Bean public DaoAuthenticationProvider authProvider() {
        DaoAuthenticationProvider p = new DaoAuthenticationProvider();
        p.setUserDetailsService(userDetailsService);
        p.setPasswordEncoder(passwordEncoder());
        return p;
    }
}
```

## 4. 登录接口与 UserDetailsService

```java
@Service
@RequiredArgsConstructor
public class UserDetailsServiceImpl implements UserDetailsService {
    private final UserMapper userMapper;
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User u = userMapper.selectByUsername(username);
        if (u == null) throw new UsernameNotFoundException("用户不存在");
        return org.springframework.security.core.userdetails.User
                .withUsername(u.getUsername())
                .password(u.getPassword())
                .disabled(!u.getEnabled())
                .authorities(u.getRoles().stream()
                        .map(r -> new SimpleGrantedAuthority("ROLE_" + r)).toList())
                .build();
    }
}
```

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {
    private final AuthenticationManager authManager;
    private final JwtUtil jwtUtil;

    @PostMapping("/login")
    public Result<LoginVO> login(@Valid @RequestBody LoginDTO dto) {
        Authentication a = authManager.authenticate(
                new UsernamePasswordAuthenticationToken(dto.getUsername(), dto.getPassword()));
        UserDetails u = (UserDetails) a.getPrincipal();
        String token = jwtUtil.generate(u);
        return Result.ok(new LoginVO(token, jwtUtil.getExpireSeconds()));
    }
}
```

## 5. JWT 工具类

```java
@Component
public class JwtUtil {
    @Value("${app.jwt.secret}") private String secret;
    @Value("${app.jwt.expire:7200}") private long expire;   // 秒

    private SecretKey key() {
        return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }

    public String generate(UserDetails u) {
        return Jwts.builder()
            .subject(u.getUsername())
            .claim("roles", u.getAuthorities().stream().map(GrantedAuthority::getAuthority).toList())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expire * 1000))
            .signWith(key())
            .compact();
    }

    public Jws<Claims> parse(String token) {
        return Jwts.parser().verifyWith(key()).build().parseSignedClaims(token);
    }
    public boolean validate(String token) {
        try { parse(token); return true; } catch (JwtException | IllegalArgumentException e) { return false; }
    }
    public String getUsername(String token) { return parse(token).getPayload().getSubject(); }
}
```

> 密钥至少 256 bit（HS256），不要硬编码，放配置中心/环境变量。

## 6. JWT 过滤器

```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {
    private final JwtUtil jwtUtil;
    private final UserDetailsService uds;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String header = req.getHeader(HttpHeaders.AUTHORIZATION);
        if (header == null || !header.startsWith("Bearer ")) { chain.doFilter(req, res); return; }
        String token = header.substring(7);
        try {
            String username = jwtUtil.getUsername(token);
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails u = uds.loadUserByUsername(username);
                if (jwtUtil.validate(token)) {
                    var auth = new UsernamePasswordAuthenticationToken(u, null, u.getAuthorities());
                    auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(req));
                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        } catch (Exception e) {
            SecurityContextHolder.clearContext();     // 无效 token 直接放行到 401
        }
        chain.doFilter(req, res);
    }
}
```

## 7. 方法级授权

```java
@PreAuthorize("hasRole('ADMIN')")
@PreAuthorize("hasAuthority('order:delete')")
@PreAuthorize("#userId == authentication.principal.id")     // 只能改自己
@PostAuthorize("returnObject.userId == authentication.principal.id")
@PreFilter / @PostFilter
public void update(@P("userId") Long userId) {}
```

> `@PreAuthorize` 走 AOP，**同类自调用失效**（和事务一样）。

## 8. JWT 无状态化的取舍

| 问题 | 说明 | 缓解 |
|---|---|---|
| 无法主动失效 | 签发后有效期内一直有效 | 短 expire + Refresh Token + 黑名单（Redis） |
| 续期麻烦 | 过期需重新登录 | Refresh Token 或滑动续期 |
| 载荷变大 | 每次请求携带 | 只放必要字段 |
| 密钥泄露 | 全站沦陷 | 定期轮换、非对称 RS256 |

```java
// 黑名单：退出登录时把 jti 放入 Redis，过期时间=剩余有效期
redis.opsForValue().set("jwt:black:" + jti, "1", Duration.ofSeconds(remaining));
```

## 9. 记住我 / 会话 / OAuth2

- 会话模式：`SessionCreationPolicy.IF_REQUIRED` + Redis Session（`spring-session-data-redis`）。
- OAuth2 登录 / 资源服务器见 [48](48-oauth2-resource-server.md)。

## 10. 密码安全

```java
@Bean PasswordEncoder encoder() { return new BCryptPasswordEncoder(10); }
// 或 Argon2PasswordEncoder / Pbkdf2PasswordEncoder（更安全）
```

- 禁止 MD5/SHA1 明文存储；禁止在日志打印密码。
- 登录失败次数限制（Redis 计数 + 锁定）。

## 11. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 引入 security 后所有接口 401 | 默认全需认证 | 配置 `permitAll` |
| POST 403 | CSRF 拦截 | 无状态 JWT 关 CSRF；或携带 token |
| 静态资源 404/401 | 被拦截 | `requestMatchers("/static/**").permitAll()` |
| `WebSecurityConfigurerAdapter` 找不到 | Boot 3 编译错 | 改成 `SecurityFilterChain` Bean |
| `@PreAuthorize` 不生效 | 无权限控制 | 加 `@EnableMethodSecurity` |
| 密码匹配失败 | 登录失败 | 确认存的是 BCrypt 密文且编码器一致 |
| 过滤器抛异常不走全局处理器 | 返回默认 401 | Filter 早于 DispatcherServlet，需 `AuthenticationEntryPoint` |
| Security 吞掉 CORS | 预检 403 | `http.cors(...)` + MVC 配置 |

## 12. 面试速答

- **Security 的核心是一组 Filter 链**：`SecurityContextPersistenceFilter` → `UsernamePasswordAuthenticationFilter` → `ExceptionTranslationFilter` → `AuthorizationFilter`。
- **Boot 3 配置方式变化**：不再继承 `WebSecurityConfigurerAdapter`，改为注入 `SecurityFilterChain` Bean（Lambda 风格）。
- **JWT 的优缺点**：无状态、易扩展；但无法主动失效，需配合短有效期 + Refresh Token/黑名单。
- **认证 vs 授权**：认证是确认身份（`AuthenticationManager`），授权是判断权限（`AccessDecisionManager` / `@PreAuthorize`）。
