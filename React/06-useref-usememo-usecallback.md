# 06 - useRef / useMemo / useCallback

## 1. useRef：跨渲染的可变盒子

```tsx
const ref = useRef<T>(initial)

ref.current        // 读写，修改不触发重渲染
```

三大用途：

### 1.1 DOM 引用

```tsx
function TextInput() {
  const inputRef = useRef<HTMLInputElement>(null)
  return (
    <>
      <input ref={inputRef} />
      <button onClick={() => inputRef.current?.focus()}>聚焦</button>
    </>
  )
}
```

规则：渲染期间**不读写** ref.current（除初始化）；读写在事件回调/effect 中。

### 1.2 存储不触发渲染的可变值（定时器 id、上一次值、实例）

```tsx
const timerRef = useRef<ReturnType<typeof setInterval>>()
useEffect(() => {
  timerRef.current = setInterval(tick, 1000)
  return () => clearInterval(timerRef.current!)
}, [])
```

### 1.3 前值缓存

```tsx
const prevRef = useRef<number>()
useEffect(() => {
  prevRef.current = count        // 在 effect 里更新，供下次渲染对比
})
```

ref vs state 选型：**「影响 UI 的值用 state，不影响的用 ref」**。判断法：这个值变了需要界面更新吗？

## 2. useRef 的快照陷阱

```tsx
function handleLog() {
  console.log(ref.current)   // 读到的是最新值（ref 不受渲染快照隔离）
  console.log(count)         // state 是当次渲染快照
}
```

ref 永远指向最新 current；state 是快照。两者语义完全不同，混用是常见 bug 来源。

## 3. useMemo：缓存计算结果

```tsx
const filtered = useMemo(
  () => bigList.filter(i => i.kw === kw),   // 昂贵计算
  [bigList, kw],
)
```

两大用途：
1. **跳过昂贵计算**（万级列表过滤、大对象变换）
2. **稳定引用**（见 §5 memo 配合）

原则：**默认不 memo，profile 出热点再加**；已在 memo 边界的才考虑引用稳定化。

## 4. useCallback：缓存函数引用

```tsx
const handleSubmit = useCallback((v: string) => {
  api.submit(v)
}, [])
```

本质是 `useMemo(() => fn, deps)` 的语法糖。唯一目的：**给 memo 子组件/依赖数组提供稳定引用**。

```tsx
// ❌ 每次 Lesson 渲染都新建 videoPlayerProps → useMemo 失效
const videoPlayerProps = useMemo(() => ({ ...commonProps, onClick: () => play() }), [commonProps])

// ✅ 依赖里全是稳定引用才有效
const onClick = useCallback(() => play(), [play])
const videoPlayerProps = useMemo(() => ({ ...commonProps, onClick }), [commonProps, onClick])
```

## 5. memo：跳过子组件重渲染

```tsx
import { memo } from 'react'

const Row = memo(function Row({ item, onSelect }: RowProps) {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>
})

// 自定义比较（慎用）
memo(Row, (prev, next) => prev.item.id === next.item.id)
```

memo 生效前提：**所有传入 props 引用稳定**——所以 memo、useCallback、useMemo 是三件套：

```
父组件 memo 化的列表项
  item —— 数据本身来自 state（引用稳定 ✅）
  onSelect —— useCallback 稳定 ✅
  否则 memo 白包
```

React 19：`memo` 仍有效；编译器方向（React Compiler）将自动做 memoization（见 `10-react-18-19.md`）。

## 6. 三者的联合决策树

```
需要 DOM/跨渲染值？            → useRef
昂贵计算且输入少变？           → useMemo
传给 memo 子组件的回调？       → useCallback
纯展示大列表的行组件？         → memo + 上两条保证 props 稳定
其他情况？                     → 什么都不用（YAGNI）
```

## 7. useRef 与受控/非受控

```tsx
// 非受控：DOM 自己持值，React 用 ref 取
const inputRef = useRef<HTMLInputElement>(null)
<form onSubmit={() => submit(inputRef.current!.value)}>
  <input defaultValue="初始值" />
</form>
```

非受控适用：文件上传、一次性表单、集成第三方 DOM 库。常规表单受控（见 `32-forms.md`）。

## 8. forwardRef 与 useImperativeHandle

```tsx
// React 19：ref 可直接作为 prop，无需 forwardRef
function MyInput({ ref, ...props }: { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />
}

// 暴露命令式 API（React 18 及以前）
useImperativeHandle(ref, () => ({
  focus: () => inputRef.current?.focus(),
  clear: () => { inputRef.current!.value = '' },
}))
```

原则：优先 props/回调（声明式），ref 命令式仅用于 focus/播放器等 DOM 能力。

## 常见坑

1. **渲染期间读写 ref.current**：违反纯函数，行为不可预测。
2. **useCallback/useMemo 依赖不诚实**：缓存了旧闭包，读到旧 state。
3. **memo 包了但 props 引用不稳**：无效 memo 白白增加比较成本。
4. **useMemo 存 DOM 节点/组件实例**：React 不保证保留，可能丢弃。
5. **用 ref 存「应该触发 UI 的值」**：改了不渲染，界面陈旧。
6. **useImperativeHandle 暴露整个内部**：最小化命令式 API。

## Checklist

- [ ] ref 只在事件/effect 中读写
- [ ] memo/useCallback/useMemo 按 profile 结果添加，不滥用
- [ ] memo 子组件的 props 全部引用稳定
- [ ] 依赖数组诚实
- [ ] 命令式 API 最小暴露
- [ ] 明确 ref（最新值）与 state（快照）语义差异
