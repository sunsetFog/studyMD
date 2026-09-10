# React 知识点总结

## 1. React 简介

React 是 Facebook 开源的用于构建用户界面的 JavaScript 库，采用组件化开发和虚拟 DOM 机制，通过声明式编程高效更新界面。

特点：
- 组件化、声明式
- 虚拟 DOM，diff 算法高效更新
- 单向数据流
- JSX 语法
- 生态丰富（React Router、Redux、Next.js）

## 2. JSX 语法

JSX 是 JavaScript 的语法扩展，可以像写 HTML 一样写组件结构。

```
const name = 'world'
const element = <h1>Hello, {name}</h1>

// 内联样式，属性用驼峰命名
const style = { color: 'red', fontSize: '20px' }
const el = <div style={style}>内容</div>

// 条件渲染
const show = true
const node = show ? <p>显示</p> : null
```

注意：class 要写成 `className`，for 要写成 `htmlFor`。

## 3. 函数组件与类组件

函数组件是当前主流写法，配合 Hooks 实现状态和副作用；类组件是传统写法，通过 this 访问 props/state。

```
// 函数组件（推荐）：接收 props，返回 JSX
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>
}

// 类组件：继承 React.Component，用 render 返回 JSX
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>
  }
}
```

## 4. Props 父传子

父组件通过属性向子组件传值，props 是只读的。

```
function Child(props) {
  // props 是只读对象，包含父组件传入的所有属性
  return <p>{props.message}</p>
}

function Parent() {
  // 通过 JSX 属性向子组件传值
  return <Child message="你好" />
}
```

## 5. useState 状态管理

函数组件中用 useState 管理状态。

```
import { useState } from 'react'

function Counter() {
  // useState(0)：返回 [状态值, 更新函数]，0 是初始值
  const [count, setCount] = useState(0)

  return (
    <div>
      <p>{count}</p>
      {/* 点击时调用 setCount 更新状态，触发组件重新渲染 */}
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  )
}
```

## 6. useState 注意事项

setState 是异步的，更新函数式写法可避免闭包问题。

```
const [count, setCount] = useState(0)

// 基于旧值更新，用函数式
setCount(prev => prev + 1)
setCount(prev => prev + 1)   // 最终 +2

// 不要直接修改状态
const [list, setList] = useState([])
setList([...list, newItem])   // 正确，返回新数组
```

## 7. useEffect 副作用

在渲染后执行副作用（请求、订阅、DOM 操作），可用返回函数清理。

```
import { useEffect, useState } from 'react'

function App() {
  const [data, setData] = useState(null)

  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(setData)

    // 清理函数
    return () => {
      console.log('卸载或依赖变化时清理')
    }
  }, [])   // 空数组：仅在挂载时执行一次

  return <div>{data}</div>
}
```

## 8. useEffect 依赖数组

- 不传：每次渲染都执行
- 传空数组 []：仅挂载时执行一次
- 传依赖 [a, b]：依赖变化时执行

```
useEffect(() => {
  console.log('每次渲染执行')
})

useEffect(() => {
  console.log('仅挂载执行一次')
}, [])

useEffect(() => {
  console.log('count 变化时执行')
}, [count])
```

## 9. useMemo 记忆化值

缓存计算结果，依赖不变时直接返回缓存值。

```
import { useMemo, useState } from 'react'

function App() {
  const [count, setCount] = useState(0)
  const [name, setName] = useState('')

  const expensive = useMemo(() => {
    return count * 100   // 复杂计算
  }, [count])            // 仅 count 变化时重新计算

  return <div>{expensive}</div>
}
```

## 10. useCallback 记忆化函数

缓存函数引用，避免子组件不必要的重渲染。

```
import { useCallback, useState } from 'react'

function App() {
  const [count, setCount] = useState(0)

  const handleClick = useCallback(() => {
    setCount(c => c + 1)
  }, [])

  return <Child onClick={handleClick} />
}
```

## 11. useRef 引用

获取 DOM 元素，或保存可变值（不触发重渲染）。

```
import { useRef } from 'react'

function App() {
  const inputRef = useRef(null)
  const countRef = useRef(0)

  const focus = () => {
    inputRef.current.focus()
    countRef.current++   // 修改不触发渲染
  }

  return (
    <div>
      <input ref={inputRef} />
      <button onClick={focus}>聚焦</button>
    </div>
  )
}
```

## 12. 自定义 Hook

