# 07 - React 核心

## 1. 组件基础

```tsx
// 函数组件（唯一主流形态，类组件已成历史遗产，面试只需了解）
type Props = { title: string; onOk?: () => void };
function Card({ title, onOk }: Props) {
  return <div onClick={onOk}>{title}</div>;
}
```

- JSX → `React.createElement`（或编译产物 `_jsx`）→ 虚拟 DOM 对象
- 组件纯函数约定：同样 props 输出同样 UI（StrictMode 双调用于暴露副作用）

## 2. 状态与 Props

- props 单向流动、不可变；状态提升（lifting state up）是跨组件共享的朴素方案
- `useState`：返回 `[state, setState]`；更新是**替换不是合并**（对象需展开）

```tsx
const [form, setForm] = useState({ a: 1, b: 2 });
setForm(prev => ({ ...prev, a: 2 }));   // 函数式更新：基于最新 state，避免闭包旧值
```

- 批量更新：React 18 起所有场景自动批处理（含 setTimeout/Promise 内）

## 3. 事件系统

- 合成事件（SyntheticEvent）：统一跨浏览器行为 + 事件委托到根容器
- React 17+ 委托到 root 容器而非 document（多版本共存安全）
- `e.stopPropagation`/`e.preventDefault` 语义一致
- 命名：onClick/onChange（注意 React 的 onChange≈input 事件）

## 4. 受控与非受控（必考）

```tsx
// 受控：表单值由 state 驱动
<input value={name} onChange={e => setName(e.target.value)} />
// 非受控：DOM 自己持有，需要时再取
const ref = useRef<HTMLInputElement>(null);
ref.current?.value
```

- 受控适合联动校验；非受控适合大表单/文件输入（受控 file 只读）

## 5. 条件渲染与列表

```tsx
{loading ? <Spin/> : <List/>}
{err && <Alert/>}
{users.map(u => <li key={u.id}>{u.name}</li>)}   // key 稳定唯一，禁 index
```

- Fragment：`<></>` 不产生多余 DOM
- Portal：`createPortal(node, el)` 弹层脱离父级

## 6. 组件通信

| 场景 | 方案 |
|---|---|
| 父→子 | props |
| 子→父 | 回调 props |
| 兄弟/远层 | 状态提升 → Context → 状态库 |
| 全局 | Zustand/Redux/Jotai |
| 修改子 DOM | forwardRef + useImperativeHandle |

Context 使用要点：value 变化则所有消费组件重渲染——大对象拆分 Context 或配合 memo。

## 7. 与 Vue 的心智差异（前置认知，详见 09）

- React：**不可变数据 + 显式 setState 驱动**，全量重渲染靠 reconciliation 调和
- Vue：**可变数据 + 依赖追踪**，精准知道谁变了
- 这决定了 Hooks 依赖数组（React）vs 自动依赖收集（Vue）的根本差异

## 常见追问

1. 为什么 key 不能用 index？——列表重排时元素复用错位，状态串行
2. setState 后立刻读到旧值？——闭包捕获的是当次渲染快照；函数式更新解决
3. 类组件时代考点（了解）：生命周期对应、this 绑定问题、HOC 与 render props 为何被 Hooks 取代

## Checklist

- [ ] 受控/非受控能选型
- [ ] 函数式更新解决闭包旧值能讲清
- [ ] 通信方案按场景列举无遗漏
