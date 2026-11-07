# 19 - 认证与安全（全栈整合）

## 1. 认证方案演进（必考⭐⭐）

### Session/Cookie

```
登录 → 服务端存 Session（内存/Redis）→ 下发 SessionId Cookie → 每次请求携带
```

- 优点：可控（踢人/改权限立即生效）、HttpOnly 防 XSS 窃取
- 缺点：分布式需共享存储（Redis）、跨域/APP 不友好

### JWT

```
Header.Payload.Signature（前两段 Base64URL，可解码！不是加密）
签名 = HMAC/RSA(前两段, secret) → 服务端无状态校验
```

- 优点：无状态（水平扩展友好）、跨服务/跨端
- 缺点：**无法主动失效**、payload 泄露风险、token 变大
- 失效方案：短 access token（15min）+ 长 refresh token（7d，Redis 存储可吊销）；黑名单（登出/改密后 jti 拉黑）

### 选型答题

```
单体/内部系统：Session + Redis（可控性最好）
多端/微服务：JWT 双 token + Redis 黑名单（兼顾无状态与可吊销）
```

## 2. OAuth 2.0（第三方登录必考）

```
授权码模式：A 重定向到授权页 → 用户同意 → 回调带 code → 后端用 code+secret 换 access_token → 拉取用户信息
```

- 为什么 code 两步走：code 经浏览器（不可信），token 只在后端与授权服务器间（带 secret 校验客户端身份）
- 四种模式：授权码（标准）/隐式（废弃）/密码（仅高度信任）/客户端凭证（服务间）
- OIDC：OAuth 之上加 id_token（JWT）做认证

## 3. SSO 单点登录（自研思路）

```
统一认证中心 cas.example.com
子系统未登录 → 302 到 CAS?redirect=当前页
CAS 登录 → 发 ticket → 子系统后端拿 ticket 换用户信息（建本地会话）
其他子系统共享 CAS 会话 Cookie（同一顶级域）→ 免登
```

- 跨顶级域：CAS 票据机制天然支持；Cookie 需 SameSite/Domain 配置
- 生产常直接用 Keycloak/Spring Security OAuth 客户端

## 4. Spring Security 6 要点（后端）

```java
@Bean
SecurityFilterChain chain(HttpSecurity http) throws Exception {
    http.csrf(AbstractHttpConfigurer::disable)          // 无状态 API 关闭
        .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/auth/**").permitAll()
            .anyRequest().authenticated())
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
}
```

- 过滤器链模型、认证（AuthenticationManager）vs 授权（hasRole/hasAuthority）
- 密码存储：BCrypt（自带盐，慢哈希）——禁 MD5

## 5. 前端配合（必考实践）

- token 存哪：
  - localStorage：方便但 XSS 可读
  - Cookie HttpOnly：防 XSS，需处理 CSRF + 跨域凭证
  - 内存 + refresh token 在 HttpOnly Cookie（主流安全方案）
- axios 拦截器：请求带 token、401 自动 refresh 并重放原请求（并发 401 加锁只刷一次）
- 路由守卫：全局 beforeEach 校验登录态 + 按钮级权限（自定义指令 v-permission）

## 6. 安全攻击清单（前后端对照）

| 攻击 | 前端防御 | 后端防御 |
|---|---|---|
| XSS | 框架默认转义、避免 v-html/dangerouslySetInnerHTML、CSP | 输出编码、HttpOnly、输入过滤 |
| CSRF | - | CSRF Token、SameSite=Lax/Strict、校验 Origin |
| SQL 注入 | - | 预编译 #{}、白名单 |
| 越权 | UI 隐藏只是体验 | **服务端校验数据归属**（水平/垂直越权必查） |
| 重放 | - | nonce+timestamp、幂等 token |

关键认知：**前端隐藏按钮 ≠ 权限控制**，后端必须独立鉴权。

## 常见追问

1. JWT 泄露了怎么办？——短有效期 + 刷新机制 + 黑名单，必要时改密强制下线
2. Cookie 和 Token 认证的 CORS 差异？——Cookie 需 `credentials: 'include'` + 后端 Allow-Credentials + 精确 Origin
3. 密码怎么存？——BCrypt；传输 HTTPS；忘记密码用邮件链接而非找回原密码

## Checklist

- [ ] Session vs JWT 能按场景选并说优缺点
- [ ] OAuth2 授权码流程能画图
- [ ] refresh token 并发刷新竞态有方案
- [ ] 越权校验在后端有意识
