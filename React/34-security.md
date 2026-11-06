# 34 - 安全实践

## 1. 威胁地图

| 威胁 | 入口 | 防线 |
|---|---|---|
| XSS | dangerouslySetInnerHTML、URL 注入 | 消毒/避免原始 HTML |
| CSRF | cookie 自动携带 | token in header / SameSite |
| 敏感泄漏 | localStorage、日志、sourcemap | 白名单/脱敏 |
| 依赖漏洞 | npm 包 | audit + lockfile |
| 开放重定向 | redirect 参数 | 白名单校验 |
| 权限绕过 | 仅前端隐藏 | 后端鉴权双保险 |

## 2. XSS（重点）

React 默认安全模型：**JSX 插值与属性绑定自动转义**——不用 dangerouslySetInnerHTML 就基本安全。

```tsx
// 唯一的原生 HTML 注入口
<div dangerouslySetInnerHTML={{ __html: sanitized }} />
```

纪律：
- 用户输入**永不**进 dangerouslySetInnerHTML
- 富文本（CMS）必须服务端/前端 DOMPurify 消毒：`DOMPurify.sanitize(dirty)`
- URL 注入：`<a href={userUrl}>` 的 `javascript:` 协议——白名单 `^https?://` 校验

```tsx
const safeUrl = /^https?:\/\//.test(url) ? url : '#'
```

## 3. Token 存储

| 方案 | XSS 风险 | 建议 |
|---|---|---|
| 内存（Zustand store） + refresh token | 最低 | 主流方案 |
| httpOnly cookie | JS 不可读（需 CSRF 防护） | 同域可控 |
| localStorage | XSS 可偷 | 不推荐存 token |

```
accessToken 内存 + refreshToken httpOnly cookie
localStorage 只存 UI 偏好（白名单）
```

## 4. CSRF

- token 走 Authorization header 天然免疫大部分 CSRF
- cookie 方案：SameSite=Lax/Strict + CSRF token + Origin 校验（服务端）

## 5. 敏感数据纪律

| 规则 | 说明 |
|---|---|
| localStorage 白名单 + 脱敏 | 手机号 138****1234 |
| 日志/埋点脱敏 | reporter 前统一 sanitize |
| 生产去 console | esbuild drop / terser |
| sourcemap 私有化 | 上传错误监控平台不公开 |
| VITE_ 环境变量无密钥 | 前端代码皆公开 |
| Server Action/接口鉴权 | 每个端点独立校验 |

## 6. 依赖安全

```bash
npm audit --production
# Dependabot / Renovate 自动升级 PR
```

- lockfile 提交锁定
- 高危 CVE 当周处理
- 私有 npm 源防 typosquatting

## 7. 权限双保险

```tsx
// 前端：体验层隐藏
const { hasPermission } = useAuth()
{hasPermission('order:export') && <ExportButton />}

// 后端：真正的防线（每个接口独立鉴权）
```

铁律：**前端隐藏 ≠ 后端允许**。Route guard / Server Action / API 全部独立鉴权。

## 8. CSP

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; img-src 'self' https://cdn.example.com
```

- React 运行时（production build）无 eval，可配严格 CSP
- 注意内联脚本处理（nonce）与第三方（监控/地图）白名单

## 9. 第三方脚本与 iframe

- `<script integrity="sha384-…">` SRI 校验
- iframe `sandbox` 属性 + `allow` 白名单
- postMessage 通信校验 `event.origin`

## 10. Next.js 专属注意

- Server Action 是公开端点：内必鉴权
- Route Handler 同理
- `dangerouslySetInnerHTML` 与 RSC 流式组合时同样要消毒

## 常见坑

1. **dangerouslySetInnerHTML 渲染接口富文本未消毒**：存储型 XSS。
2. **token 进 localStorage**：任意 XSS 即丢。
3. **redirect 未校验**：登录后开放重定向。
4. **生产 console 泄漏数据**：构建 drop。
5. **CSP 与运行时插件冲突**（dev 工具）：环境区分配置。
6. **前端隐藏当权限**：接口被直接调用。

## Checklist

- [ ] dangerouslySetInnerHTML 白名单 + DOMPurify
- [ ] token 内存 + refresh，localStorage 白名单
- [ ] 生产 drop console + sourcemap 私有化
- [ ] audit 定期 + 高危快速修
- [ ] redirect 校验
- [ ] 前后端权限双保险
- [ ] CSP 评估部署