以 `use` 开头的函数，复用逻辑。

```
function useWindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight
  })

  useEffect(() => {
    const handler = () => setSize({
      width: window.innerWidth,
      height: window.innerHeight
    })
    window.addEventListener('resize', handler)
    return () => window.removeEventListener('resize', handler)
  }, [])

  return size
}

// 使用
const { width, height } = useWindowSize()
```

## 13. useContext 上下文

跨层级传递数据，避免 props 逐层传递。

```
import { createContext, useContext, useState } from 'react'

// createContext('light')：创建上下文，'light' 是默认值
const ThemeContext = createContext('light')

function App() {
  return (
    // Provider 提供数据，value 会覆盖默认值
    <ThemeContext.Provider value="dark">
      <Child />
    </ThemeContext.Provider>
  )
}

function Child() {
  // useContext 读取最近的 Provider 提供的值（无需逐层传 props）
  const theme = useContext(ThemeContext)
  return <div>{theme}</div>
}
```

## 14. useReducer 复杂状态

适合复杂的状态逻辑，类似 Redux 的 reducer。

```
import { useReducer } from 'react'

const initialState = { count: 0 }

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 }
    case 'decrement':
      return { count: state.count - 1 }
    default:
      return state
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState)

  return (
    <div>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
    </div>
  )
}
```

## 15. 列表渲染与 key

用 map 渲染列表，每个元素需要唯一 key，key 帮助 React 高效 diff。

```
function List({ items }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  )
}
```

注意：不要用数组下标 index 作为 key（列表顺序变化会导致错误更新）。

## 16. 事件处理

React 事件用驼峰命名，事件对象是合成事件。

```
function App() {
  const handleClick = (e) => {
    e.preventDefault()            // 阻止默认行为（如表单提交、链接跳转）
    console.log('clicked', e)     // e 是 React 合成事件对象
  }

  return (
    // 事件用驼峰命名 onClick，传入函数引用而非调用结果
    <button onClick={handleClick}>点击</button>
  )
}
```

## 17. 受控组件与非受控组件

- 受控：表单值由 React state 控制。
- 非受控：表单值由 DOM 自己管理（用 ref 读取）。

```
// 受控组件
function Form() {
  const [value, setValue] = useState('')
  return <input value={value} onChange={e => setValue(e.target.value)} />
}

// 非受控组件
function Form2() {
  const inputRef = useRef(null)
  const submit = () => console.log(inputRef.current.value)
  return <input ref={inputRef} />
}
```

## 18. 条件渲染

JSX 中用 && 短路或三元表达式实现条件渲染。

```
function App({ isLogin }) {
  return (
    <div>
      {/* && 短路：isLogin 为 true 时才渲染 */}
      {isLogin && <p>欢迎回来</p>}

      {/* 三元表达式：二选一渲染 */}
      {isLogin ? <p>已登录</p> : <button>登录</button>}
    </div>
  )
}
```

## 19. 组件生命周期（类组件）

类组件中的生命周期方法。

```
class App extends React.Component {
  componentDidMount() {
    console.log('挂载完成')
  }
  componentDidUpdate(prevProps) {
    console.log('更新完成')
  }
  componentWillUnmount() {
    console.log('卸载前')
  }
  render() {
    return <div>类组件</div>
  }
}
```

函数组件中对应：mount → useEffect([])、update → useEffect([deps])、unmount → useEffect 返回函数。

## 20. 高阶组件 HOC

接收组件返回新组件的函数，用于复用逻辑。

```
function withLoading(WrappedComponent) {
  return function WithLoading(props) {
    if (props.loading) {
      return <div>加载中...</div>
    }
    return <WrappedComponent {...props} />
  }
}

const ListWithLoading = withLoading(List)
```

## 21. Render Props

通过函数 prop 共享渲染逻辑。

```
function Mouse({ render }) {
  const [pos, setPos] = useState({ x: 0, y: 0 })
  const handle = e => setPos({ x: e.clientX, y: e.clientY })

  return <div onMouseMove={handle}>{render(pos)}</div>
}

// 使用
<Mouse render={({ x, y }) => <p>{x}, {y}</p>} />
```

## 22. React.memo 与性能优化

`React.memo` 对函数组件浅比较 props，props 不变则跳过渲染。

```
import { memo } from 'react'

const Child = memo(function Child({ value }) {
  console.log('渲染')
  return <div>{value}</div>
})

// 父组件中 value 不变时，Child 不会重渲染
```

