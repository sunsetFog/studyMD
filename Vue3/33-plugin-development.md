# 33 - 插件开发

## 1. 插件是什么

插件 = **暴露 install 方法的对象（或函数）**，用于注入全局能力：全局组件/指令、provide、原型方法、路由/状态扩展、构建期转换。

```ts
interface Plugin {
  install(app: App, options?: any): void
}
```

## 2. 基本结构

```ts
// plugins/tracker.ts
import type { App } from 'vue'

export interface TrackerOptions {
  endpoint: string
  debug?: boolean
}

export function createTracker(options: TrackerOptions) {
  const send = (event: string, payload: Record<string, unknown>) => {
    if (options.debug) console.debug('[tracker]', event, payload)
    navigator.sendBeacon(options.endpoint, JSON.stringify({ event, payload, ts: Date.now() }))
  }
  return send
}

export default {
  install(app: App, options: TrackerOptions) {
    const send = createTracker(options)
    // 1. 全局 provide（推荐的方式）
    app.provide('track', send)
    // 2. 全局属性（模板可用，类型需声明）
    app.config.globalProperties.$track = send
    // 3. 组合式注入点：模块级单例 + 导出函数
    setGlobalTracker(send)
  },
}
```

## 3. 注入方式的层级

| 方式 | 场景 | 注意 |
|---|---|---|
| `app.provide` + inject | 逻辑访问（推荐） | 类型用 InjectionKey |
| `globalProperties` | 模板中 `$xxx` | TS 需 declare module；组合式 API 内不可用 |
| 模块级单例导出 | 简单库风格 | SSR 需每请求实例 |
| `app.component/directive` | 注册 UI 能力 | — |

globalProperties 声明：

```ts
declare module 'vue' {
  interface ComponentCustomProperties { $track: TrackFn }
}
```

## 4. 实战：全局指令 + 组件插件

```ts
// plugins/ui.ts
export default {
  install(app: App) {
    app.directive('focus', vFocus)
    app.component('BaseModal', BaseModal)
    // 混合注入配置
    app.provide(UI_CONFIG_KEY, { zIndex: 1000 })
  },
}

// main.ts
app.use(uiPlugin, { zIndex: 1200 })
```

## 5. 实战：给 Router 添加扩展

```ts
// plugins/router-audit.ts —— 审计所有路由跳转
export default {
  install(app: App, options: AuditOptions) {
    const router = app.config.globalProperties.$router as Router
    router.beforeEach((to, from) => {
      options.onNavigate?.(to.fullPath, from.fullPath)
    })
  },
}
```

插件间协作通过 `app.config.globalProperties` 或共享模块，避免直接依赖具体实现。

## 6. 组件库风格插件（自带配置与按需）

```ts
// install 含配置的完整插件
const plugin: Plugin = {
  install(app, options = {}) {
    const merged = { ...defaultConfig, ...options }
    app.provide(CONFIG_KEY, merged)
    // 批量注册
    Object.entries(components).forEach(([name, comp]) => app.component(name, comp))
  },
}
export default plugin

// 使用方按需（unplugin-vue-components 场景插件）
// vite.config.ts
Components({ resolvers: [MyUIResolver()] })
```

## 7. 生态型插件 API 约定

成熟插件（Pinia/Router 风格）的通用签名：

```ts
export function createMyPlugin(options: Options = {}) {
  return {
    install(app: App) {
      const api = createApi(options)
      app.provide(KEY, api)
      // 挂到 config 上供其他插件获取
      ;(app.config.globalProperties as any).$myPlugin = api
    },
  }
}
```

约定：
- `createXxx(options)` 工厂返回插件对象（可携带内部状态）
- 类型随包导出（`PluginOptions`、`InjectionKey`）
- SSR 双端安全（服务端分支）

## 8. 构建期「插件」（Vite 层面）

Vue 生态扩展很多发生在构建期，与运行时插件区分：

| 类型 | 例子 |
|---|---|
| Vite 插件 | @vitejs/plugin-vue、unplugin-auto-import |
| unplugin 按需 | components resolvers |
| 编译宏 | defineProps/defineModel 等编译期注入 |

## 9. 插件质量清单

```
✅ install 幂等（重复 use 不重复注入）
✅ 选项类型导出 + 默认值
✅ SSR 安全（服务端不访问 DOM）
✅ TS 完整（declare module 增强）
✅ 不污染 globalProperties 之外的隐藏状态
✅ 文档说明注入方式与用法
```

## 常见坑

1. **install 非 idempotent**：测试中多次 use 报重复注册。
2. **插件在 mount 后 use**：不生效，统一在 createApp 后立刻注册。
3. **globalProperties 类型缺失**：模板可用但 TS 报错，需 declare module。
4. **模块级单例 + SSR**：状态串请求。
5. **插件内部硬编码依赖具体 router/store 版本**：声明 peerDependencies 并防御性获取。
6. **provide 的 key 用字符串**：冲突风险，统一 Symbol InjectionKey。

## Checklist

- [ ] install 幂等且可传选项
- [ ] 注入方式明确（provide 优先）
- [ ] 类型增强完整（ComponentCustomProperties / GlobalComponents）
- [ ] SSR 场景已验证
- [ ] 有文档与最小示例
