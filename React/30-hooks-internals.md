# 30 - Hooks 实现原理

## 1. Hooks 存在哪里

每个 Fiber 节点的 `memoizedState` 保存该组件的 **Hooks 链表**——按调用顺序连接的单链表，每节点一个 hook：

```ts
interface Hook {
  memoizedState: any        // 当前状态（useState 的值 / effect / ref）
  queue: UpdateQueue        // 更新队列（useState/useReducer）
  next: Hook | null         // 下一个 hook
}
```

```
function App() {
  const [a, setA] = useState(0)     // hook 1
  const [b, setB] = useState(0)     // hook 2
  useEffect(() => {})               // hook 3
}
// fiber.memoizedState: hook1 → hook2 → hook3
```

**调用顺序即索引**——这就是「Hooks 必须顶层调用、不能条件调用」的根源：第二次渲染时 React 按同样顺序遍历链表取状态，顺序变了就对错位。

## 2. useState 的更新流程

```tsx
setA(1)
// 1. 创建 update 对象 { action: 1, lane: 优先级 }
// 2. 挂到对应 hook 的 queue（环状链表）
// 3. 标记 fiber.lanes + 向上冒泡到根，调度渲染
// 4. 下次 render 时：从 baseState 开始，依次应用队列中的 update（函数式 = action 是函数）
// 5. 计算 new state → 写入 hook.memoizedState → 组件拿到新值
```

函数式更新 `setA(c => c + 1)`：action 存函数，应用时执行——**这就是队列机制让连续 setState 正确叠加的原因**（不依赖当次渲染快照）。

批处理：多个 update 在队列中攒着，一次 render 全部应用（自动批处理 = 多次 dispatch 排队一次渲染）。

## 3. 渲染期间 Hooks 的执行

组件函数执行时：

```
renderWithHooks(fiber)：
  - 判断 mount（无 alternate）还是 update
  - mount：每个 Hook 调用创建 hook 节点挂链表（mountState / mountEffect）
  - update：按顺序遍历已有链表（updateState = 从队列计算新状态）
  - 「当前 hook」指针（currentlyRenderingFiber + workInProgressHook）在每次 Hook API 调用时推进
```

所以 `useState(0)` 的参数在 update 渲染中被忽略——初始值只在 mount 时建链表用。

## 4. useEffect 实现

```ts
// mount：创建 effect 对象 { create, destroy, deps } 挂 fiber.updateQueue
// update：比较 deps（Object.is 逐项）：
//   不变 → 复用（不执行）
//   变化/mount → 标记 HasEffect，commit 后执行：
//     先跑上次 destroy（旧闭包）→ 再跑新 create（新闭包）
// unmount：按队列执行所有 destroy
```

passive effects 的异步执行：commit 完成后由调度器在绘制后 flush——这解释了 effect 里读到的是「已提交 DOM」+「当次渲染闭包」。

## 5. useRef / useMemo / useCallback 实现

| Hook | 本质 |
|---|---|
| useRef | `memoizedState = { current: initial }`——跨渲染持久的盒子（不参与比较，改动不触发渲染） |
| useMemo | `memoizedState = [value, deps]`；render 时比较 deps，变了重算 |
| useCallback | `useMemo(() => fn, deps)` 的别名 |
| useContext | `readContext`：订阅当前 Context 的 fiber 标记 |
| useReducer | 同 useState 队列，action 走 reducer 计算 |
| useTransition | 内部 state + startTransition 标记 lane |
| useSyncExternalStore | 订阅 + getSnapshot 一致性检查（防 tearing） |

useMemo 的语义注意：**它是缓存不是保证**——React 可能在并发场景丢弃缓存重算，所以 useMemo 内不应有副作用。

## 6. Rules of Hooks 的完整解释

1. 只能在组件/Hook 顶层调用 → 链表顺序依赖
2. 只能在函数组件/自定义 Hook 中调用 → 需要当前渲染的 Fiber 上下文（currentlyRenderingFiber），普通函数没有
3. 循环/条件/嵌套函数中调用 → 顺序漂移状态错乱

ESLint `react-hooks/rules-of-hooks` 是静态检查这条链表不变量的守门员。

## 7. tearing 与 useSyncExternalStore

并发渲染期间，不同 Fiber 子树可能在不同时间片读取外部 store——**同一个状态读到不同值**（tearing 撕裂）。useSyncExternalStore 通过快照一致性检查解决：

```
render 中读 getSnapshot() → commit 前再检查 → 不一致则同步重渲染
```

这就是状态库（Zustand/Jotai）绑定它的原因。

## 8. React Compiler 的视角

Compiler 编译期自动插入等价 useMemo/useCallback——能做的前提依然是：
- 渲染纯（不改输入、无副作用）
- Hooks 规则遵守

理解原理后你会发现：**Compiler 不是让你放弃不可变与纯函数，而是替你写记忆化代码**。

## 常见坑（原理层）

| 现象 | 原理 |
|---|---|
| 条件调用 Hook 崩溃/错乱 | 链表顺序索引 |
| 初始值参数每次都执行 | 无惰性包裹；惰性初始化用函数 |
| useMemo 结果偶尔「被重算」 | 并发丢弃 + 缓存非保证 |
| setState 后同步读旧值 | update 队列在下一次 render 应用 |
| effect 闭包旧值 | effect 存的是创建时的闭包快照 |

## Checklist

- [ ] 能画出 memoizedState 链表结构
- [ ] 能解释 update 队列与批处理
- [ ] 能用链表顺序解释 Rules of Hooks
- [ ] 理解 effect 的 deps 比较与 destroy/create 时序
- [ ] 知道 tearing 与 useSyncExternalStore 的关系
