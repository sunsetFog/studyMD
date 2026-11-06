# 29 - SSR 与 Nuxt

## 1. 为什么要 SSR

| 模式 | 首屏 | SEO | 服务器成本 | 开发复杂度 |
|---|---|---|---|---|
| CSR（默认 Vite） | 慢（白屏→js→渲染） | 差 | 低 | 低 |
| SSR | 快（直出 HTML） | 好 | 高 | 中高 |
| SSG | 最快 | 好 | 低 | 低 |

决策：**内容/营销页要 SEO → SSR/SSG；纯中后台工具 → CSR 足够**。别为了技术新鲜度上 SSR。

## 2. 手写 SSR 核心流程（理解原理）

```
浏览器请求 → server 端 createSSRApp → renderToString 得 HTML
         → 数据预取写入 state → 响应 HTML + payload
浏览器 → 水合（hydration）：复用服务端状态接管交互
```

```ts
// server.js 骨架
import { createSSRApp } from 'vue'
import { renderToString } from 'vue/server-renderer'

const app = createSSRApp(App)
const html = await renderToString(app)
res.send(`<!DOCTYPE html>${html}<script>window.__INITIAL_STATE__=${serialize(state)}</script>`)
```

关键概念：**双端同构**（一份代码两处跑）与**水合**（客户端接管已有 DOM）。

## 3. 手写 SSR 的约束（为什么建议直接用 Nuxt）

- 服务端代码不能访问 window/document（生命周期守卫）
- 每请求新 app/pinia 实例（防状态串请求）
- 数据预取与组件树对齐（Suspense 协调）
- 路由/构建双端配置

这些工程问题 Nuxt 全部内置解决 → **生产项目直接 Nuxt**，手写仅用于理解。

## 4. Nuxt 核心概念

```
nuxt project/
├── pages/            # 文件即路由（user/[id].vue → /user/:id）
├── components/       # 自动导入
├── composables/      # 自动导入
├── server/           # 服务端 API（Nitro）
├── layouts/          # 布局
├── plugins/
├── middleware/       # 路由中间件
└── nuxt.config.ts
```

## 5. 数据获取三件套

```ts
// 页面级数据（SSR 预取 + 客户端缓存键）
const { data: user, error } = await useAsyncData('user', () => api.getUser(id))
// URL 同步数据
const { data: list, refresh } = await useFetch('/api/orders', { query: { page } })
// 仅客户端
const clientData = useLazyFetch('/api/stats')
```

- `useAsyncData/useFetch` 必须在 setup 同步调用（Nuxt 通过编译注入上下文）
- payload 机制：服务端取到的数据随 HTML 传递，客户端**不再重复请求**
- key 一致才能命中 payload 缓存

## 6. 渲染模式选择（Nuxt 3 routeRules）

```ts
// nuxt.config.ts
routeRules: {
  '/': { prerender: true },            // 首页静态化
  '/blog/**': { isr: 600 },            // 增量静态
  '/admin/**': { ssr: false },         // 后台纯 CSR
  '/api/**': { cors: true },
}
```

现代 SSR 的正确姿势：**按路由混合渲染模式**，而非整站一种。

## 7. 水合错误

水合不一致（服务端 HTML ≠ 客户端首次渲染）是最常见坑：

| 场景 | 修复 |
|---|---|
| 时间戳/随机数 | 客户端 mount 后再算（`import.meta.client` 分支）或统一时区序列化 |
| localStorage 数据首屏 | 客户端 flag 渲染 |
| 日期格式化地区差异 | 统一 SSR 客户端 locale 注入 |

```vue
<ClientOnly>
  <ClockWidget />          <!-- 仅客户端渲染，附 fallback 插槽 -->
</ClientOnly>
```

## 8. SSR 中的第三方库

| 类型 | 处理 |
|---|---|
| 纯 UI 库 | 一般可用（element-plus/nuxt 官方模块） |
| 操作 DOM 的库 | ClientOnly 包裹或动态 import |
| 图表/地图 | 动态 import + ClientOnly |

## 9. 性能与部署

- 部署：Nitro 输出 Node 服务 / serverless / 边缘运行时
- 缓存层：CDN 缓存静态化路由、页面级 SWR
- 监控：服务端渲染耗时、水合错误上报

## 常见坑

1. **useFetch key 缺失**：同页多处同 key 数据冲突。
2. **服务端访问 window**：`import.meta.client` 守卫或 ClientOnly。
3. **状态跨请求泄漏**：自定义 SSR 忘每请求新建 pinia。
4. **水合警告忽略**：轻微不一致可能引发整树重渲染，必须修。
5. **SSR 下组件库样式闪烁**：CSS 提取配置（nuxt 自动，手写 SSR 需 extract）。
6. **服务器内存**：每请求完整 app 树，高并发需压测与缓存。
7. **过渡使用 isr/prerender**：鉴权页静态化会泄漏数据。

## Checklist

- [ ] SSR/CSR/混合模式按路由决策
- [ ] useAsyncData/useFetch key 规范
- [ ] ClientOnly 处理 DOM 依赖组件
- [ ] 水合错误零容忍（上报+修复）
- [ ] 敏感/动态页不走静态化
- [ ] 服务端渲染耗时与错误监控就位
