# 15 - VueUse 与 hooks 生态

## 1. VueUse 是什么

`@vueuse/core`：200+ 个高质量 composables 集合，覆盖传感器、状态、元素、网络、动画等，全部遵循 `13-composables.md` 的规范（MaybeRefOrGetter 输入、自动清理、SSR 安全）。

```bash
npm i @vueuse/core
```

## 2. 常用分类速查

### 2.1 状态

| API | 用途 |
|---|---|
| `useLocalStorage(key, def)` | localStorage 双向响应式（自动序列化） |
| `useSessionStorage` | 同上，会话级 |
| `useStorageAsync` | 异步存储适配 |
| `createGlobalState` | 定义跨组件全局状态（轻量 Pinia 替代） |
| `createSharedComposable` | 多组件共享同一 composable 实例 |
| `useAsyncState` | 异步请求三件套（state/isReady/execute） |

### 2.2 元素 / 传感器

| API | 用途 |
|---|---|
| `useMouse` / `useMousePressed` | 鼠标位置/按压 |
| `useIntersectionObserver` | 进入视口（懒加载、曝光埋点） |
| `useResizeObserver` | 尺寸变化（自适应组件） |
| `useElementSize` | 元素宽高 |
| `useVirtualList` | 大列表虚拟滚动 |
| `useInfiniteScroll` | 无限加载 |
| `useDraggable` | 拖拽 |
| `useFocusWithin` | 焦点管理 |

### 2.3 定时 / 动画

| API | 用途 |
|---|---|
| `useDebounceFn` / `useThrottleFn` | 防抖节流函数 |
| `refDebounced` / `refThrottled` | 防抖节流的 ref |
| `useIntervalFn` / `useTimeoutFn` | 可暂停定时器（自动清理） |
| `useRafFn` | requestAnimationFrame 循环 |
| `useTransition` | 数值过渡动画 |
| `useNow` | 响应式当前时间 |

### 2.4 浏览器

| API | 用途 |
|---|---|
| `useEventListener` | 自动清理事件监听 |
| `useMediaQuery('(min-width: 768px)')` | 响应式媒体查询 |
| `usePreferredDark` | 暗色偏好 |
| `useClipboard` | 剪贴板 |
| `usePermission` | 权限状态（相机/麦克风） |
| `useGeolocation` | 地理位置 |
| `useBroadcastChannel` | 跨 tab 通信 |
| `useWebSocket` | WebSocket（自动重连） |
| `useFps` | FPS 监测（性能调试） |
| `useIdle` | 用户空闲检测 |

### 2.5 UI / 杂项

`useTitle` `useDark` `useToggle` `useFullscreen` `useSwipe` `useScroll` `useDocumentVisibility` `useWindowScroll` `useIntersectionObserver`。

## 3. 典型用法示例

```ts
// 本地存储持久化
const settings = useLocalStorage('app-settings', { theme: 'light', size: 'default' })

// 曝光埋点
const { stop } = useIntersectionObserver(target, ([entry]) => {
  if (entry.isIntersecting) { report(); stop() }
}, { threshold: 0.5 })

// 自动清理的全局事件
useEventListener(window, 'resize', onResize)   // 组件卸载自动移除

// 可暂停倒计时
const { pause, resume, remaining } = useIntervalFn(tick, 1000)

// 跨 tab 同步
const { data, post } = useBroadcastChannel({ name: 'auth' })
watch(data, d => d?.type === 'logout' && doLogout())
```

## 4. createGlobalState vs Pinia

```ts
// createGlobalState：适合简单的全局单例状态
const useCounter = createGlobalState(() => {
  const count = ref(0)
  const inc = () => count.value++
  return { count, inc }
})
```

| 维度 | createGlobalState | Pinia |
|---|---|---|
| devtools | 弱 | 完整（时间旅行/动作追踪） |
| 模块划分 | 函数级 | store 级 |
| 生态/插件 | 少 | 持久化/订阅等生态成熟 |
| 适用 | 简单全局状态 | 中大型应用状态层 |

原则：**业务状态一律 Pinia；createGlobalState 用于纯前端工具性状态**（如全局主题实例）。

## 5. 使用规范

1. **按需导入**，tree-shaking 自动生效
2. **升级策略**：锁次版本，关注 breaking（配置项偶有变动）
3. **自己封装一层**：业务中不直接散用 API，在 `composables/` 里做薄封装（统一 SSR 分支、统一埋点）

```ts
// composables/usePageVisibilityLog.ts —— 业务封装示例
import { useDocumentVisibility } from '@vueuse/core'
export function usePageVisibilityLog(onShow: () => void, onHide: () => void) {
  const v = useDocumentVisibility()
  watch(v, s => s === 'visible' ? onShow() : onHide())
}
```

4. **SSR**：大多数 API 自动处理（返回默认值），涉及 DOM 的在 client 环境判断

## 常见坑

1. **useLocalStorage 共享 key**：不同组件传不同默认值，后挂载覆盖先挂载。
2. **useIntersectionObserver 忘记 stop**：一次性曝光场景需要手动 stop。
3. **useIntervalFn 默认立即启动**：需要手动控制时传 `{ immediate: false }`。
4. **SSR 中 useLocalStorage 报 window 未定义**：确认 SSR 环境（旧版本个别 API 需手动判断）。
5. **useDebounceFn 与 watch 组合错位**：对 ref 防抖应使用 `refDebounced` 或 `watchDebounced`，而不是 debounce 函数后再 watch（时序差一拍）。

## Checklist

- [ ] 引入 @vueuse/core 并统一在 composables 封装层使用
- [ ] 一次性观察器正确 stop
- [ ] 定时器类 API 用 pause/resume 管理生命周期
- [ ] 业务状态走 Pinia，工具状态才用 createGlobalState
- [ ] localStorage key 有统一命名空间前缀
