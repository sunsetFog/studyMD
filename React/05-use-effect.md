# 05 - useEffect 深入

## 1. useEffect 是什么

**不是生命周期钩子**，而是「把副作用与渲染结果同步」的机制：Effect 在**提交到 DOM 之后**异步执行，且依赖不变时跳过。

```tsx
useEffect(() => {
  const conn = createConnection(roomId)
  conn.connect()
  return () => conn.disconnect()      // cleanup：下次执行前/卸载时调用
}, [roomId])
```

心智模型：**Effect = 「让外部系统与当前 props/state 保持同步」**。每次依赖变化 = 外部系统的参数变了 → 先清理旧配置，再按新参数重配。

## 2. 执行时机

```
渲染（函数执行）→ React 提交 DOM → 浏览器绘制 → useEffect 执行
                                  （layoutEffect 在绘制前，见 §7）
```

- **首次渲染后**执行一次
- **依赖变化的重渲染后**：先 cleanup（旧闭包）再 effect（新闭包）
- **卸载时**：最后一次 cleanup
- StrictMode 开发期：mount → effect → cleanup → effect（检测非幂等副作用）

Effect 内读到的 props/state 是**当次渲染的闭包快照**。

## 3. 依赖数组规则

```tsx
useEffect(fn)           // 每次渲染后执行（慎用）
useEffect(fn, [])       // 仅 mount 后（+ unmount cleanup）
useEffect(fn, [a, b])   // a/b Object.is 变化后
```

规则：
- **诚实声明**：effect 内用到的所有响应式值都必须在依赖里，lint（exhaustive-deps）强制
- 不想要的依赖不是靠删依赖解决，而是**改代码**（把值移进 effect、提取到组件外、用 reducer 合并状态）

## 4. 竞态与清理（异步请求标准模板）

```tsx
useEffect(() => {
  let active = true
  setLoading(true)
  fetchUser(id)
    .then(user => { if (active) { setUser(user); setLoading(false) } })
    .catch(e => { if (active) setError(e) })
  return () => { active = false }
}, [id])
```

或 AbortController：

```tsx
useEffect(() => {
  const ctrl = new AbortController()
  fetch(`/api/user/${id}`, { signal: ctrl.signal })
    .then(r => r.json()).then(setUser)
    .catch(e => { if (e.name !== 'AbortError') setError(e) })
  return () => ctrl.abort()
}, [id])
```

**生产建议**：请求类逻辑直接用 TanStack Query（自带竞态/缓存/重试），见 `19-tanstack-query.md`；手写 effect 请求仅适合学习理解。

## 5. 不需要 Effect 的场景（官方重点）

| 场景 | 错误写法 | 正确写法 |
|---|---|---|
| props 派生值 | effect 同步进 state | 渲染期直接计算 |
| 事件响应后更新 | 在 effect 里监听 | 事件回调里直接处理 |
| state 变化触发另一个 state | 链式 effect | 单事件里一起 set / useReducer |
| 表单提交后跳转 | effect 监听 submitted | 回调里 navigate |
| 全局 JS 状态联动 | effect 桥接 | 状态收口单一来源 |

```tsx
// ❌ effect 链
const [firstName, setFirstName] = useState('')
const [fullName, setFullName] = useState('')
useEffect(() => { setFullName(firstName + ' ' + lastName) }, [firstName, lastName])

// ✅ 渲染期计算
const fullName = `${firstName} ${lastName}`
```

原则：**effect 只做「与外部系统同步」**（网络、DOM 库、订阅、定时器、localStorage）；组件内部的状态联动用渲染计算/事件/reducer。

## 6. 惰性事件订阅与 re-attach 模式

```tsx
// 浏览器事件随依赖重绑
useEffect(() => {
  const onResize = () => setSize({ w: innerWidth, h: innerHeight })
  window.addEventListener('resize', onResize)
  return () => window.removeEventListener('resize', onResize)
}, [])   // onResize 引用稳定的写法见 06 useCallback / 或函数定义进 effect 内
```

封装成自定义 hook（useEventListener）统一管理，见 `09-custom-hooks.md`。

## 7. useLayoutEffect vs useEffect

| | useEffect | useLayoutEffect |
|---|---|---|
| 时机 | 绘制后异步 | DOM 变更后、绘制前同步 |
| 用途 | 常规副作用 | 需要在绘制前测量/修改 DOM（防闪烁） |
| 性能 | 不阻塞绘制 | 阻塞绘制，滥用掉帧 |

典型场景：Tooltip 定位测量、防止初始闪烁的位置校正。**默认 useEffect，只有看到视觉闪烁才换 layout**。

## 8. useSyncExternalStore（订阅外部 store 的正确姿势）

```tsx
const width = useSyncExternalStore(
  (cb) => { window.addEventListener('resize', cb); return () => window.removeEventListener('resize', cb) },
  () => window.innerWidth,     // getSnapshot（必须稳定返回缓存值）
  () => 0,                     // SSR snapshot
)
```

订阅外部系统（非 React 状态）时**优先于 useEffect + setState**：无 tearing、天然并发安全。Zustand/Jotai 内部即基于它。

## 9. Effect 滥用信号自查

```
信号：effect 里 setState 一个「能算出来的值」     → 渲染期计算
信号：effect A 的输出触发 effect B               → 合并或 useReducer
信号：effect 依赖数组被 lint 红线逼着撒谎        → 重构状态结构
信号：同一数据在 effect 里被多次 fetch            → 提升为 Query 缓存
```

## 常见坑

1. **忘 cleanup**：订阅/定时器/连接泄漏，组件卸载后仍执行。
2. **竞态**：快速切换 id，旧请求后返回覆盖新数据。
3. **依赖撒谎**：lint 屏蔽后读旧闭包，出现「状态不更新」诡异 bug。
4. **effect 链**：状态 A→effect 设 B→effect B 触发 C，重渲染风暴。
5. **把 effect 当生命周期用**：`[]` 里塞「只跑一次」的逻辑但读了响应式值 → StrictMode 双执行 + 旧闭包。
6. **对象/数组/函数依赖**：每次渲染新引用 → effect 每次都跑；依赖收敛为具体字段或用 memo。
7. **cleanup 里异步未防序**：cleanup 与 effect 交错执行，写脏状态。

## Checklist

- [ ] effect 只做外部系统同步，无内部状态联动
- [ ] 所有订阅/定时器有 cleanup
- [ ] 异步请求有竞态保护（active 标记 / Abort / Query）
- [ ] 依赖数组诚实，不屏敝 lint
- [ ] 无 effect 链与 effect 中 setState 派生
- [ ] 外部 store 订阅考虑 useSyncExternalStore
