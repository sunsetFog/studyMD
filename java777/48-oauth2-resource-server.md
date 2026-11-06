# 48 · OAuth2 与 JWT 资源服务器

## 1. OAuth2 四种授权模式

| 模式 | 流程 | 适用 |
|---|---|---|
| **授权码**（authorization_code） | 前端跳授权页 → 拿 code → 后端用 code 换 token | ✅ 最安全，主用 |
| 授权码 + PKCE | 免 client_secret | 移动端/SPA 推荐 |
| 客户端凭证（client_credentials） | 直接 client_id/secret 换 token | **服务间调用** |
| 密码模式（password） | 用户把账号密码给客户端 | ❌ 已废弃，仅遗留系统 |
| 简化模式（implicit） | 直接返回 token 到前端 | ❌ 已废弃 |

## 2. 角色

```
Resource Owner   资源拥有者（用户）
Client           客户端（Web/App/第三方）
Authorization Server  授权服务器（签发 token）
Resource Server  资源服务器（校验 token，提供业务 API）
```

## 3. 搭建资源服务器（Boot 3 + Security 6）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### 方式一：JWT（自包含，无需远程校验）

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: http://auth-server:9000/oauth2/jwks     # 公钥端点
          # 或 issuer-uri: http://auth-server:9000   （自动发现）
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
          .csrf(c -> c.disable())
          .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
          .authorizeHttpRequests(a -> a
              .requestMatchers("/api/public/**").permitAll()
              .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
              .requestMatchers(HttpMethod.GET, "/api/orders/**").hasAuthority("SCOPE_order.read")
              .anyRequest().authenticated())
          .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()));
        return http.build();
    }
}
```

> JWT 里的 `scope` 会被映射为 `SCOPE_xxx` 权限；`roles` 需手动转换（见下）。

### 从 JWT 提取自定义权限

```java
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter scopes = new JwtGrantedAuthoritiesConverter();
    scopes.setAuthorityPrefix("SCOPE_");
    scopes.setAuthoritiesClaimName("scope");

    JwtAuthenticationConverter c = new JwtAuthenticationConverter();
    c.setJwtGrantedAuthoritiesConverter(jwt -> {
        Collection<GrantedAuthority> auths = new ArrayList<>(scopes.convert(jwt));
        // 提取 roles 数组
        Collection<String> roles = jwt.getClaimAsStringList("roles");
        if (roles != null) roles.forEach(r -> auths.add(new SimpleGrantedAuthority("ROLE_" + r)));
        return auths;
    });
    c.setPrincipalClaimName("sub");
    return c;
}
```

### 方式二：Opaque Token（不透明，需远程校验）

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: http://auth-server:9000/oauth2/introspect
          client-id: my-client
          client-secret: secret
```

| 对比 | JWT | Opaque |
|---|---|---|
| 校验 | 本地（公钥） | 每次远程调用 |
| 性能 | 高 | 低（可缓存） |
| 吊销 | 难（需黑名单） | 易（服务端删除） |
| 信息 | 自包含 | 需查询 |

## 4. 获取当前用户

```java
// 方式1：SecurityContext
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String userId = auth.getName();

// 方式2：方法参数（推荐）
@GetMapping("/me")
public User me(@AuthenticationPrincipal Jwt jwt) {
    return new User(jwt.getSubject(), jwt.getClaim("nickname"));
}

// 方式3：自定义 principal
public User me(@AuthenticationPrincipal UserPrincipal p) {}
```

## 5. 搭建授权服务器（Spring Authorization Server）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
```
> Spring Security OAuth2 项目已停止，官方接管为 **Spring Authorization Server**（支持 OAuth2.1 + OIDC）。

```java
@Configuration
@Import(OAuth2AuthorizationServerConfiguration.class)
public class AuthServerConfig {

