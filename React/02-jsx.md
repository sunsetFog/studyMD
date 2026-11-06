# 02 - JSX 详解

## 1. JSX 是什么

JS 的语法扩展，编译为 `React.createElement` / `jsx()` 函数调用（返回 JS 对象——虚拟 DOM）。

```tsx
const el = <h1 className="title">Hello</h1>
// 编译产物（automatic runtime）
import { jsx as _jsx } from 'react/jsx-runtime'
const el = _jsx('h1', { className: 'title', children: 'Hello' })
```

本质：**JSX 是表达式，不是模板字符串**——可以在任何 JS 表达式位置使用。

## 2. 基本规则

| 规则 | 说明 |
|---|---|
| 必须单根 | 返回多个元素需包裹：Fragment `<>…</>` 或数组 |
| 大小写敏感 | 小写=原生标签，大写=组件 |
| className / htmlFor | 替代 class / for（避免 JS 关键字冲突） |
| 属性 camelCase | `strokeWidth`、`tabIndex`（data-*/aria-* 保留原样） |
| 自闭合 | `<img />` `<br />` 必须闭合 |
| {} 内是 JS 表达式 | 不能写语句（if/for），可用三元、`&&`、map |

## 3. 表达式与条件渲染

```tsx
// 文本插值
<h1>{user.name}</h1>

// 条件渲染
{isReady && <Panel />}                       // && 注意左值为 0 的问题
{status === 'loading' ? <Spinner /> : <List />}   // 三元最安全
{error ? <ErrorView /> : data ? <List /> : <Empty />}

// ⚠️ && 陷阱：count && <Badge />，count 为 0 时渲染出 "0"
{!!count && <Badge />}        // 正确
{count > 0 && <Badge />}      // 正确
```

`&&` 陷阱成因：JSX 渲染 falsy 值时 `false/null/undefined` 不渲染，但 **0 和 NaN 会渲染**。

## 4. 列表渲染

```tsx
{todos.map(todo => (
  <li key={todo.id}>{todo.text}</li>
))}
```

- key 必须稳定唯一，禁止 index（有增删排序时），与 Vue 同理（见 `29-diff.md`）
- map 返回数组作为 children，React 自动展开

## 5. 样式

```tsx
// className（字符串拼接）
<div className={`card ${isActive ? 'card--active' : ''}`}>

// clsx / classnames 库（推荐）
<div className={clsx('card', isActive && 'card--active', size === 'lg' && 'card--lg')}>

// style 内联对象（camelCase，数字自动补 px）
<div style={{ color: textColor, fontSize: 16, '--custom-var': val } as React.CSSProperties}>
```

## 6. 事件处理

```tsx
<button onClick={handleClick}>方法引用</button>
<button onClick={() => handleClick(id)}>带参（箭头包装）</button>
<button onClick={e => { e.stopPropagation(); onClick() }}>事件对象</button>
```

- 事件对象是合成事件（SyntheticEvent），React 17+ 不再做事件池
- `e.stopPropagation()` 阻冒泡、`e.preventDefault()` 阻默认行为
- React 17+ 事件委托挂在 root 容器（不再是 document）

## 7. Fragment

```tsx
<>…</>                            // 简写，无 key 能力
<Fragment key={id}>…</Fragment>   // 列表中需要 key 时用完整写法
```

不产生额外 DOM 节点——解决「必须单根」约束。

## 8. 组件即标签

```tsx
<UserCard user={user} />                    // 静态组件
const Comp = componentMap[type]             // 动态组件：变量引用
return <Comp {...props} />
<comp />                                    // ❌ 小写被当原生标签！
```

动态组件三要素：**变量名大写**（`const Comp`）、传 props、key 稳定。

## 9. children 与嵌套

```tsx
<Card>
  <Header />          {/* children 数组元素 */}
  文本内容
  {list.map(...)}     {/* 表达式子节点 */}
</Card>

// children 是特殊 prop，组件内通过 props.children 访问
function Card({ children }: { children: React.ReactNode }) { … }
```

## 10. JSX 中的注释与陷阱

```tsx
{/* 这是 JSX 注释 */}
<div
  // 属性间可用 JS 注释
  onClick={handler}
/>
```

| 陷阱 | 说明 |
|---|---|
| 条件渲染返回 `0` | `count && …` 数字 0 会被渲染 |
| 数组直接渲染对象 | `{[obj1, obj2]}` 报错 objects are not valid |
| 未闭合标签 | 编译报错，但 HTML 内嵌场景注意 `<br>` |
| key 放在 Fragment 简写上 | 简写 `<>` 不能接 key |
| style 字符串 | style 只接受对象 |

## 常见坑

1. **`&&` 渲染出 0/NaN**：条件渲染统一三元或显式布尔转换。
2. **className 写成 class**：React 会警告且样式失效。
3. **动态组件小写变量**：`const modal = …; <modal />` 渲染成未知标签。
4. **内联 style 字符串**（`style="color:red"`）：必须对象。
5. **在 JSX 中写 if 语句**：{} 只允许表达式，用三元/IIFE/提前 return。
6. **map 忘 key**：控制台警告 + 更新错乱。

## Checklist

- [ ] 条件渲染统一安全写法（三元 / `!!x &&`）
- [ ] 所有 map 列表有稳定 key
- [ ] 动态组件变量大写
- [ ] 样式类拼接用 clsx
- [ ] 无未闭合标签与字符串 style