## 23. 虚拟 DOM 与 Diff 算法

React 通过虚拟 DOM 描述 UI，更新时与旧树 diff，只更新变化的部分。

Diff 策略：
- 同层比较，不跨层比较
- 类型不同则直接替换，不递归子节点
- 通过 key 判断列表中元素是否变化

## 24. 状态提升

把多个组件共享的状态提升到最近的共同父组件。

```
function Parent() {
  // 共享状态提升到共同父组件，统一管理
  const [value, setValue] = useState('')

  return (
    <div>
      {/* 两个子组件共享同一个 value 和 setValue，保持同步 */}
      <InputA value={value} onChange={setValue} />
      <InputB value={value} onChange={setValue} />
    </div>
  )
}
```

## 25. 状态管理 Redux

集中管理全局状态，单向数据流。

```
// store.js
import { createStore } from 'redux'

const reducer = (state = { count: 0 }, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 }
    default:
      return state
  }
}

const store = createStore(reducer)
export default store

// 组件中
store.dispatch({ type: 'INCREMENT' })  // 派发 action，触发状态更新
store.getState().count                 // 读取当前状态
store.subscribe(() => {})              // 订阅状态变化（回调在每次更新后触发）
```

## 26. 状态管理 Zustand

轻量级状态管理库，无需 Provider。

```
import { create } from 'zustand'

// create 返回一个 Hook，内部定义状态和修改方法
const useStore = create((set) => ({
  count: 0,                                    // 状态
  increment: () => set(state => ({ count: state.count + 1 })),  // 修改状态
  decrement: () => set(state => ({ count: state.count - 1 }))
}))

// 组件中
function Counter() {
  const { count, increment } = useStore()
  return <button onClick={increment}>{count}</button>
}
```

## 27. React Router 路由

通过路由表定义页面与 URL 的映射，支持动态参数和编程式导航。

```
import { BrowserRouter, Routes, Route, Link, useParams, useNavigate } from 'react-router-dom'

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/user/:id" element={<User />} />
      </Routes>
    </BrowserRouter>
  )
}

function User() {
  const { id } = useParams()
  const navigate = useNavigate()
  return (
    <div>
      <p>用户 {id}</p>
      <button onClick={() => navigate('/')}>返回首页</button>
    </div>
  )
}
```

## 28. 组件通信方式汇总

| 方式 | 场景 |
|------|------|
| props | 父传子 |
| 回调函数 | 子传父 |
| Context | 跨层级 |
| Redux / Zustand | 全局状态 |
| 事件总线 | 任意组件（不推荐） |

## 29. 错误边界 Error Boundary

类组件实现，捕获子组件渲染错误，防止整个应用崩溃。

```
class ErrorBoundary extends React.Component {
  state = { hasError: false }

  static getDerivedStateFromError() {
    return { hasError: true }
  }

  componentDidCatch(error, info) {
    console.log(error, info)
  }

  render() {
    if (this.state.hasError) {
      return <h1>出错了</h1>
    }
    return this.props.children
  }
}

// 使用
<ErrorBoundary>
  <MyComponent />
</ErrorBoundary>
```

## 30. 代码分割与懒加载

使用 `React.lazy` 和 `Suspense` 实现按需加载。

```
import { lazy, Suspense } from 'react'

const HeavyComponent = lazy(() => import('./HeavyComponent'))

function App() {
  return (
    <Suspense fallback={<div>加载中...</div>}>
      <HeavyComponent />
    </Suspense>
  )
}
```

## 31. StrictMode 严格模式

帮助发现潜在问题，开发环境下会重复渲染和调用副作用。

```
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
)
```

## 32. React 18 新特性

- 并发特性（Concurrent Features）
- 自动批处理（Automatic Batching）
- 新的根 API：`createRoot`
- 新增 Hook：`useId`、`useTransition`、`useDeferredValue`

```
import { useTransition, useDeferredValue } from 'react'

// useTransition：标记更新为非紧急
const [isPending, startTransition] = useTransition()
startTransition(() => setSearch(input))

// useDeferredValue：延迟更新某个值
const deferredValue = useDeferredValue(searchText)
```

## 33. 常用 Hook 速查

