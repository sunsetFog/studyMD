# 08 - useReducer 与状态逻辑内聚

## 1. useReducer 是什么

把「状态 + 更新逻辑」收敛到一个纯函数 reducer 中，适合**多个子值、复杂更新规则、联动状态**：

```tsx
import { useReducer } from 'react'

type State = { count: number; step: number }
type Action =
  | { type: 'increment' }
  | { type: 'setStep'; step: number }
  | { type: 'reset' }

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment': return { ...state, count: state.count + state.step }
    case 'setStep':   return { ...state, step: action.step }
    case 'reset':     return { count: 0, step: state.step }
  }
}

const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 })
dispatch({ type: 'increment' })
```

要点：
- reducer 必须**纯函数**：不改入参、不副作用（请求在 effect/事件层）
- dispatch 是稳定引用（不随渲染变化）——天然适合传给 memo 子组件
- `Object.is` 比较返回的 state，相同引用跳过重渲染

## 2. 什么时候用 useReducer 而不是多个 useState

| 信号 | 用 useReducer |
|---|---|
| 多个 state 字段联动更新 | ✅ 一次 action 原子更新 |
| 更新逻辑依赖复杂前值 | ✅ 逻辑集中可测 |
| 同一状态被多处/多类型事件修改 | ✅ action 命名即文档 |
| 下一状态转移需要校验/守卫 | ✅ reducer 内统一守卫 |
| 简单独立字段 | ❌ useState 更轻 |

## 3. 状态机模式（reducer + 联合类型）

```tsx
type FetchState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: User[] }
  | { status: 'error'; message: string }

type FetchAction =
  | { type: 'start' }
  | { type: 'success'; data: User[] }
  | { type: 'error'; message: string }
  | { type: 'reset' }

function reducer(s: FetchState, a: FetchAction): FetchState {
  switch (a.type) {
    case 'start':   return { status: 'loading' }
    case 'success': return { status: 'success', data: a.data }
    case 'error':   return { status: 'error', message: a.message }
    case 'reset':   return { status: 'idle' }
  }
}
```

好处：**非法状态不可表示**（loading 与 error 不可能同时），渲染层 switch status 全覆盖（TS 穷尽检查）。

## 4. 表单 reducer 实战

```tsx
type FormState = { values: Record<string, string>; errors: Record<string, string>; touched: Record<string, boolean> }
type FormAction =
  | { type: 'change'; field: string; value: string }
  | { type: 'blur'; field: string }
  | { type: 'submitFailed'; errors: Record<string, string> }
  | { type: 'reset'; initial: Record<string, string> }

function formReducer(s: FormState, a: FormAction): FormState {
  switch (a.type) {
    case 'change': {
      const values = { ...s.values, [a.field]: a.value }
      const errors = { ...s.errors }
      delete errors[a.field]                       // 输入即清错
      return { ...s, values, errors }
    }
    case 'blur':
      return { ...s, touched: { ...s.touched, [a.field]: true } }
    case 'submitFailed':
      return { ...s, errors: a.errors, touched: markAllTouched(s) }
    case 'reset':
      return { values: a.initial, errors: {}, touched: {} }
  }
}
```

## 5. 初始化与惰性初始化

```tsx
useReducer(reducer, initialArg)              // 直接初值
useReducer(reducer, initialArg, init)        // 第三参惰性初始化：init(initialArg)
useReducer(reducer, url, parseStateFromUrl)  // 典型：从 URL/localStorage 解析初始状态
```

## 6. dispatch 的稳定性优势

```tsx
const Row = memo(function Row({ onSelect }: { onSelect: (id: number) => void }) { … })

// dispatch 引用永不变 → 直接传，无需 useCallback
<Row onSelect={(id) => dispatch({ type: 'select', id })} />   // ❌ 内联函数仍每次新建
// 更优：子组件直接拿 dispatch
<Row dispatch={dispatch} />                                   // ✅ 稳定引用
```

大量列表 + memo 场景下，useReducer 的稳定 dispatch 是免 useCallback 的利器。

## 7. useReducer + Context：迷你状态管理（ reducer 模式）

```tsx
const StateContext = createContext<State | null>(null)
const DispatchContext = createContext<Dispatch<Action> | null>(null)

export function TodosProvider({ children }: Props) {
  const [state, dispatch] = useReducer(todosReducer, [])
  return (
    <StateContext.Provider value={state}>
      <DispatchContext.Provider value={dispatch}>
        {children}
      </DispatchContext.Provider>
    </StateContext.Provider>
  )
}
```

两个 Context 拆分：读状态的组件不因 dispatch 对象重建而重渲染（虽然 dispatch 本身稳定，拆分还能让只写组件不订阅读）。小型应用的全局状态无需引入 Redux/Zustand 即可覆盖。

## 8. 与外部交互的边界

```
组件交互（用户输入、点击）  → dispatch（同步、纯）
外部系统（网络返回、定时器） → 在 effect 中 dispatch
reducer 内                  → 永远不发请求、不碰 DOM、不 console.log 业务
```

## 常见坑

1. **reducer 不纯**：内部发请求/改外部变量，StrictMode 双执行导致重复副作用。
2. **action 类型字符串散落魔法值**：用联合类型 + `as const` 集中定义。
3. **state 内嵌套过深直接改**：展开更新繁琐时考虑 immer 或扁平化状态。
4. **忘记 switch default 穷尽性**：TS 用 `never` 检查兜底：
   ```ts
   default: { const _exhaustive: never = action; return state }
   ```
5. **把所有简单状态都塞 reducer**：单字段独立状态 useState 更简洁。
6. **dispatch 后立即读 state**：拿旧快照；后续逻辑放 effect 或事件流。

## Checklist

- [ ] reducer 纯函数，无副作用
- [ ] Action/State 联合类型 + 穷尽检查
- [ ] 非法状态不可表示（状态机）
- [ ] dispatch 稳定性被利用（免 useCallback）
- [ ] 复杂全局状态评估 useReducer+Context vs 状态库
- [ ] 简单状态不被过度 reducer 化