    @Bean
    public RegisteredClientRepository clientRepository() {
        return new InMemoryRegisteredClientRepository(
            RegisteredClient.withId("order-client")
                .clientId("order-client")
                .clientSecret("{noop}secret")       // 生产用 BCrypt
                .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
                .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
                .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
                .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
                .redirectUri("http://localhost:8080/login/oauth2/code/order")
                .scope("order.read").scope("order.write").scope("admin")
                .clientSettings(ClientSettings.builder().requireAuthorizationConsent(false).build())
                .tokenSettings(TokenSettings.builder()
                        .accessTokenTimeToLive(Duration.ofHours(2))
                        .refreshTokenTimeToLive(Duration.ofDays(7))
                        .build())
                .build());
    }

    @Bean public AuthorizationServerSettings settings() {
        return AuthorizationServerSettings.builder().issuer("http://auth:9000").build();
    }
}
```

端点：`/oauth2/authorize`、`/oauth2/token`、`/oauth2/jwks`、`/oauth2/introspect`、`/oauth2/revoke`、`/userinfo`、`/connect/logout`。

## 6. 服务间调用（Client Credentials）

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          internal:
            client-id: order-client
            client-secret: secret
            authorization-grant-type: client_credentials
            scope: order.read
        provider:
          internal:
            token-uri: http://auth:9000/oauth2/token
```

```java
@Bean
@LoadBalanced
public OAuth2AuthorizedClientManager clientManager(...) { ... }

// 在 Feign 拦截器里取 token
String token = manager.authorize(OAuth2AuthorizeRequest.withClientRegistrationId("internal")
        .principal("internal").build()).getAccessToken().getTokenValue();
```

## 7. 与 Spring Cloud Gateway 配合

常见模式：**网关做 token 校验（资源服务器），内部服务信任网关传递的用户头**。

```java
// 网关过滤器：校验 JWT → 解析用户信息 → 放 X-User-Id 头 → 转发
// 内部服务：自定义过滤器信任 X-User-Id（需保证内网不可被外部直接访问）
// 更严格方案：Token Relay（网关转发原始 token，内部服务各自校验）
```
```yaml
# Token Relay
spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay=      # 需 spring-boot-starter-oauth2-client
```

## 8. 安全要点

| 要点 | 说明 |
|---|---|
| 用 HTTPS | 否则 token 明文传输 |
| 短有效期 | access token 15min~2h，配 refresh token |
| 密钥管理 | RS256 非对称，私钥在授权服务器，公钥下发 |
| 校验 `aud`/`iss` | 防止 token 被跨系统复用 |
| 刷新令牌轮换 | 用后即换，防重放 |
| 注销 | 黑名单/短 TTL/授权服务器吊销 |
| 权限最小化 | scope 粒度控制 |
| 防止 XSS | token 别存 localStorage（SPA 用 httpOnly cookie 或内存） |

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 401 invalid_token | token 过期/签名不一致 | 检查 issuer、密钥、时钟偏移（允许 60s 时钟偏移） |
| 403 但已登录 | 权限不匹配 | 权限前缀 `SCOPE_` / `ROLE_` 要一致 |
| `aud` 校验失败 | 跨服务 | 配置 `audience` 或自定义校验器 |
| 时钟偏移 | token 刚签就过期 | 服务器 NTP 同步；设 `JwtTimestampValidator` 容差 |
| 无法登出 | JWT 仍有效 | 短 TTL + 黑名单 |
| 网关透传头被伪造 | 越权 | 内部网络隔离 + 签名头 |

## 10. 面试速答

- **OAuth2 是什么**：授权框架，允许用户授权第三方应用访问其资源而无需共享密码；核心是**令牌**而非密码。
- **JWT vs Opaque Token**：JWT 自包含、本地校验、难吊销；Opaque 需 introspection、易吊销、有网络开销。
- **资源服务器怎么校验**：`spring-security-oauth2-resource-server` + `jwk-set-uri`（JWKS 公钥）+ `oauth2ResourceServer().jwt()`。
- **scope 与 role**：scope 是**授权范围**（能访问哪些 API），role 是**角色**（用户在系统中的身份）；Security 中 scope 映射为 `SCOPE_xxx` 权限。
- **OAuth2.1 变化**：废弃密码模式与隐式模式，授权码强制 PKCE，Refresh Token 必须轮换。
