# 14 - Portal / ErrorBoundary / Suspense

## 1. Portal（传送门）

把子节点渲染到父组件 DOM 层级之外的容器——**逻辑上是子组件，DOM 位置任意**：

```tsx
import { createPortal } from 'react-dom'

function Modal({ children }: { children: React.ReactNode }) {
  const [el] = useState(() => document.createElement('div'))   // 稳定实例
  useEffect(() => {
    document.body.appendChild(el)
    return () => document.body.removeChild(el)                  // 卸载清理
  }, [el])
  return createPortal(children, el)
}
```

事件冒泡沿 **React 组件树**（不是 DOM 树）——Portal 内 onXxx 仍能被组件层级的父捕获；但原生事件委托行为按 DOM 层级。

用途：Modal/Drawer/Tooltip/通知，解决父级 `overflow/transform/z-index` 层叠上下文限制（与 Vue Teleport 同理，见 vue3/11）。

z-index 治理：全局层级规范变量 + 统一浮层管理器（考虑 zustand 存浮层栈做滚动锁计数）。

## 2. ErrorBoundary（错误边界）

**class 组件独有能力**（React 尚无 Hook 版），捕获子树渲染期/生命周期错误：

```tsx
import { Component, type ReactNode } from 'react'

type Props = { fallback: ReactNode | ((e: Error) => ReactNode); children: ReactNode; onError?: (e: Error) => void }
type State = { error: Error | null }

export class ErrorBoundary extends Component<Props, State> {
  state: State = { error: null }

  static getDerivedStateFromError(error: Error): State {
    return { error }
  }
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    this.props.onError?.(error)
    reportError(error, { componentStack: info.componentStack })
  }

  reset = () => this.setState({ error: null })

  render() {
    const { error } = this.state
    if (error) {
      const fb = typeof this.props.fallback === 'function' ? this.props.fallback(error) : this.props.fallback
      return <ErrorContext.Provider value={{ reset: this.reset }}>{fb}</ErrorContext.Provider>
    }
    return this.props.children
  }
}
```

### 2.1 部署粒度与边界

| 能捕获 | 不能捕获 |
|---|---|
| 子树渲染期错误 | 事件回调错误（自己 try/catch） |
| 生命周期错误 | 异步代码错误（promise reject） |
| constructor 错误 | SSR 错误、ErrorBoundary 自身错误 |

部署策略：**路由级包一层**（页面崩溃不拖垮全局）+ 关键独立区块（图表/第三方嵌入）单独包。fallback 内提供重试按钮（reset 或 :key 重挂载子树）。

### 2.2 react-error-boundary 库（推荐直接用）

```tsx
import { ErrorBoundary, useErrorBoundary } from 'react-error-boundary'

<ErrorBoundary FallbackComponent={ErrorFallback} onReset={…} resetKeys={[userId]}>
  <PageView />
</ErrorBoundary>

// 函数组件内主动抛错
const { showBoundary } = useErrorBoundary()
api.get().catch(showBoundary)     // 把异步错误交给边界
```

## 3. Suspense

**声明式 loading 边界**：子树内任何「悬挂」（suspend）的组件显示 fallback：

```tsx
<Suspense fallback={<PageSkeleton />}>
  <Dashboard />
</Suspense>
```

触发 suspend 的三种来源：
1. **懒加载组件**（React.lazy）——最经典
2. **use(promise)**（React 19）
3. **框架数据获取**（Next.js use / RSC 流式、TanStack Query 的 suspense 模式）

### 3.1 React.lazy + 代码分割

```tsx
import { lazy } from 'react'
const HeavyChart = lazy(() => import('./HeavyChart'))

<Suspense fallback={<ChartSkeleton />}>
  {showChart && <HeavyChart />}
</Suspense>
```

- lazy 组件必须挂在 Suspense 内
- chunk 失败需 reload 兜底（同 vue3/09）：监听 window error 或重试包装
- 懒加载选择同 Vue：重组件（图表/编辑器）才值得

### 3.2 Suspense 的语义细节

- fallback 与内容切换**是显示替换**（不卸载已挂载兄弟）
- 嵌套 Suspense：内层先 resolve 内层显示（局部骨架体验）
- 事务性：数据未就绪不显示「半成品 UI」——这是与「组件内 loading 布尔」的本质区别
- 并发可中断：fallback 显示期间新数据到达可直接跳过 fallback

### 3.3 与 TanStack Query 结合

```tsx
const { data } = useSuspenseQuery({ queryKey: ['user', id], queryFn: … })
// promise 挂起 → 自动交给最近 Suspense；数据缓存后不再挂起
```

## 4. 三件套组合：页面标准骨架

```tsx
function Page() {
  return (
    <ErrorBoundary FallbackComponent={PageErrorFallback} resetKeys={[routeKey]}>
      <Suspense fallback={<PageSkeleton />}>
        <PageContent />
      </Suspense>
    </ErrorBoundary>
  )
}
```

层次约定：**ErrorBoundary 在外（捕获 Suspense 子树内错误），Suspense 在内**。

## 5. 过渡动画与三者协作

```tsx
// lazy 切换 + 过渡：用 startTransition 防止 fallback 闪现
const [show, setShow] = useState(false)
startTransition(() => setShow(true))   // 挂起新内容时保留旧 UI 而非闪 fallback
```

## 常见坑

1. **Portal 忘清理容器节点**：DOM 泄漏。
2. **ErrorBoundary 捕不了事件/异步错误**：回调 try/catch 或 showBoundary 转交。
3. **边界粒度过粗**：一处崩全页灰；路由级 + 区块级分层。
4. **lazy 组件没包 Suspense**：报错 requires a Suspense。
5. **fallback 闪烁**：startTransition 包切换 / fallback 延迟显示包装。
6. **Portal 内滚动锁未计数管理**：多层 Modal 嵌套滚动失控。
7. **use(promise) 每次渲染新 Promise**：永远 pending（见 10）。

## Checklist

- [ ] 浮层统一 Portal + z-index 规范 + 滚动锁计数
- [ ] 路由级 ErrorBoundary + 重试机制
- [ ] 异步错误转交边界（showBoundary）
- [ ] lazy 组件有 Suspense + chunk 失败兜底
- [ ] 页面骨架 ErrorBoundary 外层 + Suspense 内层
- [ ] 无渲染中的裸 Promise 调用
