# 18 - Suspense 与异步组件实践

## 1. Suspense 解决什么问题

组件的 `setup` 可以是 async（等待异步依赖、顶层 await），此时组件变为「异步组件」——**父组件不知道它何时渲染完成**。Suspense 提供子树的统一加载协调：

```vue
<template>
  <Suspense>
    <!-- 默认插槽：异步子树 -->
    <Dashboard />
    <!-- fallback：等待时的占位 -->
    <template #fallback>
      <PageSkeleton />
    </template>
  </Suspense>
</template>
```

## 2. 工作机制

```
1. Suspense 挂载 → 尝试渲染默认插槽
2. 子树内任一 async setup 未 resolve → 显示 fallback
3. 所有异步依赖 resolve → 整树切换渲染（默认进入无过渡）
4. 子树内出现新的异步依赖（如异步组件加载）→ 回到 pending
```

关键语义：
- **协调是子树级**：一个页面多个 async 组件，全部就绪才显示（瀑布阻塞效果）
- 单个异步组件想「各自 loading」就不要都放 async setup，用 `18-dynamic-async-components.md` 的 defineAsyncComponent

## 3. 配合 Transition

```vue
<Transition name="page" mode="out-in">
  <Suspense :key="route.fullPath">
    <Dashboard />
    <template #fallback><PageSkeleton /></template>
  </Suspense>
</Transition>
```

fallback 与内容间的切换动画：在 `<Suspense>` 外层包 Transition，mode="out-in" 防重叠。

## 4. 事件钩子

```vue
<Suspense @resolve="onResolved" @pending="onPending" @fallback="onFallback">
```

典型用途：resolve 后再初始化依赖 DOM 的第三方库、上报首屏耗时。

## 5. 错误处理

**Suspense 不处理错误**——async setup 抛错会传播到最近的 `onErrorCaptured`：

```ts
// 父组件
onErrorCaptured((err, instance, info) => {
  showError.value = true
  logger.error(err, { info })
  return false   // 阻止继续向上传播
})
```

标准页面骨架：

```vue
<template>
  <ErrorBoundary v-if="error" @retry="reload" />
  <Suspense v-else :key="reloadKey">
    <Dashboard />
    <template #fallback><PageSkeleton /></template>
  </Suspense>
</template>
```

## 6. async setup 的实践规范

### 6.1 什么时候用 async setup

| 场景 | 用 async setup？ |
|---|---|
| 页面首屏必须依赖的数据（如权限/配置） | ✅ 配 Suspense |
| 可后置的数据（列表、统计） | ❌ 组件内自行 loading |
| 重组件按需加载 | ❌ defineAsyncComponent |

原则：**Suspense 只协调「渲染前必须就绪」的依赖**，不要把所有请求都塞进 async setup——否则最慢接口决定整页呈现时间。

### 6.2 并行优化

```ts
// ❌ 串行瀑布
const user = await getUser()
const orders = await getOrders(user.id)

// ✅ 并行
const [user, config] = await Promise.all([getUser(), getConfig()])
const orders = await getOrders(user.id)   // 真正依赖链的保持串行
```

### 6.3 数据请求放哪一层

- 页面级依赖 → async setup + Suspense
- 模块级数据 → composable（useFetch）+ 局部 skeleton
- 交互触发 → 事件回调 + 状态

## 7. 深层嵌套与嵌套 Suspense

Suspense 可嵌套：内层组件自己有异步依赖时，内层 fallback 先展示，内层 resolve 后内层内容呈现，外层随之 resolve。用于「页面骨架 + 局部骨架」两级加载体验。

## 8. SSR 注意

Suspense 是 SSR 数据预取的官方协调机制（配合 Nuxt 的 useAsyncData）。手写 SSR 时需等所有 pending resolve 才能输出 HTML（详见 `29-ssr-nuxt.md`）。

## 9. 当前状态与替代

Suspense 在 3.x 长期为 experimental 状态（API 已稳定使用但文档标注实验性），生产广泛使用（Nuxt 内核）。注意点：
- 不要在 async setup 中无法捕获的地方抛错——统一 try/catch 后 return 状态对象也是稳健替代
- 若团队反馈 Suspense 时序问题难排查，可退化为「组件内 loading + defineAsyncComponent」组合，完全可控

## 常见坑

1. **所有请求塞 async setup**：最慢接口阻塞整页。
2. **async setup 抛错无兜底**：白屏；必须配 onErrorCaptured。
3. **Suspense 不带 key 复用旧内容**：路由参数变化页面不刷新，`:key="route.fullPath"`。
4. **fallback 闪烁**：快速网络下骨架屏闪一下，可给 fallback 加最小展示时长（setTimeout 延迟显示 skeleton 的包装组件）。
5. **嵌套 Suspense 与 Transition 冲突**：mode 与 key 设置不当导致动画丢失。
6. **忘卸载时 abort 请求**：async setup 的请求无法中途取消，页面切走仍完成写入——用 AbortController + onUnmounted。

## Checklist

- [ ] 仅渲染前必须就绪的依赖进 async setup
- [ ] 异步依赖 Promise.all 并行
- [ ] onErrorCaptured 兜底 + retry 机制
- [ ] Suspense 有 :key 与路由参数关联
- [ ] fallback 有防闪烁处理
- [ ] async setup 请求考虑 abort
