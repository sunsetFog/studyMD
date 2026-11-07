# 08 - React 进阶：Hooks / Fiber / 状态管理

## 1. Hooks 全家（必考）

```tsx
// useState/useReducer：本地状态
const [state, dispatch] = useReducer(reducer, init);

// useEffect：副作用（挂载/更新/卸载模型）
useEffect(() => {
  const t = setInterval(tick, 1000);
  return () => clearInterval(t);        // 清理函数
}, []);                                  // 依赖数组

// 其他核心
useMemo(() => heavy(a), [a]);            // 缓存计算结果
useCallback((x) => f(x), [f]);           // 缓存函数引用（传给 memo 子组件）
useRef<HTMLDivElement>(null);            // DOM 引用 / 可变值（不触发渲染）
useLayoutEffect(() => {...}, []);        // DOM 变更后、绘制前（测量布局用，阻塞绘制）
useId();                                 // SSR 安全的唯一 id
useTransition() / useDeferredValue();    // 并发特性：低优先级更新
```

- useEffect vs useLayoutEffect：默认 useEffect（异步、不阻塞）；测量 DOM 布局防闪烁用 useLayoutEffect
- useEffect 模拟生命周期：`[]` → mounted；依赖变化 → updated；return 清理 → unmounted。**面试陷阱：这不是生命周期，是依赖同步**

## 2. Hooks 原理（进阶高频）

- Hooks 存储在 Fiber 节点的 `memoizedState` 链表上，按调用顺序索引 → **不能放条件/循环里**（顺序漂移）
- 为什么闭包旧值：每次渲染生成新的 props/state 快照，回调捕获当次快照
- 依赖数组：告诉 React "哪些外部值变了需要重跑 effect"；eslint-plugin-react-hooks exhaustive-deps 保平安
- `Object.is` 比较依赖；依赖是对象时每次新引用都会触发 → 配合 useMemo/useCallback

## 3. Fiber 架构（⭐⭐）

- Fiber = 虚拟 DOM 节点 + 工作单元；链表结构（child/sibling/return）支持**可中断**
- 双缓冲：current 树（屏幕）+ workInProgress 树（内存构建，完成后切换）
- 渲染分两阶段：
  ```
  Render（可中断）：beginWork/completeWork 构建 WIP，diff 出副作用列表
  Commit（不可中断）：commitWork 执行 DOM 变更 + 生命周期/effect
  ```
- 调度：时间切片（time slicing，每帧 5ms）+ 优先级 lanes；concurrent 特性（transition）允许被打断重跑
- 价值：长任务不再阻塞交互与渲染

## 4. 重渲染优化（必考⭐）

```tsx
// 三层拦截
const Child = React.memo(({ user, onSelect }) => {...});   // props 浅比较
const onSelect = useCallback((id) => pick(id), []);        // 引用稳定
const sorted = useMemo(() => sort(users), [users]);        // 引用稳定

// Context 拆分：Provider 的 value 用 useMemo 包裹
```

React 19 相关：`React Compiler`（自动 memo 化）正在改变心智——面试可提"编译期自动优化，手写 memo 逐步减少"。

- 何时优化：先用 React DevTools Profiler 定位再优化，**盲加 memo 是反模式**

## 5. 状态管理选型

| 库 | 模型 | 适用 |
|---|---|---|
| Zustand | 单 store、hook 订阅选择器 | 中小项目主流 |
| Redux Toolkit | 单向数据流、devtools 强 | 大型复杂状态/团队规范 |
| Jotai/Recoil | 原子化状态 | 细粒度依赖 |
| TanStack Query | **服务端状态**（缓存/重试/失效） | 所有接口数据——现代必答 |

关键认知：**服务端状态与客户端状态分离**——接口数据交给 TanStack Query/SWR，本地 UI 状态用 Zustand，不再手写 loading/error 缓存逻辑。

## 6. React 18/19 特性速览

- 18：并发渲染、automatic batching、Suspense for data、`startTransition`
- 19：Actions（表单 `useActionState`）、`use()` API、ref 作为 prop、文档元数据、编译器

## 常见追问

1. useEffect 第二个参数省略/[]/有值分别什么行为？
2. 为什么 memo 后组件还重渲染？——非 memo 的子元素、context 变化、props 引用不稳
3. Fiber 之前为什么不能中断？——递归同步调和，栈不可暂停；Fiber 链表 + 自己管栈
4. useMemo/useCallback 滥用的成本？——记忆化本身有比较与存储开销

## Checklist

- [ ] Fiber 两阶段与可中断原理能画图讲
- [ ] Hooks 链表顺序约束说得清
- [ ] 有 Profiler 定位再优化的意识
- [ ] 服务端状态分离是加分项
