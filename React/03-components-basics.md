# 03 - 组件基础：props 与 state

## 1. 组件定义

```tsx
// 函数组件（唯一推荐写法）
type Props = { title: string; count?: number }

export default function UserCard({ title, count = 0 }: Props) {
  return <div>{title} - {count}</div>
}

// 组件命名：PascalCase，文件同名
```

组件是**纯函数**：相同 props/state 输入，相同 JSX 输出，无副作用（副作用进 useEffect/事件处理）。

## 2. props：只读的单向数据流

```
父 → props → 子（只读）
子 → 回调函数 → 父
```

```tsx
// 父
<UserCard title="张三" onUpdate={() => …} />

// 子：props 不可修改
function UserCard(props: Props) {
  // props.title = 'x'  // ❌ TypeScript 报错（readonly）
}
```

铁律（与 Vue 单向流同源）：
- props 只读；需要「改」就通过回调请父层改
- 默认值用解构默认值（`count = 0`）而非 defaultProps（已废弃）

## 3. children 与插槽等价物

```tsx
// React 的「插槽」就是 children prop + 函数 children
<Card footer={<Button>确定</Button>}>        {/* 具名：传 JSX prop */}
  默认内容                                    {/* 默认：children */}
</Card>

function Card({ children, footer }: { children: ReactNode; footer?: ReactNode }) {
  return <div>{children}{footer && <div className="footer">{footer}</div>}</div>
}
```

## 4. state：组件私有可变数据

```tsx
const [count, setCount] = useState(0)
```

state 更新规则（不可变更新）：

```tsx
setCount(c => c + 1)                  // ✅ 函数式更新（依赖旧值必用）
setUser({ ...user, name: 'b' })       // ✅ 对象：展开新对象
setList([...list, item])              // ✅ 数组：新增
setList(list.filter(i => i.id !== id))// ✅ 删除
setList(list.map(i => i.id === id ? { ...i, done: true } : i))  // ✅ 更新项
user.name = 'b'                       // ❌ 直接改不触发渲染（引用没变）
```

关键：**state 必须替换引用**，React 用 `Object.is` 比较决定是否重渲染。

## 5. setState 的批处理与异步性

```tsx
function handleClick() {
  setCount(count + 1)   // ❌ 两次都基于同一个旧快照 → 只 +1
  setCount(count + 1)
}

function handleClick() {
  setCount(c => c + 1)  // ✅ 函数式：队列依次基于最新值
  setCount(c => c + 1)  // 结果 +2
}
```

- React 18+ 自动批处理：事件回调、promise、setTimeout 中多次 setState 合并为一次渲染
- setState 后立即读 state 拿到的是**旧值**（当次快照）；需要「更新后做什么」进 useEffect 或回调

## 6. 状态该放哪：状态归属判定

```
1. 列出使用该状态的所有组件
2. 找它们的最近公共父组件
3. 状态放父组件，通过 props 下发，回调上报
   ↓ 复杂层级
4. 跨层级/全局 → Context / 状态库（见 15）
5. 服务端数据 → 不放组件 state，进 TanStack Query（见 19）
```

**状态提升**（lifting state up）是 React 解决兄弟通信的唯一正途（或状态库）。

## 7. props 展开与透传

```tsx
<Button {...restProps} type="primary">   // {...props} 展开透传
```

注意：展开顺序决定覆盖关系；自定义组件透传给原生元素时注意无效属性警告（见 `13-component-patterns.md`）。

## 8. 组件纯函数纪律

以下操作禁止在渲染期间执行：

```
❌ 直接修改外部变量 / props / 上一次渲染创建的对象
❌ Math.random() / Date.now() 直接用于渲染输出（不一致输出）
❌ 发请求、订阅事件（这些放 useEffect / 事件回调）
```

需要「不稳定值」时放事件回调或 useEffect；需要随机初始值时 `useState(() => Math.random())` 惰性初始化。

## 常见坑

1. **直接修改 state 对象/数组**：引用不变 → 不渲染，或渲染了但 diff 错乱。
2. **连续 setState 用旧值**：必须函数式更新。
3. **依赖 props 的衍生值存进 state**：props 变了 state 不跟着变——用渲染期计算（见 05「不需要 effect 的场景」）。
4. **子组件修改 props.children 引用**：children 是父的快照。
5. **渲染期副作用**（发请求、改全局变量）：StrictMode 下双执行，产生重复请求。

## Checklist

- [ ] state 更新全部不可变 + 函数式（依赖旧值时）
- [ ] 组件渲染期无副作用
- [ ] 状态归属经公共父组件判定，未滥用 Context/全局库
- [ ] props 默认值用解构默认值
- [ ] 无「props 拷贝进 state」的冗余状态
