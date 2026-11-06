# 13 - 组件设计模式

## 1. 模式总览

| 模式 | 解决什么 |
|---|---|
| 组合 vs 继承 | React 唯一推荐「组合」 |
| 容器/展示分离 | 数据与视图解耦 |
| children/插槽模式 | 结构注入 |
| render props | 逻辑注入（现代多被 Hook 替代） |
| 受控/非受控双模式 | 组件灵活性 |
| HOC（历史） | 增强组件（现已边缘化） |
| Compound Components | 多部件共享隐式状态 |
| Variants（CVA 思路） | 样式变体管理 |

## 2. 组合优于继承（官方第一原则）

```tsx
// ❌ 继承思维：BaseList extends
// ✅ 组合：组件通过 props/children 拼装
<DataList
  items={users}
  renderItem={(u) => <UserRow user={u} />}
  empty={<EmptyState />}
  footer={<Pagination total={100} />}
/>
```

用「配置 + 插槽位」表达所有变体，而不是造一个巨型 Base 组件。

## 3. 容器 / 展示分离（现代版）

```
容器（页面级）：取数（Query/hook）、编排状态 —— 少样式
展示组件：纯 props 进、回调出 —— 不发请求，可测可复用
```

现代演进：容器逻辑收进自定义 Hook（`useUserListPage`），页面 JSX 纯粹化。展示组件配合 memo 做性能边界。

## 4. children 与函数 children

```tsx
// 结构注入
<Modal title="确认">{content}</Modal>

// 函数 children（逻辑注入：把内部状态暴露给调用方）
<MouseTracker>
  {({ x, y }) => <span>{x}, {y}</span>}
</MouseTracker>

function MouseTracker({ children }: { children: (pos: Pos) => ReactNode }) {
  const pos = useMouse()
  return <div>{children(pos)}</div>
}
```

函数 children 在 React 中多被 Hook 替代（`const pos = useMouse()` 直接在调用组件里用），保留场景：**逻辑与渲染结构强绑定**（如 Fetcher 组件、Tab 激活态传递）。

## 5. 受控 / 非受控双模式组件

```tsx
type InputProps = {
  value?: string                    // 受控值
  defaultValue?: string             // 非受控初值
  onChange?: (v: string) => void
}

function AutoInput({ value, defaultValue, onChange }: InputProps) {
  const [inner, setInner] = useState(defaultValue ?? '')
  const isControlled = value !== undefined
  const current = isControlled ? value : inner

  return (
    <input
      value={current}
      onChange={e => {
        if (!isControlled) setInner(e.target.value)
        onChange?.(e.target.value)
      }}
    />
  )
}
```

成熟组件库（Dialog、Select）的标准能力：调用方可选接管状态。

## 6. Compound Components（复合组件）

```tsx
<Tabs defaultActive="a">
  <Tabs.List>
    <Tabs.Tab id="a">一</Tabs.Tab>
    <Tabs.Tab id="b">二</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel id="a">内容一</Tabs.Panel>
  <Tabs.Panel id="b">内容二</Tabs.Panel>
</Tabs>
```

实现核心：**隐式状态通过 Context 共享给子部件**：

```tsx
const TabsCtx = createContext<TabsCtxValue | null>(null)

export function Tabs({ defaultActive, children }: TabsProps) {
  const [active, setActive] = useState(defaultActive)
  const ctx = useMemo(() => ({ active, setActive }), [active])
  return <TabsCtx.Provider value={ctx}>{children}</TabsCtx.Provider>
}

Tabs.Tab = function Tab({ id, children }: TabProps) {
  const ctx = useContext(TabsCtx)!
  return (
    <button
      className={ctx.active === id ? 'tab tab--active' : 'tab'}
      onClick={() => ctx.setActive(id)}
    >{children}</button>
  )
}
```

适用：语义相关联的多部件组件（Tabs/Accordion/Menu/Table 列定义）。子部件挂主组件静态属性（`Tabs.Tab = …`），配合 TS 命名空间类型。

## 7. render props（了解即可）

```tsx
<DataFetcher url="/api/users">
  {({ data, loading, error }) => loading ? <Spinner /> : <List data={data!} />}
</DataFetcher>
```

Hook 时代绝大多数 render props 应改为自定义 Hook（`09-custom-hooks.md`）；保留于需要「每次挂载独立请求实例 + 模板绑定」的场景。

## 8. HOC（历史模式，新代码禁用）

```tsx
// 增强组件：withAuth(Component)
// 问题：包装地狱、props 来源不明、TS 泛型推导复杂、ref 转发繁琐
```

替换方案映射：数据增强 → Hook/Query；权限增强 → Hook + 组件；重渲染优化 → memo。

## 9. 变体样式模式（CVA + tailwind）

```tsx
import { cva, type VariantProps } from 'class-variance-authority'

const button = cva('btn', {
  variants: {
    intent: { primary: 'btn--primary', danger: 'btn--danger' },
    size: { sm: 'btn--sm', md: 'btn--md' },
  },
  defaultVariants: { intent: 'primary', size: 'md' },
})

type ButtonProps = React.ButtonHTMLAttributes<HTMLButtonElement> & VariantProps<typeof button>

export function Button({ intent, size, className, ...rest }: ButtonProps) {
  return <button className={cn(button({ intent, size }), className)} {...rest} />
}
```

变体类型自动推导 + 透传原生属性（`...rest` 透传到 button，组件 API 与原生一致）。

## 10. 组件 API 设计原则

| 原则 | 说明 |
|---|---|
| props 最小化 | 每多一个 prop 是一份契约；布尔开关 >3 个考虑插槽/变体 |
| 透传原生属性 | `{...rest}` 支持 onClick/className/style |
| 受控可选 | value/onChange 可选，非受控兜底 |
| 一个概念一个 prop | 不造 `mode: 'a'|'b'|'c'` 组合爆炸 |
| 数据下行、事件上行 | 命名 `onXxx` |

## 常见坑

1. **巨型 Base 组件 + 布尔 prop 开关爆炸**：改组合/变体。
2. **HOC 链 + 类型丢失**：迁移到 Hook。
3. **Compound 忘 Context null 断言**：子部件独立使用时崩溃提示不清。
4. **函数 children 每次渲染新函数**：子树 diff 成本；注意 memo 边界。
5. **受控组件内部存副本不同步**：双状态源；见 §5 模式实现。
6. **透传 rest 时覆盖了关键 props**（onChange 被 rest 覆盖）：展开顺序注意。

## Checklist

- [ ] 无继承式组件，全部组合
- [ ] 展示组件不发请求
- [ ] 复用逻辑优先自定义 Hook
- [ ] 复合组件 Context 断言 + 类型命名空间
- [ ] 组件 props 原生透传（rest）+ 变体类型化
- [ ] 布尔 prop 不超 3 个
