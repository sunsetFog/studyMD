# 10 - 前端工程化与性能

## 1. 构建工具

| 工具 | 特点 |
|---|---|
| Vite | dev 用原生 ESM + esbuild 预构建（秒启动）；prod 用 Rollup 打包 |
| Webpack | bundle 一切、生态最全；配置重，dev 慢 |
| Turbopack/Rspack | Rust 实现，webpack 兼容提速 |

Vite 冷启动快的原因：**不打包 dev 依赖图**，按需编译请求的模块；预构建把 CJS 转 ESM 并合并请求。
Webpack 核心概念：entry/loader（转换）/plugin（钩子介入构建全程）/chunk 拆分。

## 2. 代码分割与加载策略

```js
// 路由级懒加载（React/Vue 通用思路）
const Page = lazy(() => import('./Page'));        // React
component: () => import('./Page.vue');            // Vue Router
```

- 第三方分包：`manualChunks` 拆 vendor（缓存友好，业务变更不失效框架包）
- tree-shaking：ESM 静态分析 + `sideEffects: false` 标注
- 按需引入组件库（unplugin-auto-import/unplugin-vue-components）

## 3. 首屏性能优化（必考⭐）

指标：**FCP 首屏、LCP 最大内容、CLS 布局偏移、INP 交互延迟**（Core Web Vitals）

手段分层：
1. **网络层**：HTTP2、CDN、gzip/brotli、资源预加载（preload/prefetch/dns-prefetch）
2. **资源层**：代码分割、图片懒加载 + WebP/AVIF、字体子集化、关键 CSS 内联
3. **渲染层**：SSR/SSG（见 4）、骨架屏、防白屏
4. **运行时**：长任务拆分（scheduler）、虚拟列表、Web Worker

虚拟列表原理：只渲染可视区条目（+缓冲），总高度用占位容器撑起，滚动偏移换算 startIndex。

## 4. SSR / SSG / ISR

| 方案 | 原理 | 适用 |
|---|---|---|
| CSR | 空壳 + JS 渲染 | 后台系统 |
| SSR | 每次请求服务端渲染（Node） | 首屏敏感 + 动态内容 |
| SSG | 构建期生成静态页 | 文档/博客 |
| ISR/混合 | 静态 + 增量再生 | 电商详情 |

- SSR 关键考点：hydration（水合：客户端接手服务端 HTML 绑定事件）、同构代码注意 window 守卫、Nginx 转发 Node 服务
- 流式 SSR（React 18/Suspense、Vue3 renderToNodeStream）改善 TTFB

## 5. 微前端（了解+场景）

- qiankun（沙箱+样式隔离+生命周期）、Module Federation（webpack/vite 运行时共享模块）
- 适用：多团队巨石应用拆分、技术栈共存
- 必提代价：复杂度、样式/JS 沙箱豁口、公共依赖治理

## 6. 工程规范

- ESLint + Prettier + husky + lint-staged + commitlint（Git 钩子卡质量）
- Monorepo：pnpm workspace / Turborepo（任务编排缓存）
- 环境变量分层：.env.development/.env.production，Vite 只暴露 VITE_ 前缀

## 7. 监控与告警（加分项）

```
错误监控：window.onerror / unhandledrejection / 资源错误捕获 → 上报（sendBeacon）
性能监控：Performance API / web-vitals 库 采集 FCP/LCP/INP
用户行为：埋点（PV/UV/点击流）、sourcemap 还原堆栈
```

## 常见追问

1. Vite 为什么生产还要 Rollup？——ESM 按需在大量模块+代码分割场景网络瀑布与优化不如打包
2. tree-shaking 失效场景？——CJS、有副作用的模块未标注、动态访问
3. 白屏排查路径？——Network 看 JS 加载、Console 报错、Performance 火焰图、CDN/缓存

## Checklist

- [ ] 能讲 Vite dev 快的本质
- [ ] 首屏优化按网络/资源/渲染/运行时分层答
- [ ] SSR 的 hydration 能解释