| Hook | 作用 |
|------|------|
| useState | 状态管理 |
| useEffect | 副作用 |
| useMemo | 缓存值 |
| useCallback | 缓存函数 |
| useRef | 引用/可变值 |
| useContext | 上下文 |
| useReducer | 复杂状态 |
| useLayoutEffect | 渲染前同步执行副作用 |
| useId | 生成唯一 ID |
| useTransition | 非紧急更新 |
| useImperativeHandle | 暴露子组件方法给父组件 |
| useDebugValue | 调试自定义 Hook |

## 34. 类组件：state 与 setState

类组件用 state 管理状态，用 setState 更新。

```
import React from 'react'

class Counter extends React.Component {
  // 方式一：构造函数中初始化 state
  constructor(props) {
    super(props)                    // 必须调用 super
    this.state = {
      count: 0
    }
  }

  // 方式二：类属性初始化（更简洁）
  // state = { count: 0 }

  increment = () => {
    // setState 接受对象，会浅合并
    this.setState({ count: this.state.count + 1 })

    // 基于旧值更新，用函数式（避免异步问题）
    this.setState((prevState) => ({ count: prevState.count + 1 }))
  }

  render() {
    return (
      <div>
        <p>{this.state.count}</p>
        <button onClick={this.increment}>+1</button>
      </div>
    )
  }
}
```

## 35. 类组件：事件与 this 绑定

类组件中普通方法的 this 默认是 undefined，需要绑定。

```
class App extends React.Component {
  // 方式一：类属性箭头函数（推荐，自动绑定 this）
  handleClick = () => {
    console.log(this)   // this 指向组件实例
  }

  // 方式二：构造函数中 bind
  constructor(props) {
    super(props)
    this.handleClick2 = this.handleClick2.bind(this)
  }
  handleClick2() {
    console.log(this)
  }

  // 方式三：调用时用箭头函数包裹（每次渲染创建新函数，不推荐）
  handleClick3() {
    console.log(this)
  }

  render() {
    return (
      <div>
        <button onClick={this.handleClick}>方式一</button>
        <button onClick={this.handleClick2}>方式二</button>
        <button onClick={() => this.handleClick3()}>方式三</button>
      </div>
    )
  }
}
```

## 36. 类组件：完整生命周期

类组件生命周期分挂载、更新、卸载三个阶段。

```
class App extends React.Component {
  constructor(props) {
    super(props)                // 初始化 state
    console.log('1. constructor')
  }

  static getDerivedStateFromProps(props, state) {
    // 根据 props 派生 state（少用）
    console.log('2. getDerivedStateFromProps')
    return null
  }

  shouldComponentUpdate(nextProps, nextState) {
    // 控制是否重新渲染（性能优化）
    console.log('3. shouldComponentUpdate')
    return true                 // 返回 false 跳过渲染
  }

  render() {
    console.log('4. render')
    return <div>App</div>
  }

  getSnapshotBeforeUpdate(prevProps, prevState) {
    // DOM 更新前获取快照（少用）
    console.log('5. getSnapshotBeforeUpdate')
    return null
  }

  componentDidMount() {
    // 挂载完成，可发起请求、订阅
    console.log('6. componentDidMount')
  }

  componentDidUpdate(prevProps, prevState) {
    // 更新完成
    console.log('7. componentDidUpdate')
  }

  componentWillUnmount() {
    // 卸载前，清理定时器、订阅
    console.log('8. componentWillUnmount')
  }
}
```

```
# 生命周期流程
挂载：constructor → getDerivedStateFromProps → render → componentDidMount
更新：getDerivedStateFromProps → shouldComponentUpdate → render
     → getSnapshotBeforeUpdate → componentDidUpdate
卸载：componentWillUnmount
```

## 37. Hooks 规则

使用 Hooks 必须遵守两条规则。

```
# 规则 1：只在顶层调用
# 不要在循环、条件、嵌套函数里调用 Hook

// 错误
if (condition) {
  useState(0)   // 不能放在条件里
}

// 正确
const [state, setState] = useState(0)
```

```
# 规则 2：只在函数组件或自定义 Hook 中调用
# 不要在普通函数里调用 Hook
```

```
# 原因：React 依赖 Hook 调用顺序来关联状态
# 违反规则会导致状态错乱
```

## 38. useLayoutEffect

与 useEffect 类似，但在浏览器绘制之前同步执行，适合读取布局信息。

