# 09 - 自定义 Hooks 设计规范

## 1. 什么是自定义 Hook

**以 use 开头、内部调用其他 Hooks 的函数**——React 中复用「有状态逻辑」的唯一正规手段（替代 class 时代的 HOC/mixin）。

```tsx
// hooks/useLocalStorage.ts
import { useState, useEffect } from 'react'

export function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const raw = localStorage.getItem(key)
      return raw ? (JSON.parse(raw) as T) : initial
    } catch { return initial }
  })

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value))
  }, [key, value])

  return [value, setValue] as const   // as const 保持元组类型
}
```

规则：
- 每次调用是**独立实例**（各组件的 useLocalStorage 互不隔离共享）
- 只能在组件/其他 Hook 顶层调用（同 Hooks 规则）

## 2. 设计原则

| 原则 | 说明 |
|---|---|
| 输入兼容响应式 | 参数用 `RefLike<T> | T`，内部 `toValue` 归一化（React 无内置，自实现） |
| 返回稳定引用 | 返回对象用 useMemo，函数用 useCallback，或返回元组 |
| 自动清理 | 内部订阅/定时器在 effect cleanup 中清理 |
| 竞态保护 | 异步请求带 active 标记/Abort |
| SSR 安全 | 不直接访问 window（守卫 import.meta.server / typeof window） |

### 2.1 响应式参数归一化（React 版 toValue）

```tsx
type MaybeGetter<T> = T | (() => T)

function toValue<T>(v: MaybeGetter<T>): T {
  return typeof v === 'function' ? (v as () => T)() : v
}

export function useFiltered(list: Item[] | (() => Item[]), kw: string) {
  return useMemo(() => {
    const l = toValue(list)
    const k = toValue(kw).toLowerCase()
    return k ? l.filter(i => i.name.toLowerCase().includes(k)) : l
  }, [list, kw])   // getter 引用需调用方稳定，或内部用 useLatest
}
```

## 3. 必备基础 Hooks 清单（自建最小集）

```tsx
// useEventListener：自动清理的事件订阅
export function useEventListener<K extends keyof WindowEventMap>(
  type: K, handler: (e: WindowEventMap[K]) => void, target: Window | HTMLElement = window,
) {
  const saved = useRef(handler)
  useEffect(() => { saved.current = handler })            // 永远指向最新回调
  useEffect(() => {
    const fn = (e: Event) => saved.current(e as never)
    target.addEventListener(type, fn)
    return () => target.removeEventListener(type, fn)
  }, [type, target])
}

// useDebounce / useThrottle
// useInterval（可暂停）
export function useInterval(cb: () => void, ms: number | null) {
  const saved = useRef(cb)
  useEffect(() => { saved.current = cb })
  useEffect(() => {
    if (ms === null) return
    const t = setInterval(() => saved.current(), ms)
    return () => clearInterval(t)
  }, [ms])   // ms 为 null 即暂停
}

// useMediaQuery、useOnClickOutside、useTitle、useClipboard、useInView
```

先查 **VueUse 的 React 对应物（react-use / ahooks / usehooks.com）**，避免重复造轮子。

## 4. 业务 Hook 模板：数据请求（Query 之前的手写版）

```tsx
export function useUser(id: string) {
  const [state, setState] = useState<AsyncState<User>>({ status: 'idle' })

  useEffect(() => {
    let active = true
    setState({ status: 'loading' })
    api.getUser(id)
      .then(d => active && setState({ status: 'success', data: d }))
      .catch(e => active && setState({ status: 'error', message: String(e) }))
    return () => { active = false }
  }, [id])

  return state   // { status, data?, message? } 联合类型互斥态
}
```

生产替换为 TanStack Query（`19-tanstack-query.md`）——手写版的价值是理解 + 无依赖场景。

## 5. 组合与分层

```
hooks/
├── primitives/     # 通用原语：useEventListener / useInterval / useDebounce（无业务）
├── queries/        # 服务端状态封装：useOrders / useUser（包 TanStack Query）
├── features/       # 业务流程：useCheckoutFlow / useAuth
└── ui/             # UI 状态：useModal / useToast
```

抽取时机：同一逻辑 ≥2 处使用，或组件内一段逻辑自成闭环且 >40 行。**不要为了「看起来整洁」过度抽取一次性逻辑**。

## 6. Hooks 调用规则的深层原因

为什么只能在顶层调用？——React 按**调用顺序**在 Fiber 节点上索引 Hooks（见 `30-hooks-internals.md`）：

```tsx
// ❌ 条件调用导致顺序错位
if (cond) { const [a, setA] = useState(0) }
const [b, setB] = useState(0)   // cond 变化时 b 拿到 a 的槽位 → 状态错乱
```

自定义 Hook 内部的 Hooks 计入调用组件的顺序表——所以「Hook 只能在顶层」的规则同样约束 Hook 内部。

## 7. 返回值设计

| 形式 | 适用 |
|---|---|
| 元组 `[value, setter] as const` | 两个语义对（state 模式） |
| 对象 `{ data, loading, error, reload }` | 多返回值（请求模式） |
| 单函数/值 | 命令式或单一结果 |

对象返回时 **useMemo 包一层**（或内容全为稳定引用），避免消费方依赖它时 effect 连锁重跑：

```tsx
return useMemo(() => ({ data, loading, reload }), [data, loading, reload])
```

## 常见坑

1. **Hook 内条件调用 Hooks**：顺序错位，状态串门。
2. **返回不稳定引用**：消费方 useEffect 依赖它 → 每次渲染重跑。
3. **忘清理**：useInterval/useEventListener 泄漏。
4. **异步竞态**：组件卸载后 setState（React 18 后无警告但仍是逻辑错误）。
5. **Hook 里读 props 却不接参数**：闭包旧值；该传的参数显式传。
6. **全局单例误解**：普通自定义 Hook 是每组件实例私有；共享需求走 Context/状态库。
7. **SSR 直接访问 window**：抽到 effect 或守卫。

## Checklist

- [ ] Hook 内无条件 Hooks 调用
- [ ] 返回值引用稳定（useMemo/useCallback/as const）
- [ ] 每个订阅/定时器有 cleanup
- [ ] 异步逻辑有竞态保护
- [ ] 优先复用 ahooks/react-use，不重复造轮子
- [ ] 分层目录清晰（primitives/queries/features/ui）
