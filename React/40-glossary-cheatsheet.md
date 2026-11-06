# 40 - 术语表与速查卡

## 1. 术语表

| 术语 | 含义 |
|---|---|
| JSX | JS 语法扩展，编译为 createElement/jsx 调用（虚拟 DOM） |
| Fiber | React 协调引擎；组件对应的链表节点，可中断渲染单元 |
| 协调（Reconciliation） | 新旧树的 diff 过程 |
| 双缓冲 | current / workInProgress 两棵树 + alternate 复用 |
| Lanes | 更新优先级赛道 |
| Commit | 同步应用 DOM 变更的阶段 |
| 快照（Snapshot） | 每次渲染捕获的 props/state 闭包 |
| 受控 / 非受控 | React state 作数据源 / DOM 自持值 |
| 合成事件 | React 包装的跨浏览器事件对象 |
| Portal | DOM 渲染到组件树外容器 |
| ErrorBoundary | 捕获子树渲染错误的 class 组件 |
| Suspense | 声明式 loading 边界（lazy/use(promise)） |
| RSC | React Server Components，服务端渲染组件不进 bundle |
| Server Action | Next.js 服务端函数（form action） |
| Tearing | 并发下外部 store 状态读不一致 |
| Transition | 标记低优先级可中断更新 |
| 自动批处理 | 多次 setState 合并一次渲染 |
| useMemo/useCallback | 缓存值/函数引用 |
| custom Hook | use 开头的可复用有状态逻辑函数 |
| Query Key | TanStack Query 的缓存键（含全部输入） |
| staleTime / gcTime | 新鲜窗口 / 无订阅缓存保留 |
| invalidate | 使缓存失效触发重取 |
| 乐观更新 | 先改 UI 后等响应，失败回滚 |
| storeToRefs 等价 | Zustand useShallow / Redux createSelector |
| prop drilling | props 逐层透传（Context/状态库解法） |
| compound components | 复合组件（Context 共享隐式状态） |
| render props | 函数 children 注入渲染逻辑（多被 Hook 替代） |
| CVA | class-variance-authority，样式变体管理 |

## 2. Hooks 速查

```
useState(initial|fn)          状态（函数式更新 setX(c => …)）
useReducer(reducer, init)     复杂状态逻辑
useEffect(fn, deps)           外部同步（cleanup）
useLayoutEffect               绘制前同步（防闪烁测量）
useMemo(fn, deps)             缓存计算
useCallback(fn, deps)         缓存函数引用
useRef(initial)               跨渲染可变盒/DOM
useContext(Ctx)               读 Context
useId()                       SSR 安全唯一 id
useTransition()               [isPending, startTransition]
useDeferredValue(value)       延迟值
useSyncExternalStore(sub, get, getServer)  外部 store 订阅
use(promise|context)          19：读 Promise/Context（条件可用）
useActionState(fn, init)      19：表单 action 状态
useOptimistic(state, fn)      19：乐观更新
useFormStatus()               19：form 内 pending（react-dom）
```

## 3. 事件类型速查

```
onClick: React.MouseEvent<HTMLButtonElement>
onChange: React.ChangeEvent<HTMLInputElement>
onSubmit: React.FormEvent<HTMLFormElement>
onKeyDown: React.KeyboardEvent<HTMLInputElement>
onFocus/Blur: React.FocusEvent<HTMLInputElement>
useImperativeHandle / ComponentRef<typeof X>
```

## 4. 常用模式速记

```tsx
// 函数式更新
setList(l => [...l, item])

// 竞态保护
useEffect(() => { let ok = true; api().then(r => ok && set(r)); return () => { ok = false } }, [id])

// 状态机
type P = { s: 'idle' } | { s: 'ok'; d: T } | { s: 'err' }
const [p, setP] = useState<P>({ s: 'idle' })

// 稳定引用
const onSave = useCallback((v) => dispatch({ type: 'save', v }), [dispatch])
const Row = memo(RowInner)

// Query 闭环
const qc = useQueryClient()
useMutation({ mutationFn: api.x, onSuccess: () => qc.invalidateQueries({ queryKey: keys.all }) })

// key 重置状态
<ProfileForm key={userId} userId={userId} />

// 双模式组件
const isControlled = value !== undefined

// 动态组件
const Comp = map[type]; return <Comp {...props} />
```

## 5. 一页速查：决策口诀

```
状态放哪？  刷新要留 → URL searchParams
            服务端数据 → useQuery
            多页共享 → Zustand（高频）/ Context（低频）
            否则 → useState

需要哪个 Hook？  派生值 useMemo（或渲染期算）
                 外部同步 useEffect
                 DOM/可变值 useRef
                 联动复杂 useReducer
                 慢更新 useTransition / useDeferredValue

memo 三问？  子组件渲染贵吗？→ 不贵不用
             props 引用稳定吗？→ 不稳先修
             测出瓶颈了吗？→ 没测不加

错误处理？  渲染崩 → ErrorBoundary
            异步崩 → showBoundary / try-catch
            请求崩 → Query error 态 + ApiError
```

## 6. 高频坑位 TOP10 速记

1. 原地修改 state（push/直接赋值）
2. effect 依赖撒谎/忘 cleanup
3. effect 做内部状态联动（effect 链）
4. 条件渲染渲染出 0（`count &&`）
5. key 用 index
6. setState 后同步读旧值
7. 渲染期副作用（StrictMode 双执行翻车）
8. Context 放高频状态
9. 手写 effect 管服务端数据（无缓存失效）
10. memo 白包（props 引用不稳）

---

全系列完（00-40）。落地用 `39-best-practices.md`，面试看 `35/36`，跨框架看 `37` 与同仓库 `vue3/`。
