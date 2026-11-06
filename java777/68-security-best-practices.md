# 68 · 安全最佳实践

## 1. OWASP Top 10（要点）

| 风险 | 说明 | 防护 |
|---|---|---|
| 失效的访问控制 | 越权查看他人数据 | 每个接口校验归属（水平越权）、角色（垂直越权） |
| 加密失败 | 明文传输/存储 | HTTPS、密码哈希、敏感字段加密 |
| 注入 | SQL/命令/LDAP 注入 | 预编译、参数校验 |
| 不安全设计 | 逻辑漏洞 | 威胁建模 |
| 安全配置错误 | 默认口令、暴露端点 | 最小化、关默认、关 Actuator 暴露 |
| 易受攻击组件 | 老版本依赖 | 依赖扫描、及时升级 |
| 认证识别失败 | 弱口令、会话管理差 | 强口令、MFA、令牌有效期 |
| 软件与数据完整性 | 反序列化漏洞 | 禁用不安全反序列化、校验签名 |
| 日志与监控不足 | 攻击无感知 | 审计日志、告警 |
| SSRF | 服务端请求伪造 | URL 白名单、禁内网访问 |

## 2. SQL 注入

```java
// ❌ 危险
String sql = "select * from t_user where name = '" + name + "'";
// MyBatis ${name} 也是拼接

// ✅ 正确
jdbc.query("select * from t_user where name = ?", name);        // #{} / PreparedStatement
// ${} 必须白名单（如排序列）
```

- 参数校验（长度、字符集）。
- 最小权限数据库账号（不用 root）。
- 错误信息不暴露 SQL/堆栈。

## 3. XSS（跨站脚本）

- 输入过滤 + **输出编码**（按上下文：HTML/JS/CSS/URL）。
- 响应头 `Content-Type: application/json`（JSON 接口天然安全）。
- 前端框架（Vue/React）默认转义；慎用 `v-html` / `dangerouslySetInnerHTML`。
- Cookie 设 `HttpOnly` + `Secure` + `SameSite`。

## 4. CSRF（跨站请求伪造）

- 前后端分离 + JWT（无 Cookie）**天然免疫**。
- Cookie 会话模式需开启 CSRF Token（Spring Security 默认开启）。
- 校验 `Origin`/`Referer`。
- `SameSite=Lax/Strict`。

## 5. 越权

| 类型 | 说明 | 防护 |
|---|---|---|
| 水平越权 | 看/改同角色他人的数据 | 每个查询带上归属条件：`where user_id = currentUser` |
| 垂直越权 | 普通用户访问管理员接口 | 角色/权限校验 |

```java
// ✅ 永远用当前登录用户作为条件，不要相信前端传的 userId
@GetMapping("/orders/{id}")
public OrderVO get(@PathVariable Long id) {
    Long uid = SecurityUtil.currentUserId();       // ★ 从 token 取，不用参数
    return orderService.findByIdAndUser(id, uid);  // SQL 带 and user_id = ?
}
```

## 6. 认证与口令

- 口令哈希：`BCryptPasswordEncoder`（或 Argon2/PBKDF2），**不要** MD5/SHA1。
- 强口令策略、登录失败锁定（Redis 计数）。
- JWT：短有效期 + Refresh Token + 黑名单/密钥轮换；密钥不要硬编码。
- 敏感操作二次校验（短信/邮箱/密码）。
- 验证码要防刷（限流 + 图形/滑块）。

## 7. 敏感数据

| 数据 | 处理 |
|---|---|
| 密码 | 哈希存储，**永不输出** |
| 手机号/身份证/银行卡 | 展示脱敏（`138****8888`），存储加密（AES） |
| token/密钥 | 不写日志、不进代码库、用配置中心/Secret |
| 日志 | 脱敏后再打；禁止打印完整请求体 |
| 传输 | 全站 HTTPS；内部调用也尽量 TLS/mTLS |

```java
// 统一脱敏
@JsonSerialize(using = MobileDesensitize.class) private String mobile;
```

## 8. 文件上传安全

- 校验类型（**后缀 + 魔数**）、大小。
- 重命名（UUID），不复用原文件名（防路径穿越 `../`）。
- 存储目录不放在 Web 根目录；禁止执行权限。
- 最好直传对象存储，返回 URL。

## 9. 依赖与组件安全

```bash
mvn dependency-check:check              # OWASP 检查
trivy image registry/app:1.0             # 镜像扫描
```
- 定期升级（Spring、Log4j2、Fastjson 等历史漏洞高发）。
- 关注 CVE 与官方公告。
- 移除无用依赖。

## 10. 配置与运维安全

| 项 | 做法 |
|---|---|
| Actuator | 独立端口 + 内网 + 鉴权；不暴露 `/env`、`/heapdump` |
| 默认账号 | 全部修改（Nacos、Redis、Rabbit、Druid 监控页） |
| 中间件 | 不暴露公网，设强口令与 ACL |
| 密钥 | 环境变量 / K8s Secret / Vault，绝不入库 |
| 容器 | 非 root、只读文件系统、镜像扫描 |
| 备份 | 定期备份 + 恢复演练 |
| 审计 | 关键操作留痕（谁、何时、做了什么） |

## 11. Spring Security 关键配置

```java
http
  .csrf(c -> c.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
  .headers(h -> h
      .contentSecurityPolicy(c -> c.policyDirectives("default-src 'self'"))
      .referrerPolicy(r -> r.policy(ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
      .frameOptions(f -> f.deny()))
  .sessionManagement(s -> s
      .sessionFixation().changeSessionId()
      .maximumSessions(1));
```

```yaml
server:
  error:
    include-message: never          # 不返回内部错误信息
    include-stacktrace: never
spring:
  mvc:
    problemdetails: {enabled: true} # 结构化但不含敏感信息
```

## 12. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 越权查数据 | 遍历 ID 看到他人数据 | 查询带归属条件 |
| 日志泄露 | 日志里有身份证/密码 | 脱敏 |
| Actuator 暴露 | `/env` 拿到密码 | 关闭或鉴权 |
| 依赖漏洞 | Log4Shell 类事件 | 定期扫描升级 |
| 反序列化 RCE | Redis/Jackson 默认类型 | 禁用 `enableDefaultTyping`，限制基类 |
| 上传漏洞 | 上传 webshell | 类型+魔数校验、重命名、不可执行 |
| 弱口令 | 被爆破 | 强口令 + 失败锁定 |
| SSRF | 内网被探测 | URL 白名单，禁私有网段 |

## 13. 面试速答

- **SQL 注入怎么防**：预编译（`PreparedStatement`/`#{}`）+ 参数校验 + 最小权限账号；`${}` 必须白名单。
- **XSS 与 CSRF 区别**：XSS 是注入脚本窃取/冒充用户；CSRF 是诱导用户浏览器发出非本意请求。防护分别是输出编码/转义与 Token/SameSite。
- **越权怎么防**：水平越权靠「查询条件带当前用户」；垂直越权靠角色/权限校验；不要信任前端传的身份标识。
- **密码怎么存**：BCrypt/Argon2/PBKDF2 加盐哈希，绝不明文或 MD5。
- **Actuator 安全**：独立管理端口、不暴露到公网、关闭或鉴权敏感端点（env/heapdump/loggers）。
