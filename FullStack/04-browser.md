# 04 - 浏览器原理与安全

## 1. 渲染流程（必考）

```
解析 HTML → DOM 树
解析 CSS  → CSSOM
合成渲染树（不含 display:none）
布局 Layout（计算几何位置）
分层 Layer + 绘制 Paint
合成 Composite（GPU）
```

- JS 阻塞解析：script 默认阻塞 DOM；`defer`（DOM 后按序执行）/ `async`（下载完即执行）
- CSS 不阻塞 DOM 解析但阻塞渲染
- `display:none` 不参与渲染树；`visibility:hidden` 参与（占位）

## 2. 重绘与回流（必考）

- **回流（reflow）**：几何变化（尺寸/位置/增删节点）→ 布局重算，代价大
- **重绘（repaint）**：颜色/背景等样式变化 → 只重绘
- 合成层属性（transform/opacity）只触发 composite，性能最好

优化手段：
1. 批量修改：`classList`、`cssText`、DocumentFragment
2. 读写分离（避免布局抖动 layout thrashing）
3. 动画用 transform/opacity，`will-change` 提升合成层
4. `requestAnimationFrame` 对齐帧

## 3. 事件机制

```
捕获（window→target）→ 目标 → 冒泡（target→window）
addEventListener(type, fn, { capture, once, passive })
e.stopPropagation() 停止传播 / preventDefault() 阻止默认行为
```

- 事件委托：监听父级 + `e.target`（高频：万级列表只挂一个监听器）
- passive listener：滚动事件标记后不阻塞合成

## 4. 浏览器存储

| 方案 | 容量 | 生命周期 | 特点 |
|---|---|---|---|
| Cookie | 4KB | 可设过期 | 随请求发送、HttpOnly/Secure/SameSite |
| localStorage | 5-10MB | 永久 | 同源共享、同步 API |
| sessionStorage | 5MB | 会话级 | 标签页隔离 |
| IndexedDB | 大 | 永久 | 异步、事务、索引，离线应用 |

- Cookie 的 `SameSite=Lax`（默认）防 CSRF；`HttpOnly` 防 JS 读取（配合 XSS 防御）
- 存储事件 `storage` 可做同源标签页通信

## 5. 跨域（必考）

同源 = 协议 + 域名 + 端口一致。

| 方案 | 原理 | 场景 |
|---|---|---|
| CORS（主流） | 服务端响应 `Access-Control-Allow-Origin` 等 | 生产标准 |
| 预检 OPTIONS | 非简单请求（自定义头/PUT/JSON）先发预检 | 附带 cookie 需 `Allow-Credentials` 且 Origin 不能为 * |
| Nginx 反向代理 | 同源转发 | 前端工程常用 |
| postMessage / WebSocket | 消息通道 | 特殊场景 |

简单请求判定：GET/POST/HEAD + 常规头 + 常规 Content-Type。

## 6. Web 安全三板斧（必考）

| 攻击 | 原理 | 防御 |
|---|---|---|
| XSS | 注入脚本执行 | 输出编码、CSP、HttpOnly Cookie、React/Vue 默认转义（`dangerouslySetInnerHTML`/`v-html` 是豁口） |
| CSRF | 冒用 Cookie 发起请求 | SameSite、CSRF Token、校验 Origin/Referer |
| 点击劫持 | iframe 嵌套 | `X-Frame-Options: DENY` / CSP frame-ancestors |

后端联动：见 19（认证安全全链路）。

## 7. 其他高频点

- **内存泄漏排查**：Chrome DevTools Memory 快照对比，常见泄漏源（未清理的定时器/监听器、闭包、脱离 DOM 的引用）
- **Performance/Lighthouse**：FCP/LCP/CLS/INP 指标（见 10 性能篇）
- **浏览器多进程架构**：主进程/渲染进程/GPU/网络插件进程；渲染进程沙箱
- **Service Worker / PWA**：离线缓存、拦截请求

## 常见追问

1. 为什么 transform 动画不触发回流？——合成器线程独立处理，主线程不参与
2. localStorage 的同步 IO 问题？——大存储阻塞主线程，大数据用 IndexedDB
3. Cookie 里 session id 怎么防窃取？——HttpOnly + Secure + SameSite 组合

## Checklist

- [ ] 能完整讲渲染流程并区分重绘/回流
- [ ] CORS 预检与凭证规则清晰
- [ ] XSS/CSRF 能各给两个防御手段
