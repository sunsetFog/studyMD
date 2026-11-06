# 38 - 反模式清单

> 按「出现频率 × 危害」排序，每条：症状 → 修复。

## A. 状态类

### A1. 原地修改 state
```tsx
user.name = 'x'; arr.push(i)   // ❌ 引用未变不渲染 / diff 错乱
setUser({...user, name: 'x'}); setArr([...arr, i])   // ✅
```

### A2. 连续 setState 用旧快照
```tsx
setCount(count + 1); setCount(count + 1)   // ❌ 只 +1
setCount(c => c + 1); setCount(c => c + 1) // ✅
```

### A3. 镜像 props 进 state
```tsx
const [name, setName] = useState(props.name)   // ❌ 同步地狱
// ✅ 渲染期计算 / key 重置 / 完全受控
```

### A4. 能算的存 state（冗余态）
`total` 从 price/qty 派生却单独存 → 两处数据源漂移；渲染期计算。

### A5. 布尔组合表达互斥状态
`loading/error/data` 三布尔出非法态；联合类型 Phase。

## B. Effect 类

### B1. effect 做内部状态联动
```tsx
useEffect(() => setFullName(first + last), [first, last])   // ❌ effect 链
const fullName = `${first} ${last}`                          // ✅ 渲染期计算
```

### B2. 依赖撒谎 / 屏蔽 exhaustive-deps
读旧闭包诡异 bug；诚实声明 + 改结构收敛依赖。

### B3. effect 里发请求无竞态保护
快速切换 id 旧响应覆盖新数据；active 标记 / Abort / Query。

### B4. 忘 cleanup
订阅/定时器/Abort 泄漏。

### B5. `[]` 塞了读响应式值的「只跑一次」逻辑
StrictMode 双执行 + 旧闭包；诚实依赖或 Query。

## C. 组件与 JSX 类

### C1. v 系列不适用但等价错误：条件渲染渲染出 0
`{count && <Badge/>}` count=0 渲染 "0"；`{!!count && …}` 或三元。

### C2. 列表 key 用 index / 随机数
增删排序状态串位；稳定 id。

### C3. 动态组件小写变量
`const modal = ...; <modal />` 渲染未知标签；`const Modal = ...; <Modal />`。

### C4. 巨型组件 800+ 行
拆子组件 / 自定义 Hook / 状态下沉。

### C5. 渲染期副作用
组件函数里发请求/改全局变量/ref 读写；事件回调或 effect。

## D. 性能类

### D1. 无 profile 滥用 memo/useCallback/useMemo
负优化；Profiler 驱动。

### D2. 状态放顶层导致全树重渲染
状态下沉优先于 memo。

### D3. Context 放高频状态
输入/动画卡顿；组件 state / Zustand。

### D4. memo 子组件 props 引用不稳定
memo 白包；三件套配套或状态下沉。

### D5. 大列表无虚拟滚动
万级节点；react-virtual。

## E. 请求与状态库类

### E1. 手写 useEffect 管服务端数据
无缓存/失效/去重；TanStack Query。

### E2. queryKey 漏输入变量
不同筛选命中同一缓存；key 工厂。

### E3. 写后忘 invalidate
界面陈旧；mutation onSuccess 失效闭环。

### E4. Redux selector 返回新对象
重渲染风暴；createSelector。

### E5. Context 消费者未 useMemo value
value 每渲染新引用；useMemo。

## F. 工程类

### F1. react-hooks lint 未开 / 被屏蔽
Hooks 规则裸奔；error 级开启。

### F2. CI 不跑 tsc --noEmit
类型错误上线。

### F3. 组件库 react 未 external
双实例 Invalid hook call。

### F4. dangerouslySetInnerHTML 未消毒
XSS；DOMPurify 白名单。

### F5. 错误只有 console
三路捕获 + 上报 + 边界。

### F6. token 进 localStorage
内存 + refresh。

## 使用方式

- Code Review 按分区扫描
- ESLint 能拦截的（hooks 规则/依赖/key/tsx 类型）配好规则兜底
- 新人入职第一周通读
