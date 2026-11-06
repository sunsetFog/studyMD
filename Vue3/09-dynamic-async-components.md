# 09 - 动态组件 / 异步组件

## 1. 动态组件

```vue
<component :is="currentTab" />
```

- `is` 可以是组件对象（setup 中 import 的）或注册名字符串
- 切换默认**销毁重建**，需要保留状态时包 KeepAlive（见 `12-keepalive.md`）

```vue
<script setup lang="ts">
import TabA from './TabA.vue'
import TabB from './TabB.vue'
const tabs = { TabA, TabB }   // 组件对象映射，避免字符串魔法值
const current = shallowRef(TabA)
</script>
<template>
  <component :is="current" />
</template>
```

`shallowRef` 存组件对象可避免组件被深度响应式代理（大组件树代理开销大）。

## 2. 异步组件 defineAsyncComponent

```ts
import { defineAsyncComponent } from 'vue'

const AsyncChart = defineAsyncComponent(() => import('./HeavyChart.vue'))

// 完整配置
const AsyncPage = defineAsyncComponent({
  loader: () => import('./Page.vue'),
  delay: 200,        // 加载组件延迟显示（防闪烁），默认 200ms
  timeout: 10000,    // 超时报错
  errorComponent: ErrorView,
  loadingComponent: Skeleton,
})
```

### 2.1 使用场景

| 场景 | 理由 |
|---|---|
| 首屏外的重组件（图表、编辑器） | 缩小首屏 chunk |
| 条件触发的弹窗/抽屉 | 点击才加载 |
| 路由懒加载 | Router 自带（`() => import(...)`） |

不必要场景：小组件（几百行）拆 chunk 得不偿失。

### 2.2 全局异步组件

```ts
app.component('AsyncChart', defineAsyncComponent(() => import('./Chart.vue')))
```

## 3. defineAsyncComponent vs Suspense

| 维度 | defineAsyncComponent | Suspense |
|---|---|---|
| 处理对象 | 异步**组件** | 异步 **setup**（含异步依赖） |
| 粒度 | 单组件级 | 子树级 |
| 典型 | 懒加载重组件 | 页面级加载/骨架屏 |
| 错误处理 | errorComponent | onErrorCaptured / fallback |

两者可组合：页面用 Suspense 管异步 setup，重组件再用 defineAsyncComponent 拆 chunk。详见 `18-suspense-async.md`。

## 4. 异步组件加载失败与重试

```ts
const RetryComp = defineAsyncComponent({
  loader: () => import('./Page.vue'),
  retry: 2,             // 自实现：wrapper 内 catch 后延时重试
  errorComponent: () => import('./ErrorRetry.vue'),
})

// 通用重试包装
function retryLoader(loader: () => Promise<any>, times = 3, delay = 1000) {
  return () =>
    loader().catch((e) =>
      times > 0 ? new Promise((res) => setTimeout(res, delay)).then(retryLoader(loader, times - 1)) : Promise.reject(e)
    )
}
```

弱网/灰度发版场景 chunk 404 是常见线上问题，重试 + `window.location.reload()` 兜底是标配。

## 5. 路由级懒加载

```ts
const routes = [
  { path: '/admin', component: () => import('@/views/Admin.vue') },
]
```

配合 vite 的 chunk 命名：

```ts
import(/* @vite-ignore */ url)
() => import(/* webpackChunkName: "admin" */ '@/views/Admin.vue')
```

Vite 中按目录分组的实践：`import.meta.glob` 批量注册：

```ts
const modules = import.meta.glob('@/views/**/*.vue')
const routes = Object.entries(modules).map(([path, load]) => ({
  path: path.replace('/src/views', '').replace('.vue', ''),
  component: load,
}))
```

## 6. 动态组件与 props 传递

```vue
<component :is="current" v-bind="sharedProps" @confirm="onConfirm" />
```

注意：不同组件 props 契约不同时，`v-bind` 大对象容易传错——要么统一组件接口（推荐），要么分别渲染。

## 常见坑

1. **`is` 用字符串但组件未注册**（setup 局部导入的组件不吃字符串）：改用组件对象。
2. **异步组件在 v-if 里频繁重建**：每次显示重新加载，配合 KeepAlive 或提出条件到外层。
3. **chunk 404 无兜底**：发版后旧页面引用旧 chunk 报错白屏，需重试 + reload 兜底。
4. **`import()` 变量路径**：Vite 需 `/* @vite-ignore */` 或 `import.meta.glob`，否则构建报错。
5. **动态组件切换丢状态**：需要 KeepAlive 但没包。
6. **loadingComponent 阈值不当**：delay 设 0 导致快速网络下闪烁。

## Checklist

- [ ] 重组件（图表/编辑器/富页面）已异步化
- [ ] 异步组件有 errorComponent 与超时
- [ ] chunk 加载失败有重试与 reload 兜底
- [ ] 动态组件用对象映射而非字符串
- [ ] 需要保状态的切换包了 KeepAlive
