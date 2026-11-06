# 31 - 并发特性与调度

## 1. 并发渲染的本质

React 18 前渲染是同步的：一次 setState → 阻塞式渲染到完成。并发模式（createRoot）让渲染：

```
可中断：高优更新插队时暂停低优渲染
可恢复：从中断点继续（Fiber 链表）
可丢弃：被打断的低优渲染整体作废，重头再来
```

前提是 **render 阶段纯计算**（见 28）。

## 2. 优先级模型（Lanes）

```
SyncLane          同步（flushSync）
InputContinuousLane  连续输入（拖拽/输入）
DefaultLane       普通更新
TransitionLane    startTransition 标记
IdleLane          空闲任务
```

- 低优渲染中来了高优 → 暂停低优 → 先做高优 → 回来时低优可能作废重算
- 组件函数可能因此**多次执行**——纯函数纪律的深层原因

## 3. useTransition：标记低优先级

```tsx
const [isPending, startTransition] = useTransition()

function onTabChange(id: string) {
  setActive(id)                          // 高优：立即切换
  startTransition(() => {
    setTabContent(loadHeavyContent(id))  // 低优：可中断
  })
}
return { isPending && <TabSkeleton /> }
```

语义：**「这个更新不急，让用户输入先走」**。典型场景：Tab 切换、搜索列表、路由内容切换。

判断法：更新慢导致输入/点击卡顿 → 该更新包 transition。

## 4. useDeferredValue：延迟值

```tsx
const deferredQuery = useDeferredValue(query)
const results = useMemo(() => bigSearch(deferredQuery), [deferredQuery])
const isStale = query !== deferredQuery   // 列表显示旧值期间可降透明度
```

与 useTransition 对比：

| | useTransition | useDeferredValue |
|---|---|---|
| 包什么 | 更新动作（自己写的 setState） | 值（props 或别人的更新） |
| 控制方 | 调用处 | 使用处 |
| 场景 | 代码可控的切换逻辑 | 组件收到的外部值 |

## 5. Suspense 与并发的关系

- Suspense 的 fallback 切换由并发协调：**低优挂起时保留旧 UI**（不闪 fallback）
- `startTransition(() => setShow(true))` 包含 lazy 组件切换 → 内容就绪前维持旧界面
- 这是「防 fallback 闪烁」的标准手段（见 14）

## 6. flushSync：强制同步

```tsx
import { flushSync } from 'react-dom'

flushSync(() => setCount(1))       // 立即渲染（跳过并发调度）
console.log(domRef.current!.textContent)   // 现在能读到
```

仅在「必须在同帧读取更新后 DOM」的罕见场景使用（如第三方动画库测量）；滥用 = 掉帧。

## 7. 并发时代的副作用纪律

| 场景 | 并发下的行为 | 纪律 |
|---|---|---|
| 渲染期副作用 | 可能双执行/丢弃 | 禁止；放 effect |
| effect 请求 | StrictMode 双执行 | cleanup 幂等 / Query 去重 |
| 事件回调副作用 | 只执行一次 | 放事件回调 |
| DOM 测量 | render 中读到中间态 | layout effect / commit 后 |

## 8. 自动批处理边界

```tsx
// React 18+ 全场景批处理
setA(1); setB(2)                     // 事件内
promise.then(() => { setA(1); setB(2) })   // 异步内也批处理
setTimeout(() => { setA(1); setB(2) }, 0)  // 同上

// 退出批处理
flushSync(() => setA(1))
```

## 9. 实战决策速查

```
输入 + 昂贵过滤     → useDeferredValue（值侧）
Tab/路由切换慢      → useTransition（更新侧）
lazy 切换闪 fallback → startTransition 包状态切换
必须同帧读 DOM      → useLayoutEffect / flushSync（慎）
一般请求           → Query（并发安全内建）
```

## 10. 与 Vue 的调度对比

| | React | Vue |
|---|---|---|
| 中断渲染 | 有（Fiber） | 无（渲染轻，不需要） |
| 优先级 | lanes 多级 | 无显式优先级（队列顺序） |
| transition 语义 | 显式 API | 无直接对应（另有 transition-group 是动画概念） |
| 批处理 | 自动（18+） | 自动（队列） |

## 常见坑

1. **transition 包了同步昂贵计算**：transition 只降低更新优先级，不减少计算量；计算本身要拆/Worker。
2. **isPending 当 loading 滥用**：isPending 是「低优渲染未完成」不是网络 loading。
3. **useDeferredValue 无 useMemo 配合**：每次渲染重算，白延迟。
4. **flushSync 滥用**：性能劣化。
5. **假设 setState 立即生效写逻辑**：快照模型不变。
6. **低优渲染作废导致 effect 双跑**：请求类必须幂等/去重。

## Checklist

- [ ] 能解释可中断渲染与纯函数的关系
- [ ] transition/deferredValue 场景分得清
- [ ] fallback 闪烁用 startTransition 解决
- [ ] flushSync 仅限必要场景
- [ ] 副作用纪律在并发语义下重新审视过
