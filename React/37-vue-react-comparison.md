# 37 - Vue vs React 对照

> 给同时维护两套技术栈的工程师的横向对照。配合同仓库 `vue3/` 全系列阅读。

## 1. 哲学差异（一切的根源）

| 维度 | React | Vue3 |
|---|---|---|
| 核心模型 | **重算**：状态变 → 重新执行组件函数 → 协调 diff | **追踪**：依赖收集，谁变了精确更新谁 |
| UI 表达 | JSX（JS 全能力） | SFC 模板（受约束，但可编译优化） |
| 心智负担来源 | 闭包快照/引用稳定/依赖数组 | 响应式丢失/解包规则 |
| 设计取向 | 库：小核心，生态自由组合 | 框架：官方全家桶约定 |

**重算 vs 追踪**是一切差异的母题：
- React 的所有痛点（memo/useCallback/useMemo、依赖数组、快照闭包）都源于「重新执行函数」
- Vue 的所有痛点（ref .value、解构丢响应、顶层解包）都源于「Proxy 依赖追踪」

## 2. 概念映射表

| 概念 | React | Vue3 |
|---|---|---|
| 组件 | 函数组件 | SFC `<script setup>` |
| 模板 | JSX | `<template>` |
| 状态 | `useState` | `ref` / `reactive` |
| 派生值 | `useMemo`（无缓存依赖时手动） | `computed`（内建缓存） |
| 副作用 | `useEffect` | `watch` / `watchEffect` |
| 跨层传递 | Context | provide/inject |
| 全局状态 | Zustand/Redux | Pinia |
| 服务端状态 | TanStack Query | 组合式封装/Query |
| 路由 | React Router | Vue Router |
| 双向绑定 | value + onChange | v-model |
| 条件渲染 | 三元/&& | v-if/v-show |
| 列表 | map + key | v-for + key |
| 插槽 | children/props 传 JSX | slot/slot props |
| 传送门 | Portal | Teleport |
| 错误边界 | ErrorBoundary | onErrorCaptured |
| 异步协调 | Suspense | Suspense |
| 缓存组件 | 无内建（React KeepAlive 需库） | KeepAlive |
| ref/模板引用 | useRef | useTemplateRef |
| 生命周期 | useEffect 时序 | onMounted 等 |
| 单文件工具 | 无（逻辑在 Hook） | composables |
| 样式 | 全生态自选 | scoped/BEM 约定 |

## 3. 相同思想的写法对照

### 3.1 竞态保护

```tsx
// React
useEffect(() => {
  let active = true
  fetchUser(id).then(u => active && setUser(u))
  return () => { active = false }
}, [id])
```

```ts
// Vue
watch(id, async (v) => {
  const seq = ++seqRef.value
  const u = await fetchUser(v)
  if (seq === seqRef.value) user.value = u
})
```

### 3.2 状态机

```tsx
// React：useState<Phase> 联合类型（见 react/04）
type Phase = { s: 'idle' } | { s: 'ok'; data: T } | …
```

```ts
// Vue：shallowRef<Phase> 联合类型（见 vue3/34）
```

### 3.3 列表 key

两者同律：**稳定唯一 id，禁 index**（diff 语义略不同：React lastPlacedIndex / Vue LIS，但用户侧结论一致）。

## 4. 互相迁移时的注意点

### React → Vue

| React 习惯 | Vue 中要改 |
|---|---|
| `useMemo` 缓存一切 | computed 已内建缓存，别手动 memo |
| useCallback 免依赖 | computed/直接写，无引用稳定焦虑 |
| 依赖数组管理 | watch 显式源；注意 `() => state.x` |
| setState 函数式 | 直接赋值 `x.value++` |
| key/单根约束 | Fragment 免疫（Vue 支持多根） |

### Vue → React

| Vue 习惯 | React 中要改 |
|---|---|
| 直接改对象属性 | 不可变更新 `{...s}` |
| v-model | value + onChange |
| .number/.trim 修饰符 | 手动转换 |
| computed 派生 | useMemo（或渲染期计算） |
| watch 深层自动 | useEffect 显式依赖 |
| KeepAlive 内建 | 第三方（react-activation）或路由级设计 |
| 双向绑定组件契约 | defineModel 无对应，手写 value/onChange |

## 5. 生态对照

| 能力 | React | Vue |
|---|---|---|
| 构建 | Vite | Vite（官方） |
| 状态 | Zustand/RTK | Pinia |
| 路由 | React Router | Vue Router |
| Query | TanStack Query | TanStack Query（也适用） |
| UI 库 | antd/MUI/radix | element-plus/naive |
| 表单 | RHF + zod | UI 库自带/VeeValidate |
| SSR | Next.js | Nuxt |
| 动画 | framer-motion | Transition 内建 |
| 测试 | Vitest + RTL | Vitest + test-utils |

## 6. 选型建议

| 场景 | 倾向 |
|---|---|
| 团队类型/推导要求高、生态自由 | React |
| 中后台快速开发、约定统一 | Vue |
| 长期大型项目、多团队并行 | 都可，看招聘与规范成本 |
| 已有团队技能栈 | 跟随团队（切换成本 > 框架差异） |

## 7. 通用能力（两框架同款结论）

以下内容两框架完全一致，直接互参：

- 状态分层三问（刷新/共享/派生）—— react/15、vue3/23
- 服务端状态走 Query —— react/19、vue3/23
- key 铁律 —— react/12、vue3/02
- 竞态三防线 —— react/27、vue3/04
- 错误三路捕获 —— react/33、vue3/36
- 安全双保险 —— react/34、vue3/37
- 性能先测量 —— react/25、vue3/28

## Checklist

- [ ] 能用「重算 vs 追踪」解释两框架的核心差异
- [ ] 概念映射表熟练（面试/结对时快速换脑子）
- [ ] 迁移时的写法差异清单过一遍
- [ ] 通用工程结论直接复用，不重复学习