```
import { useLayoutEffect, useEffect, useRef } from 'react'

function App() {
  const ref = useRef(null)

  // useEffect：绘制后执行（异步，用户可能看到闪烁）
  useEffect(() => {
    ref.current.style.color = 'red'
  })

  // useLayoutEffect：绘制前执行（同步，避免闪烁）
  useLayoutEffect(() => {
    ref.current.style.color = 'red'
  })

  return <div ref={ref}>文字</div>
}
```

```
# 何时用 useLayoutEffect
# 需要同步修改 DOM、读取布局尺寸时
# 大多数情况用 useEffect 即可（不阻塞渲染）
```

## 39. useImperativeHandle 与 forwardRef

让父组件调用子组件暴露的方法（命令式操作）。

```
import { forwardRef, useImperativeHandle, useRef } from 'react'

// forwardRef 让子组件接收 ref
const Input = forwardRef((props, ref) => {
  const inputRef = useRef(null)

  // 暴露给父组件的方法
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    clear: () => { inputRef.current.value = '' }
  }))

  return <input ref={inputRef} />
})

// 父组件
function Parent() {
  const inputRef = useRef(null)

  return (
    <div>
      <Input ref={inputRef} />
      <button onClick={() => inputRef.current.focus()}>聚焦</button>
      <button onClick={() => inputRef.current.clear()}>清空</button>
    </div>
  )
}
```

## 40. useId

生成跨服务端/客户端一致的唯一 ID，用于无障碍属性。

```
import { useId } from 'react'

function Form() {
  const id = useId()   // 生成唯一 ID

  return (
    <div>
      <label htmlFor={id}>姓名</label>
      <input id={id} />
    </div>
  )
}
```

## 41. useDebugValue

在 React DevTools 中为自定义 Hook 显示调试信息。

```
import { useDebugValue, useState } from 'react'

function useOnlineStatus() {
  const [online, setOnline] = useState(true)
  useDebugValue(online ? '在线' : '离线')   // DevTools 中显示
  return online
}
```

## 42. 状态管理：Redux Toolkit（现代 Redux）

Redux Toolkit 是官方推荐的现代 Redux 写法，简化了 store 配置和 reducer 定义。

```
# 安装
npm install @reduxjs/toolkit react-redux
```

```
// store.js
import { configureStore, createSlice } from '@reduxjs/toolkit'

// createSlice 同时生成 action 和 reducer
const counterSlice = createSlice({
    name: 'counter',              // slice 名
    initialState: { count: 0 },
    reducers: {
        increment: (state) => { state.count += 1 },   // 可直接修改（内部用 immer）
        decrement: (state) => { state.count -= 1 }
    }
})

export const { increment, decrement } = counterSlice.actions

export const store = configureStore({
    reducer: {
        counter: counterSlice.reducer
    }
})
```

```
// 组件中
import { useSelector, useDispatch } from 'react-redux'
import { increment } from './store'

function Counter() {
    const count = useSelector(state => state.counter.count)   // 读取状态
    const dispatch = useDispatch()                            // 派发 action

    return (
        <div>
            <p>{count}</p>
            <button onClick={() => dispatch(increment())}>+1</button>
        </div>
    )
}
```

## 43. Redux Toolkit：异步请求

用 createAsyncThunk 处理异步操作。

```
import { createAsyncThunk, createSlice } from '@reduxjs/toolkit'

// 异步 action
export const fetchUser = createAsyncThunk('user/fetch', async (id) => {
    const res = await fetch(`/api/user/${id}`)
    return res.json()     // 返回值作为 fulfilled 的 payload
})

const userSlice = createSlice({
    name: 'user',
    initialState: { data: null, loading: false },
    reducers: {},
    extraReducers: (builder) => {
        builder
            .addCase(fetchUser.pending, (state) => {
                state.loading = true            // 请求中
            })
            .addCase(fetchUser.fulfilled, (state, action) => {
                state.loading = false
                state.data = action.payload     // 请求成功
            })
            .addCase(fetchUser.rejected, (state) => {
                state.loading = false           // 请求失败
            })
    }
})
```

## 44. 状态管理：Context + useReducer

React 内置方案，无需第三方库，适合中小型应用的全局状态。

