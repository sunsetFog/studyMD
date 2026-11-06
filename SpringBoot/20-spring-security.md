# 20 · Spring Security 6

## 一、核心心智模型

```
请求 → FilterChainProxy → 一条过滤器链（15+ 个 Filter）→ Controller
                              ↑
                     SecurityContext 持有当前认证信息
```

两大问题域：
- **Authentication（认证）**：你是谁 → 用户名密码/token/OAuth
- **Authorization（授权）**：你能干什么 → URL/方法/数据权限

## 二、依赖与 SecurityFilterChain（Boot 3 写法）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)                 // 无状态 API 关 CSRF
            .cors(Customizer.withDefaults())                       // 启用 CORS 配置
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/v1/**").authenticated()
                .anyRequest().authenticated())
            .exceptionHandling(e -> e
                .authenticationEntryPoint(this::unauthorized)      // 401 返回 JSON
                .accessDeniedHandler(this::forbidden))             // 403 返回 JSON
            .addFilterBefore(jwtAuthFilter,
                UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

> Boot 3 不再支持 `WebSecurityConfigurerAdapter`（已删除），统一用 SecurityFilterChain Bean。

## 三、密码加密

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();      // 内置盐，慢哈希抗暴力破解
}

// 注册
user.setPassword(passwordEncoder.encode(rawPassword));

// 校验
passwordEncoder.matches(rawPassword, encoded);
```

禁止 MD5/SHA 裸哈希存密码。BCrypt 强度默认 10，可按硬件调 10-12。

## 四、认证流程（自定义 UserDetailsService）

```java
@Service
@RequiredArgsConstructor
public class DbUserDetailsService implements UserDetailsService {

    private final UserMapper userMapper;

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userMapper.selectByUsername(username);
        if (user == null) throw new UsernameNotFoundException("用户不存在");
        return org.springframework.security.core.userdetails.User.builder()
                .username(user.getUsername())
                .password(user.getPassword())
                .disabled(user.getStatus() == 0)
                .authorities(user.getRoles().stream()
                        .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
                        .toArray(GrantedAuthority[]::new))
                .build();
    }
}
```

## 五、方法级授权

```java
// 启动类：@EnableMethodSecurity

@PreAuthorize("hasRole('ADMIN')")                    // 需要 ROLE_ADMIN
public void deleteAll() { }

@PreAuthorize("#userId == authentication.principal.id")   // 数据级：只能操作自己
public void updateUser(Long userId, UserDTO dto) { }

@PostAuthorize("returnObject.ownerId == authentication.principal.id")
public OrderDTO getOrder(Long id) { }
```

## 六、从 SecurityContext 取当前用户

```java
// 方式 1
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();

// 方式 2（参数注入，推荐）
public Result<Void> me(@AuthenticationPrincipal LoginUser user) { }
```

## 七、授权体系设计（RBAC）

```
用户表 users
角色表 roles
用户-角色 user_role
角色-权限 role_permission
权限表 permissions（如 user:delete）
```

数据权限（同一接口不同人看不同数据）通常在 SQL 层拼条件（如只看本部门），配合 AOP 注解注入部门过滤条件。

## 常见坑

- **坑 1**：Boot 3 老写法 WebSecurityConfigurerAdapter → 类不存在，编译报错
- **坑 2**：`.anyRequest().permitAll()` 放底部 → 所有接口裸奔；permitAll 白名单放前面，兜底 authenticated
- **坑 3**：前后端分离没配 CORS/没放行 OPTIONS → 预检失败
- **坑 4**：ROLE_ 前缀混乱 → `hasRole("ADMIN")` 要求 authority 是 `ROLE_ADMIN`；用 authorities 装配时统一处理
- **坑 5**：SecurityContext 的 ThreadLocal 在异步线程丢失 → DelegatingSecurityContextAsyncTaskExecutor 或手动传递
- **坑 6**：接口返回 403 但预期 401 → 权限不足与未认证要区分清楚（entryPoint vs accessDeniedHandler）

## Checklist

- [ ] 无状态 API：CSRF 关闭 + Session STATELESS + JWT 过滤器
- [ ] 密码 BCrypt 存储与校验
- [ ] 白名单集中管理（auth/public/文档）
- [ ] 方法级授权 + 数据权限方案明确
