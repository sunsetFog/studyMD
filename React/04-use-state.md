# 04 - useState 深入

## 1. 基本用法

```tsx
const [state, setState] = useState<Type>(initial)
```

- 返回 `[当前快照值, 更新函数]`（数组解构，命名自由但约定 `[x, setX]`）
- initial 只在**首次渲染**生效，之后传入的初始值被忽略
- 更新后组件**重新执行**，拿到新快照

## 2. 惰性初始化

```tsx
// ❌ 每次渲染都执行 expensiveInit（虽然结果被忽略）
const [data, setData] = useState(expensiveInit())

// ✅ 惰性初始化：只在首次渲染执行
const [data, setData] = useState(() => expensiveInit())

// 典型场景：从 localStorage / URL 解析 / 大计算
const [filter, setFilter] = useState<Filter>(() => parseFilter(searchParams))
```

## 3. 更新语义：替换 vs 函数式

```tsx
setCount(5)              // 直接替换
setCount(c => c + 1)     // 函数式：基于队列中的最新值
```

函数式必用场景：

```tsx
// 1. 同一事件中多次更新
setCount(c => c + 1)
setCount(c => c * 2)

// 2. 更新依赖旧值且旧值可能来自闭包（定时器/异步回调）
useEffect(() => {
  const timer = setInterval(() => setCount(c => c + 1), 1000)  // ✅ 永远正确
  return () => clearInterval(timer)
}, [])   // 空依赖也不会读到旧闭包

// 3. 数组/对象嵌套更新
setUsers(us => us.map(u => u.id === id ? { ...u, active: true } : u))
```

## 4. 不可变更新模板库

```tsx
// 对象
setState(s => ({ ...s, key: v }))                    // 改单字段
setState(s => ({ ...s, nested: { ...s.nested, k: v } }))  // 嵌套改

// 数组
setArr(a => [...a, item])                            // 尾加
setArr(a => [item, ...a])                            // 头插
setArr(a => a.filter(x => x.id !== id))              // 删
setArr(a => a.map(x => x.id === id ? { ...x, ...patch } : x))  // 改
setArr(a => a.slice(0, i).concat(item, a.slice(i)))  // 插入
setArr(a => a.sort(...))                             // ❌ sort/splice 原地改！
setArr(a => [...a].sort(...))                        // ✅ 先拷贝再原地方法

// 深层结构复杂时 → 拆 state / useReducer / immer
```

## 5. state 拆分与合并原则

| 原则 | 说明 |
|---|---|
| 经常一起变的合并为一个对象 | 减少多次 setState |
| 独立变化的拆开 | 避免不必要的重渲染与复杂更新 |
| 能计算出来的不存 state | `const total = price * qty` 渲染期算 |
| 冗余/镜像 state 是 bug 温床 | props 已有值不复制进 state |

```tsx
// ❌ 镜像 state：props 变化要手动同步，必然出 bug
const [name, setName] = useState(props.name)

// ✅ 要「初始值语义」用 key 重置；要「可编辑副本」见 05
```

## 6. key 重置组件状态（重要模式）

```tsx
// 需求：切换用户时表单完全重置
<ProfileForm key={userId} userId={userId} />
```

key 变化 → React 卸载旧组件、挂载新组件 → state 全新初始化。**这是显式声明「重置」的标准手段**，优于 useEffect 同步 props。

## 7. 基于前值派生的正确姿势

```tsx
// ❌ useEffect 同步两个 state（链式 effect 地狱）
const [page, setPage] = useState(1)
const [list, setList] = useState([])
useEffect(() => { fetchList(page).then(setList) }, [page])   // 见 05 的分析

// ✅ 渲染期直接根据 page 触发请求的方案：TanStack Query
const { data: list } = useQuery({ queryKey: ['list', page], queryFn: () => fetchList(page) })
```

## 8. 状态机替代布尔组合

```tsx
// ❌ 三布尔组合出非法态
const [loading, setLoading] = useState(false)
const [error, setError] = useState(null)
const [data, setData] = useState(null)   // loading && error 可同时为真

// ✅ 联合类型单 state，状态互斥
type Phase = { s: 'idle' } | { s: 'loading' } | { s: 'ok'; data: T } | { s: 'err'; e: Error }
const [phase, setPhase] = useState<Phase>({ s: 'idle' })
```

## 常见坑

1. **setState 后立即读 state**：拿到旧值（快照模型）；需要后续动作进 useEffect。
2. **原地修改数组方法**（push/splice/sort 直接调用）：不触发渲染或 diff 混乱。
3. **非惰性初始化昂贵计算**：每次渲染白算。
4. **镜像 props 进 state**：同步地狱；用 key 重置或渲染期计算。
5. **闭包陷阱**：setTimeout/事件监听里的 state 是旧快照——函数式更新解决。
6. **一个巨型 state 对象**：每次改任意字段全量替换，重渲染失控；拆分或 useReducer。

## Checklist

- [ ] 依赖旧值的更新全部函数式
- [ ] 数组更新无原地修改（push/splice/sort 前先拷贝）
- [ ] 无镜像/冗余 state
- [ ] 重置场景用 key 而非 effect 同步
- [ ] 异步状态用互斥联合类型而非布尔组合
- [ ] 初始值昂贵时惰性初始化