```
import { createContext, useContext, useReducer } from 'react'

// 创建 Context
const AppContext = createContext(null)

// reducer
function reducer(state, action) {
    switch (action.type) {
        case 'ADD':
            return { items: [...state.items, action.payload] }
        case 'REMOVE':
            return { items: state.items.filter(i => i.id !== action.payload) }
        default:
            return state
    }
}

// Provider 组件
function AppProvider({ children }) {
    const [state, dispatch] = useReducer(reducer, { items: [] })

    return (
        <AppContext.Provider value={{ state, dispatch }}>
            {children}
        </AppContext.Provider>
    )
}

// 子组件使用
function TodoList() {
    const { state, dispatch } = useContext(AppContext)

    return (
        <ul>
            {state.items.map(item => (
                <li key={item.id}>{item.text}</li>
            ))}
        </ul>
    )
}
```

## 45. 状态管理库对比

| 方案 | 特点 | 适用场景 |
|------|------|----------|
| useState | 局部状态 | 单组件内 |
| useReducer | 复杂局部状态 | 状态逻辑多 |
| Context + useReducer | 内置、无依赖 | 中小应用全局状态 |
| Redux Toolkit | 强大、规范、生态好 | 大型应用、团队协作 |
| Zustand | 极简、无需 Provider | 中小应用 |
| MobX | 响应式、自动追踪 | 数据复杂、强调响应式 |
| Jotai | 原子化、细粒度 | 性能敏感场景 |

```
# 选型建议
局部状态 → useState / useReducer
简单全局 → Context + useReducer 或 Zustand
大型应用/团队 → Redux Toolkit
性能敏感/细粒度 → Jotai

## 46. 状态管理：MobX

MobX 是响应式状态管理库，状态变化自动驱动组件更新，无需手动 dispatch，写法更贴近面向对象。

```
# 安装
npm install mobx mobx-react-lite
```

```
# 核心概念
observable   可观察状态（被追踪的属性）
action       修改状态的方法
computed     派生值（依赖变化自动重算，有缓存）
observer     让组件响应状态变化
```

```
// store.js
import { makeAutoObservable } from 'mobx'

class CounterStore {
    count = 0

    constructor() {
        makeAutoObservable(this)   // 自动把属性转 observable、方法转 action
    }

    increment = () => {
        this.count++               // 直接修改，无需返回新对象
    }

    decrement = () => {
        this.count--
    }

    get double() {
        return this.count * 2      // computed：依赖 count，自动缓存
    }
}

export default new CounterStore()
```

```
// 组件中（用 observer 包裹）
import { observer } from 'mobx-react-lite'
import counterStore from './store'

const Counter = observer(() => {
    return (
        <div>
            <p>{counterStore.count}</p>
            <p>double: {counterStore.double}</p>
            <button onClick={counterStore.increment}>+1</button>
            <button onClick={counterStore.decrement}>-1</button>
        </div>
    )
})
```

## 47. MobX：异步 action

异步操作中用 runInAction 修改状态。

```
import { makeAutoObservable, runInAction } from 'mobx'

class UserStore {
    data = null
    loading = false

    constructor() {
        makeAutoObservable(this)
    }

    async fetchUser(id) {
        this.loading = true
        try {
            const res = await fetch(`/api/user/${id}`)
            const data = await res.json()
            runInAction(() => {        // 异步回调中修改状态需用 runInAction
                this.data = data
                this.loading = false
            })
        } catch (e) {
            runInAction(() => { this.loading = false })
        }
    }
}
```

## 48. MobX 与 Redux 对比

| 维度 | MobX | Redux Toolkit |
|------|------|---------------|
| 范式 | 响应式、面向对象 | 单向数据流、函数式 |
| 状态修改 | 直接改（mutable） | 不可变（immer） |
| 代码量 | 少 | 较多 |
| 学习曲线 | 平缓 | 较陡 |
| 数据流 | 隐式、自动追踪 | 显式、可预测 |
| 适用 | 数据模型复杂、强调响应式 | 大型应用、需要可预测性 |

```
# 选择建议
喜欢面向对象、数据模型复杂 → MobX
需要严格可预测的数据流、团队规范 → Redux Toolkit
```

## 49. 类组件：props 默认值与类型校验

用 defaultProps 设默认值，用 PropTypes 校验类型。

```
import React from 'react'
import PropTypes from 'prop-types'

class Child extends React.Component {
    render() {
        return <p>{this.props.name} - {this.props.age}</p>
    }
}

// 默认值：未传入时使用
Child.defaultProps = {
    name: '默认名字',
    age: 0
}

