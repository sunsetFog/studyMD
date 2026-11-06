# 05 - 生命周期

## 1. 全景图

```
setup()                          ← 组合式入口，替代 beforeCreate/created
    ↓
onBeforeMount   → 模板编译完成，未挂载 DOM
    ↓
onMounted       → DOM 已挂载（首次可访问 $el / ref）
    ↓
onBeforeUpdate  → 状态变更，DOM 未更新
    ↓
onUpdated       → DOM 已更新（勿在此改状态 → 死循环）
    ↓
onBeforeUnmount → 卸载前（清理定时器/订阅/事件）
    ↓
onUnmounted     → 已卸载

onErrorCaptured  → 捕获子孙组件错误
onActivated / onDeactivated → KeepAlive 缓存组件激活/失活
onRenderTracked / onRenderTriggered → 调试用，追踪渲染依赖
```

> Vue3 中 `beforeCreate / created` 由 `setup` 本身替代，其余钩子加 `on` 前缀。

## 2. 基本用法（script setup）

```vue
<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'

const size = ref({ w: 0, h: 0 })

onMounted(() => {
  window.addEventListener('resize', onResize)
})
onUnmounted(() => {
  window.removeEventListener('resize', onResize)
})
function onResize() {
  size.value = { w: innerWidth, h: innerHeight }
}
</script>
```

## 3. 挂载期常见用途

| 钩子 | 典型场景 |
|---|---|
| `onBeforeMount` | 极少用；最后一刻修改状态（不触发重渲染） |
| `onMounted` | 请求首屏数据、初始化第三方库（echarts/编辑器）、测量 DOM、绑定全局事件 |

注意：**子组件 onMounted 先于父组件执行**（挂载自底向上完成）。依赖子组件 DOM 的逻辑放 `onMounted` + `nextTick`。

## 4. 更新期

```ts
onBeforeUpdate(() => { /* DOM 更新前读旧值 */ })
onUpdated(() => { /* DOM 已同步，适合测量/日志，禁改状态 */ })
```

陷阱：`onUpdated` 中改响应式状态 → 触发新一轮更新 → 无限循环（dev 会警告）。

## 5. 卸载期清理清单

`onBeforeUnmount / onUnmounted` 必须清理的资源：

```
✅ addEventListener（window/document 级）
✅ setInterval / setTimeout
✅ WebSocket / EventSource 订阅
✅ 第三方实例（chart.dispose()、editor.destroy()）
✅ IntersectionObserver / ResizeObserver / MutationObserver
✅ 全局 store 中的订阅 / 事件总线监听
```

最佳实践：**创建与清理写在同一作用域**，一眼看清配对：

```ts
onMounted(() => { timer = setInterval(tick, 1000) })
onBeforeUnmount(() => { clearInterval(timer) })
```

## 6. KeepAlive 的生命周期

被 `<KeepAlive>` 缓存的组件切换时不触发 unmount/mount，而是：

```ts
onActivated(() => { /* 重新可见：刷新数据、恢复轮询 */ })
onDeactivated(() => { /* 隐藏：暂停轮询、停止视频播放 */ })
```

典型 bug：列表页被 KeepAlive 缓存后返回不刷新数据——数据刷新逻辑应放 `onActivated` 而非 `onMounted`（或两者兼顾）。

## 7. 异步组件与 Suspense 的影响

配合 `<Suspense>`，异步 setup 的钩子时序改变：子组件 `onMounted` 在 resolve 后才触发。涉及加载骨架屏时需注意「父 mounted 时子可能还没 mounted」。详见 `18-suspense-async.md`。

## 8. 生命周期内禁止的操作

| 禁止 | 原因 |
|---|---|
| onUpdated 中修改响应式状态 | 无限循环 |
| onUnmounted 中发起新请求 | 无意义且易内存泄漏 |
| onMounted 中同步大量计算 | 阻塞首屏，改分片或 nextTick |
| 在 setup 顶层注册但不清理 | 泄漏 |

## 9. 父子时序总结

```
创建: 父 beforeCreate → 父 created → 子… → 子 mounted → 父 mounted
更新: 父 beforeUpdate → 子 beforeUpdate → 子 updated → 父 updated
卸载: 父 beforeUnmount → 子 beforeUnmount → 子 unmounted → 父 unmounted
```

## 常见坑

1. **忘记卸载清理**：路由切换后事件还在触发、定时器还在跑。
2. **KeepAlive 页面用 onMounted 刷新数据**：返回时数据陈旧。
3. **父 onMounted 里测子组件 DOM**：子已 mounted（OK），但若子是异步组件则未就绪，需 Suspense 或 nextTick。
4. **onUpdated 里改状态**：死循环。
5. **async setup 阻塞渲染**：未配 Suspense 时组件不渲染且无提示，必须配 `<Suspense>`。
6. **在模块顶层写 onMounted**：钩子必须在 setup 同步作用域内注册，异步回调中注册不生效。

## Checklist

- [ ] 每个全局监听/定时器都有对应清理
- [ ] KeepAlive 页面数据刷新放 onActivated
- [ ] onUpdated 无状态修改
- [ ] 首屏请求放 onMounted（或 setup 顶层 SSR 感知处理）
- [ ] 第三方实例初始化与 dispose 配对
