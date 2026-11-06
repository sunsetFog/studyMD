# 21 · JWT 无状态认证完整方案

## 一、JWT 结构

```
Header.Payload.Signature
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMSJ9.xxxxx

Header:  {"alg":"HS256","typ":"JWT"}
Payload: {"sub":"1","username":"tom","roles":["ADMIN"],"exp":1730000000}
Signature: HMAC-SHA256(base64(header).base64(payload), secret)
```

**要点**：Payload 只是 Base64 编码，**不是加密**——任何拿到 token 的人可解码。绝不能放密码/手机号等敏感信息。

## 二、依赖与工具类

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

```java
@Component
@RequiredArgsConstructor
public class JwtUtil {

    private final JwtProperties props;

    private SecretKey key() {
        return Keys.hmacShaKeyFor(props.getSecret().getBytes(StandardCharsets.UTF_8));
    }

    public String generate(Long userId, String username, List<String> roles) {
        Instant now = Instant.now();
        return Jwts.builder()
                .subject(String.valueOf(userId))
                .claim("username", username)
                .claim("roles", roles)
                .issuedAt(Date.from(now))
                .expiration(Date.from(now.plus(props.getExpireMinutes(), ChronoUnit.MINUTES)))
                .signWith(key())
                .compact();
    }

    public Claims parse(String token) {
        return Jwts.parser().verifyWith(key()).build()
                .parseSignedClaims(token).getPayload();   // 过期抛 ExpiredJwtException
    }

    public boolean isValid(String token) {
        try { parse(token); return true; }
        catch (JwtException | IllegalArgumentException e) { return false; }
    }
}
```

## 三、认证过滤器

```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final UserMapper userMapper;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            try {
                Claims claims = jwtUtil.parse(token);
                Long userId = Long.valueOf(claims.getSubject());

                // 可选：查询最新用户状态/角色（封禁即时生效），配合 Redis 缓存
                LoginUser user = buildLoginUser(userId, claims);
                UsernamePasswordAuthenticationToken auth =
                        new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (ExpiredJwtException e) {
                response.sendError(401, "token 已过期");
                return;
            } catch (JwtException e) {
                response.sendError(401, "token 无效");
                return;
            }
        }
        chain.doFilter(request, response);
    }
}
```

## 四、登录接口

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtUtil jwtUtil;
    private final UserService userService;

    @PostMapping("/login")
    public Result<LoginVO> login(@RequestBody @Valid LoginDTO dto) {
        // 走 Security 认证（校验密码 + 装配角色）
        Authentication auth = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(dto.getUsername(), dto.getPassword()));

        LoginUser user = (LoginUser) auth.getPrincipal();
        String token = jwtUtil.generate(user.getId(), user.getUsername(),
                user.getRoles());
        return Result.ok(new LoginVO(token, user.getUsername()));
    }

    @PostMapping("/register")
    public Result<Long> register(@RequestBody @Valid RegisterDTO dto) {
        return Result.ok(userService.register(dto));
    }
}
```

SecurityConfig 中注册 AuthenticationManager：

```java
@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
        throws Exception {
    return config.getAuthenticationManager();
}
```

## 五、刷新 token 与注销

**刷新令牌模式**：accessToken（15-30 分钟）+ refreshToken（7 天，存 Redis）。

```
登录 → 颁发 access + refresh
access 过期 → 用 refresh 换新 access
refresh 过期 → 重新登录
```

**注销**（JWT 无状态的代价）：token 签出后服务端默认无法作废 → 维护黑名单：

```java
// 注销：把 token 剩余有效期存入 Redis 黑名单
redisTemplate.opsForValue().set("jwt:blacklist:" + token, "1",
        Duration.ofMillis(remainingMillis));

// 过滤器校验：黑名单即拒绝
```

## 六、双 token 安全增强

- secret 长度 ≥ 32 字节（HS256 要求），走配置中心/环境变量
- refreshToken 支持轮换：每次刷新旧 refresh 作废，被盗用可检测
- 高敏感操作二次校验（密码/验证码）

## 常见坑

- **坑 1**：把敏感信息放 payload → 可解码泄露
- **坑 2**：secret 写死在代码/提交进 git → 用环境变量，长度不足启动报错
- **坑 3**：注销了但 token 还能用 → 黑名单或短过期 + refresh 轮换
- **坑 4**：token 过期时间太长（如 7 天的 accessToken）→ 风险大
- **坑 5**：过滤器对 OPTIONS 请求抛 401 → 放行预检
- **坑 6**：修改用户角色后旧 token 仍携带旧角色 → 短过期 + 过滤器查库/Redis 取最新角色

## Checklist

- [ ] accessToken 短过期 + refreshToken 机制
- [ ] secret 走环境变量，≥32 字节
- [ ] 注销黑名单已实现
- [ ] payload 仅含 id/用户名/角色
- [ ] 角色变更的时效性方案明确