// 类型校验：类型不符会在控制台告警
Child.propTypes = {
    name: PropTypes.string,
    age: PropTypes.number.isRequired,   // 必填
    onClick: PropTypes.func
}
```

## 50. 类组件：PureComponent 性能优化

PureComponent 自动浅比较 props/state，不变则跳过渲染。

```
import React from 'react'

// 继承 PureComponent 而非 Component
class Child extends React.PureComponent {
    render() {
        console.log('渲染')
        return <div>{this.props.value}</div>
    }
}

class Parent extends React.Component {
    state = { value: 1, other: 2 }

    render() {
        // 修改 other 时，Child 的 value 没变，不会重渲染
        return <Child value={this.state.value} />
    }
}
```

```
# 注意：PureComponent 是浅比较
# 传入新对象/数组/函数时，每次都视为变化（会重渲染）
# 相当于函数组件的 React.memo
```

## 51. 类组件：refs 与 DOM

用 createRef 获取 DOM 元素。

```
import React from 'react'

class App extends React.Component {
    constructor(props) {
        super(props)
        this.inputRef = React.createRef()   // 创建 ref
    }

    focus = () => {
        this.inputRef.current.focus()       // 访问 DOM 元素
    }

    render() {
        return (
            <div>
                <input ref={this.inputRef} />
                <button onClick={this.focus}>聚焦</button>
            </div>
        )
    }
}
```

## 52. 类组件：Context

用 static contextType 在类组件中读取 Context。

```
import React from 'react'

const ThemeContext = React.createContext('light')

class Child extends React.Component {
    static contextType = ThemeContext   // 声明要消费的 Context

    render() {
        return <div>当前主题：{this.context}</div>   // this.context 读取
    }
}

class App extends React.Component {
    render() {
        return (
            <ThemeContext.Provider value="dark">
                <Child />
            </ThemeContext.Provider>
        )
    }
}
```

## 53. 类组件：父组件调用子组件方法

父组件通过 ref 拿到子组件实例，调用其方法。

```
import React from 'react'

class Child extends React.Component {
    doSomething() {
        console.log('子组件方法被调用')
    }

    render() {
        return <div>子组件</div>
    }
}

class Parent extends React.Component {
    constructor(props) {
        super(props)
        this.childRef = React.createRef()
    }

    callChild = () => {
        this.childRef.current.doSomething()   // 调用子组件方法
    }

    render() {
        return (
            <div>
                <Child ref={this.childRef} />
                <button onClick={this.callChild}>调用子组件方法</button>
            </div>
        )
    }
}
```

## 54. 类组件：受控表单

用 state 控制表单值。

```
import React from 'react'

class Form extends React.Component {
    state = {
        name: '',
        age: ''
    }

    handleChange = (e) => {
        // 用 name 属性区分字段
        this.setState({ [e.target.name]: e.target.value })
    }

    handleSubmit = (e) => {
        e.preventDefault()               // 阻止默认提交
        console.log(this.state)          // { name, age }
    }

    render() {
        return (
            <form onSubmit={this.handleSubmit}>
                <input name="name" value={this.state.name} onChange={this.handleChange} />
                <input name="age" value={this.state.age} onChange={this.handleChange} />
                <button type="submit">提交</button>
            </form>
        )
    }
}
```

## 55. 类组件：条件渲染与列表渲染

类组件同样支持条件渲染和列表渲染。

```
import React from 'react'

class List extends React.Component {
    state = {
        items: [
            { id: 1, name: 'a' },
            { id: 2, name: 'b' }
        ]
    }

    render() {
        const { items } = this.state

        return (
            <div>
                {/* 条件渲染 */}
                {items.length > 0 ? (
                    <ul>
                        {/* 列表渲染 */}
                        {items.map(item => (
                            <li key={item.id}>{item.name}</li>
                        ))}
                    </ul>
                ) : (
                    <p>暂无数据</p>
                )}
            </div>
        )
    }
}
```

## 56. 类组件写法速查

| 功能 | 类组件 | 函数组件 |
|------|--------|----------|
| 状态 | this.state + setState | useState |
| 副作用 | componentDidMount 等 | useEffect |
| 计算值 | get 方法/自定义 | useMemo |
| ref | createRef | useRef |
| 上下文 | static contextType | useContext |
| 性能优化 | PureComponent | React.memo |
| 记忆化 | shouldComponentUpdate | useMemo/useCallback |

```
# 类组件 vs 函数组件选择
新项目 → 优先函数组件 + Hooks
维护旧项目/错误边界 → 仍需类组件
```
