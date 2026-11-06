# 37 - 安全实践

## 1. 前端安全威胁地图

| 威胁 | 入口 | 前端防线 |
|---|---|---|
| XSS | v-html、innerHTML、URL 注入 | 消毒/避免原始 HTML |
| CSRF | cookie 自动携带 | token in header / SameSite |
| 敏感信息泄漏 | localStorage、日志、source map | 白名单存储、脱敏日志 |
| 点击劫持 | iframe 嵌套 | X-Frame-Options（服务端） |
| 依赖漏洞 | npm 包 | lockfile + audit + 私有源 |
| 开放重定向 | redirect 参数 | 白名单校验 |

## 2. XSS（重点）

### 2.1 默认安全模型

Vue 模板插值 `{{ }}` 与属性绑定**自动转义**——不用 v-html 就基本安全。

### 2.2 v-html 纪律

```vue
<!-- 数据来源必须可信：CMS 富文本（服务端已消毒） -->
<div v-html="sanitizedHtml"></div>
```

规则：
- 用户输入**永不** v-html；富文本必须服务端 DOMPurify 消毒后下发
- 前端兜底消毒：`import DOMPurify from 'dompurify'`；`DOMPurify.sanitize(dirty)`
- 禁止拼接 HTML 字符串再注入

### 2.3 URL 注入

```vue
<a :href="userUrl">     <!-- javascript: 协议注入 -->
```

```ts
const safeUrl = computed(() => {
  const u = props.userUrl
  return /^https?:\/\//.test(u) ? u : '#'
})
```

## 3. Token 存储

| 方案 | XSS 风险 | 适用 |
|---|---|---|
| Authorization header + 内存/内存型 store | 最低 | 主流方案（刷新走 refresh token） |
| httpOnly cookie | 无 JS 可读（但需 CSRF 防护） | 同域后端可控 |
| localStorage | XSS 可偷 | 不推荐存敏感 token |

实践：**accessToken 内存（Pinia）+ refreshToken httpOnly cookie**；localStorage 仅存非敏感 UI 偏好。

## 4. CSRF

- token 走 header（非 cookie）天然免疫大部分 CSRF
- cookie 方案：SameSite=Strict/Lax + CSRF token + 校验 Origin/Referer（服务端）

## 5. 敏感数据纪律

| 规则 | 说明 |
|---|---|
| 不在 localStorage 存身份证/手机号全文 | 脱敏（138****1234） |
| 日志/埋点脱敏 | reporter 前统一 sanitize |
| 生产禁 console | 构建期 esbuild drop console |
| source map 不公开 | 私有上传到错误监控 |
| env 不含真实密钥 | 前端代码皆公开 |

## 6. 依赖安全

```bash
npm audit --production
npm i -D dependabot-adjacent  # 或 GitHub Dependabot / Renovate 自动 PR
```

- lockfile 提交且锁定
- 定期 audit + 升级；高危 CVE 当周处理
- 私有 npm 源 + 提交校验（防 typosquatting）

## 7. 权限控制的双保险

前端权限控制是**体验层**，不是安全边界：

```
前端：隐藏无权按钮/菜单（v-permission 指令，见 16）
后端：每个接口独立鉴权（真正防线）
```

铁律：**前端隐藏 ≠ 后端允许**；接口防越权永远是服务端职责。

## 8. 内容安全策略（CSP）

```
Content-Security-Policy: default-src 'self'; script-src 'self'; img-src 'self' https://cdn.example.com; object-src 'none'
```

- Vite 构建产物可配严格 CSP（无内联脚本需处理 nonce）
- 注意 Vue 运行时模板编译（runtime-only 无 eval，可用严格 CSP；用完整版含编译器则不行）

## 9. iframe 与第三方脚本

- 第三方脚本白名单 + SRI（`integrity` 属性）
- iframe 沙箱 `sandbox` 属性
- postMessage 通信校验 origin

## 常见坑

1. **v-html 渲染接口返回的富文本未消毒**：存储型 XSS。
2. **token 进 localStorage**：任意 XSS 即丢 token。
3. **redirect 未校验**：登录后开放重定向钓鱼。
4. **生产 console 泄漏业务数据**：构建 drop console。
5. **sourcemap 随 dist 上传 CDN**：源码公开。
6. **以为前端隐藏=安全**：接口被直接调用。
7. **CSP 与 runtime compiler 冲突**：eval 被拦截页面挂掉。

## Checklist

- [ ] v-html 数据源全部经过消毒且有清单
- [ ] token 内存存储 + refresh 机制
- [ ] 生产构建 drop console + sourcemap 私有化
- [ ] 依赖 audit 定期跑、高危快速修
- [ ] redirect 参数白名单校验
- [ ] 前端权限隐藏 + 后端鉴权双保险
- [ ] CSP 策略评估与部署
