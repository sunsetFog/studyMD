# 35 - 面试题精讲（一）：基础与 Hooks

> 每题：考察点 → 参考答案（要点式）→ 加分项。

## Q1：函数组件与 class 组件的区别？

**考察点**：心智模型。

**要点**：函数组件捕获渲染快照（props/state 是当次闭包），class 组件通过 this 读最新值（this 可变）；Hooks 无生命周期心智、逻辑可组合复用；函数组件是唯一官方发展方向。

**加分**：举快照例子（setTimeout 里读 state 的差异）；说出「渲染即纯函数」。

## Q2：useState 的更新机制？为什么连续 setState 只加一次/加两次？

**要点**：
- setState 触发的是 update 入队 + 调度渲染，不是立即赋值
- `setCount(count + 1)` 两次基于同一快照 → +1；`setCount(c => c + 1)` 队列依次应用 → +2
- React 18 自动批处理：事件/异步回调中多次 setState 合并一次渲染

**加分**：update 队列与 fiber.memoizedState 的关系（见 30）。

## Q3：useEffect 的执行时机与 cleanup 语义？

**要点**：
- commit 后异步执行（绘制后）；依赖变化先 cleanup（旧闭包）再 effect（新闭包）；卸载执行最后 cleanup
- 心智模型：与外部系统同步，不是生命周期
- StrictMode 开发期 mount→cleanup→mount 检测幂等性

**加分**：与 useLayoutEffect 时机差异；「不需要 effect 的场景」清单（渲染期计算/事件回调）。

## Q4：useMemo/useCallback/memo 的关系与使用原则？

**要点**：
- memo：props 浅比较跳过子组件渲染；useCallback/useMemo 提供稳定引用使其有效
- 三件套配套；默认不 memo，profile 驱动
- React Compiler 方向：编译期自动 memoization

**加分**：能说浅比较对对象/函数 props 失效的原理；引用「状态下沉优先」。

## Q5：ref 和 state 的区别？

**要点**：ref.current 跨渲染持久、改动不触发渲染、读写最新值；state 是快照、触发渲染。「影响 UI 用 state，不影响用 ref」。渲染期禁止读写 ref.current。

**加分**：ref 三用途（DOM/可变值/前值缓存）；非受控组件场景。

## Q6：useEffect 依赖数组的规则与常见错误？

**要点**：诚实声明所有响应式依赖（exhaustive-deps）；不想要的依赖靠改代码解决（移进 effect/提取/reducer）；对象/数组/函数依赖每次新引用导致每次执行。

**加分**：依赖撒谎的 bug 案例；用 useReducer 合并状态收敛依赖。

## Q7：useContext 的重渲染问题与解法？

**要点**：Provider value 变 → 子树全重渲染；解法：value useMemo、Context 拆分（高频/低频分箱）、memo 边界、换 Zustand（useSyncExternalStore 精确订阅）。Context 适合低频全局值。

**加分**：tearing 与 useSyncExternalStore。

## Q8：自定义 Hook 是什么？与普通函数/HOC/mixin 的区别？

**要点**：内部调用其他 Hook 的 use 函数；每次调用独立实例；React 复用有状态逻辑的唯一正规手段（替代 HOC/mixin）。只能在组件/Hook 顶层调用（链表顺序）。

**加分**：现场设计 useEventListener（最新回调 ref + cleanup）。

## Q9：key 的作用？为什么不能用 index？

**要点**：key 是 children diff 的身份标识；稳定 key 复用 + 最小移动；index 在增删/排序时身份错位 → 状态串位（非受控输入）。

**加分**：React 两轮 diff 策略；与 Vue3 LIS 对比（见 29/37）。

## Q10：受控与非受控组件？

**要点**：受控 = React state 是数据源（value+onChange）；非受控 = DOM 持有（defaultValue+ref/FormData）。受控支持即时校验联动；非受控性能轻、适合文件/第三方。

**加分**：双模式组件实现（value !== undefined 判断）；RHF 的非受控内核。

## Q11：useReducer 什么时候优于多个 useState？

**要点**：多字段联动更新（原子 action）、复杂前值依赖、状态机（非法态不可表示）、dispatch 引用稳定（免 useCallback 传 memo 子组件）。

**加分**：联合类型 FetchState 示例；reducer 纯函数可测性。

## Q12：useTransition / useDeferredValue 的区别？

**要点**：transition 包更新动作（低优可中断）；deferredValue 包值（外部来的）。场景：Tab 切换慢 → transition；搜索框 + 大列表 → deferredValue + useMemo。

**加分**：并发渲染语义（可中断/可丢弃）与纯函数的关系。

## Q13：为什么 React 组件首字母必须大写？

**要点**：JSX 编译按标识符大小写区分：大写 = 变量引用（组件），小写 = 字符串（原生标签）。动态组件必须大写变量引用。

## Q14：React 18 的自动批处理与 React 17 的差异？

**要点**：17 只批处理 React 事件内；18 全场景（promise/setTimeout/原生事件）；flushSync 退出批处理。

---

## 延伸练习

1. 手写 useDebounce（含 cleanup 与最新回调）
2. 解释 `useEffect(() => { fetch(x) }, [])` 中 x 变化不重新请求的问题与三种解法
3. 现场实现受控/非受控双模式 Input
