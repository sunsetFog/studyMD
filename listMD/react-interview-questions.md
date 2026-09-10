# React 高价值面试题清单（含详细答案）

> 涵盖 React 核心原理、Hooks、性能优化、状态管理、源码解析等高频考点。

---

## 目录

1. [基础概念](#一基础概念)
2. [Hooks 深度解析](#二hooks-深度解析)
3. [状态管理](#三状态管理)
4. [性能优化](#四性能优化)
5. [React 原理与源码](#五react-原理与源码)
6. [路由与数据流](#六路由与数据流)
7. [实战场景题](#七实战场景题)
8. [TypeScript + React](#八typescript--react)
9. [组件设计模式](#九组件设计模式)
10. [React 测试策略](#十react-测试策略)
11. [React 19 新特性](#十一react-19-新特性)
12. [常见陷阱与高阶题](#十二常见陷阱与高阶题)
13. [系统设计题](#十三系统设计题)
14. [React 工程化实践](#十四react-工程化实践)
  15. [React 生态深度对比](#十五react-生态深度对比)
16. [React 安全与最佳实践](#十六react-安全与最佳实践)
17. [Next.js App Router 深度解析](#十七nextjs-app-router-深度解析)
18. [React 服务器组件与流式渲染](#十八react-服务器组件与流式渲染)
19. [前端架构与工程化进阶](#十九前端架构与工程化进阶)
20. [高级 Hooks 与渲染机制](#二十高级-hooks-与渲染机制)
21. [React 实战进阶技巧](#二十一react-实战进阶技巧)
22. [工程化与未来趋势](#二十二工程化与未来趋势)

---

## 一、基础概念

### 1. React 中的 Virtual DOM 是什么？它是如何工作的？

**Virtual DOM（虚拟 DOM）** 是真实 DOM 的 JavaScript 对象表示。React 在内存中维护一棵虚拟 DOM 树，当状态变化时，通过 Diff 算法比较新旧虚拟 DOM，计算出最小变更集，最后批量更新真实 DOM。

**工作流程：**

```
状态变化 → 生成新 VDOM 树 → Diff 算法对比新旧 VDOM → 计算 Patch → 批量更新真实 DOM
```

**为什么需要 Virtual DOM？**

- **性能优化**：直接操作真实 DOM 代价高昂，VDOM 在 JS 层面做 Diff，减少不必要的 DOM 操作。
- **跨平台**：VDOM 是对渲染层的抽象，可以渲染到不同平台（Web、Native、Canvas）。
- **声明式编程**：开发者只需关心"UI 应该是什么样"，不需要手动操作 DOM。

**代码示例：**

```jsx
// VDOM 节点的大致结构
const vdom = {
  type: 'div',
  props: {
    id: 'app',
    children: [
      { type: 'h1', props: { children: 'Hello' } },
      { type: 'p', props: { children: 'World' } }
    ]
  }
};
```

---

### 2. React 的 Diff 算法策略是什么？

React 采用 **三层 Diff 策略**，将 O(n³) 复杂度优化到 O(n)：

**（1）Tree Diff（树层级）**
- 只对同层节点进行对比，不跨层级比较。
- 如果节点类型不同，直接删除旧节点及其子树，创建新节点。

**（2）Component Diff（组件层级）**
- 同一类型的组件：继续按照 Tree Diff 比较子节点。
- 不同类型的组件：判定为 dirty component，直接替换整个组件及其子树。

**（3）Element Diff（元素层级）**
- 同一层级的子节点，通过 **key** 属性来识别。
- 没有 key 时通过下标逐个对比，有 key 时可复用 DOM 元素。

**为什么需要 key？**

```jsx
// 无 key：逐个对比，移动代价大
// 有 key：React 可以识别哪些元素只是移动了位置
<ul>
  {items.map(item => (
    <li key={item.id}>{item.text}</li> // key 必须唯一且稳定
  ))}
</ul>
```

**key 的使用原则：**
- 使用唯一且稳定的标识（如数据 ID）。
- **不要使用数组索引作为 key**（列表重排序时会导致错误复用）。
- key 只在兄弟节点之间需要保持唯一。

---

### 3. 类组件和函数组件的区别？

| 维度 | 类组件 | 函数组件 |
|------|--------|----------|
| 定义方式 | `class XX extends React.Component` | `function XX(props)` |
| 状态管理 | `this.state` + `this.setState` | `useState` / `useReducer` |
| 生命周期 | 完整生命周期方法 | `useEffect` / `useLayoutEffect` 模拟 |
| this 绑定 | 需要绑定 this | 无 this，无绑定问题 |
| 性能 | 实例化开销较大 | 轻量，无实例化 |
| 逻辑复用 | HOC / Render Props | 自定义 Hooks |
| 代码量 | 较多模板代码 | 更简洁 |

**函数组件的优势：**
- 更简洁，减少样板代码。
- 逻辑复用更方便（自定义 Hooks）。
- 更符合 React 未来的发展方向（Concurrent Mode 优先支持函数组件）。
- 没有 this 绑定问题，更容易理解和测试。

---

### 4. `setState` 是同步还是异步的？

**React 18 之前：**

```jsx
// 在 React 合成事件和生命周期中：异步（批量更新）
handleClick = () => {
  this.setState({ count: this.state.count + 1 });
  console.log(this.state.count); // 还是旧值
};

// 在原生事件和 setTimeout 中：同步
setTimeout(() => {
  this.setState({ count: this.state.count + 1 });
  console.log(this.state.count); // 新值
}, 0);
```

**React 18（自动批处理）：**

```jsx
// React 18 中，所有更新都会自动批处理
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // 只会触发一次重渲染！
}, 0);
```

**核心结论：**
- React 18 之前，合成事件/生命周期中异步批量处理，原生事件/setTimeout 中同步。
- React 18 之后，所有更新默认批量处理（可通过 `flushSync` 强制同步）。
- `setState` 传入函数可以获取最新 state：`setState(prev => prev + 1)`。

---

### 5. 受控组件 vs 非受控组件

**受控组件（Controlled Component）：**

```jsx
// 表单数据由 React state 控制
function ControlledInput() {
  const [value, setValue] = useState('');
  
  return (
    <input 
      value={value} 
      onChange={e => setValue(e.target.value)} 
    />
  );
}
```

**非受控组件（Uncontrolled Component）：**

```jsx
// 表单数据由 DOM 自身管理，通过 ref 获取值
function UncontrolledInput() {
  const inputRef = useRef(null);
  
  const handleSubmit = () => {
    console.log(inputRef.current.value);
  };
  
  return <input ref={inputRef} defaultValue="hello" />;
}
```

**选择建议：**

| 场景 | 推荐 |
|------|------|
| 需要实时验证 | 受控组件 |
| 需要条件禁用按钮 | 受控组件 |
| 表单字段很少 | 受控组件 |
| 文件上传 | 非受控组件（文件 input 只能非受控） |
| 与第三方非 React 库集成 | 非受控组件 |
| 性能敏感的大型表单 | 非受控组件 + ref |

---

### 6. React 合成事件（SyntheticEvent）是什么？

React 封装了一套跨浏览器的事件系统，称为 **合成事件**。

**核心特点：**

```jsx
function handleClick(e) {
  // e 是 SyntheticEvent，不是原生 Event
  e.preventDefault();   // 跨浏览器一致
  e.stopPropagation();  // 跨浏览器一致
  console.log(e.nativeEvent); // 可通过 nativeEvent 获取原生事件
}
```

**事件委托机制：**
- React 17 之前：所有事件委托到 `document`。
- React 17 之后：委托到根容器 `root` 节点。
- 好处：减少内存占用，组件卸载时自动清理。

**为什么需要合成事件？**
1. **跨浏览器兼容**：抹平不同浏览器的 API 差异。
2. **性能优化**：通过事件委托减少事件监听器数量。
3. **事件池**（React 17 前）：复用事件对象，减少 GC 压力（React 17 已移除事件池）。

---

## 二、Hooks 深度解析

### 7. `useState` 的原理和注意事项

**基本用法：**

```jsx
const [state, setState] = useState(initialValue);

// 函数式更新（推荐用于依赖前值的情况）
setState(prev => prev + 1);

// 直接更新
setState(newValue);
```

**核心原理（简化实现）：**

```jsx
// 简化版 useState 实现思路
let memorizedStates = [];
let cursor = 0;

function useState(initialValue) {
  const currentCursor = cursor;
  memorizedStates[currentCursor] = 
    memorizedStates[currentCursor] ?? initialValue;
  
  const setState = (newValue) => {
    memorizedStates[currentCursor] = newValue;
    render(); // 触发重渲染
  };
  
  cursor++;
  return [memorizedStates[currentCursor], setState];
}
```

**注意 ⚠️：**
- `useState` 的更新是**异步批处理**的（React 18 全部批处理）。
- 连续调用多个 `setState`，React 会将它们合并到一次渲染中。
- **不要在循环、条件或嵌套函数中调用 Hooks**（必须在函数顶层调用）。

---

### 8. `useEffect` 的执行时机和清理机制

**三种使用场景：**

```jsx
// 1. 每次渲染后都执行（没有依赖数组）
useEffect(() => {
  console.log('每次渲染都执行');
});

// 2. 仅在挂载时执行一次（空依赖数组）
useEffect(() => {
  console.log('仅在 mount 时执行');
}, []);

// 3. 依赖项变化时执行
useEffect(() => {
  console.log('count 变化时执行', count);
}, [count]);
```

**清理函数（Cleanup）：**

```jsx
useEffect(() => {
  const timer = setInterval(() => {
    console.log('tick');
  }, 1000);
  
  // 清理函数：组件卸载或依赖变化前执行
  return () => {
    clearInterval(timer);
  };
}, []);
```

**执行时序：**

```
Mount:   渲染 → useEffect
Update:  渲染 → 执行上次 cleanup → useEffect
Unmount: 执行最后一次 cleanup
```

**`useLayoutEffect` 与 `useEffect` 的区别：**

| | `useEffect` | `useLayoutEffect` |
|------|-------------|-------------------|
| 执行时机 | 浏览器绘制完成后异步执行 | DOM 更新后、浏览器绘制前同步执行 |
| 是否阻塞渲染 | 不阻塞 | 阻塞渲染 |
| 使用场景 | 数据请求、订阅、日志 | 测量 DOM、同步更新 DOM 避免闪烁 |

---

### 9. `useRef` 的多种用途

```jsx
function RefExamples() {
  // 1. 访问 DOM 元素
  const inputRef = useRef(null);
  const focusInput = () => inputRef.current?.focus();

  // 2. 保存可变值（不触发重渲染）
  const countRef = useRef(0);
  const increment = () => {
    countRef.current++;
    console.log(countRef.current); // 变化但不会重渲染
  };

  // 3. 保存上一次的值
  const [count, setCount] = useState(0);
  const prevCountRef = useRef(count);
  
  useEffect(() => {
    prevCountRef.current = count; // 记录上一次的值
  }, [count]);

  // 4. 解决闭包陷阱
  const callbackRef = useRef(null);
  callbackRef.current = () => {
    console.log(count); // 始终能拿到最新值
  };

  return <input ref={inputRef} />;
}
```

**`useRef` vs `createRef`：**
- `useRef` 在每次渲染时返回**同一个对象**引用。
- `createRef` 每次调用都创建新的 ref 对象。
- 函数组件中必须使用 `useRef`。

---

### 10. `useMemo` 和 `useCallback` 的区别

```jsx
function Parent({ a, b }) {
  // useMemo：缓存计算结果（值）
  const expensiveResult = useMemo(() => {
    return computeExpensiveValue(a, b);
  }, [a, b]);

  // useCallback：缓存函数引用
  const handleClick = useCallback(() => {
    doSomething(a, b);
  }, [a, b]);

  // useCallback(fn, deps) 等价于 useMemo(() => fn, deps)

  return <Child onClick={handleClick} result={expensiveResult} />;
}
```

**核心区别：**

| | `useMemo` | `useCallback` |
|------|-----------|---------------|
| 返回值 | 缓存计算结果的**值** | 缓存函数**本身** |
| 用途 | 避免重复执行昂贵计算 | 避免子组件因函数引用变化而重渲染 |
| 本质 | `useCallback(fn, deps)` ≈ `useMemo(() => fn, deps)` |

**什么时候该用？**
- **`useMemo`**：计算量大的派生数据（如数组过滤、排序、复杂计算）。
- **`useCallback`**：传给子组件（用 `React.memo` 包裹）的回调函数。
- **不要过早优化**：大部分场景不需要用，只有在性能瓶颈出现时才使用。

---

### 11. `useReducer` 的适用场景

```jsx
// Reducer 函数（纯函数）
function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return { count: action.payload };
    default:
      throw new Error('Unknown action type');
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  
  return (
    <>
      Count: {state.count}
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset', payload: 0 })}>
        Reset
      </button>
    </>
  );
}
```

**适用场景：**
- 状态逻辑复杂、包含多个子值。
- 下一个状态依赖上一个状态。
- 多个状态更新之间有相互依赖关系。
- 类似 Redux 但不需要全局状态管理的场景。

---

### 12. 自定义 Hooks 的设计原则

```jsx
// 1. useDebounce - 防抖 Hook
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// 2. usePrevious - 获取上一次的值
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}

// 3. useFetch - 数据请求封装
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    
    setLoading(true);
    fetch(url)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) setData(data);
      })
      .catch(err => {
        if (!cancelled) setError(err);
      })
      .finally(() => {
        if (!cancelled) setLoading(false);
      });

    return () => { cancelled = true; }; // 竞态条件处理
  }, [url]);

  return { data, loading, error };
}
```

**设计原则：**
1. **以 `use` 开头**：这是 React 识别 Hook 的约定。
2. **单一职责**：一个 Hook 只做一件事。
3. **返回值清晰**：返回易于使用的 API。
4. **处理清理**：在 `useEffect` 中返回清理函数。
5. **处理竞态条件**：组件卸载或依赖变化时要取消异步操作。

---

### 13. Hooks 闭包陷阱及解决方案

**问题场景：**

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      // 闭包捕获了初始的 count=0，永远打印 1
      setCount(count + 1); // 每次都是 0 + 1 = 1
    }, 1000);
    
    return () => clearInterval(timer);
  }, []); // 空依赖，effect 只执行一次

  return <div>{count}</div>; // 始终显示 1
}
```

**解决方案：**

```jsx
// 方案1：使用函数式更新（推荐）
useEffect(() => {
  const timer = setInterval(() => {
    setCount(prev => prev + 1); // 获取最新值
  }, 1000);
  return () => clearInterval(timer);
}, []);

// 方案2：将依赖加入数组（每次重建定时器）
useEffect(() => {
  const timer = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(timer);
}, [count]); // 每次 count 变化重建定时器

// 方案3：使用 useRef 保存最新值
const countRef = useRef(count);
countRef.current = count;

useEffect(() => {
  const timer = setInterval(() => {
    setCount(countRef.current + 1);
  }, 1000);
  return () => clearInterval(timer);
}, []);
```

**推荐方案排序：1 > 3 > 2**。函数式更新最简洁高效。

---

## 三、状态管理

### 14. Context API 的使用和注意事项

```jsx
// 1. 创建 Context
const ThemeContext = React.createContext('light');

// 2. Provider 提供值
function App() {
  const [theme, setTheme] = useState('light');
  
  const contextValue = useMemo(
    () => ({ theme, setTheme }),
    [theme]  // 关键：防止不必要的重渲染
  );

  return (
    <ThemeContext.Provider value={contextValue}>
      <Toolbar />
    </ThemeContext.Provider>
  );
}

// 3. Consumer 消费值
function Toolbar() {
  const { theme, setTheme } = useContext(ThemeContext);
  
  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      当前主题: {theme}
    </button>
  );
}
```

**注意事项：**
- Provider value 变化时，所有 Consumer 都会重新渲染（即使只用了部分值）。
- 解决方案：拆分 Context 或使用 `useMemo` 缓存 value。
- Context 不适合频繁变化的全局状态（如每秒更新一次的数据）。
- 对于高频更新的状态，考虑使用 Redux / Zustand / Jotai 等状态管理库。

---

### 15. Redux 的核心概念和中间件

**三大原则：**
- **单一数据源**：整个应用的状态存储在唯一的 Store 中。
- **State 只读**：唯一修改方式是通过 Action。
- **使用纯函数（Reducer）修改**：接收旧 state 和 action，返回新 state。

**数据流：**

```
View → dispatch(Action) → Reducer → Store(新State) → View 重新渲染
```

**Redux Toolkit（现代写法）：**

```jsx
// store/counterSlice.js
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1 },
    decrement: (state) => { state.value -= 1 },
    incrementByAmount: (state, action) => {
      state.value += action.payload;
    },
  },
});

export const { increment, decrement, incrementByAmount } = counterSlice.actions;
export default counterSlice.reducer;
```

**中间件（Middleware）：**

```jsx
// Redux Thunk：处理异步逻辑
const fetchUser = (id) => async (dispatch) => {
  dispatch({ type: 'user/loading' });
  try {
    const user = await api.getUser(id);
    dispatch({ type: 'user/success', payload: user });
  } catch (error) {
    dispatch({ type: 'user/error', payload: error.message });
  }
};

// Redux Saga：通过 Generator 管理副作用
function* fetchUserSaga(action) {
  try {
    const user = yield call(api.getUser, action.payload);
    yield put({ type: 'user/success', payload: user });
  } catch (e) {
    yield put({ type: 'user/error', payload: e.message });
  }
}
```

**性能注意事项：**
- 使用 `useSelector` 时，尽量只选择需要的数据，避免返回新对象。
- 使用 `shallowEqual` 比较器避免不必要的重渲染。
- 考虑使用 `createSelector`（Reselect）缓存派生数据。

---

### 16. React 中有哪些状态管理方案？如何选择？

| 方案 | 适用场景 | 特点 |
|------|----------|------|
| **useState** | 组件内部简单状态 | 最基础，无需额外依赖 |
| **useReducer** | 组件内部复杂状态逻辑 | 有明确的状态转换流程 |
| **Context + useReducer** | 小到中型应用的全局状态 | React 内置，无需引入第三方库 |
| **Redux Toolkit** | 中大型应用，需要完善的 devtools | 生态成熟，中间件丰富 |
| **Zustand** | 轻量全局状态，追求简洁 API | 极简 API，无 Provider 包裹 |
| **Jotai / Recoil** | 原子化状态，希望精准重渲染 | 按原子粒度更新，天然解决重新渲染问题 |
| **React Query (TanStack Query)** | 服务端状态管理 | 缓存、自动重请求、乐观更新 |

**选择建议：**

```
组件内部简单状态      → useState
组件内部复杂逻辑      → useReducer
少量全局状态          → Context + useReducer
大量全局状态          → Zustand / Redux Toolkit
服务端数据（API 请求） → React Query / SWR
```

---

## 四、性能优化

### 17. React.memo、useMemo、useCallback 的合理使用

**性能优化工具箱：**

```jsx
// 1. React.memo：避免 props 不变时的重渲染
const Child = React.memo(({ name, onClick }) => {
  console.log('Child render');
  return <button onClick={onClick}>{name}</button>;
});

// 2. useMemo：缓存计算结果
function Parent({ items }) {
  const sortedItems = useMemo(
    () => [...items].sort((a, b) => a - b),
    [items]
  );
  
  return <List items={sortedItems} />;
}

// 3. useCallback：缓存函数引用（配合 React.memo）
function Parent() {
  const handleClick = useCallback((id) => {
    console.log(id);
  }, []);
  
  return <Child onClick={handleClick} />;
}
```

**常见反模式：**

```jsx
// ❌ 过度使用 useMemo/useCallback，反而增加开销
const value = useMemo(() => 1 + 1, []);  // 简单计算不需要

// ❌ 用 useMemo 包裹 JSX（React.memo 才是正确的）
const element = useMemo(() => <Child />, []);

// ✅ 正确的做法
const MemoizedChild = React.memo(Child);
```

**核心原则：测过再优化。** 使用 React DevTools Profiler 找到瓶颈，针对性优化。

---

### 18. 代码分割（Code Splitting）和懒加载

```jsx
import { lazy, Suspense } from 'react';

// 懒加载组件
const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}
```

**路由级别的代码分割：**

```jsx
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageLoader />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

**错误边界（Error Boundary）配合使用：**

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <h1>Something went wrong.</h1>;
    }
    return this.props.children;
  }
}

// 使用
<Suspense fallback={<Loading />}>
  <ErrorBoundary>
    <LazyComponent />
  </ErrorBoundary>
</Suspense>
```

---

### 19. 长列表优化方案

```jsx
// 1. 虚拟列表（推荐：react-window / react-virtualized）
import { FixedSizeList as List } from 'react-window';

function VirtualList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      {items[index].name}
    </div>
  );

  return (
    <List
      height={400}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </List>
  );
}

// 2. 分页加载
function PaginatedList() {
  const [page, setPage] = useState(1);
  const [items, setItems] = useState([]);
  const pageSize = 20;

  const loadMore = () => {
    fetchItems(page, pageSize).then(newItems => {
      setItems(prev => [...prev, ...newItems]);
      setPage(p => p + 1);
    });
  };

  return (
    <>
      {items.map(item => <ItemCard key={item.id} item={item} />)}
      <button onClick={loadMore}>加载更多</button>
    </>
  );
}

// 3. 无限滚动（Intersection Observer）
function InfiniteScroll() {
  const loaderRef = useRef(null);
  const [items, setItems] = useState([]);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        loadMoreItems().then(newItems => 
          setItems(prev => [...prev, ...newItems])
        );
      }
    });
    
    if (loaderRef.current) observer.observe(loaderRef.current);
    return () => observer.disconnect();
  }, []);

  return (
    <>
      {items.map(item => <ItemCard key={item.id} item={item} />)}
      <div ref={loaderRef}>加载中...</div>
    </>
  );
}
```

---

### 20. 如何排查和避免不必要的重渲染？

**排查工具：**

```jsx
// 1. React DevTools Profiler - 录制渲染过程分析性能

// 2. 使用 why-did-you-render 追踪重渲染原因
// 安装: npm install @welldone-software/why-did-you-render
import React from 'react';

if (process.env.NODE_ENV === 'development') {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true,
  });
}

// 在组件上加静态属性
Component.whyDidYouRender = true;

// 3. 手动追踪渲染次数
function useRenderCount(componentName) {
  const renderCount = useRef(0);
  useEffect(() => {
    renderCount.current += 1;
    console.log(`${componentName} rendered ${renderCount.current} times`);
  });
}
```

**常见重渲染原因和解决方案：**

| 原因 | 解决方案 |
|------|----------|
| 父组件重渲染导致子组件重渲染 | `React.memo` 包裹子组件 |
| 内联函数/对象作为 props | `useCallback` / `useMemo` |
| Context value 变化 | 拆分 Context，使用 `useMemo` 包裹 value |
| 状态提升过高 | 状态下沉到需要的最近父组件 |
| 不稳定的 key | 使用稳定且唯一的 key |

---

## 五、React 原理与源码

### 21. React Fiber 架构是什么？解决了什么问题？

**问题背景：**
React 15 使用 Stack Reconciler（递归遍历），一旦开始渲染便无法中断。当组件树很深时，JS 长期占用主线程，导致页面卡顿、动画丢帧。

**Fiber 架构的核心思路：**

```
将递归的不可中断的更新 → 改为可中断的异步更新
```

**Fiber 是什么？**
- Fiber 是一个 JavaScript 对象，对应一个"工作单元"。
- 每个 React 元素都有对应的 Fiber 节点。
- Fiber 节点之间通过 `child`、`sibling`、`return` 形成链表结构。

```jsx
// Fiber 节点简化结构
{
  type: 'div',          // 组件类型
  key: null,            // key 属性
  stateNode: domNode,   // 对应的 DOM 节点
  child: childFiber,    // 第一个子 Fiber
  sibling: null,        // 下一个兄弟 Fiber
  return: parentFiber,  // 父 Fiber
  alternate: oldFiber,  // 指向旧 Fiber（双缓冲）
  pendingProps: {},     // 新的 props
  memoizedProps: {},    // 上一次的 props
  memoizedState: {},    // 上一次的 state
  effectTag: 'Update',  // 副作用标记
  lane: 0,              // 优先级
}
```

**双缓冲机制（Double Buffering）：**
- `current` 树：屏幕上显示的 Fiber 树。
- `workInProgress` 树：正在内存中构建的新 Fiber 树。
- 两棵树通过 `alternate` 属性互相引用。
- 构建完成后直接切换指针（ `current = workInProgress` ），无需销毁重建。

**工作流程：**

```
Scheduler 调度 → Render 阶段(可中断) → Commit 阶段(不可中断)

Render 阶段（找出变更）：
beginWork:  向下遍历，比较新旧 Fiber，标记 effectTag
completeWork: 向上归并，收集副作用链表

Commit 阶段（提交变更）：
beforeMutation → mutation → layout
```

**解决的问题：**
- 任务可中断、可恢复、可优先级调度。
- 高优先级更新（如用户输入）可打断低优先级更新（如数据渲染）。
- 避免主线程被长时间阻塞。

---

### 22. React 18 的 Concurrent Mode 是什么？

Concurrent Mode（并发模式）是 React 18 的核心特性，它是一个**底层架构改进**，支持渲染可中断。

**核心特性：**

**（1）自动批处理（Automatic Batching）**

```jsx
// React 17：只有在合成事件和生命周期中会批处理
setTimeout(() => {
  setCount(c => c + 1);  // 触发一次渲染
  setFlag(f => !f);      // 触发一次渲染
  // 总共 2 次渲染！
}, 0);

// React 18：所有更新都自动批处理
setTimeout(() => {
  setCount(c => c + 1);  // \
  setFlag(f => !f);      //  → 合并为 1 次渲染
}, 0);
```

**（2）Transitions（过渡更新）**

```jsx
import { useTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    // 紧急更新（输入框立即响应）
    setQuery(e.target.value);
    
    // 非紧急更新（可被中断）
    startTransition(() => {
      setSearchResults(filterResults(e.target.value));
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <Results />}
    </>
  );
}
```

**（3）Suspense 改进**

```jsx
// React 18 支持 Streaming SSR + Suspense
import { Suspense } from 'react';

function App() {
  return (
    <Suspense fallback={<Loader />}>
      <Comments />
      <Suspense fallback={<MiniLoader />}>
        <Sidebar />
      </Suspense>
    </Suspense>
  );
}
```

**（4）`useDeferredValue`**

```jsx
import { useDeferredValue } from 'react';

function SearchResults({ query }) {
  // 延迟更新，优先响应用户输入
  const deferredQuery = useDeferredValue(query);
  const results = useMemo(
    () => filterData(deferredQuery),
    [deferredQuery]
  );
  
  return <ResultList results={results} />;
}
```

---

### 23. React 的调度机制（Scheduler）

React 使用 `lane` 模型来管理更新的优先级：

```
Lane 优先级（从高到低）：
SyncLane（同步，如点击事件）
  → InputContinuousLane（连续输入）
    → DefaultLane（默认优先级）
      → TransitionLane（过渡优先级）
        → IdleLane（空闲时执行）
        → OffscreenLane（离屏/不可见）
```

**时间切片（Time Slicing）：**

```jsx
// 简化的工作循环
function workLoop(deadline) {
  let shouldYield = false;
  
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    // 检查剩余时间，如果不够就交还主线程
    shouldYield = deadline.timeRemaining() < 5;
  }
  
  if (!nextUnitOfWork) {
    // 所有工作完成，进入 Commit 阶段
    commitRoot();
  } else {
    // 还有工作，交给浏览器调度
    requestIdleCallback(workLoop); // 或 Scheduler 的 MessageChannel 实现
  }
}
```

**为什么不用 `requestIdleCallback`？**
- 浏览器兼容性不好（Safari 不支持）。
- 触发频率不稳定（每 50ms 最多一次）。
- React 自研了 Scheduler 包，基于 `MessageChannel` 实现。

---

### 24. React 的 Hooks 在源码层面是如何实现的？

```
Fiber 节点上存储 Hooks 的数据结构：

Fiber.memoizedState → Hook1 → Hook2 → Hook3 → null
                       ↓        ↓        ↓
                    useState  useEffect useState
```

```jsx
// Hook 对象的简化结构
{
  memoizedState: null,   // 当前状态值
  baseState: null,       // 基础状态
  queue: null,           // 更新队列（环状链表）
  next: nextHook,        // 指向下一个 Hook
}

// useState 更新队列节点的结构
{
  action: newValue,      // 要更新的值
  eagerState: null,      // 提前计算的新状态
  next: nextUpdate,      // 下一个更新
  priority: lane,        // 优先级
}
```

**为什么 Hooks 不能放在条件语句中？**

```jsx
// ❌ 错误：条件调用 Hook
function Component({ flag }) {
  if (flag) {
    const [name, setName] = useState('');  // Hook1 可能不存在
  }
  const [age, setAge] = useState(0);       // Hook2 的位置不固定
}

// 原因：React 依赖 Hook 的调用顺序来关联 Fiber 和 Hook 链表。
// 如果顺序变了，Hook2 可能拿到 Hook1 的状态，导致数据错乱。
```

**`useState` 的简单实现思路：**

```jsx
let workInProgressHook = null; // 当前工作的 Hook
let isMount = true;            // 是否是首次渲染

function useState(initial) {
  let hook;
  
  if (isMount) {
    // 首次渲染：创建 Hook 并挂载到链表
    hook = {
      memoizedState: initial,
      queue: { pending: null },
      next: null,
    };
    if (!workInProgressHook) {
      currentlyRenderingFiber.memoizedState = hook;
    } else {
      workInProgressHook.next = hook;
    }
    workInProgressHook = hook;
  } else {
    // 更新：从链表中取出对应 Hook
    hook = workInProgressHook;
    workInProgressHook = hook.next;
  }
  
  // 计算最新状态
  let baseState = hook.memoizedState;
  if (hook.queue.pending) {
    let firstUpdate = hook.queue.pending.next;
    let update = firstUpdate;
    do {
      baseState = typeof update.action === 'function'
        ? update.action(baseState)
        : update.action;
      update = update.next;
    } while (update !== firstUpdate);
  }
  
  hook.memoizedState = baseState;
  
  const dispatch = (action) => {
    // 创建 update 加入队列，触发调度
    const update = { action, next: null };
    // ... 加入环状更新队列
    scheduleUpdateOnFiber(currentlyRenderingFiber);
  };
  
  return [baseState, dispatch];
}
```

---

## 六、路由与数据流

### 25. React Router v6 的核心变化

```jsx
// v5 写法（旧）
<Switch>
  <Route exact path="/" component={Home} />
  <Route path="/about" component={About} />
  <Route path="/user/:id" component={User} />
</Switch>

// v6 写法（新）
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/about" element={<About />} />
  <Route path="/user/:id" element={<User />} />
</Routes>
```

**v6 核心变化：**

| 特性 | v5 | v6 |
|------|----|----|
| 路由匹配 | `<Switch>` + `exact` | `<Routes>`（自动精确匹配） |
| 组件渲染 | `component={Comp}` / `render={fn}` | `element={<Comp />}` |
| 嵌套路由 | 需要手动定义 | 使用 `<Outlet />` 占位 |
| 导航 | `useHistory()` | `useNavigate()` |
| 路由参数 | `useParams()` / `useRouteMatch()` | `useParams()` |
| 相对链接 | 不支持 | 支持 `<Link to="details">` |

**嵌套路由示例：**

```jsx
function App() {
  return (
    <Routes>
      <Route path="/dashboard" element={<Dashboard />}>
        <Route index element={<DashboardHome />} />
        <Route path="stats" element={<Stats />} />
        <Route path="settings" element={<Settings />} />
      </Route>
    </Routes>
  );
}

function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <nav>
        <Link to=".">Home</Link>          {/* 相对路径 */}
        <Link to="stats">Stats</Link>
      </nav>
      <Outlet /> {/* 子路由渲染位置 */}
    </div>
  );
}
```

---

### 26. 客户端渲染（CSR）与服务端渲染（SSR）的区别

| 维度 | CSR | SSR |
|------|-----|-----|
| 渲染位置 | 浏览器 | 服务端 |
| 首屏速度 | 慢（需下载 JS bundle） | 快（直接返回 HTML） |
| SEO | 差（爬虫可能不执行 JS） | 好（HTML 已就绪） |
| 服务器负载 | 低 | 高（每次请求都需要渲染） |
| 交互体验 | 快（后续页面切换无需刷新） | 首次加载快，但页面间切换较慢 |
| TTFB | 快（静态资源） | 慢（需等待渲染） |
| FCP | 慢 | 快 |

**Next.js 的混合方案：**

```jsx
// SSR（Server Side Rendering）
export async function getServerSideProps(context) {
  const data = await fetchData();
  return { props: { data } };
}

// SSG（Static Site Generation）
export async function getStaticProps() {
  const data = await fetchData();
  return { props: { data }, revalidate: 60 }; // ISR
}

// React 18 Server Components
// 默认是 Server Component（无法使用 hooks、事件处理）
async function ServerComponent() {
  const data = await db.query('SELECT * FROM users');
  return <UserList users={data} />;
}

// 需要交互的用 Client Component
'use client';
function ClientComponent() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

---

## 七、实战场景题

### 27. 实现一个高亮搜索关键词的组件

```jsx
function HighlightText({ text, keyword }) {
  if (!keyword.trim()) return <span>{text}</span>;

  // 转义特殊正则字符
  const escapedKeyword = keyword.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  const regex = new RegExp(`(${escapedKeyword})`, 'gi');
  const parts = text.split(regex);

  return (
    <span>
      {parts.map((part, index) =>
        regex.test(part) ? (
          <mark key={index} style={{ backgroundColor: '#FFEB3B' }}>
            {part}
          </mark>
        ) : (
          <span key={index}>{part}</span>
        )
      )}
    </span>
  );
}
```

**考察点：**
- 正则表达式的使用（边界情况处理）。
- `split` 和正则捕获组的配合。
- JSX 中动态渲染 `key` 的处理。

---

### 28. 实现一个可复用的 Modal 组件

```jsx
import { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';

function Modal({ isOpen, onClose, title, children }) {
  const overlayRef = useRef(null);

  // ESC 键关闭
  useEffect(() => {
    const handleEscape = (e) => {
      if (e.key === 'Escape') onClose();
    };
    
    if (isOpen) {
      document.addEventListener('keydown', handleEscape);
      document.body.style.overflow = 'hidden'; // 禁止页面滚动
    }
    
    return () => {
      document.removeEventListener('keydown', handleEscape);
      document.body.style.overflow = '';
    };
  }, [isOpen, onClose]);

  // 点击遮罩层关闭
  const handleOverlayClick = (e) => {
    if (e.target === overlayRef.current) onClose();
  };

  if (!isOpen) return null;

  // Portal 将 Modal 渲染到 body 下，避免 CSS 层级问题
  return createPortal(
    <div
      ref={overlayRef}
      onClick={handleOverlayClick}
      style={{
        position: 'fixed',
        inset: 0,
        backgroundColor: 'rgba(0,0,0,0.5)',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        zIndex: 1000,
      }}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      <div
        style={{
          background: '#fff',
          borderRadius: 8,
          padding: 24,
          minWidth: 400,
          maxWidth: '90vw',
        }}
      >
        <div style={{ display: 'flex', justifyContent: 'space-between' }}>
          <h2 id="modal-title">{title}</h2>
          <button onClick={onClose} aria-label="关闭">✕</button>
        </div>
        <div>{children}</div>
      </div>
    </div>,
    document.body
  );
}
```

**考察点：**
- Portal 的使用和原理。
- 事件处理（ESC 关闭、点击遮罩关闭）。
- 无障碍访问（`role`、`aria-modal`、`aria-labelledby`）。
- 副作用清理。

---

### 29. 实现一个防抖/节流的搜索输入框

```jsx
import { useState, useEffect, useRef, useCallback } from 'react';

// 防抖 Hook
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// 节流 Hook
function useThrottle(value, delay) {
  const [throttledValue, setThrottledValue] = useState(value);
  const lastRan = useRef(Date.now());

  useEffect(() => {
    const handler = setTimeout(() => {
      if (Date.now() - lastRan.current >= delay) {
        setThrottledValue(value);
        lastRan.current = Date.now();
      }
    }, delay - (Date.now() - lastRan.current));

    return () => clearTimeout(handler);
  }, [value, delay]);

  return throttledValue;
}

// 搜索组件
function SearchBox() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const debouncedQuery = useDebounce(query, 300);
  const abortControllerRef = useRef(null);

  // 请求取消 + 竞态处理
  useEffect(() => {
    if (!debouncedQuery.trim()) {
      setResults([]);
      return;
    }

    // 取消上一次请求
    abortControllerRef.current?.abort();
    abortControllerRef.current = new AbortController();

    setLoading(true);
    fetch(`/api/search?q=${debouncedQuery}`, {
      signal: abortControllerRef.current.signal,
    })
      .then(res => res.json())
      .then(data => {
        setResults(data);
        setLoading(false);
      })
      .catch(err => {
        if (err.name !== 'AbortError') {
          setLoading(false);
        }
      });
  }, [debouncedQuery]);

  // 高亮关键词
  const highlightMatch = useCallback((text) => {
    const regex = new RegExp(`(${debouncedQuery})`, 'gi');
    const parts = text.split(regex);
    return parts.map((part, i) =>
      regex.test(part) ? <mark key={i}>{part}</mark> : part
    );
  }, [debouncedQuery]);

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="搜索..."
      />
      {loading && <div>搜索中...</div>}
      <ul>
        {results.map(item => (
          <li key={item.id}>
            {highlightMatch(item.title)}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

**考察点：**
- 防抖（输入停止后才触发）vs 节流（固定频率触发）的区别。
- 请求竞态处理（AbortController 取消过期请求）。
- 多种 Hook 组合使用。

---

### 30. 如何设计一个大型 React 项目的目录结构？

```
src/
├── assets/              # 静态资源（图片、字体、CSS）
├── components/          # 通用组件
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx
│   │   ├── Button.module.css
│   │   └── index.ts     # 统一导出
│   ├── Modal/
│   └── Table/
├── features/            # 业务功能模块（按领域拆分）
│   ├── auth/
│   │   ├── components/  # 模块专用组件
│   │   ├── hooks/       # 模块专用 Hooks
│   │   ├── services/    # API 调用
│   │   ├── store/       # 模块状态
│   │   ├── types/       # 类型定义
│   │   └── utils/       # 模块工具函数
│   ├── dashboard/
│   └── orders/
├── hooks/               # 全局共享 Hooks
├── services/            # 全局 API 服务
├── store/               # 全局状态管理
├── types/               # 全局类型定义
├── utils/               # 全局工具函数
├── layouts/             # 布局组件
├── pages/               # 页面组件（路由入口）
├── routes/              # 路由配置
└── constants/           # 常量定义
```

**设计原则：**
- **按功能/领域拆分**，而非按文件类型拆分。
- **就近原则**：组件相关的样式、测试、类型放在同一个文件夹下。
- **桶模式（Barrel Export）**：每个文件夹使用 `index.ts` 统一导出。
- **显式优于隐式**：`import { Button } from '@/components/Button'` 比 `'@/components'` 更好（Tree Shaking 优化）。

---

## 八、TypeScript + React

### 31. React 中常用的 TypeScript 类型

```tsx
// 1. Props 类型
interface ButtonProps {
  children: React.ReactNode;
  onClick?: (event: React.MouseEvent<HTMLButtonElement>) => void;
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  disabled?: boolean;
}

function Button({ children, onClick, variant = 'primary' }: ButtonProps) {
  return <button onClick={onClick}>{children}</button>;
}

// 2. 事件类型
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  console.log(e.target.value);
};

const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
};

const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
  console.log(e.currentTarget);
};

// 3. useRef 的类型
const inputRef = useRef<HTMLInputElement>(null);
const timerRef = useRef<ReturnType<typeof setInterval> | null>(null);

// 4. useState 的类型
const [user, setUser] = useState<User | null>(null);
const [status, setStatus] = useState<'idle' | 'loading' | 'success' | 'error'>('idle');

// 5. 泛型组件
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>
          {renderItem(item, index)}
        </li>
      ))}
    </ul>
  );
}

// 使用
<List
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
  keyExtractor={(user) => user.id}
/>

// 6. 提取组件 Props 类型
type ButtonReactProps = React.ComponentProps<typeof Button>;
type DivNativeProps = React.ComponentPropsWithoutRef<'div'>;

// 7. ReactNode vs ReactElement vs JSX.Element
// ReactNode：最宽泛，可以渲染的所有类型
type Node = React.ReactNode; 
// ReactElement：具体的一个 JSX 元素
type Element = React.ReactElement;
// JSX.Element：等同于 ReactElement
type JSXElement = JSX.Element;
```

---

### 32. 常见错误处理和边界情况

**（1）useEffect 异步函数处理：**

```tsx
// ❌ 错误：useEffect 不能直接传 async 函数
useEffect(async () => {
  const data = await fetchData();
}, []);

// ✅ 正确：内部定义并调用
useEffect(() => {
  const loadData = async () => {
    try {
      const data = await fetchData();
      setData(data);
    } catch (err) {
      setError(err);
    }
  };
  loadData();
}, []);
```

**（2）空值安全渲染：**

```tsx
function UserProfile({ user }: { user: User | null | undefined }) {
  // 早期返回
  if (!user) return <div>用户不存在</div>;
  
  // 可选链 + 空值合并
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.profile?.bio ?? '暂无简介'}</p>
      <img src={user.avatar ?? '/default-avatar.png'} alt={user.name} />
    </div>
  );
}
```

**（3）数组安全的 map 渲染：**

```tsx
function ItemList({ items }: { items?: Item[] }) {
  return (
    <ul>
      {items?.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
      {(!items || items.length === 0) && (
        <li>暂无数据</li>
      )}
    </ul>
  );
}
```

---

## 九、组件设计模式

### 33. 高阶组件（HOC）的原理和常见用途

高阶组件（Higher-Order Component）是一个**函数**，接收一个组件作为参数，返回一个新的增强组件。

```jsx
// HOC 的基本形式
function withEnhancement(WrappedComponent) {
  return function EnhancedComponent(props) {
    // 注入逻辑... 返回增强后的组件
    return <WrappedComponent {...props} extraProp="value" />;
  };
}
```

**常见用途：**

```jsx
// 1. 权限控制 HOC
function withAuth(WrappedComponent) {
  return function AuthComponent(props) {
    const { isAuthenticated, user } = useAuth();

    if (!isAuthenticated) {
      return <Navigate to="/login" replace />;
    }

    return <WrappedComponent {...props} user={user} />;
  };
}

// 使用
const ProtectedDashboard = withAuth(Dashboard);

// 2. 日志记录 HOC
function withLogging(WrappedComponent, componentName) {
  return function LoggingComponent(props) {
    useEffect(() => {
      console.log(`${componentName} mounted`);
      return () => console.log(`${componentName} unmounted`);
    }, []);

    useEffect(() => {
      console.log(`${componentName} updated`, props);
    });

    return <WrappedComponent {...props} />;
  };
}

// 3. 数据获取 HOC
function withData(WrappedComponent, fetchFn) {
  return function DataComponent(props) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
      fetchFn(props).then(result => {
        setData(result);
        setLoading(false);
      });
    }, []);

    if (loading) return <Loading />;

    return <WrappedComponent {...props} data={data} />;
  };
}
```

**注意事项：**

```jsx
// ⚠️ 1. 必须透传 ref（使用 React.forwardRef）
function withSomething(WrappedComponent) {
  function WithSomething(props, ref) {
    return <WrappedComponent {...props} ref={ref} />;
  }
  return React.forwardRef(WithSomething);
}

// ⚠️ 2. 复制静态方法
function withSomething(WrappedComponent) {
  class WithSomething extends React.Component { /* ... */ }
  hoistNonReactStatic(WithSomething, WrappedComponent); // 使用 hoist-non-react-statics
  return WithSomething;
}

// ⚠️ 3. 设置 displayName 方便调试
function withSomething(WrappedComponent) {
  function WithSomething(props) {
    return <WrappedComponent {...props} />;
  }
  WithSomething.displayName = `WithSomething(${
    WrappedComponent.displayName || WrappedComponent.name || 'Component'
  })`;
  return WithSomething;
}
```

**HOC vs Hooks：**

| 维度 | HOC | Hooks |
|------|-----|-------|
| 复用范围 | 任何组件 | 函数组件 |
| 嵌套层级 | 多层 HOC 会产生"包装地狱" | 无嵌套，直接调用 |
| ref 传递 | 需要 `forwardRef` | 直接使用 `useRef` |
| Props 冲突 | 可能发生命名冲突 | 自行管理，不注入 props |

---

### 34. Render Props 模式

Render Props 是指通过一个**值为函数的 prop** 来共享代码的组件模式。

```jsx
// 经典 Render Props：共享鼠标位置
class MouseTracker extends React.Component {
  state = { x: 0, y: 0 };

  handleMouseMove = (e) => {
    this.setState({ x: e.clientX, y: e.clientY });
  };

  render() {
    return (
      <div onMouseMove={this.handleMouseMove} style={{ height: '100vh' }}>
        {/* 调用 render prop，将状态传给子组件 */}
        {this.props.render(this.state)}
      </div>
    );
  }
}

// 使用
function App() {
  return (
    <MouseTracker
      render={({ x, y }) => (
        <h1>鼠标位置: ({x}, {y})</h1>
      )}
    />
  );
}
```

**Render Props vs HOC vs Hooks（同一需求三种写法）：**

```jsx
// 需求：获取窗口大小并共享

// 1. HOC 写法
function withWindowSize(Component) {
  return function (props) {
    const [size, setSize] = useState({ width: window.innerWidth, height: window.innerHeight });
    useEffect(() => {
      const handler = () => setSize({ width: window.innerWidth, height: window.innerHeight });
      window.addEventListener('resize', handler);
      return () => window.removeEventListener('resize', handler);
    }, []);
    return <Component {...props} windowSize={size} />;
  };
}
// 使用: export default withWindowSize(MyComponent)

// 2. Render Props 写法
function WindowSize({ children }) {
  const [size, setSize] = useState({ width: window.innerWidth, height: window.innerHeight });
  useEffect(() => {
    const handler = () => setSize({ width: window.innerWidth, height: window.innerHeight });
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);
  return <>{children(size)}</>;
}
// 使用: <WindowSize>{size => <div>...</div>}</WindowSize>

// 3. 自定义 Hook 写法（✅ 最简洁）
function useWindowSize() {
  const [size, setSize] = useState({ width: window.innerWidth, height: window.innerHeight });
  useEffect(() => {
    const handler = () => setSize({ width: window.innerWidth, height: window.innerHeight });
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);
  return size;
}
// 使用: const size = useWindowSize();
```

---

### 35. Compound Components（组合组件模式）

Compound Components 通过隐式共享状态，让多个子组件协同工作，提供灵活的 API。

```jsx
// 经典例子：自定义 Select 组件
const SelectContext = React.createContext();

function Select({ children, onChange, defaultValue }) {
  const [selectedValue, setSelectedValue] = useState(defaultValue);
  const [isOpen, setIsOpen] = useState(false);

  const contextValue = useMemo(() => ({
    selectedValue,
    isOpen,
    select: (value) => {
      setSelectedValue(value);
      onChange?.(value);
      setIsOpen(false);
    },
    toggle: () => setIsOpen(prev => !prev),
  }), [selectedValue, isOpen, onChange]);

  return (
    <SelectContext.Provider value={contextValue}>
      <div className="select">{children}</div>
    </SelectContext.Provider>
  );
}

// 子组件通过 Context 共享状态
Select.Trigger = function SelectTrigger({ children }) {
  const { selectedValue, toggle } = useContext(SelectContext);
  return (
    <button onClick={toggle} className="select-trigger">
      {children || selectedValue || '请选择'}
    </button>
  );
};

Select.Options = function SelectOptions({ children }) {
  const { isOpen } = useContext(SelectContext);
  if (!isOpen) return null;
  return <ul className="select-options">{children}</ul>;
};

Select.Option = function SelectOption({ value, children }) {
  const { selectedValue, select } = useContext(SelectContext);
  return (
    <li
      className={`select-option ${value === selectedValue ? 'selected' : ''}`}
      onClick={() => select(value)}
    >
      {children}
    </li>
  );
};

// 使用：高度灵活的 API
function App() {
  return (
    <Select defaultValue="react" onChange={console.log}>
      <Select.Trigger />
      <Select.Options>
        <Select.Option value="react">React</Select.Option>
        <Select.Option value="vue">Vue</Select.Option>
        <Select.Option value="angular">Angular</Select.Option>
      </Select.Options>
    </Select>
  );
}
```

**优点：**
- 子组件组合方式由使用者决定（UI 布局灵活）。
- 无需通过 props 层层传递配置。
- 每个子组件职责单一，易于维护。

**适用场景：** Tabs、Accordion、Menu、Dropdown、Form 等需要多个子组件协同工作的复合组件。

---

### 36. Provider Pattern（提供者模式）和 Context 进阶

```jsx
// 多级 Provider 组合
function AppProviders({ children }) {
  return (
    <ThemeProvider>
      <AuthProvider>
        <NotificationProvider>
          <QueryClientProvider client={queryClient}>
            {children}
          </QueryClientProvider>
        </NotificationProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}

// 自定义 Provider + useReducer 实现复杂状态管理
const AuthContext = createContext(null);

function AuthProvider({ children }) {
  const [state, dispatch] = useReducer(authReducer, initialState);

  const value = useMemo(() => ({
    user: state.user,
    isAuthenticated: state.isAuthenticated,
    login: async (credentials) => {
      dispatch({ type: 'LOGIN_START' });
      try {
        const user = await api.login(credentials);
        dispatch({ type: 'LOGIN_SUCCESS', payload: user });
      } catch (error) {
        dispatch({ type: 'LOGIN_FAILURE', payload: error.message });
      }
    },
    logout: () => dispatch({ type: 'LOGOUT' }),
  }), [state]);

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

// 封装 useAuth Hook（避免暴露 context 创建细节）
function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}
```

---

### 37. Presentational vs Container Components（展示/容器组件分离）

```jsx
// ========== 容器组件（Container）：负责数据和逻辑 ==========
function UserListContainer() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchUsers()
      .then(setUsers)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);

  const handleDelete = (id) => {
    deleteUser(id).then(() => {
      setUsers(prev => prev.filter(u => u.id !== id));
    });
  };

  // 将数据和回调传给展示组件
  return (
    <UserList
      users={users}
      loading={loading}
      error={error}
      onDelete={handleDelete}
    />
  );
}

// ========== 展示组件（Presentational）：只负责渲染 ==========
function UserList({ users, loading, error, onDelete }) {
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error.message} />;
  if (users.length === 0) return <EmptyState />;

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>
          <span>{user.name}</span>
          <button onClick={() => onDelete(user.id)}>删除</button>
        </li>
      ))}
    </ul>
  );
}
```

**好处：**
- 展示组件可复用（纯渲染，容易测试）。
- 关注点分离：逻辑和 UI 解耦。
- 便于 Storybook 等工具管理组件。

> **注意**：随着 Hooks 的普及，自定义 Hook + 函数组件可以兼顾两者优势，不必严格区分。但核心思想（关注点分离）仍然适用。

---

## 十、React 测试策略

### 38. React 测试的层级和工具选择

**测试金字塔：**

```
          /\
         /E2E\          ← Playwright / Cypress（少量，最慢）
        /------\
       /集成测试\        ← React Testing Library（适量，适中）
      /----------\
     /  单元测试  \      ← Jest / Vitest（大量，最快）
    /--------------\
```

**工具对比：**

| 工具 | 类型 | 特点 |
|------|------|------|
| **Jest** | 单元测试 | 快照测试、Mock 系统、覆盖率报告 |
| **Vitest** | 单元测试 | 更快的 Jest 替代品，原生 ESM 支持 |
| **React Testing Library** | 集成测试 | 模拟用户行为测试，不测试实现细节 |
| **Playwright** | E2E 测试 | 跨浏览器、自动等待、录屏回放 |
| **Cypress** | E2E 测试 | 实时重载、时间旅行调试 |

**React Testing Library 核心理念：**

```jsx
// ❌ 测试实现细节（脆弱，重构容易出错）
test('button has correct class', () => {
  render(<Button />);
  expect(screen.getByRole('button')).toHaveClass('btn-primary');
});

// ✅ 测试用户行为（稳定的测试）
test('button triggers callback on click', async () => {
  const onClick = jest.fn();
  render(<Button onClick={onClick}>提交</Button>);

  await userEvent.click(screen.getByRole('button', { name: /提交/ }));
  expect(onClick).toHaveBeenCalledTimes(1);
});
```

---

### 39. 常见测试场景实战

```jsx
import { render, screen, waitFor, within } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

// ===== 1. 表单提交测试 =====
function LoginForm({ onSubmit }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    onSubmit({ email, password });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        placeholder="邮箱"
        value={email}
        onChange={e => setEmail(e.target.value)}
      />
      <input
        type="password"
        placeholder="密码"
        value={password}
        onChange={e => setPassword(e.target.value)}
      />
      <button type="submit">登录</button>
    </form>
  );
}

// 测试
test('submits form with email and password', async () => {
  const onSubmit = jest.fn();

  render(<LoginForm onSubmit={onSubmit} />);

  await userEvent.type(screen.getByPlaceholderText('邮箱'), 'test@example.com');
  await userEvent.type(screen.getByPlaceholderText('密码'), 'password123');
  await userEvent.click(screen.getByRole('button', { name: /登录/ }));

  expect(onSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'password123',
  });
});

// ===== 2. 异步数据加载测试 =====
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    fetch(`/api/users/${userId}`)
      .then(res => {
        if (!res.ok) throw new Error('Failed');
        return res.json();
      })
      .then(data => {
        setUser(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, [userId]);

  if (loading) return <div>加载中...</div>;
  if (error) return <div role="alert">{error}</div>;
  return <div>{user.name}</div>;
}

// 测试
test('shows loading then user name', async () => {
  // Mock fetch
  global.fetch = jest.fn(() =>
    Promise.resolve({
      ok: true,
      json: () => Promise.resolve({ id: 1, name: '张三' }),
    })
  );

  render(<UserProfile userId={1} />);

  // 先看到 loading 状态
  expect(screen.getByText('加载中...')).toBeInTheDocument();

  // 等数据加载完成后验证
  await waitFor(() => {
    expect(screen.getByText('张三')).toBeInTheDocument();
  });
});

// ===== 3. 自定义 Hook 测试 =====
// renderHook 来自 @testing-library/react
import { renderHook, act } from '@testing-library/react';

function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  const reset = () => setCount(initial);
  return { count, increment, decrement, reset };
}

test('useCounter should increment and decrement', () => {
  const { result } = renderHook(() => useCounter(10));

  expect(result.current.count).toBe(10);

  act(() => result.current.increment());
  expect(result.current.count).toBe(11);

  act(() => result.current.decrement());
  expect(result.current.count).toBe(10);

  act(() => {
    result.current.increment();
    result.current.increment();
  });
  expect(result.current.count).toBe(12);

  act(() => result.current.reset());
  expect(result.current.count).toBe(10);
});

// ===== 4. 列表渲染测试 =====
test('renders list of todos and handles delete', async () => {
  const todos = [
    { id: 1, text: '买菜' },
    { id: 2, text: '写代码' },
    { id: 3, text: '跑步' },
  ];
  const onDelete = jest.fn();

  render(<TodoList todos={todos} onDelete={onDelete} />);

  // 验证所有项都存在
  const listItems = screen.getAllByRole('listitem');
  expect(listItems).toHaveLength(3);

  // 使用 within 限定查找范围
  const secondItem = listItems[1];
  await userEvent.click(within(secondItem).getByRole('button', { name: /删除/ }));

  expect(onDelete).toHaveBeenCalledWith(2);
});
```

**测试原则（React Testing Library 哲学）：**
1. 按用户使用方式测试，不测试实现细节。
2. 基于可访问性选择元素（`getByRole` > `getByLabelText` > `getByText` > `getByTestId`）。
3. 使用 `userEvent` 而非 `fireEvent`（更接近真实用户行为）。
4. 测试行为（发生了什么），不测试状态（内部是什么）。

---

## 十一、React 19 新特性

### 40. React 19 有哪些重要更新？

**（1）React Server Components（RSC）正式集成**

```jsx
// Server Component（默认）：无状态、无交互，可访问数据库
async function NoteList({ searchParams }) {
  // 直接在服务端查询数据库
  const notes = await db.note.findMany({
    where: { title: { contains: searchParams.q } },
  });
  return (
    <ul>
      {notes.map(note => (
        <NoteItem key={note.id} note={note} />
      ))}
    </ul>
  );
}

// Client Component：有状态、有交互
'use client';
function LikeButton({ initialLikes }) {
  const [likes, setLikes] = useState(initialLikes);
  return <button onClick={() => setLikes(l => l + 1)}>{likes} 赞</button>;
}
```

**（2）Actions（表单操作）**

```jsx
// 传统方式
function Form() {
  const [loading, setLoading] = useState(false);
  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    try {
      await updateProfile(formData);
    } finally {
      setLoading(false);
    }
  };
}

// React 19 Actions 方式
async function updateProfileAction(formData) {
  'use server';
  const name = formData.get('name');
  await db.user.update({ name });
  revalidatePath('/profile'); // 自动刷新页面数据
}

function Form() {
  return (
    <form action={updateProfileAction}>
      <input name="name" required />
      <button type="submit">保存</button>
    </form>
  );
}
```

**（3）`useOptimistic` — 乐观更新**

```jsx
import { useOptimistic } from 'react';

function TodoList({ initialTodos }) {
  const [todos, setTodos] = useState(initialTodos);
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (state, newTodo) => [...state, { ...newTodo, sending: true }]
  );

  async function handleAdd(formData) {
    // 立即显示（乐观更新）
    addOptimistic({ text: formData.get('text') });

    // 实际上发请求
    const newTodo = await createTodo(formData);
    setTodos(prev => [...prev, newTodo]);
  }

  return (
    <>
      <form action={handleAdd}>
        <input name="text" />
        <button type="submit">添加</button>
      </form>
      {optimisticTodos.map(todo => (
        <div key={todo.id} className={todo.sending ? 'opacity-50' : ''}>
          {todo.text}
        </div>
      ))}
    </>
  );
}
```

**（4）`useFormStatus` — 表单状态**

```jsx
'use client';
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? '提交中...' : '提交'}
    </button>
  );
}
```

**（5）`use` API — 在渲染中读取 Promise / Context**

```jsx
import { use } from 'react';

// 在组件渲染过程中读取 Promise（不需要 Suspense）
function UserProfile({ userPromise }) {
  const user = use(userPromise); // 直接消费 Promise
  return <div>{user.name}</div>;
}

// 也可以读取 Context（跳过 useContext 的限制）
function ThemedButton() {
  const theme = use(ThemeContext); // 可以在条件语句中使用！
  return <button className={theme}>Click</button>;
}
```

**（6）`ref` 作为普通 prop**

```jsx
// React 18：需要 forwardRef
const Input = React.forwardRef((props, ref) => (
  <input {...props} ref={ref} />
));

// React 19：ref 是普通 prop，不需要 forwardRef
function Input({ ref, ...props }) {
  return <input {...props} ref={ref} />;
}
```

**（7）Document Metadata（原生支持 `<title>`、`<meta>`）**

```jsx
// React 19：在组件中直接写 title、meta 标签即可生效
function BlogPost({ post }) {
  return (
    <article>
      <title>{post.title} - My Blog</title>
      <meta name="description" content={post.excerpt} />
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}
```

---

### 41. React 19 与 React 18 的关键差异总结

| 特性 | React 18 | React 19 |
|------|----------|----------|
| ref 传递 | 必须用 `forwardRef` | 直接作为普通 prop |
| `<title>` 等标签 | 需要 react-helmet 等库 | 原生支持 |
| 表单处理 | 手动管理 state + loading | Actions + `useFormStatus` |
| 乐观更新 | 手动实现 | `useOptimistic` |
| Context | 不能在条件中使用 | `use()` API 支持条件使用 |
| Promise 读取 | 只能在 useEffect 中 | `use()` API 直接读取 |
| Server Components | 实验性（Next.js App Router） | 正式标准化 |

---

## 十二、常见陷阱与高阶题

### 42. 为什么 React 中列表的 map 需要 key？

```jsx
// ❌ 没有 key 的问题场景
function TodoList({ todos }) {
  const [list, setList] = useState(todos);

  const addToTop = () => {
    setList([{ id: Date.now(), text: '新任务' }, ...list]);
  };

  return (
    <ul>
      {list.map((todo, index) => (
        // 用 index 作 key：新项目插入头部时，所有 DOM 都会错误复用
        <li key={index}>
          <input type="checkbox" />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
// 问题：如果第一个 item 的 checkbox 已勾选，
// 用 index 作 key 时，插入新头部后，checkbox 状态会错误地保留在"新第一项"上。
```

**key 的工作机制：**

```
旧 VDOM 子节点：      [A(key=0), B(key=1), C(key=2)]
插入头部后新 VDOM：    [D(key=0), A(key=1), B(key=2), C(key=3)]
React 认为：A→D(update), B→A(update), C→B(update), C(insert)
实际应做的事情：D(insert) → 复用 A, B, C
```

**key 的正确选择：**

```jsx
// ✅ 使用唯一稳定的 ID
{list.map(todo => (
  <li key={todo.id}>{todo.text}</li>
))}

// ✅ 如果数据没有 ID，可以自己生成
{list.map((todo, index) => (
  <li key={`${todo.text}-${index}`}>{todo.text}</li>
))}
// ⚠️ 注意：这仍然有隐患，最好让后端提供唯一 ID
```

---

### 43. `useEffect` 依赖数组的陷阱

```jsx
// 陷阱 1：遗漏依赖 — 闭包过期
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // 永远打印初始值 0（闭包陷阱）
    }, 1000);
    return () => clearInterval(id);
  }, []); // ESLint 会警告：missing dependency: count

  // 正确做法：使用函数式更新
  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1); // 不需要 count 在依赖里
    }, 1000);
    return () => clearInterval(id);
  }, []);
}

// 陷阱 2：对象/数组作为依赖 — 无限循环
function SearchResults({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    fetchResults(query).then(setResults);
  }, [query]); // ✅ 原始值，稳定对比
  // ❌ 如果 query 是对象 { search: 'react' }，每次渲染都是新对象 → 无限循环
}

// 陷阱 3：函数作为依赖
function Parent() {
  const handleClick = () => console.log('click'); // 每次渲染都是新函数

  return <Child onClick={handleClick} />;
}
// 子组件如果 useEffect 依赖了 onClick，会导致效果频繁执行
// 解决：useCallback 稳定引用
```

---

### 44. 如何安全地处理 dangerouslySetInnerHTML？

```jsx
import DOMPurify from 'dompurify';

function SafeHTML({ html }) {
  // ⚠️ 必须使用 DOMPurify 消毒 HTML，防止 XSS 攻击
  const sanitizedHTML = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br', 'ul', 'li'],
    ALLOWED_ATTR: ['href', 'target', 'rel'],
  });

  return (
    <div
      dangerouslySetInnerHTML={{ __html: sanitizedHTML }}
    />
  );
}

// 使用
<SafeHTML html="<b>Hello</b><script>alert('XSS')</script>" />
// 输出: <b>Hello</b>（script 标签被移除）
```

**XSS 防范清单：**
- `dangerouslySetInnerHTML` 必须配合 sanitizer 使用（推荐 DOMPurify）。
- 不要在 JSX `{}` 中渲染用户输入作为 URL（`href="javascript:..."`）。
- 使用 `rel="noopener noreferrer"` 配合 `target="_blank"`。
- CSP（Content Security Policy）作为最后一道防线。

---

### 45. 什么是 Strict Mode？为什么开发环境组件会渲染两次？

```jsx
// index.js
<React.StrictMode>
  <App />
</React.StrictMode>
```

**Strict Mode 的双重渲染：**

```jsx
// React 18 在开发模式下，StrictMode 会：
// 1. 组件函数体执行两次
// 2. useState 的初始化函数执行两次
// 3. useEffect 执行两次（mount → cleanup → mount）

function Counter() {
  console.log('render'); // 开发模式下会打印两次！

  const [count, setCount] = useState(() => {
    console.log('initializer'); // 开发模式下会打印两次！
    return 0;
  });

  useEffect(() => {
    console.log('effect mount');
    return () => console.log('effect cleanup');
  }, []);

  return <div>{count}</div>;
}
```

**为什么这样设计？**
- 帮助发现副作用没有正确清理的问题。
- 确保组件是"纯函数"（相同输入产生相同输出）。
- 模拟组件的卸载→重新挂载场景。

**重要：** 双重渲染只在**开发模式**下发生，生产构建中不会有此行为。

---

### 46. 如何实现组件的"惰性初始化"？

```jsx
// ❌ 每次渲染都创建新数组（即使不被使用）
function Component() {
  const [items, setItems] = useState(
    expensiveInitialCalculation() // 每次渲染都调用！
  );
}

// ✅ 传入函数，只在首次渲染时调用
function Component() {
  const [items, setItems] = useState(() => {
    // 惰性初始化：只在首次渲染时计算
    return expensiveInitialCalculation();
  });
}

// 另一个例子：从 localStorage 恢复状态
function usePersistedState(key, defaultValue) {
  const [state, setState] = useState(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : defaultValue;
    } catch {
      return defaultValue;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(state));
  }, [key, state]);

  return [state, setState];
}
```

---

### 47. React 中如何实现 Keep-Alive（缓存组件状态）？

```jsx
// 需求：切换 Tab 时保留 Tab 的内容状态，不销毁重建

// 方案1：CSS 显隐（简单但 DOM 不释放）
function TabContainer() {
  const [activeTab, setActiveTab] = useState(0);

  return (
    <div>
      <nav>
        {tabs.map((tab, i) => (
          <button key={i} onClick={() => setActiveTab(i)}>{tab.label}</button>
        ))}
      </nav>
      {tabs.map((tab, i) => (
        <div key={i} style={{ display: i === activeTab ? 'block' : 'none' }}>
          <tab.Component /> {/* 所有组件都存在 DOM 中 */}
        </div>
      ))}
    </div>
  );
}

// 方案2：状态提升（推荐）
function TabContainer() {
  const [activeTab, setActiveTab] = useState(0);
  // 把状态提升到父组件，切换 Tab 只是为了切换展示
  const [tab1State, setTab1State] = useState({});
  const [tab2State, setTab2State] = useState({});

  return (
    <div>
      <nav>
        {tabs.map((tab, i) => (
          <button key={i} onClick={() => setActiveTab(i)}>{tab.label}</button>
        ))}
      </nav>
      {activeTab === 0 && <Tab1 state={tab1State} onChange={setTab1State} />}
      {activeTab === 1 && <Tab2 state={tab2State} onChange={setTab2State} />}
    </div>
  );
}

// 方案3：第三方库（react-activation / react-keep-alive）
// 适合复杂场景，在 VDOM 层面实现缓存
import KeepAlive from 'react-activation';

function TabContainer() {
  const [activeTab, setActiveTab] = useState(0);

  return (
    <div>
      <nav>{/* Tab 按钮 */}</nav>
      <KeepAlive>
        {activeTab === 0 && <Tab1 />}
        {activeTab === 1 && <Tab2 />}
      </KeepAlive>
    </div>
  );
}
```

---

## 十三、系统设计题

### 48. 设计一个大型表单方案（动态表单）

```jsx
// 表单配置驱动（JSON Schema）
const formSchema = {
  fields: [
    {
      name: 'username',
      label: '用户名',
      type: 'text',
      required: true,
      validators: [
        { rule: 'minLength', value: 3, message: '最少 3 个字符' },
        { rule: 'maxLength', value: 20, message: '最多 20 个字符' },
      ],
    },
    {
      name: 'email',
      label: '邮箱',
      type: 'email',
      required: true,
      validators: [
        { rule: 'pattern', value: /^[^\s@]+@[^\s@]+\.[^\s@]+$/, message: '邮箱格式不正确' },
      ],
    },
    {
      name: 'role',
      label: '角色',
      type: 'select',
      required: true,
      options: [
        { label: '管理员', value: 'admin' },
        { label: '编辑者', value: 'editor' },
        { label: '访客', value: 'viewer' },
      ],
    },
    {
      name: 'bio',
      label: '个人简介',
      type: 'textarea',
    },
    {
      name: 'skills',
      label: '技能',
      type: 'array', // 动态增删字段
      itemType: {
        type: 'text',
      },
    },
  ],
};

// 通用 DynamicForm 组件
function DynamicForm({ schema, onSubmit, initialValues = {} }) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});

  const validate = useCallback((name, value) => {
    const field = schema.fields.find(f => f.name === name);
    if (!field?.validators) return '';

    for (const { rule, value: ruleValue, message } of field.validators) {
      if (rule === 'required' && !value) return message;
      if (rule === 'minLength' && value.length < ruleValue) return message;
      if (rule === 'maxLength' && value.length > ruleValue) return message;
      if (rule === 'pattern' && !ruleValue.test(value)) return message;
    }
    return '';
  }, [schema.fields]);

  const handleChange = (name, value) => {
    setValues(prev => ({ ...prev, [name]: value }));
    if (touched[name]) {
      setErrors(prev => ({ ...prev, [name]: validate(name, value) }));
    }
  };

  const handleBlur = (name) => {
    setTouched(prev => ({ ...prev, [name]: true }));
    setErrors(prev => ({ ...prev, [name]: validate(name, values[name]) }));
  };

  // 数组字段处理
  const handleArrayChange = (name, index, value) => {
    setValues(prev => {
      const arr = [...(prev[name] || [])];
      arr[index] = value;
      return { ...prev, [name]: arr };
    });
  };

  const handleArrayAdd = (name) => {
    setValues(prev => ({ ...prev, [name]: [...(prev[name] || []), ''] }));
  };

  const handleArrayRemove = (name, index) => {
    setValues(prev => ({
      ...prev,
      [name]: (prev[name] || []).filter((_, i) => i !== index),
    }));
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    // 全量校验
    const newErrors = {};
    schema.fields.forEach(field => {
      const error = validate(field.name, values[field.name]);
      if (error) newErrors[field.name] = error;
    });
    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      setTouched(Object.keys(newErrors).reduce((acc, k) => ({ ...acc, [k]: true }), {}));
      return;
    }
    onSubmit(values);
  };

  const renderField = (field) => {
    switch (field.type) {
      case 'select':
        return (
          <select
            value={values[field.name] || ''}
            onChange={e => handleChange(field.name, e.target.value)}
            onBlur={() => handleBlur(field.name)}
          >
            <option value="">请选择</option>
            {field.options?.map(opt => (
              <option key={opt.value} value={opt.value}>{opt.label}</option>
            ))}
          </select>
        );

      case 'textarea':
        return (
          <textarea
            value={values[field.name] || ''}
            onChange={e => handleChange(field.name, e.target.value)}
            onBlur={() => handleBlur(field.name)}
          />
        );

      case 'array':
        return (
          <div>
            {(values[field.name] || []).map((item, index) => (
              <div key={index} style={{ display: 'flex', gap: 8 }}>
                <input
                  value={item}
                  onChange={e => handleArrayChange(field.name, index, e.target.value)}
                  placeholder={`${field.label} ${index + 1}`}
                />
                <button type="button" onClick={() => handleArrayRemove(field.name, index)}>
                  删除
                </button>
              </div>
            ))}
            <button type="button" onClick={() => handleArrayAdd(field.name)}>
              + 添加{field.label}
            </button>
          </div>
        );

      default:
        return (
          <input
            type={field.type}
            value={values[field.name] || ''}
            onChange={e => handleChange(field.name, e.target.value)}
            onBlur={() => handleBlur(field.name)}
          />
        );
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {schema.fields.map(field => (
        <div key={field.name} className="form-field">
          <label>
            {field.label}
            {field.required && <span className="required">*</span>}
          </label>
          {renderField(field)}
          {errors[field.name] && touched[field.name] && (
            <span className="error">{errors[field.name]}</span>
          )}
        </div>
      ))}
      <button type="submit">提交</button>
    </form>
  );
}
```

**考察点：**
- JSON Schema 驱动的动态表单设计。
- 表单校验（声明式 + 命令式结合）。
- 受控组件的状态管理。
- 数组字段的动态增删。
- touched 状态管理（只在 blur 后显示错误）。

---

### 49. 设计一个权限路由系统

```jsx
// 权限模型
const ROLES = {
  ADMIN: 'admin',
  EDITOR: 'editor',
  VIEWER: 'viewer',
};

const PERMISSIONS = {
  VIEW_DASHBOARD: 'view_dashboard',
  EDIT_POSTS: 'edit_posts',
  DELETE_USERS: 'delete_users',
  MANAGE_SETTINGS: 'manage_settings',
};

// 角色-权限映射
const rolePermissions = {
  [ROLES.ADMIN]: [PERMISSIONS.VIEW_DASHBOARD, PERMISSIONS.EDIT_POSTS, PERMISSIONS.DELETE_USERS, PERMISSIONS.MANAGE_SETTINGS],
  [ROLES.EDITOR]: [PERMISSIONS.VIEW_DASHBOARD, PERMISSIONS.EDIT_POSTS],
  [ROLES.VIEWER]: [PERMISSIONS.VIEW_DASHBOARD],
};

// ===== 1. 鉴权 Hook =====
function usePermission() {
  const { user } = useAuth();

  const hasPermission = useCallback((permission) => {
    if (!user) return false;
    return rolePermissions[user.role]?.includes(permission) ?? false;
  }, [user]);

  const hasAnyPermission = useCallback((...permissions) => {
    return permissions.some(p => hasPermission(p));
  }, [hasPermission]);

  const hasAllPermissions = useCallback((...permissions) => {
    return permissions.every(p => hasPermission(p));
  }, [hasPermission]);

  return { hasPermission, hasAnyPermission, hasAllPermissions };
}

// ===== 2. 权限路由组件 =====
function ProtectedRoute({ children, permission, redirectTo = '/403' }) {
  const { hasPermission } = usePermission();
  const { isAuthenticated } = useAuth();

  if (!isAuthenticated) {
    return <Navigate to="/login" replace />;
  }

  if (permission && !hasPermission(permission)) {
    return <Navigate to={redirectTo} replace />;
  }

  return children;
}

// ===== 3. 权限按钮组件 =====
function PermissionButton({ permission, children, fallback = null, ...props }) {
  const { hasPermission } = usePermission();

  if (!hasPermission(permission)) return fallback;

  return <button {...props}>{children}</button>;
}

// ===== 4. 权限路由配置 =====
const routeConfig = [
  {
    path: '/',
    element: <Layout />,
    children: [
      { index: true, element: <Dashboard />, permission: PERMISSIONS.VIEW_DASHBOARD },
      { path: 'posts', element: <Posts />, permission: PERMISSIONS.EDIT_POSTS },
      {
        path: 'admin',
        permission: PERMISSIONS.MANAGE_SETTINGS,
        children: [
          { index: true, element: <AdminDashboard /> },
          { path: 'users', element: <UserManagement />, permission: PERMISSIONS.DELETE_USERS },
          { path: 'settings', element: <Settings />, permission: PERMISSIONS.MANAGE_SETTINGS },
        ],
      },
    ],
  },
  { path: '/login', element: <Login /> },
  { path: '/403', element: <Forbidden /> },
  { path: '*', element: <NotFound /> },
];

// ===== 5. 递归生成路由 =====
function renderRoutes(routes) {
  return routes.map(({ children, permission, element, ...route }) => {
    const routeElement = permission ? (
      <ProtectedRoute permission={permission}>{element}</ProtectedRoute>
    ) : (
      element
    );

    return (
      <Route
        key={route.path || 'index'}
        {...route}
        element={children ? <RouteWrapper permission={permission}>{routeElement}</RouteWrapper> : routeElement}
      >
        {children && renderRoutes(children)}
      </Route>
    );
  });
}

// ===== 6. 根据权限生成菜单 =====
function useMenuItems() {
  const { hasAnyPermission } = usePermission();

  const allMenuItems = [
    { key: 'dashboard', label: '仪表盘', path: '/', permission: PERMISSIONS.VIEW_DASHBOARD },
    { key: 'posts', label: '文章管理', path: '/posts', permission: PERMISSIONS.EDIT_POSTS },
    {
      key: 'admin',
      label: '系统管理',
      permission: PERMISSIONS.MANAGE_SETTINGS,
      children: [
        { key: 'users', label: '用户管理', path: '/admin/users', permission: PERMISSIONS.DELETE_USERS },
        { key: 'settings', label: '系统设置', path: '/admin/settings', permission: PERMISSIONS.MANAGE_SETTINGS },
      ],
    },
  ];

  // 递归过滤不可见菜单
  const filterMenu = (items) =>
    items
      .filter(item => item.permission ? hasAnyPermission(item.permission) : true)
      .map(item => ({
        ...item,
        children: item.children ? filterMenu(item.children) : undefined,
      }));

  return filterMenu(allMenuItems);
}
```

**考察点：**
- RBAC（基于角色的访问控制）模型设计。
- 多层权限控制：路由级、组件级、数据级。
- 菜单和路由的权限联动。
- 递归路由和菜单的生成。

---

### 50. 设计一个支持撤销/重做的编辑器

```jsx
function useUndoRedo(initialState) {
  // 历史栈
  const [history, setHistory] = useState([initialState]);
  const [currentIndex, setCurrentIndex] = useState(0);

  const currentState = history[currentIndex];
  const canUndo = currentIndex > 0;
  const canRedo = currentIndex < history.length - 1;

  const pushState = useCallback((newStateOrUpdater) => {
    setHistory(prev => {
      // 如果当前不是最新，截断后续历史（新操作覆盖旧 redo 栈）
      const truncated = prev.slice(0, currentIndex + 1);
      const newState = typeof newStateOrUpdater === 'function'
        ? newStateOrUpdater(prev[currentIndex])
        : newStateOrUpdater;

      // 去重：避免连续相同状态占空间
      if (JSON.stringify(newState) === JSON.stringify(prev[currentIndex])) {
        return prev;
      }

      setCurrentIndex(truncated.length);
      return [...truncated, newState];
    });
  }, [currentIndex]);

  const undo = useCallback(() => {
    if (canUndo) setCurrentIndex(prev => prev - 1);
  }, [canUndo]);

  const redo = useCallback(() => {
    if (canRedo) setCurrentIndex(prev => prev + 1);
  }, [canRedo]);

  // 快捷键支持：Ctrl+Z / Ctrl+Shift+Z
  useEffect(() => {
    const handler = (e) => {
      if ((e.ctrlKey || e.metaKey) && e.key === 'z') {
        e.preventDefault();
        if (e.shiftKey) {
          redo();
        } else {
          undo();
        }
      }
    };
    window.addEventListener('keydown', handler);
    return () => window.removeEventListener('keydown', handler);
  }, [undo, redo]);

  return {
    state: currentState,
    pushState,
    undo,
    redo,
    canUndo,
    canRedo,
    historySize: history.length,
    currentStep: currentIndex + 1,
  };
}

// ===== 使用示例：文本编辑器 =====
function TextEditor() {
  const { state, pushState, undo, redo, canUndo, canRedo } = useUndoRedo({
    title: '',
    content: '',
  });

  const handleTitleChange = (e) => {
    pushState(prev => ({ ...prev, title: e.target.value }));
  };

  const handleContentChange = (e) => {
    pushState(prev => ({ ...prev, content: e.target.value }));
  };

  return (
    <div className="editor">
      <div className="toolbar">
        <button onClick={undo} disabled={!canUndo}>撤销 (Ctrl+Z)</button>
        <button onClick={redo} disabled={!canRedo}>重做 (Ctrl+Shift+Z)</button>
      </div>
      <input
        placeholder="标题"
        value={state.title}
        onChange={handleTitleChange}
      />
      <textarea
        placeholder="内容..."
        value={state.content}
        onChange={handleContentChange}
        rows={20}
      />
    </div>
  );
}
```

**考察点：**
- 不可变数据 + 历史栈的设计思路。
- 新操作截断 redo 栈的细节。
- 状态去重优化（避免无意义的历史记录）。
- 快捷键系统集成。
- 自定义 Hook 的抽象能力。

---

## 总结

### 面试准备建议

1. **掌握基础**：Virtual DOM、Diff 算法、Hooks 原理是高频考点，必须深入理解。
2. **手写关键代码**：`useState`、`useEffect` 的简化实现能帮助你理解本质。
3. **性能优化**：`React.memo`、`useMemo`、`useCallback` 的适用场景和反模式。
4. **关注 React 18 / 19 新特性**：并发模式、自动批处理、Suspense、RSC、Actions。
5. **设计模式**：HOC、Render Props、Compound Components、Provider 等模式的实际应用。
6. **实战经验**：能够讲述复杂组件的设计思路、优化的实际案例。
7. **测试能力**：RTL 的使用、常见场景的测试策略。
8. **源码阅读**：Fiber 架构、调度机制、Hooks 链表会让你在面试中脱颖而出。

### 推荐学习路径

```
基础 React → Hooks 深入 → 性能优化 → 状态管理 → 设计模式
    → React 18 新特性 → 测试 → 源码原理 → React 19 新特性
```

### 50 道题目难度分布

| 难度 | 题号 | 说明 |
|------|------|------|
| ⭐ 初级 | 1, 3, 5, 7, 25 | 基础概念，必会题 |
| ⭐⭐ 中级 | 2, 4, 8-12, 14-18, 26-29 | 日常工作高频使用 |
| ⭐⭐⭐ 高级 | 6, 13, 19-24, 30-32 | 深入原理和优化 |
| ⭐⭐⭐⭐ 专家 | 33-37, 42-47 | 设计模式和陷阱 |
| ⭐⭐⭐⭐⭐ 架构 | 38-41, 48-50 | 测试、新特性、系统设计 |

---

> **面试技巧**：不要死记硬背答案。理解核心概念后，用自己的话组织答案。结合你实际项目中的经验和踩过的坑来回答，会让面试官印象深刻。每一道题都可以扩展讲 10 分钟以上——展示你的深度思考。

---

## 十四、React 工程化实践

### 51. Webpack 与 Vite 在 React 项目中的配置要点

**Webpack 配置关键点：**

```js
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = {
  entry: './src/index.tsx',

  resolve: {
    extensions: ['.tsx', '.ts', '.js', '.jsx'],
    alias: {
      '@': path.resolve(__dirname, 'src'), // 路径别名
    },
  },

  module: {
    rules: [
      // TS/JS 处理
      {
        test: /\.(ts|tsx|js|jsx)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              ['@babel/preset-env', { targets: '> 0.25%, not dead' }],
              ['@babel/preset-react', { runtime: 'automatic' }], // React 17+ JSX 自动引入
              '@babel/preset-typescript',
            ],
          },
        },
      },
      // CSS Modules
      {
        test: /\.module\.css$/,
        use: [MiniCssExtractPlugin.loader, {
          loader: 'css-loader',
          options: { modules: { localIdentName: '[name]__[local]--[hash:base64:5]' } },
        }],
      },
      // 全局 CSS
      { test: /\.css$/, exclude: /\.module\.css$/, use: [MiniCssExtractPlugin.loader, 'css-loader'] },
      // 图片/字体
      { test: /\.(png|jpg|gif|svg)$/, type: 'asset' },
    ],
  },

  // 代码分割配置
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10,
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },
  },

  plugins: [
    new HtmlWebpackPlugin({ template: './public/index.html' }),
    new MiniCssExtractPlugin({ filename: '[name].[contenthash].css' }),
  ],
};
```

**Vite 配置关键点：**

```js
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [
    react({
      // 启用 React Fast Refresh（HMR 保留状态）
      // Babel 配置可选
      babel: {
        plugins: [['@babel/plugin-proposal-decorators', { legacy: true }]],
      },
    }),
  ],

  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },

  // 开发服务器
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },

  // 构建优化
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'router-vendor': ['react-router-dom'],
        },
      },
    },
    chunkSizeWarningLimit: 1000,
    sourcemap: false,
  },

  // 环境变量前缀（只有 VITE_ 开头的会暴露给客户端）
  // import.meta.env.VITE_API_URL
});
```

**Webpack vs Vite 对比：**

| 维度 | Webpack | Vite |
|------|---------|------|
| 冷启动速度 | 慢（需全量打包） | 快（ESM 按需编译） |
| HMR 速度 | 随项目体积增大而变慢 | 极快（ESM 精确更新） |
| 生态插件 | 极其丰富 | 快速发展中（兼容 Rollup 插件） |
| 生产构建 | Webpack 5（成熟稳定） | Rollup（ESBuild 预处理） |
| 配置复杂度 | 较高 | 低（开箱即用） |
| 底层机制 | Bundle 打包（CommonJS/ESM → bundle） | 开发时 Native ESM，构建时 Rollup |

---

### 52. React 项目的 CI/CD 流水线设计

```yaml
# .github/workflows/ci.yml（GitHub Actions）
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # === 1. 代码质量检查 ===
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      # ESLint 检查
      - run: npm run lint

      # TypeScript 类型检查
      - run: npm run type-check

      # 单元测试 + 覆盖率
      - run: npm run test -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  # === 2. 构建检查 ===
  build:
    needs: lint-and-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm run build

      # 检查构建产物体积（可选）
      - run: npx bundlesize

      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  # === 3. 部署到测试环境 ===
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
      - run: |
          # 部署到 S3 / 阿里云 OSS / 腾讯云 COS
          aws s3 sync dist/ s3://staging-bucket/ --delete
      - run: |
          # 刷新 CDN 缓存
          aws cloudfront create-invalidation --distribution-id ${{ secrets.CF_DIST_ID }} --paths "/*"

  # === 4. 部署到生产环境（需审批）===
  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
      - run: |
          # 生产部署脚本
          npm run deploy:prod
```

**关键检查项清单：**

```
□ ESLint                    # 代码规范
□ Prettier                  # 代码格式化
□ TypeScript 类型检查        # tsc --noEmit
□ Jest 单元测试 + 覆盖率     # --coverage（阈值：80%+）
□ Bundle 体积分析            # @next/bundle-analyzer / bundlesize
□ Lighthouse 性能检查        # CI 中集成 lhci
□ 构建成功验证              # npm run build 无报错
```


---

### 53. React 中的 CSS 方案全景对比

```tsx
// ===== 1. CSS Modules（推荐大多数项目） =====
// Button.module.css
.button { padding: 8px 16px; border-radius: 4px; }
.primary { background: #1890ff; color: #fff; }
.danger { background: #ff4d4f; color: #fff; }

// Button.tsx
import styles from './Button.module.css';

function Button({ variant = 'primary', children }) {
  return (
    <button className={`${styles.button} ${styles[variant]}`}>
      {children}
    </button>
  );
}
// ✅ 优点：零运行时、天然作用域隔离、TypeScript 友好
// ❌ 缺点：动态样式麻烦、变量共享需额外处理

// ===== 2. Tailwind CSS（原子化 CSS） =====
function Button({ variant = 'primary', children }) {
  const base = 'px-4 py-2 rounded transition-colors';
  const variants = {
    primary: 'bg-blue-500 text-white hover:bg-blue-600',
    danger: 'bg-red-500 text-white hover:bg-red-600',
  };
  return (
    <button className={`${base} ${variants[variant]}`}>
      {children}
    </button>
  );
}
// ✅ 优点：快速开发、约束性强、JIT 编译树摇
// ❌ 缺点：HTML 可能很长、学习成本、团队约束

// ===== 3. CSS-in-JS（Styled Components / Emotion） =====
import styled from 'styled-components';

const Button = styled.button`
  padding: 8px 16px;
  border-radius: 4px;
  background: ${({ variant }) =>
    variant === 'danger' ? '#ff4d4f' : '#1890ff'
  };
  color: #fff;
  border: none;
  cursor: pointer;

  &:hover {
    opacity: 0.9;
  }
`;

<Button variant="danger">删除</Button>
// ✅ 优点：动态样式能力强、组件内聚、自动 vendor prefix
// ❌ 缺点：运行时开销、SSR 复杂、Bundle 体积、React 19/Server Components 兼容性差

// ===== 4. Panda CSS / Vanilla Extract（零运行时 CSS-in-JS） =====
// 编译时生成静态 CSS，零运行时开销
import { css } from '../styled-system/css';

function Button({ variant }) {
  return (
    <button className={css({
      padding: '8px 16px',
      borderRadius: 'md',
      bg: variant === 'danger' ? 'red.500' : 'blue.500',
    })}>
      Click
    </button>
  );
}
// ✅ 优点：零运行时、类型安全、RSC 兼容
// ❌ 缺点：相对较新，生态较小
```

**选择决策树：**

```
需要 Server Components 支持？
├── 是 → Panda CSS / Vanilla Extract / Tailwind CSS / CSS Modules
└── 否 → 继续往下判断
    ├── 团队小、追求开发速度？ → Tailwind CSS
    ├── 需要动态主题/大量交互样式？ → Styled Components / Emotion
    ├── 习惯传统 CSS + 需要作用域隔离？ → CSS Modules
    └── 大型团队、需要强约束？ → Tailwind CSS + 设计 Token
```


---

### 54. 微前端架构在 React 中的应用

```jsx
// ===== 微前端方案对比 =====

// 方案1：qiankun（阿里，基于 single-spa）
// 主应用
import { registerMicroApps, start } from 'qiankun';

registerMicroApps([
  {
    name: 'app-react',        // 子应用名称
    entry: '//localhost:3001', // 子应用地址
    container: '#subapp-container',
    activeRule: '/app-react',
    props: { user: currentUser, token }, // 向子应用传数据
  },
  {
    name: 'app-vue',
    entry: '//localhost:3002',
    container: '#subapp-container',
    activeRule: '/app-vue',
  },
]);

start({ sandbox: { experimentalStyleIsolation: true } });

// 子应用入口（React）
let root = null;
function render(props = {}) {
  const { container } = props;
  root = createRoot(container ? container.querySelector('#root') : document.getElementById('root'));
  root.render(<App {...props} />);
}

// 独立运行
if (!window.__POWERED_BY_QIANKUN__) {
  render();
}

// 生命周期钩子
export async function bootstrap() { /* 初始化 */ }
export async function mount(props) { render(props); }
export async function unmount() { root?.unmount(); }

// ===== 方案2：Module Federation（Webpack 5 原生） =====
// 主应用 webpack.config.js
new ModuleFederationPlugin({
  name: 'host',
  remotes: {
    app1: 'app1@http://localhost:3001/remoteEntry.js',
    app2: 'app2@http://localhost:3002/remoteEntry.js',
  },
  shared: {
    react: { singleton: true, eager: true },
    'react-dom': { singleton: true, eager: true },
  },
});

// 使用远程组件
const RemoteHeader = lazy(() => import('app1/Header'));
const RemoteDashboard = lazy(() => import('app2/Dashboard'));

// 方案3：wujie（无极，腾讯）
import { startApp } from 'wujie';

startApp({
  name: 'react-child',
  url: '//localhost:3001',
  el: '#wujie-container',
  sync: true, // 同步路由
  props: { data: 'from parent' },
});
```

**方案对比：**

| 方案 | 隔离方式 | 通信 | 技术栈 | 适合场景 |
|------|---------|------|--------|----------|
| **qiankun** | JS沙箱 + 样式隔离 | props + globalState | 不限 | 已有多个不同技术栈项目 |
| **Module Federation** | 运行时共享 | shared state | 不限 | 新项目、同技术栈 |
| **wujie** | iframe + WebComponent | props + EventBus | 不限 | 需要最好隔离性 |
| **micro-app** | WebComponent | 全局通信 | 不限 | 简单接入 |

**微前端通信方案：**

```jsx
// 1. 自定义事件通信
window.dispatchEvent(new CustomEvent('user-login', { detail: user }));

// 2. 共享 Store（主应用维护，通过 props 下发）
const globalStore = create((set) => ({
  token: null,
  setToken: (token) => set({ token }),
}));

registerMicroApps([{ name: 'app1', /*...*/, props: { store: globalStore } }]);

// 3. URL 参数传递（适合简单场景）
navigate(`/app-react?token=${token}&userId=${userId}`);
```

---

### 55. React 前端监控体系搭建

```jsx
// 1. 错误监控（Error Boundary + 上报）
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // 上报到监控平台
    monitor.reportError({
      type: 'react_error',
      message: error.message,
      stack: error.stack,
      componentStack: errorInfo.componentStack,
      timestamp: Date.now(),
      url: window.location.href,
    });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <ErrorFallback error={this.state.error} />;
    }
    return this.props.children;
  }
}

// 2. 性能监控
function usePerformanceMonitor(componentName) {
  useEffect(() => {
    const startTime = performance.now();

    return () => {
      const duration = performance.now() - startTime;
      if (duration > 100) { // 渲染超过 100ms 的组件
        monitor.reportMetric({
          type: 'slow_component',
          component: componentName,
          duration,
          timestamp: Date.now(),
        });
      }
    };
  });

  // 监控重渲染次数
  const renderCount = useRef(0);
  renderCount.current++;
  useEffect(() => {
    if (renderCount.current > 20) {
      console.warn(`[${componentName}] 重渲染 ${renderCount.current} 次，可能存在性能问题`);
    }
  });
}

// 3. 核心 Web Vitals 监控
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals';

function reportWebVitals() {
  onCLS(metric => sendToAnalytics('CLS', metric));
  onINP(metric => sendToAnalytics('INP', metric));
  onLCP(metric => sendToAnalytics('LCP', metric));
  onFCP(metric => sendToAnalytics('FCP', metric));
  onTTFB(metric => sendToAnalytics('TTFB', metric));
}

// 4. 自定义埋点 Hook
function useTrack() {
  const track = useCallback((event, data = {}) => {
    // 上报用户行为
    monitor.track({
      event,
      data,
      url: window.location.href,
      timestamp: Date.now(),
      userId: getCurrentUserId(),
    });
  }, []);

  const trackPageView = useCallback((pageName) => {
    track('page_view', { page: pageName });
  }, [track]);

  const trackClick = useCallback((element, extra = {}) => {
    track('click', {
      element: element.dataset?.trackId || element.id || '',
      text: element.textContent?.slice(0, 50) || '',
      ...extra,
    });
  }, [track]);

  return { track, trackPageView, trackClick };
}

// 5. API 请求监控（Axios 拦截器）
const apiMonitor = axios.create();

apiMonitor.interceptors.request.use(config => {
  config.metadata = { startTime: Date.now() };
  return config;
});

apiMonitor.interceptors.response.use(
  response => {
    const duration = Date.now() - response.config.metadata.startTime;
    if (duration > 3000) {
      monitor.reportMetric({ type: 'slow_api', url: response.config.url, duration });
    }
    return response;
  },
  error => {
    monitor.reportError({
      type: 'api_error',
      url: error.config?.url,
      status: error.response?.status,
      message: error.message,
    });
    return Promise.reject(error);
  }
);
```

**监控体系分层：**

```
第一层：错误监控（Sentry / 自建）
  ├── JS 运行时错误
  ├── React Error Boundary 捕获的错误
  ├── Promise 未捕获异常
  ├── 接口错误（4xx / 5xx）
  └── 静态资源加载失败

第二层：性能监控（自建 / Lighthouse CI）
  ├── Web Vitals（LCP、INP、CLS、FCP、TTFB）
  ├── 组件渲染耗时
  ├── 接口响应耗时
  └── Bundle 体积变化

第三层：用户行为监控（埋点）
  ├── PV/UV
  ├── 用户操作路径
  ├── 功能使用率
  └── 页面停留时长
```


---

## 十五、React 生态深度对比

### 56. React Query（TanStack Query）深度用法

```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// ===== 基础数据获取 =====
function UserList() {
  const {
    data: users,
    isLoading,
    isError,
    error,
    refetch,     // 手动重新请求
    isFetching,  // 是否正在请求（包括后台刷新）
  } = useQuery({
    queryKey: ['users', { page, status }], // 唯一键，支持参数
    queryFn: () => fetchUsers({ page, status }),
    staleTime: 5 * 60 * 1000,     // 5 分钟内不重新请求
    gcTime: 30 * 60 * 1000,       // 30 分钟后清除缓存（React Query v5: gcTime 替代 cacheTime）
    retry: 3,                      // 失败重试 3 次
    refetchOnWindowFocus: false,   // 窗口聚焦时不自动刷新
  });
}

// ===== 乐观更新（Optimistic Update） =====
function useCreateTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (newTodo) => createTodo(newTodo),

    // 请求发出前立即更新 UI
    onMutate: async (newTodo) => {
      // 取消正在进行的查询，避免覆盖乐观更新
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // 保存旧数据用于回滚
      const previousTodos = queryClient.getQueryData(['todos']);

      // 乐观更新缓存
      queryClient.setQueryData(['todos'], (old) => [
        ...old,
        { ...newTodo, id: 'temp-' + Date.now(), optimistic: true },
      ]);

      return { previousTodos };
    },

    // 失败时回滚
    onError: (err, newTodo, context) => {
      queryClient.setQueryData(['todos'], context.previousTodos);
      toast.error('添加失败，已回滚');
    },

    // 请求成功后刷新
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
}

// ===== 依赖查询（串行） =====
function UserDetail({ userId }) {
  // 第一步：获取用户
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    enabled: !!userId,
  });

  // 第二步：基于用户获取部门（依赖第一步结果）
  const { data: department } = useQuery({
    queryKey: ['department', user?.departmentId],
    queryFn: () => fetchDepartment(user.departmentId),
    enabled: !!user?.departmentId, // 用户数据拿到后才查询
  });
}

// ===== 无限滚动 =====
function InfinitePosts() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 1 }) => fetchPosts(pageParam),
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    initialPageParam: 1,
  });

  const loadMoreRef = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting && hasNextPage) {
        fetchNextPage();
      }
    });
    if (loadMoreRef.current) observer.observe(loadMoreRef.current);
    return () => observer.disconnect();
  }, [hasNextPage, fetchNextPage]);

  return (
    <div>
      {data?.pages.flatMap(page => page.items).map(post => (
        <PostCard key={post.id} post={post} />
      ))}
      <div ref={loadMoreRef}>
        {isFetchingNextPage ? '加载更多...' : hasNextPage ? '上滑加载' : '没有更多了'}
      </div>
    </div>
  );
}

// ===== 预加载（Prefetch） =====
function NavigationMenu() {
  const queryClient = useQueryClient();

  const prefetchUsers = () => {
    queryClient.prefetchQuery({
      queryKey: ['users'],
      queryFn: fetchUsers,
      staleTime: 60000,
    });
  };

  // 鼠标悬停链接时预加载数据
  return (
    <Link
      to="/users"
      onMouseEnter={prefetchUsers}
      onFocus={prefetchUsers}
    >
      用户管理
    </Link>
  );
}
```

**React Query 的缓存生命周期：**

```
数据请求 → fresh（新鲜期，staleTime 内不重新请求）
         → stale（过期但仍可用，触发后台重新请求）
         → inactive（无组件使用，gcTime 后清除缓存）
```

---

### 57. Zustand 与 Jotai：轻量状态管理新范式

```jsx
// ===== Zustand：类 Redux 但更简洁 =====
import { create } from 'zustand';
import { persist, devtools } from 'zustand/middleware';

// 创建 Store
const useUserStore = create(
  devtools(
    persist(
      (set, get) => ({
        user: null,
        isAuthenticated: false,

        login: async (credentials) => {
          const user = await api.login(credentials);
          set({ user, isAuthenticated: true });
        },

        logout: () => {
          set({ user: null, isAuthenticated: false });
        },

        updateProfile: (updates) => {
          set((state) => ({
            user: { ...state.user, ...updates },
          }));
        },

        // getter：从其他 Store 派生数据
        getDisplayName: () => {
          const { user } = get();
          return user?.nickname || user?.username || '匿名用户';
        },
      }),
      {
        name: 'user-storage',     // localStorage key
        partialize: (state) => ({ // 只持久化部分字段
          user: state.user,
          isAuthenticated: state.isAuthenticated,
        }),
      }
    ),
    { name: 'UserStore' }  // Redux DevTools 中显示的名称
  )
);

// 使用（无需 Provider！）
function Header() {
  const user = useUserStore(s => s.user);
  const logout = useUserStore(s => s.logout);
  const getDisplayName = useUserStore(s => s.getDisplayName);

  return (
    <div>
      <span>{user ? getDisplayName() : '未登录'}</span>
      {user && <button onClick={logout}>退出</button>}
    </div>
  );
}

// Zustand 高级用法：切片模式 + 中间件链
const createBearSlice = (set) => ({
  bears: 0,
  addBear: () => set((state) => ({ bears: state.bears + 1 })),
});

const createFishSlice = (set) => ({
  fishes: 0,
  addFish: () => set((state) => ({ fishes: state.fishes + 1 })),
});

const useBoundStore = create((...a) => ({
  ...createBearSlice(...a),
  ...createFishSlice(...a),
}));


// ===== Jotai：原子化状态管理 =====
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';
import { atomWithStorage, atomFamily } from 'jotai/utils';

// 原子（Atoms）
const countAtom = atom(0);
const doubledAtom = atom((get) => get(countAtom) * 2); // 派生 atom

// 可写派生 atom（双向）
const celsiusAtom = atom(0);
const fahrenheitAtom = atom(
  (get) => get(celsiusAtom) * 9 / 5 + 32,
  (get, set, newFahrenheit) => set(celsiusAtom, (newFahrenheit - 32) * 5 / 9)
);

// 持久化 atom
const themeAtom = atomWithStorage('theme', 'light');

// atomFamily：参数化 atom（类似 Recoil）
const todoAtomFamily = atomFamily((id) =>
  atom(async () => {
    const res = await fetch(`/api/todos/${id}`);
    return res.json();
  })
);

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const doubled = useAtomValue(doubledAtom); // 只读
  const [theme, setTheme] = useAtom(themeAtom);

  return (
    <div>
      <p>Count: {count} (x2 = {doubled})</p>
      <button onClick={() => setCount(c => c + 1)}>+1</button>
      <p>Theme: {theme}</p>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
    </div>
  );
}

function TodoItem({ id }) {
  // 每个 id 独立缓存，精准渲染
  const [todo] = useAtom(todoAtomFamily(id));
  return <div>{todo?.title}</div>;
}
```

**Zustand vs Jotai vs Redux 对比：**

| 维度 | Redux Toolkit | Zustand | Jotai |
|------|--------------|---------|-------|
| Bundle 大小 | ~12KB | ~2KB | ~3KB |
| Provider | 需要 | 不需要 | 可选（Provider 作用域控制） |
| DevTools | 丰富（RTK Query 等） | 支持 | 支持 |
| 学习曲线 | 中高 | 低 | 低 |
| 模板代码 | 较多（slice + thunk） | 极少 | 极少 |
| 渲染优化 | 需手动 selector | selector 自动浅比较 | 原子粒度自动优化 |
| 适合场景 | 大型复杂应用 | 中大型应用 | 细粒度状态、表单、编辑器 |

---

### 58. React vs Vue vs Svelte 核心差异

| 维度 | React | Vue 3 | Svelte |
|------|-------|-------|--------|
| **设计理念** | 函数式、不可变 | 响应式、可变 | 编译器优先 |
| **响应式机制** | 手动 setState（不可变） | Proxy 自动追踪 | 编译器注入响应式代码 |
| **虚拟 DOM** | ✅ Fiber 架构 | ✅ 编译器优化 | ❌ 无 VDOM，直接操作 DOM |
| **运行时体积** | ~40KB | ~30KB | ~2KB（编译后几乎为 0） |
| **组件模型** | 函数组件 + Hooks | SFC + Composition API | .svelte 单文件组件 |
| **模板语法** | JSX（JS 表达式） | Template（指令系统） | 增强 HTML 模板 |
| **状态管理** | Context/Redux/Zustand | Pinia（官方） | Store（内置） |
| **CSS 方案** | 社区方案（CSS Modules 等） | `<style scoped>` 内置 | `<style>` 默认作用域隔离 |
| **编译产物** | JS + VDOM Runtime | JS + VDOM Runtime | 纯 JS（直接操作 DOM） |
| **TypeScript** | 一等公民 | 良好支持 | 支持中 |
| **生态** | 🌟🌟🌟🌟🌟 最丰富 | 🌟🌟🌟🌟 官方主导 | 🌟🌟 快速增长中 |
| **学习曲线** | 中等（需要理解不可变、闭包） | 较低（HTML/CSS/JS 分离） | 低（接近原生写法） |

**代码风格对比（同一计数器）：**

```jsx
// React
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

```vue
<!-- Vue 3 Composition API -->
<script setup>
import { ref } from 'vue';
const count = ref(0);
</script>
<template>
  <button @click="count++">{{ count }}</button>
</template>
```

```svelte
<!-- Svelte -->
<script>
  let count = 0;
</script>
<button on:click={() => count++}>{count}</button>
```

**选择建议：**
- **React**：大团队、复杂项目、生态依赖多的场景。
- **Vue**：中小团队、追求开发效率、中文社区友好。
- **Svelte**：对包体积敏感、简单页面、嵌入式场景。

---

### 59. Signals（信号）是什么？对 React 有何影响？

Signals 是一种**细粒度响应式原语**，由 Solid.js 带火，目前在 React 社区通过 Preact Signals 等库可用。

```jsx
// ===== Preact Signals 在 React 中使用 =====
import { signal, computed, effect } from '@preact/signals-react';

// 1. 创建 signal（绕开 React 渲染机制）
const count = signal(0);
const doubled = computed(() => count.value * 2);

// 自动追踪 + 自动清理
const dispose = effect(() => {
  console.log(`count: ${count.value}, doubled: ${doubled.value}`);
});

function Counter() {
  return (
    <div>
      <p>Count: {count}</p>       {/* 直接使用 signal，无需 useState */}
      <p>Doubled: {doubled}</p>   {/* 自动依赖 computed */}
      <button onClick={() => count.value++}>+1</button>
      {/* count 变化时，只有用到 count 的组件重新渲染 */}
    </div>
  );
}

// 2. signals 解决的核心问题：精准渲染
const userSignal = signal({ name: 'Alice', age: 25 });

function UserName() {
  // 只读 name，age 变化时不会重新渲染！
  return <span>{userSignal.value.name}</span>;
}

function UserAge() {
  // 只读 age，name 变化时不会重新渲染！
  return <span>{userSignal.value.age}</span>;
}

// 3. React 状态 vs Signals 对比
// React 方案：状态变化 → 整个组件树重新渲染 → 通过 memo 跳过不必要渲染
// Signals 方案：值变化 → 只有用到该值的组件重新渲染（自动精准，无需 memo）

// 4. React 19 的 use() 与 Signals 的相似之处
// use() 可以读取 Promise/Context，但不改变 React 的更新机制
// Signals 改变了更新粒度，是更大的范式变化
```

**Signals 的优势与挑战：**

| 优势 | 挑战 |
|------|------|
| 自动精准渲染，无需 `React.memo` | 与 React 单向数据流理念不同 |
| 无依赖数组，无闭包陷阱 | 破坏 React DevTools 的状态追踪 |
| 极小的代码量 | 学习新范式 |
| 天然跨组件通信 | 需要 `useSignals()` 包装组件 |

---

### 60. React Native 跨端方案对比

| 方案 | 架构 | 性能 | 生态 | 适合场景 |
|------|------|------|------|----------|
| **React Native (新架构)** | JSI + Fabric + TurboModules | 接近原生 | 丰富 | 跨 iOS/Android 移动端 |
| **Expo** | 基于 RN，托管服务 | 接近原生 | 丰富 | 快速开发，不需要原生模块 |
| **Taro 3** | 编译时 + 运行时适配 | 良好 | 国内丰富 | 跨小程序 + H5 + RN |
| **Flutter** | 自绘引擎 Skia | 优秀 | 丰富 | 需要高性能渲染的 App |
| **Electron** | Chromium + Node.js | 一般 | Web 生态 | 桌面应用 |

```jsx
// Taro 跨端示例（写 React 代码，编译到多端）
import { View, Text, Button } from '@tarojs/components';
import Taro from '@tarojs/taro';

function HomePage() {
  const handleClick = () => {
    Taro.navigateTo({ url: '/pages/detail/index' });
  };

  return (
    <View>
      <Text>Hello Taro</Text>
      <Button onClick={handleClick}>跳转</Button>
    </View>
  );
}
// 可编译为：微信小程序 / 支付宝小程序 / H5 / React Native
```


---

## 十六、React 安全与最佳实践

### 61. React 中的安全防护清单

```jsx
// 1. XSS 防护
// React 默认对 JSX 中的变量进行 HTML 转义
function SafeComponent({ userInput }) {
  return <div>{userInput}</div>; // 自动转义，安全
}

// 危险：dangerouslySetInnerHTML（必须消毒）
import DOMPurify from 'dompurify';
function SafeHTML({ html }) {
  return <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(html) }} />;
}

// 2. URL 安全
function SafeLink({ url }) {
  // 防止 javascript: 协议的 XSS
  const isSafe = url.startsWith('http://') || url.startsWith('https://') || url.startsWith('/');

  return (
    <a
      href={isSafe ? url : '#'}
      target="_blank"
      rel="noopener noreferrer" // 防止 tabnabbing 攻击
    >
      访问
    </a>
  );
}

// 3. 敏感信息不暴露到前端
function PaymentForm() {
  // ❌ 密钥、Token 等不应在前端硬编码
  // const STRIPE_KEY = 'sk_live_xxxx'; // 生产密钥暴露！

  // ✅ 通过环境变量 + 后端接口获取
  const [config, setConfig] = useState(null);
  useEffect(() => {
    fetch('/api/payment/config') // 后端返回公开 key
      .then(r => r.json())
      .then(setConfig);
  }, []);
}

// 4. 防止 CSRF（单页应用）
// 后端设置 SameSite=Strict/Lax 的 Cookie
// 前端在请求头中加入 CSRF Token
const api = axios.create({
  headers: {
    'X-CSRF-Token': getCookie('csrf_token'),
  },
});

// 5. 防止 iframe 嵌套（点击劫持）
// 后端设置 Content-Security-Policy: frame-ancestors 'none'
// 前端可选检测
if (window.top !== window.self) {
  window.top.location = window.self.location;
}

// 6. 依赖安全审计
// npm audit / yarn audit / Snyk 定期扫描
```

---

### 62. React 代码规范与最佳实践

```jsx
// ===== 1. 组件命名和结构 =====

// ✅ 好的组件结构顺序
function MyComponent({ prop1, prop2, ...rest }) {
  // 1. Hooks（全部放在顶部）
  const [state, setState] = useState();
  const ref = useRef();
  const value = useMemo(() => compute(prop1), [prop1]);

  // 2. 派生状态 / 计算值
  const derivedValue = state * 2;

  // 3. 事件处理函数
  const handleClick = useCallback(() => {
    setState(s => s + 1);
  }, []);

  // 4. 副作用
  useEffect(() => {
    fetchData(prop1);
  }, [prop1]);

  // 5. 条件返回（早 return）
  if (!prop1) return <EmptyState />;
  if (state === 'error') return <ErrorState />;

  // 6. 主渲染
  return (
    <div {...rest}>
      <button onClick={handleClick}>{derivedValue}</button>
    </div>
  );
}

// ===== 2. 避免常见反模式 =====

// ❌ 在渲染中创建组件（每次渲染都会卸载重建）
function Parent() {
  const Child = () => <div>Child</div>; // 不要在渲染中定义组件！
  return <Child />;
}

// ❌ 使用 index 作为 key
{items.map((item, i) => <div key={i}>{item}</div>)}

// ❌ 直接在 JSX 中使用复杂逻辑
{items.filter(i => i.active).map(i => i.name).join(', ')}

// ✅ 提取为变量或 useMemo
const activeNames = useMemo(
  () => items.filter(i => i.active).map(i => i.name).join(', '),
  [items]
);
{activeNames}

// ❌ 把整个组件树包裹在 Context.Provider 值变化时全部重渲染
// ✅ 拆分 Context 或使用 useMemo 稳定 value
const themeValue = useMemo(() => ({ theme, setTheme }), [theme]);

// ===== 3. 数据请求最佳实践 =====

// ✅ 使用 useReducer 管理请求状态
const initialState = { data: null, loading: false, error: null };

function dataReducer(state, action) {
  switch (action.type) {
    case 'FETCH_START':  return { ...state, loading: true, error: null };
    case 'FETCH_SUCCESS': return { data: action.payload, loading: false, error: null };
    case 'FETCH_ERROR':  return { ...state, loading: false, error: action.payload };
    default: return state;
  }
}

// ===== 4. 性能优化的使用原则 =====
// - 不要过早优化（Profiler 先找出瓶颈）
// - 不要在"静态" props 的子组件上使用 React.memo
// - useCallback 只在传给 memo 子组件时使用
// - useMemo 只在计算量明显大时使用
```

---

### 63. 从零到一的 React 项目搭建决策

```
           ┌─ 是否需要 SEO？
           │
       ┌───┴───┐
      是        否
       │         │
   需要 SSR    纯 SPA
       │         │
   ┌───┴───┐     │
 Next.js  Remix    │
                    │
            ┌───────┴────────┐
            │                 │
        简单项目           复杂项目
            │                 │
       Vite + React      Vite + React
            │                 │
    ┌───────┴───────┐    ┌────┴────┐
    │               │    │         │
  状态管理         CSS   路由      状态管理
    │               │    │         │
  Zustand       Tailwind  React   Redux Toolkit
  (轻量)        CSS      Router   (大型)
                        v6

工具链标配：
├── Vite (构建工具)
├── TypeScript (类型安全)
├── ESLint + Prettier (代码规范)
├── Husky + lint-staged (Git Hooks)
├── Vitest + React Testing Library (测试)
├── TanStack Query (服务端状态)
└── Zustand / Jotai (客户端状态)
```


---

## 十七、Next.js App Router 深度解析

### 64. App Router vs Pages Router 核心差异

```jsx
// ===== Pages Router（旧） =====
// pages/blog/[slug].js
export async function getServerSideProps({ params }) {
  const post = await getPost(params.slug);
  return { props: { post } };
}

function BlogPost({ post }) {
  return <article>{post.content}</article>;
}

// ===== App Router（新） =====
// app/blog/[slug]/page.js
// 默认是 Server Component，可直接 async
async function BlogPost({ params }) {
  const post = await getPost(params.slug); // 直接在组件内 await
  return <article>{post.content}</article>;
}
```

| 维度 | Pages Router | App Router |
|------|-------------|------------|
| 组件模型 | 默认 Client | 默认 Server（RSC） |
| 数据获取 | `getServerSideProps` / `getStaticProps` | 组件内直接 `async/await` |
| 布局 | `_app.js` + `_document.js` 手动组合 | `layout.js`（嵌套布局、状态保持） |
| 路由分组 | 不支持 | `(group)` 目录（不影响 URL） |
| 并行路由 | 不支持 | `@slot` 目录（Slot 机制） |
| 流式渲染 | 手动配置 | `loading.js` + `Suspense` 原生支持 |
| 错误处理 | `_error.js` 顶层 | `error.js` 各级粒度 |
| 缓存策略 | 4 种函数 | 更精细的缓存控制（fetch cache、revalidate） |
| Streaming | 不支持 | `Suspense` 原生流式 SSR |

---

### 65. App Router 文件约定与路由系统

```
app/
├── layout.js              # 根布局（必须，包裹所有页面）
├── page.js                # 首页 /
├── loading.js             # 页面加载状态（自动包裹 Suspense）
├── error.js               # 错误边界（自动包裹 Error Boundary）
├── not-found.js           # 404 页面
├── global-error.js        # 全局错误（替代根 error.js）
│
├── blog/
│   ├── layout.js          # 博客子布局
│   ├── page.js            # /blog
│   ├── loading.js         # /blog 加载状态
│   ├── [slug]/            # 动态路由
│   │   └── page.js        # /blog/my-post
│   └── [...catchAll]/     # 捕获所有路由
│       └── page.js        # /blog/a/b/c
│
└── (marketing)/           # 路由分组（不影响 URL）
    ├── layout.js          # marketing 专属布局
    ├── about/page.js      # /about
    └── contact/page.js    # /contact
```

```jsx
// ===== layout.js：嵌套布局，页面切换时保持状态 =====
// app/blog/layout.js
export default function BlogLayout({ children }) {
  return (
    <div>
      <BlogSidebar />       {/* 导航切换时不重新渲染 */}
      <main>{children}</main> {/* 只有 children 变化 */}
    </div>
  );
}

// ===== loading.js：自动 Loading UI（基于 Suspense） =====
// app/blog/loading.js
export default function BlogLoading() {
  return (
    <div className="skeleton">
      <SkeletonLine />
      <SkeletonLine width="80%" />
      <SkeletonLine width="60%" />
    </div>
  );
}

// ===== error.js：细粒度错误处理 =====
'use client'; // error.js 必须是 Client Component
// app/blog/error.js
export default function BlogError({ error, reset }) {
  return (
    <div>
      <h2>加载文章失败</h2>
      <p>{error.message}</p>
      <button onClick={() => reset()}>重试</button>
    </div>
  );
}

// ===== template.js：重新挂载的布局 =====
// 与 layout.js 不同，每次导航都重新挂载（适合动画场景）
export default function Template({ children }) {
  return <div className="page-transition">{children}</div>;
}
```

---

### 66. 服务端数据获取与缓存策略

```jsx
// ===== 1. 静态数据：构建时获取，永久缓存 =====
// app/blog/page.js
async function BlogPage() {
  const posts = await fetch('https://api.example.com/posts');
  // 默认 cache: 'force-cache'，等同于 getStaticProps
  return <PostList posts={posts} />;
}

// ===== 2. 动态数据：每次请求都获取 =====
async function BlogPage() {
  const posts = await fetch('https://api.example.com/posts', {
    cache: 'no-store', // 等同于 getServerSideProps
  });
  return <PostList posts={posts} />;
}

// ===== 3. ISR：定时重新生成 =====
async function BlogPage() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { revalidate: 3600 }, // 每 1 小时重新生成
  });
  return <PostList posts={posts} />;
}

// ===== 4. 按需重新验证（On-Demand Revalidation） =====
// app/api/revalidate/route.js
import { revalidateTag, revalidatePath } from 'next/cache';

export async function POST(request) {
  const { tag, path } = await request.json();

  if (tag) {
    revalidateTag(tag);  // 清除指定 tag 的缓存
  }

  if (path) {
    revalidatePath(path); // 清除指定路径的缓存
  }

  return Response.json({ revalidated: true });
}

// 在上面的页面中使用 tag
async function BlogPage() {
  const posts = await fetch('https://api.example.com/posts', {
    next: { tags: ['posts'], revalidate: 3600 },
  });
  return <PostList posts={posts} />;
}

// ===== 5. 使用 unstable_cache 缓存任意异步函数 =====
import { unstable_cache } from 'next/cache';

const getCachedPost = unstable_cache(
  async (slug) => {
    const post = await db.post.findUnique({ where: { slug } });
    return post;
  },
  ['post-detail'],  // cache key
  { revalidate: 86400, tags: ['posts'] }
);

// ===== 6. 请求去重（Request Dedup） =====
// Next.js 自动在同一渲染周期内去重相同 URL 的 fetch 请求
async function BlogPage({ params }) {
  // 这两个请求不会重复发送！
  const [post, comments] = await Promise.all([
    getPost(params.slug),
    getPost(params.slug), // 自动去重，不会再次请求
  ]);
}
```

---

### 67. Server Actions — 表单提交的新范式

```jsx
// ===== 基础 Server Action =====
// app/actions.js
'use server';

import { revalidatePath } from 'next/cache';
import { db } from '@/lib/db';
import { z } from 'zod'; // 校验

// 带校验的 mutation
export async function createPost(formData) {
  // 服务端校验
  const schema = z.object({
    title: z.string().min(3).max(100),
    content: z.string().min(10),
  });

  const parsed = schema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  });

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors };
  }

  await db.post.create({ data: parsed.data });
  revalidatePath('/blog'); // 重新验证缓存

  return { success: true };
}

// ===== 在组件中使用 =====
// app/blog/create/page.js
'use client';
import { useFormState, useFormStatus } from 'react-dom';
import { createPost } from '@/app/actions';

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? '发布中...' : '发布文章'}
    </button>
  );
}

function CreatePostForm() {
  const [state, formAction] = useFormState(createPost, null);

  return (
    <form action={formAction}>
      <input name="title" required />
      {state?.error?.title && <span>{state.error.title[0]}</span>}

      <textarea name="content" required />
      {state?.error?.content && <span>{state.error.content[0]}</span>}

      <SubmitButton />
      {state?.success && <p className="text-green-500">发布成功！</p>}
    </form>
  );
}

// ===== 非表单场景的 Server Action =====
'use server';
export async function deletePost(id) {
  await db.post.delete({ where: { id } });
  revalidatePath('/blog');
}

// 客户端按钮直接调用
'use client';
import { deletePost } from '@/app/actions';
import { useTransition } from 'react';

function DeleteButton({ postId }) {
  const [isPending, startTransition] = useTransition();

  const handleDelete = () => {
    startTransition(async () => {
      await deletePost(postId);
    });
  };

  return (
    <button onClick={handleDelete} disabled={isPending}>
      {isPending ? '删除中...' : '删除'}
    </button>
  );
}
```

---

### 68. Next.js 中间件与认证方案

```jsx
// ===== middleware.js（根目录） =====
// middleware.js
import { NextResponse } from 'next/server';
import { getToken } from 'next-auth/jwt';

export async function middleware(request) {
  const { pathname } = request.nextUrl;

  // 1. 路由保护
  const token = await getToken({ req: request, secret: process.env.NEXTAUTH_SECRET });

  const isAuthPage = pathname.startsWith('/login') || pathname.startsWith('/register');

  // 登录页：已认证则跳首页
  if (token && isAuthPage) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }

  // 受保护路由
  const protectedPaths = ['/dashboard', '/profile', '/admin'];
  const isProtected = protectedPaths.some(p => pathname.startsWith(p));

  if (!token && isProtected) {
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('callbackUrl', pathname);
    return NextResponse.redirect(loginUrl);
  }

  // 2. CORS 配置
  if (pathname.startsWith('/api')) {
    const response = NextResponse.next();
    response.headers.set('Access-Control-Allow-Origin', '*');
    response.headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
    return response;
  }

  // 3. Geo 定位（通过 Vercel 的 x-vercel-ip-country header）
  const country = request.headers.get('x-vercel-ip-country') || 'US';

  if (pathname.startsWith('/blocked') && country === 'XX') {
    return NextResponse.redirect(new URL('/403', request.url));
  }

  return NextResponse.next();
}

// 指定 middleware 匹配的路径
export const config = {
  matcher: [
    '/((?!_next|api/auth|static|favicon.ico).*)', // 排除静态资源和 auth API
  ],
};

// ===== Auth.js (NextAuth v5) 完整认证流程 =====
// app/api/auth/[...nextauth]/route.js
import { handlers } from '@/auth';
export const { GET, POST } = handlers;

// auth.js
import NextAuth from 'next-auth';
import GitHub from 'next-auth/providers/github';
import Credentials from 'next-auth/providers/credentials';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { db } from '@/lib/db';

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(db),
  providers: [
    GitHub({ clientId: process.env.GITHUB_ID, clientSecret: process.env.GITHUB_SECRET }),
    Credentials({
      credentials: { email: {}, password: {} },
      authorize: async (credentials) => {
        const user = await db.user.findUnique({ where: { email: credentials.email } });
        if (user && bcrypt.compareSync(credentials.password, user.password)) {
          return user;
        }
        return null;
      },
    }),
  ],
  callbacks: {
    jwt: ({ token, user }) => {
      if (user) token.role = user.role;
      return token;
    },
    session: ({ session, token }) => {
      session.user.role = token.role;
      return session;
    },
  },
  pages: {
    signIn: '/login',
  },
});

// 在组件中使用
// app/dashboard/page.js
import { auth } from '@/auth';

export default async function Dashboard() {
  const session = await auth(); // Server Component 直接获取 session

  return (
    <div>
      <h1>Welcome, {session?.user?.name}</h1>
      <p>Role: {session?.user?.role}</p>
    </div>
  );
}
```

---

## 十八、React 服务器组件与流式渲染

### 69. RSC（React Server Components）核心原理与限制

```jsx
// ===== Server Component 能力清单 =====
async function ServerComponent({ postId }) {
  // ✅ 可以做:
  const post = await db.post.findUnique({ where: { id: postId } }); // 直接查数据库
  const file = fs.readFileSync('./content.md', 'utf-8');              // 读文件系统
  const data = await fetch('https://internal-api/project/data', {    // 直连内部服务
    headers: { Authorization: `Bearer ${process.env.API_TOKEN}` },   // 使用服务端密钥
  });

  return (
    <article>
      <h1>{post.title}</h1>
      <Markdown content={file} /> {/* 可以引入大的依赖，不会发送到客户端 */}
    </article>
  );
}

// ❌ 不能做:
// - 使用 useState、useEffect 等 Hooks
// - 使用浏览器 API（window、document、localStorage）
// - 绑定事件处理器（onClick 等）
// - 使用 Context（Server Context 不同，通过树传递 Promise）

// ===== Server Component 与 Client Component 的边界 =====

// 1. Server → Client：通过 Props 传递（必须可序列化）
'use client';
function ClientPost({ post }) { // post 从 Server Component 传过来（JSON 序列化）
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>{post.title}</button>;
}

// 2. Client → Server：通过 children 插槽
'use client';
function ClientLayout({ children }) { // 可以接收 Server Component 作为 children
  const [theme] = useState('dark');
  return (
    <ThemeProvider value={theme}>
      <nav>...</nav>
      {children} {/* 这里是 Server Component，可以先在服务端渲染 */}
    </ThemeProvider>
  );
}

// 3. 混合模式：服务端序列化成 VDOM JSON，客户端 Reconciliation
async function HybridPage() {
  const products = await getProducts(); // 服务端

  return (
    <div>
      <ProductList products={products} />    {/* Server */}
      {'use client'; /* 标记客户端边界 */}
      <AddToCartButton productId={products[0].id} /> {/* Client */}
    </div>
  );
}

// ===== 关键原理 =====
// 1. RSC Payload：Server Component 渲染为一种特殊的 JSON 格式（不是 HTML）
// 2. 客户端收到后，React 进行 Reconciliation，不需要发 JS 代码
// 3. 只有 Client Component 的 JS 代码会被打包发送
// 4. Server Component 可以直接引入 Server 侧模块，零 Bundle 体积
```

**常见误区澄清：**

| 误区 | 真相 |
|------|------|
| RSC 就是 SSR | RSC ≠ SSR。RSC 是组件模型，SSR 是渲染策略。两者可以组合使用 |
| RSC 替代了 CSR | RSC 只在 Pages/App Router 等服务端框架中可用，纯 CSR 项目无法使用 |
| RSC 不需要客户端 JS | 页面中仍可以有 Client Component，只有 Server Component 不产生客户端 JS |
| `use server` 和 `use client` 是相反的 | 不完全。`use client` 标记 Client Component 边界；`use server` 标记 Server Action 入口 |

---

### 70. Suspense 与 Streaming SSR 深度解析

```jsx
// ===== 1. Suspense 基础：声明式加载 =====
import { Suspense } from 'react';

function Page() {
  return (
    <div>
      <Suspense fallback={<ProfileSkeleton />}>
        <UserProfile /> {/* 异步组件，resolve 前显示 fallback */}
      </Suspense>

      <Suspense fallback={<PostsSkeleton />}>
        <Posts />
      </Suspense>
      {/* 两个 Suspense 独立加载，互不阻塞 */}
    </div>
  );
}

// ===== 2. Streaming SSR：边渲染边发送 =====
// 在 Next.js App Router 中，layout.js + page.js + loading.js 自动实现 Streaming
// 手动实现：
<Suspense fallback={<Skeleton />}>
  <SlowComponent />
</Suspense>
// 流程：
// 1. 服务端渲染时，遇到 Suspense 先发送 fallback HTML + <script>占位符
// 2. SlowComponent 数据就绪后，继续渲染并流式发送结果 + 内联 <script>替换占位符
// 3. 用户看到页面逐步加载，不需要等待所有数据

// ===== 3. use() API + Suspense 组合 =====
import { use, Suspense } from 'react';

// 把 Promise 传入组件作为 prop
function AsyncData({ dataPromise }) {
  const data = use(dataPromise); // 直接在渲染中使用，自动集成 Suspense
  return <div>{data}</div>;
}

function Page() {
  const dataPromise = fetchData(); // 组件外启动请求

  return (
    <Suspense fallback={<Loading />}>
      <AsyncData dataPromise={dataPromise} />
    </Suspense>
  );
}

// ===== 4. Suspense 的嵌套顺序（重要！）=====
// ❌ 糟糕的嵌套：所有内容等最慢的
<Suspense fallback={<PageSkeleton />}>
  <Header />          {/* 快 */}
  <SlowContent />     {/* 慢：阻塞了所有 */}
  <Footer />          {/* 快 */}
</Suspense>

// ✅ 好的嵌套：每个部分独立加载
<>
  <Header />
  <Suspense fallback={<ContentSkeleton />}>
    <SlowContent />
  </Suspense>
  <Footer />
</>

// ===== 5. Suspense 与 ErrorBoundary 配合 =====
function Page() {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Suspense fallback={<Loading />}>
        <MightFailComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
// Suspense 处理加载态，ErrorBoundary 处理错误态

// ===== 6. Suspense 与 Transition 配合 =====
// app/search/page.js
'use client';
import { useTransition, Suspense } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    // startTransition 包裹：保持旧 UI 可见，避免 Suspense 回退
    startTransition(() => {
      setQuery(e.target.value);
    });
  };

  return (
    <div>
      <input value={query} onChange={handleChange} />
      <div style={{ opacity: isPending ? 0.7 : 1 }}>
        <Suspense fallback={<SearchSkeleton />}>
          <SearchResults query={query} />
        </Suspense>
      </div>
    </div>
  );
}
```

---

### 71. React 渲染性能的黄金法则

```jsx
// ===== 法则 1：减少渲染次数 =====

// 1a. 状态拆分（避免无关更新）
// ❌ 一个 state 对象：任何字段改变都触发整个组件重渲染
const [form, setForm] = useState({ name: '', email: '', bio: '' });

// ✅ 拆分 state：只更新改变的字段
const [name, setName] = useState('');
const [email, setEmail] = useState('');
const [bio, setBio] = useState('');

// 1b. 状态下沉（State Colocation）
// ❌ 状态在顶层 => 整个页面重渲染
function Page() {
  const [search, setSearch] = useState(''); // 每次输入，整个 Page 渲染
  return (
    <div>
      <input value={search} onChange={e => setSearch(e.target.value)} />
      <ExpensiveChart /> {/* 也会重渲染！ */}
    </div>
  );
}

// ✅ 状态下沉到输入组件内部
function Page() {
  return (
    <div>
      <SearchInput />     {/* 状态隔离 */}
      <ExpensiveChart />  {/* 不受影响 */}
    </div>
  );
}

// ===== 法则 2：减少渲染开销 =====

// 2a. 派生数据用 useMemo
function ProductList({ products, filterCategory }) {
  const filteredProducts = useMemo(
    () => products.filter(p => p.category === filterCategory),
    [products, filterCategory]
  ); // 只在依赖变化时重新计算

  return <List items={filteredProducts} />;
}

// 2b. 子组件提纯（提取无状态部分）
// ❌ 计算量大 + 状态混在一起
function Component({ data }) {
  const [value, setValue] = useState(0);
  const expensive = heavyCalculation(data); // 每次渲染都重新计算
  return <div onClick={() => setValue(v => v + 1)}>{expensive}</div>;
}

// ✅ 提取纯渲染部分，用 React.memo 包裹
const PureDisplay = React.memo(function({ data }) {
  const expensive = heavyCalculation(data);
  return <div>{expensive}</div>;
});

function Component({ data }) {
  const [value, setValue] = useState(0);
  return <PureDisplay data={data} onClick={() => setValue(v => v + 1)} />;
}

// ===== 法则 3：代码分割（Code Splitting）=====

// 3a. 路由级分割
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

// 3b. 组件级分割
const HeavyChart = lazy(() => import('./HeavyChart'));

function Page() {
  const [showChart, setShowChart] = useState(false);
  return (
    <div>
      <button onClick={() => setShowChart(true)}>显示图表</button>
      {showChart && (
        <Suspense fallback={<div>加载中...</div>}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}

// 3c. 条件加载（Intersection Observer）
function LazySection({ component: Comp }) {
  const ref = useRef(null);
  const [shouldLoad, setShouldLoad] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        setShouldLoad(true);
        observer.disconnect();
      }
    });
    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return (
    <div ref={ref}>
      {shouldLoad ? <Comp /> : <div style={{ height: 300 }} />}
    </div>
  );
}
```

---

### 72. 大列表与虚拟滚动的高级实践

```jsx
// ===== react-window 完整封装 =====
import { FixedSizeList } from 'react-window';

function VirtualList({ items, rowHeight = 50, height = 600 }) {
  const Row = ({ index, style }) => (
    <div style={style} className="list-item">
      {items[index].name}
    </div>
  );

  return (
    <FixedSizeList
      height={height}
      itemCount={items.length}
      itemSize={rowHeight}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}

// ===== 动态高度列表（VariableSizeList）=====
import { VariableSizeList } from 'react-window';

function DynamicVirtualList({ items }) {
  const listRef = useRef();
  const rowHeights = useRef({});

  // 获取行高
  const getItemSize = (index) => rowHeights.current[index] || 60;

  // 渲染后测量实际高度
  const Row = ({ index, style }) => {
    const rowRef = useRef();

    useEffect(() => {
      if (rowRef.current) {
        const height = rowRef.current.getBoundingClientRect().height;
        if (rowHeights.current[index] !== height) {
          rowHeights.current[index] = height;
          listRef.current?.resetAfterIndex(index); // 重置后续行位置缓存
        }
      }
    });

    return (
      <div ref={rowRef} style={style}>
        <div style={{ padding: 8 }}>{items[index].text}</div>
      </div>
    );
  };

  return (
    <VariableSizeList
      ref={listRef}
      height={600}
      itemCount={items.length}
      itemSize={getItemSize}
      width="100%"
    >
      {Row}
    </VariableSizeList>
  );
}

// ===== 支持滚动到指定位置 =====
import { FixedSizeList } from 'react-window';

function ScrollableList({ items, scrollToIndex }) {
  const listRef = useRef();

  useEffect(() => {
    if (scrollToIndex != null) {
      listRef.current?.scrollToItem(scrollToIndex, 'center');
    }
  }, [scrollToIndex]);

  return (
    <FixedSizeList ref={listRef} height={600} itemCount={items.length} itemSize={50}>
      {({ index, style }) => (
        <div style={style} className={index === scrollToIndex ? 'highlight' : ''}>
          {items[index]}
        </div>
      )}
    </FixedSizeList>
  );
}

// ===== 无限加载 + 虚拟滚动 =====
function InfiniteVirtualList({ loadMore, hasMore, items }) {
  const listRef = useRef();

  // react-window 的 onItemsRendered 回调
  const handleItemsRendered = useCallback(
    ({ visibleStopIndex }) => {
      if (visibleStopIndex >= items.length - 5 && hasMore) {
        loadMore();
      }
    },
    [items.length, hasMore, loadMore]
  );

  return (
    <FixedSizeList
      ref={listRef}
      height={600}
      itemCount={hasMore ? items.length + 1 : items.length}
      itemSize={50}
      onItemsRendered={handleItemsRendered}
    >
      {({ index, style }) => {
        if (index >= items.length) {
          return <div style={style}>加载更多...</div>;
        }
        return <div style={style}>{items[index].name}</div>;
      }}
    </FixedSizeList>
  );
}

// ===== 虚拟滚动的替代方案对比 =====
// react-window:   最小体积，适合固定/变化高度，不支持 grid 内置（用 react-virtualized-auto-sizer 补）
// react-virtual:  框架无关，支持 grid，API 更灵活，体积稍大
// react-virtuoso: 自动处理动态高度，无需传 itemSize，支持分组、粘性头部
// @tanstack/react-virtual:  最新 API，支持 solid/vue/react 多框架，高度可定制
```

---

## 十九、前端架构与工程化进阶

### 73. Monorepo 架构在 React 项目中的实践

```
my-monorepo/
├── apps/
│   ├── web/                    # 主站 (React + Next.js)
│   │   ├── next.config.js
│   │   └── package.json
│   ├── admin/                  # 管理后台 (React + Vite)
│   │   ├── vite.config.ts
│   │   └── package.json
│   └── mobile/                 # 移动端 (React Native / Expo)
├── packages/
│   ├── ui/                     # 共享 UI 组件库
│   │   ├── src/
│   │   │   ├── Button.tsx
│   │   │   ├── Modal.tsx
│   │   │   └── index.ts
│   │   └── package.json
│   ├── utils/                  # 共享工具函数
│   │   ├── src/
│   │   │   ├── format.ts
│   │   │   ├── api.ts
│   │   │   └── hooks/
│   │   └── package.json
│   ├── eslint-config/          # 共享 ESLint 配置
│   └── typescript-config/      # 共享 TS 配置
├── turbo.json                  # Turborepo 配置
├── pnpm-workspace.yaml
└── package.json
```

```jsonc
// pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

```js
// turbo.json（Turborepo 任务编排）
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],     // 先构建依赖包
      "outputs": ["dist/**", ".next/**"]
    },
    "lint": {
      "dependsOn": ["^build"]      // lint 前先构建依赖
    },
    "dev": {
      "cache": false,              // 开发模式不缓存
      "persistent": true           // 长时间运行
    },
    "test": {
      "dependsOn": ["build"],
      "inputs": ["src/**/*.tsx", "src/**/*.ts", "test/**/*.ts"]
    }
  }
}
```

```jsx
// apps/web/src/App.tsx — 引用 package 中的组件
import { Button, Modal } from '@my-org/ui';
import { formatCurrency } from '@my-org/utils';

function App() {
  return <Button onClick={() => {}}>支付 {formatCurrency(99.99)}</Button>;
}
```

**Monorepo 工具对比：**

| 工具 | 特点 | 适合 |
|------|------|------|
| **Turborepo** | 极速构建缓存、并行执行、增量构建 | Vercel 生态、JS/TS 项目 |
| **Nx** | 依赖图可视化、代码生成器、智能检测受影响项目 | 大型企业级项目 |
| **pnpm workspace** | 最轻量、依赖去重、严格模式 | 中小型项目 |
| **Lerna** | 老牌工具，现在常配合 Nx/Turborepo 使用 package 发布 | npm 包管理 |
| **Rush** | 微软出品、极严格的策略控制 | 超大型组织(100+ packages) |

---

### 74. 组件库建设与设计系统

```jsx
// ===== 组件库设计原则 =====

// 1. 原子化 + 组合化（Radix UI / shadcn/ui 哲学）
// 提供无样式基础组件 + 可组合的原语

// 2. 组件变体设计（CVA — Class Variance Authority）
import { cva } from 'class-variance-authority';

const buttonVariants = cva(
  // 基础样式
  'inline-flex items-center justify-center rounded-md font-medium transition-colors',
  {
    variants: {
      variant: {
        primary: 'bg-blue-600 text-white hover:bg-blue-700',
        secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200',
        ghost: 'hover:bg-gray-100 text-gray-700',
        danger: 'bg-red-600 text-white hover:bg-red-700',
      },
      size: {
        sm: 'h-8 px-3 text-xs',
        md: 'h-10 px-4 text-sm',
        lg: 'h-12 px-6 text-base',
      },
      fullWidth: {
        true: 'w-full',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

// 使用
function Button({ variant, size, fullWidth, className, ...props }) {
  return (
    <button
      className={buttonVariants({ variant, size, fullWidth, className })}
      {...props}
    />
  );
}

// 3. 复合组件模式（Radix 风格）
function Dialog({ children, open, onOpenChange }) {
  if (!open) return null;
  return (
    <DialogContext.Provider value={{ onOpenChange }}>
      <div className="fixed inset-0 z-50">
        <div className="fixed inset-0 bg-black/50" onClick={() => onOpenChange(false)} />
        <div className="fixed inset-0 flex items-center justify-center">
          {children}
        </div>
      </div>
    </DialogContext.Provider>
  );
}

Dialog.Title = function DialogTitle({ children }) {
  return <h2 className="text-lg font-semibold">{children}</h2>;
};

Dialog.Content = function DialogContent({ children }) {
  return (
    <div className="bg-white rounded-lg shadow-xl p-6 max-w-md w-full">
      {children}
    </div>
  );
};

Dialog.Close = function DialogClose() {
  const { onOpenChange } = useContext(DialogContext);
  return <button onClick={() => onOpenChange(false)}>关闭</button>;
};

// 使用
<Dialog open={isOpen} onOpenChange={setIsOpen}>
  <Dialog.Content>
    <Dialog.Title>确认删除</Dialog.Title>
    <p>此操作不可撤销</p>
    <Dialog.Close />
  </Dialog.Content>
</Dialog>
```

```jsx
// ===== 设计 Token 体系 =====
// 通过 CSS 变量 + Tailwind 配置统一设计语言

// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        // 语义化色值
        brand: {
          50:  'var(--color-brand-50)',
          500: 'var(--color-brand-500)',
          900: 'var(--color-brand-900)',
        },
      },
      spacing: {
        '18': '4.5rem',
      },
      borderRadius: {
        '4xl': '2rem',
      },
      // 动画 Token
      animation: {
        'fade-in': 'fadeIn 200ms ease-out',
        'slide-up': 'slideUp 300ms ease-out',
      },
    },
  },
};

// 组件库发布配置
// package.json（packages/ui/）
{
  "name": "@my-org/ui",
  "version": "1.0.0",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "sideEffects": false,  // 支持 tree-shaking
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "scripts": {
    "build": "tsup src/index.ts --format esm --dts",
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build"
  }
}
```

---

### 75. React 项目的国际化（i18n）方案

```jsx
// ===== 方案 1：next-intl（Next.js App Router 推荐）=====

// i18n.ts — 服务端配置
import { getRequestConfig } from 'next-intl/server';

export default getRequestConfig(async ({ locale }) => ({
  messages: (await import(`./messages/${locale}.json`)).default,
}));

// messages/zh.json
{
  "home": {
    "title": "欢迎来到我们的网站",
    "description": "这是首页",
    "userGreeting": "你好，{name}！你有 {count, plural, =0 {零条消息} one {一条消息} other {# 条消息}}"
  },
  "button": {
    "submit": "提交",
    "cancel": "取消"
  }
}

// app/[locale]/layout.js
import { NextIntlClientProvider } from 'next-intl';
import { getMessages } from 'next-intl/server';

export default async function LocaleLayout({ children, params: { locale } }) {
  const messages = await getMessages();
  return (
    <html lang={locale}>
      <NextIntlClientProvider messages={messages}>
        {children}
      </NextIntlClientProvider>
    </html>
  );
}

// 使用（Server Component）
import { useTranslations } from 'next-intl';

export default function HomePage() {
  const t = useTranslations('home');
  return (
    <div>
      <h1>{t('title')}</h1>
      <p>{t('userGreeting', { name: 'Alice', count: 3 })}</p>
    </div>
  );
}

// Client Component 中使用
'use client';
import { useTranslations } from 'next-intl';

function SubmitButton() {
  const t = useTranslations('button');
  return <button>{t('submit')}</button>;
}

// ===== 方案 2：react-i18next（通用方案） =====

// i18n.js
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import Backend from 'i18next-http-backend';
import LanguageDetector from 'i18next-browser-languagedetector';

i18n
  .use(Backend)          // 按需加载翻译文件
  .use(LanguageDetector) // 自动检测语言
  .use(initReactI18next)
  .init({
    fallbackLng: 'zh',
    supportedLngs: ['zh', 'en', 'ja'],
    interpolation: { escapeValue: false }, // React 已处理 XSS
    backend: {
      loadPath: '/locales/{{lng}}/{{ns}}.json', // 按需加载
    },
    detection: {
      order: ['cookie', 'localStorage', 'navigator'],
      caches: ['cookie'],
    },
  });

// 使用 React 组件
import { useTranslation, Trans } from 'react-i18next';

function Component() {
  const { t } = useTranslation();

  return (
    <div>
      <h1>{t('title')}</h1>

      {/* Trans 组件：翻译含 HTML 的文本 */}
      <Trans i18nKey="terms">
        点击即表示同意
        <a href="/terms">服务条款</a>
        和
        <a href="/privacy">隐私政策</a>
      </Trans>

      {/* 复数形式 */}
      <p>{t('itemCount', { count: items.length })}</p>
    </div>
  );
}
```

**i18n 方案选择：**

| 场景 | 推荐方案 |
|------|----------|
| Next.js App Router | `next-intl`（一流支持） |
| Next.js Pages Router | `next-i18next` |
| Create React App / Vite | `react-i18next` |
| 需要 ICU MessageFormat | `formatjs` / `react-intl` |
| 最少依赖 | 自建 Context + JSON |

---

### 76. GraphQL 在 React 中的最佳实践

```jsx
// ===== Apollo Client 完整用法 =====

// client.js
import { ApolloClient, InMemoryCache, HttpLink, ApolloLink } from '@apollo/client';
import { onError } from '@apollo/client/link/error';
import { RetryLink } from '@apollo/client/link/retry';

const httpLink = new HttpLink({ uri: '/api/graphql' });

const errorLink = onError(({ graphQLErrors, networkError }) => {
  if (graphQLErrors) {
    graphQLErrors.forEach(({ message, extensions }) => {
      if (extensions.code === 'UNAUTHENTICATED') {
        window.location.href = '/login';
      }
    });
  }
});

const retryLink = new RetryLink({
  delay: { initial: 300 },
  attempts: { max: 3, retryIf: (error) => !!error },
});

export const client = new ApolloClient({
  link: ApolloLink.from([errorLink, retryLink, httpLink]),
  cache: new InMemoryCache({
    typePolicies: {
      Query: {
        fields: {
          posts: {
            // 合并分页数据
            keyArgs: false,
            merge(existing = [], incoming) {
              return [...existing, ...incoming];
            },
          },
        },
      },
    },
  }),
});

// ===== useQuery — 基础查询 =====
import { useQuery, gql } from '@apollo/client';

const GET_POST = gql`
  query GetPost($id: ID!) {
    post(id: $id) {
      id
      title
      content
      author {
        name
        avatar
      }
    }
  }
`;

function PostDetail({ postId }) {
  const { data, loading, error } = useQuery(GET_POST, {
    variables: { id: postId },
    skip: !postId,          // 条件跳过
    pollInterval: 30000,    // 每 30 秒轮询
    fetchPolicy: 'cache-and-network', // 先显示缓存再请求网络
    onCompleted: (data) => {
      console.log('Query completed', data);
    },
  });

  if (loading) return <Skeleton />;
  if (error) return <Error message={error.message} />;
  return <article>{data.post.content}</article>;
}

// ===== useMutation — 数据变更 =====
const CREATE_POST = gql`
  mutation CreatePost($input: CreatePostInput!) {
    createPost(input: $input) {
      id
      title
    }
  }
`;

function CreatePostForm() {
  const [createPost, { loading }] = useMutation(CREATE_POST, {
    // 自动更新缓存
    update(cache, { data: { createPost } }) {
      const existingPosts = cache.readQuery({ query: GET_POSTS });
      cache.writeQuery({
        query: GET_POSTS,
        data: {
          posts: [createPost, ...existingPosts.posts],
        },
      });
    },
    onCompleted: () => toast.success('创建成功'),
    onError: (error) => toast.error(error.message),
  });

  const handleSubmit = async (e) => {
    e.preventDefault();
    await createPost({ variables: { input: { title, content } } });
  };

  return <form onSubmit={handleSubmit}>...</form>;
}

// ===== useFragment — 组件级 Colocation =====
const POST_FIELDS = gql`
  fragment PostFields on Post {
    id
    title
    excerpt
    createdAt
    author { name }
  }
`;

function PostCard({ post }) {
  // useFragment 确保只请求需要的字段，组件自行定义数据需求
  const { complete, data } = useFragment({
    fragment: POST_FIELDS,
    from: post, // 从父组件传入的缓存引用
  });

  if (!complete) return <Skeleton />;
  return <div>{data.title} by {data.author.name}</div>;
}

// 父组件不需要知道 PostCard 需要哪些字段！
function PostList() {
  const { data } = useQuery(GET_POSTS);
  return data.posts.map(post => <PostCard key={post.id} post={post} />);
}
```

**GraphQL 客户端对比：**

| 特性 | Apollo Client | URQL | Relay |
|------|---------------|------|-------|
| Bundle 大小 | ~30KB | ~8KB | ~40KB |
| 学习曲线 | 中 | 低 | 高 |
| 缓存策略 | 丰富（归一化缓存） | 文档缓存 + Graphcache | 预编译器 + 静态分析 |
| Fragment Colocation | 支持 | 支持 | 原生核心 |
| 适合场景 | 大多数项目 | 轻量级、快速启动 | 大规模、需要编译器保证 |

---

### 77. React 项目的状态管理终极指南

```jsx
// ===== 状态分类与工具选择决策树 =====

/*
Server State（服务端状态）
  - 来自 API / 数据库的数据
  - 与 UI 无关
  - 需要缓存、去重、重新验证
  → TanStack Query / SWR / Apollo Client

Client State（客户端状态）
  ├── URL State（URL 状态）
  │   - 页面路由、查询参数、分页
  │   → React Router / Next.js useSearchParams
  │
  ├── UI State（UI 交互状态）
  │   - Modal 开关、表单输入、Tab 选中
  │   → useState / useReducer（局部）
  │
  ├── Global UI State（全局 UI 状态）
  │   - 主题、语言、侧边栏折叠
  │   → Context + useReducer / Zustand / Jotai
  │
  └── Complex State（复杂状态机）
      - 多步骤表单、审批流、播放器状态
      → useReducer / XState / Zustand

Form State（表单状态）
  - 输入值、校验、提交状态
  → React Hook Form / Formik / useActionState（React 19）

Animation State（动画状态）
  - 补间值、手势状态、布局动画
  → Framer Motion / React Spring / CSS transitions

Persisted State（持久化状态）
  - 本地存储、离线数据
  → Zustand persist / redux-persist / useLocalStorage
*/
```

**各状态库的 npm 趋势选型建议（2025-2026）：**

```
React 项目规模 → 状态管理方案
├── 微型 / 个人项目
│   → useState + Context（够了）
│
├── 小型项目（< 10 页面）
│   → Zustand（极简、无 Provider）
│   → TanStack Query（服务端状态）
│
├── 中型项目（10-50 页面）
│   → Zustand（UI 状态）+ TanStack Query（服务端状态）
│   → React Hook Form（表单）
│
├── 大型项目（50+ 页面）
│   → Zustand / Redux Toolkit（UI 状态）
│   → TanStack Query（服务端状态）
│   → React Hook Form + Zod（表单 + 校验）
│
└── 企业级项目
    → Redux Toolkit + RTK Query（统一方案）
    → 或信号方案（Preact Signals / Jotai）
    → XState（复杂状态机场景）
```

---

## 二十、高级 Hooks 与渲染机制

### 78. useLayoutEffect vs useEffect 的区别与使用场景

```jsx
// ===== 执行时机差异 =====
// useEffect：浏览器绘制之后异步执行（不阻塞渲染）
// useLayoutEffect：DOM 更新之后、浏览器绘制之前同步执行（阻塞渲染）

function LayoutExample() {
  const [value, setValue] = useState(0);

  useEffect(() => {
    // 执行顺序：组件渲染 → DOM 更新 → 浏览器绘制 → useEffect
    // 如果修改 DOM，用户会看到闪烁
    if (value === 0) setValue(1);
  }, [value]);

  useLayoutEffect(() => {
    // 同步修改 DOM 不会有闪烁
    if (value === 0) setValue(1);
  }, [value]);
}

// ===== 场景 1：读取 DOM 布局信息（必须用 useLayoutEffect） =====
function Tooltip({ targetRef, content }) {
  const tooltipRef = useRef();
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    const targetRect = targetRef.current.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect();

    setPosition({
      top: targetRect.bottom + 8,
      left: targetRect.left + targetRect.width / 2 - tooltipRect.width / 2,
    });
    // ✅ useLayoutEffect：计算位置 → 渲染 Tooltip → 绘制（无闪烁）
  }, [content, targetRef]);

  return (
    <div ref={tooltipRef} style={{ position: 'fixed', ...position }}>
      {content}
    </div>
  );
}

// ===== 场景 2：Scroll 位置恢复 =====
function ScrollRestorer() {
  useLayoutEffect(() => {
    const savedPosition = sessionStorage.getItem('scrollPosition');
    if (savedPosition) {
      window.scrollTo(0, parseInt(savedPosition));
    }
  }, []);
}
```

**选择原则：** 默认用 `useEffect`，只有需要**在浏览器绘制前同步修改 DOM** 时才用 `useLayoutEffect`。

---

### 79. useImperativeHandle 与 forwardRef 进阶

```jsx
const Input = forwardRef(function Input({ label, ...props }, ref) {
  const inputRef = useRef();

  // 只暴露需要的方法给父组件，封装内部实现
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    select: () => inputRef.current.select(),
    clear: () => {
      const nativeInputValueSetter = Object.getOwnPropertyDescriptor(
        window.HTMLInputElement.prototype, 'value'
      ).set;
      nativeInputValueSetter.call(inputRef.current, '');
      inputRef.current.dispatchEvent(new Event('input', { bubbles: true }));
    },
    getValue: () => inputRef.current.value,
  }), []);

  return (
    <div className="input-group">
      {label && <label>{label}</label>}
      <input ref={inputRef} {...props} />
    </div>
  );
});

// 父组件
function Parent() {
  const inputRef = useRef();
  return (
    <div>
      <Input ref={inputRef} label="用户名" />
      <button onClick={() => { inputRef.current?.clear(); inputRef.current?.focus(); }}>
        清空
      </button>
    </div>
  );
}
```

---

### 80. useSyncExternalStore — 订阅外部 Store

```jsx
import { useSyncExternalStore } from 'react';

// 订阅浏览器 API：窗口尺寸
function useWindowSize() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('resize', callback);
      return () => window.removeEventListener('resize', callback);
    },
    () => ({ width: window.innerWidth, height: window.innerHeight }),
    () => ({ width: 1200, height: 800 }) // SSR 快照
  );
}

// 手动实现迷你 Store（Zustand 内部就用的这个）
function createStore(initialState) {
  let state = initialState;
  const listeners = new Set();
  return {
    getState: () => state,
    setState: (newState) => {
      state = typeof newState === 'function' ? newState(state) : newState;
      listeners.forEach(l => l());
    },
    subscribe: (listener) => { listeners.add(listener); return () => listeners.delete(listener); },
  };
}

const counterStore = createStore({ count: 0 });

function useCounter() {
  return useSyncExternalStore(counterStore.subscribe, counterStore.getState);
}
// ✅ useSyncExternalStore 同步读取，并发模式安全，无 tearing 问题
```

---

### 81. useDeferredValue 与 useTransition 深度对比

```jsx
// useDeferredValue：延迟更新一个「值」，旧值保留到新值计算好
function SearchBox() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  // query 立即更新 → input 不卡；deferredQuery 延迟 → 列表可中断

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <Suspense fallback={<p>搜索中...</p>}>
        <SearchResults query={deferredQuery} />
      </Suspense>
    </div>
  );
}

// useTransition：标记状态更新为低优先级，返回 isPending 标识
function TabSwitcher() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();

  const handleTabChange = (newTab) => {
    startTransition(() => setTab(newTab));
  };

  return (
    <div>
      <button onClick={() => handleTabChange('posts')}>文章</button>
      <div style={{ opacity: isPending ? 0.7 : 1 }}>
        {tab === 'posts' && <SlowPosts />}
      </div>
    </div>
  );
}

// useTransition:     你「控制」何时更新（Tab 切换、路由导航）
// useDeferredValue:  你「不控制」何时更新（用户输入、外部推送）
```

---

### 82. React 18 自动批处理（Automatic Batching）

```jsx
// React 17：setTimeout/Promise 中的更新不会批处理 → 多次渲染
// React 18：所有更新自动批处理 → 一次渲染

function Example() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  const handleAsync = () => {
    setTimeout(() => {
      setCount(c => c + 1); // React 18：合并
      setFlag(f => !f);     // React 18：合并 → 仅 1 次渲染
    }, 1000);
  };
  // React 17: 3 次 setState → 3 次渲染
  // React 18: 3 次 setState → 1 次渲染（减少 66% 不必要渲染）

  // 需要立即读取更新后状态（不批处理）：
  const handleImmediate = () => {
    flushSync(() => setCount(c => c + 1)); // 强制同步渲染
  };
}
```

---

### 83. useInsertionEffect — CSS-in-JS 专用 Hook

```jsx
// 三种 Effect Hook 执行顺序：
// DOM 更新 → useInsertionEffect → useLayoutEffect → 浏览器绘制 → useEffect

// useInsertionEffect：CSS-in-JS 库作者专用，普通开发者几乎不需要
function useDynamicStyles(styles) {
  useInsertionEffect(() => {
    const style = document.createElement('style');
    style.textContent = styles;
    document.head.appendChild(style);
    return () => document.head.removeChild(style);
  }, [styles]);
  // ✅ 在 Layout 计算前插入样式，避免样式闪烁
}
```

---

## 二十一、React 实战进阶技巧

### 84. React Hook Form + Zod 表单校验完整方案

```jsx
import { useForm, useFieldArray } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const signUpSchema = z.object({
  username: z.string().min(3, '至少3个字符').max(20).regex(/^\w+$/, '只能是字母数字下划线'),
  email: z.string().email('邮箱格式不正确'),
  password: z.string().min(8, '至少8个字符').regex(/[A-Z]/, '至少一个大写字母').regex(/[0-9]/, '至少一个数字'),
  confirmPassword: z.string(),
  age: z.number().min(18, '必须年满18岁').max(120),
}).refine(data => data.password === data.confirmPassword, {
  message: '两次密码不一致',
  path: ['confirmPassword'],
});

function SignUpForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting, isDirty } } = useForm({
    resolver: zodResolver(signUpSchema),
    mode: 'onBlur',
  });

  const onSubmit = async (data) => {
    // data 已通过校验，类型安全
    await api.register(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="max-w-md space-y-4">
      <div>
        <input {...register('username')} placeholder="用户名" />
        {errors.username && <p className="text-red-500">{errors.username.message}</p>}
      </div>
      <div>
        <input {...register('email')} type="email" placeholder="邮箱" />
        {errors.email && <p className="text-red-500">{errors.email.message}</p>}
      </div>
      <div>
        <input {...register('password')} type="password" placeholder="密码" />
        {errors.password && <p className="text-red-500">{errors.password.message}</p>}
      </div>
      <div>
        <input {...register('confirmPassword')} type="password" placeholder="确认密码" />
        {errors.confirmPassword && <p className="text-red-500">{errors.confirmPassword.message}</p>}
      </div>
      <button type="submit" disabled={isSubmitting || !isDirty}>
        {isSubmitting ? '提交中...' : '注册'}
      </button>
    </form>
  );
}
```

**React Hook Form 性能优势：** 非受控组件模式，通过 `register` 精准更新，不触发整体重渲染。

---

### 85. React 动画方案全景对比

```jsx
import { motion, AnimatePresence } from 'framer-motion';

// 列表进出动画
function AnimatedList({ items, onRemove }) {
  return (
    <AnimatePresence>
      {items.map(item => (
        <motion.div
          key={item.id}
          initial={{ opacity: 0, height: 0 }}
          animate={{ opacity: 1, height: 'auto' }}
          exit={{ opacity: 0, height: 0, x: -100 }}
          transition={{ duration: 0.3 }}
        >
          {item.text}
          <button onClick={() => onRemove(item.id)}>删除</button>
        </motion.div>
      ))}
    </AnimatePresence>
  );
}

// Layout 动画（自动处理位置变化）
function LayoutAnimation() {
  const [items, setItems] = useState([1, 2, 3]);
  return (
    <div className="flex gap-2">
      {items.map(i => (
        <motion.div key={i} layout className="w-20 h-20 bg-blue-500 rounded" />
      ))}
      <button onClick={() => setItems(prev => [...prev.reverse()])}>乱序</button>
    </div>
  );
}
```

| 场景 | 推荐方案 |
|------|----------|
| 简单过渡 | CSS `transition` |
| 复杂手势/拖拽 | Framer Motion |
| 列表进出动画 | Framer Motion `AnimatePresence` |
| Layout 变化 | Framer Motion `layout` |
| 物理弹簧 | React Spring |
| Canvas / 3D | Three.js / R3F |

---

### 86. WebSocket 实时通信 Hook

```jsx
function useWebSocket(url) {
  const [lastMessage, setLastMessage] = useState(null);
  const [readyState, setReadyState] = useState(WebSocket.CONNECTING);
  const wsRef = useRef(null);

  const connect = useCallback(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => setReadyState(WebSocket.OPEN);
    ws.onmessage = (event) => {
      try { setLastMessage(JSON.parse(event.data)); }
      catch { setLastMessage(event.data); }
    };
    ws.onclose = (event) => {
      setReadyState(WebSocket.CLOSED);
      // 非正常关闭时自动重连
      if (!event.wasClean) setTimeout(connect, 3000);
    };
    ws.onerror = () => ws.close();
  }, [url]);

  const sendMessage = useCallback((data) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(typeof data === 'string' ? data : JSON.stringify(data));
    }
  }, []);

  useEffect(() => { connect(); return () => wsRef.current?.close(); }, [connect]);

  return { lastMessage, readyState, sendMessage };
}

// 使用
function ChatRoom({ roomId, userId }) {
  const [messages, setMessages] = useState([]);
  const { lastMessage, sendMessage, readyState } = useWebSocket(
    `wss://chat.example.com/ws?room=${roomId}&user=${userId}`
  );

  useEffect(() => {
    if (lastMessage) setMessages(prev => [...prev, lastMessage]);
  }, [lastMessage]);
}
```

---

### 87. PWA 离线化关键实现

```jsx
// ===== Service Worker 注册 =====
export function register() {
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', () => {
      navigator.serviceWorker.register('/service-worker.js');
    });
  }
}

// ===== 离线检测 Hook =====
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const goOnline = () => setIsOnline(true);
    const goOffline = () => setIsOnline(false);
    window.addEventListener('online', goOnline);
    window.addEventListener('offline', goOffline);
    return () => { window.removeEventListener('online', goOnline); window.removeEventListener('offline', goOffline); };
  }, []);

  return isOnline;
}

// ===== 离线数据持久化 =====
function useOfflineData(key, fetchFn) {
  const [data, setData] = useState(() => {
    const cached = localStorage.getItem(key);
    return cached ? JSON.parse(cached) : null;
  });

  useEffect(() => {
    let cancelled = false;
    fetchFn().then(result => {
      if (!cancelled) { setData(result); localStorage.setItem(key, JSON.stringify(result)); }
    });
    return () => { cancelled = true; };
  }, [key]);
  return data;
}
```

---

### 88. React 无障碍（a11y）核心清单

```jsx
// 1. 语义化 HTML — 最重要的一条
// ❌ <div onClick={handleClick}>提交</div>
// ✅ <button onClick={handleSubmit}>提交</button>

// 2. 自定义 Tabs — 手动 ARIA
function AccessibleTabs({ tabs, activeTab, onChange }) {
  return (
    <div>
      <div role="tablist" aria-label="内容标签">
        {tabs.map(tab => (
          <button
            key={tab.id}
            role="tab"
            aria-selected={tab.id === activeTab}
            aria-controls={`panel-${tab.id}`}
            onClick={() => onChange(tab.id)}
            onKeyDown={(e) => {
              const index = tabs.findIndex(t => t.id === activeTab);
              if (e.key === 'ArrowRight') onChange(tabs[(index + 1) % tabs.length].id);
              if (e.key === 'ArrowLeft') onChange(tabs[(index - 1 + tabs.length) % tabs.length].id);
            }}
          >{tab.label}</button>
        ))}
      </div>
      {tabs.map(tab => (
        <div key={tab.id} role="tabpanel" id={`panel-${tab.id}`} aria-labelledby={`tab-${tab.id}`} hidden={tab.id !== activeTab}>
          {tab.content}
        </div>
      ))}
    </div>
  );
}

// 3. 焦点锁（Focus Trap）— Modal/Dialog 必须
function useFocusTrap(containerRef, isActive) {
  useEffect(() => {
    if (!isActive || !containerRef.current) return;
    const container = containerRef.current;
    const focusable = container.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    const first = focusable[0], last = focusable[focusable.length - 1];

    const handleTab = (e) => {
      if (e.key !== 'Tab') return;
      if (e.shiftKey && document.activeElement === first) { e.preventDefault(); last.focus(); }
      else if (!e.shiftKey && document.activeElement === last) { e.preventDefault(); first.focus(); }
    };
    container.addEventListener('keydown', handleTab);
    first?.focus();
    return () => container.removeEventListener('keydown', handleTab);
  }, [isActive]);
}
```

---

### 89. Portal 的原理与典型场景

```jsx
import { createPortal } from 'react-dom';

// 场景 1：Modal — 渲染到 body 下，避免 z-index/overflow 冲突
function Modal({ isOpen, onClose, children }) {
  if (!isOpen) return null;
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={e => e.stopPropagation()}>{children}</div>
    </div>,
    document.getElementById('modal-root')
  );
}

// 场景 2：Toast — 全局通知
function ToastContainer({ toasts }) {
  return createPortal(
    <div className="fixed top-4 right-4 space-y-2 z-50">
      {toasts.map(t => <div key={t.id} className="bg-white shadow-lg rounded p-3">{t.message}</div>)}
    </div>,
    document.body
  );
}

// ⚠️ 重要：Portal 的事件冒泡沿 React 组件树，不是 DOM 树！
function Parent() {
  const handleClick = () => console.log('Portal 内点击，Parent 也能收到！');
  return (
    <div onClick={handleClick}>
      {createPortal(<button>点击</button>, document.body)}
    </div>
  );
}
```

---

### 90. Hydration 错误排查与解决

```jsx
// 常见 Hydration mismatch 场景及修复：

// 1. 服务端/客户端渲染不一致
// ❌ <div>{Date.now()}</div> — 每次渲染值不同
// ✅ 
function FixedTime() {
  const [time, setTime] = useState(0);
  useEffect(() => setTime(Date.now()), []);
  return <div>{time || 'Loading...'}</div>;
}

// 2. 浏览器 API 条件渲染
// ❌ if (typeof window !== 'undefined' && window.innerWidth > 768) return <Desktop />
// ✅ 
function ResponsiveBanner() {
  const [isDesktop, setIsDesktop] = useState(false);
  const [hydrated, setHydrated] = useState(false);
  useEffect(() => { setIsDesktop(window.innerWidth > 768); setHydrated(true); }, []);
  if (!hydrated) return <MobileBanner />;
  return isDesktop ? <DesktopBanner /> : <MobileBanner />;
}

// 3. 非法 HTML 嵌套
// ❌ <p><div>内容</div></p> — p 内不能嵌套 div
// ❌ <table><tr><div>...</div></tr></table>
```

---

## 二十二、工程化与未来趋势

### 91. 前端日志系统设计

```jsx
const LOG_LEVELS = { DEBUG: 0, INFO: 1, WARN: 2, ERROR: 3 };

class Logger {
  constructor(options = {}) {
    this.level = LOG_LEVELS[options.level] ?? LOG_LEVELS.INFO;
    this.buffer = [];
    this.batchSize = options.batchSize || 10;
    this.endpoint = options.endpoint || '/api/logs';

    this.timer = setInterval(() => this.flush(), options.flushInterval || 5000);
    window.addEventListener('beforeunload', () => this.flush(true));
  }

  log(level, message, extra = {}) {
    if (LOG_LEVELS[level] < this.level) return;

    const entry = {
      level, message, timestamp: Date.now(),
      userId: this.userId, url: window.location.href,
      userAgent: navigator.userAgent, ...extra,
    };

    this.buffer.push(entry);
    if (level === 'ERROR') this.flush(true);
    else if (this.buffer.length >= this.batchSize) this.flush();
  }

  debug(msg, extra) { this.log('DEBUG', msg, extra); }
  info(msg, extra) { this.log('INFO', msg, extra); }
  error(msg, extra) { this.log('ERROR', msg, extra); }

  async flush(useBeacon = false) {
    if (this.buffer.length === 0) return;
    const logs = [...this.buffer]; this.buffer = [];
    const payload = JSON.stringify({ logs });
    if (useBeacon) { navigator.sendBeacon(this.endpoint, payload); }
    else {
      try { await fetch(this.endpoint, { method: 'POST', body: payload, keepalive: true }); }
      catch { this.buffer = [...logs, ...this.buffer].slice(-100); }
    }
  }
}
```

---

### 92. A/B 测试与 Feature Flags

```jsx
// ===== A/B 测试 Hook =====
function useABTest(experimentId) {
  const userId = getUserId();
  return useMemo(() => {
    const hash = hashCode(`${userId}-${experimentId}`);
    return hash % 2; // 50/50 分组
  }, [experimentId, userId]);
}

function PricingPage() {
  const variant = useABTest('pricing-redesign');
  return variant === 1 ? <NewPricingLayout /> : <OriginalPricingLayout />;
}

// ===== Feature Flags（灰度发布）=====
const featureFlags = {
  newDashboard: { enabled: true, rollout: { percentage: 20, allowedUsers: ['admin'] } },
  darkMode: { enabled: true, rollout: { percentage: 100 } },
};

function useFeatureFlag(flagName) {
  const user = useCurrentUser();
  const flag = featureFlags[flagName];
  if (!flag?.enabled) return false;
  if (flag.rollout.allowedUsers?.includes(user.id)) return true;
  return hashCode(user.id) % 100 < (flag.rollout.percentage || 0);
}
```

---

### 93. 图片优化全策略

```jsx
// 1. 响应式图片 + 现代格式
function ResponsiveImage({ src, alt }) {
  return (
    <picture>
      <source srcSet={`${src}?format=avif`} type="image/avif" />
      <source srcSet={`${src}?format=webp`} type="image/webp" />
      <img src={src} alt={alt} loading="lazy" decoding="async"
        sizes="(max-width: 768px) 100vw, 50vw"
        srcSet={`${src}?w=400 400w, ${src}?w=800 800w`} />
    </picture>
  );
}

// 2. 渐进式加载（模糊 → 清晰）
function ProgressiveImage({ src, placeholder, alt }) {
  const [loaded, setLoaded] = useState(false);
  return (
    <div className="relative">
      <img src={placeholder} alt="" aria-hidden="true"
        className={`absolute inset-0 w-full transition-opacity ${loaded ? 'opacity-0' : 'opacity-100'}`} />
      <img src={src} alt={alt} loading="lazy"
        onLoad={() => setLoaded(true)}
        className={`transition-opacity ${loaded ? 'opacity-100' : 'opacity-0'}`} />
    </div>
  );
}

// 3. CDN 参数自动化
// src = `${CDN_URL}/q_auto:good/f_auto/w_800/${publicId}`
// q_auto: 自动质量优化  f_auto: 自动格式选择（WebP/AVIF）

// 核心原则：
// 合适格式(WebP/AVIF) → 合适尺寸 → 懒加载 → CDN → 感知无损压缩
```

---

### 94. 移动端 H5 适配

```jsx
// ===== Rem 方案（375 设计稿）=====
// document.documentElement.style.fontSize = (clientWidth / 375 * 100) + 'px';
// 设计稿 24px → 0.24rem

// ===== vw 方案（推荐）：postcss-px-to-viewport 自动转换 =====
// postcss.config.js: { viewportWidth: 375, viewportUnit: 'vw' }

// ===== 安全区域（刘海屏）=====
<div style={{
  paddingTop: 'env(safe-area-inset-top)',
  paddingBottom: 'env(safe-area-inset-bottom)',
}} />

// ===== 移动端触摸手势 =====
function useTouch() {
  const [touch, setTouch] = useState({ startX: 0, startY: 0, deltaX: 0, deltaY: 0 });
  const ref = useRef();
  useEffect(() => {
    const el = ref.current;
    const handleStart = (e) => setTouch({ startX: e.touches[0].clientX, startY: e.touches[0].clientY, deltaX: 0, deltaY: 0 });
    const handleMove = (e) => setTouch(prev => ({ ...prev, deltaX: e.touches[0].clientX - prev.startX, deltaY: e.touches[0].clientY - prev.startY }));
    el.addEventListener('touchstart', handleStart, { passive: true });
    el.addEventListener('touchmove', handleMove, { passive: true });
    return () => { el.removeEventListener('touchstart', handleStart); el.removeEventListener('touchmove', handleMove); };
  }, []);
  return [ref, touch];
}
```

---

### 95. Web Worker 与密集计算

```jsx
// workers/fib.worker.js
self.onmessage = (e) => {
  const fib = (n) => (n <= 1 ? n : fib(n - 1) + fib(n - 2));
  self.postMessage({ result: fib(e.data.n), id: e.data.id });
};

// React Hook
function useWorker(workerUrl) {
  const workerRef = useRef(null);
  const callbacksRef = useRef({});

  useEffect(() => {
    workerRef.current = new Worker(new URL(workerUrl, import.meta.url));
    workerRef.current.onmessage = (e) => {
      callbacksRef.current[e.data.id]?.(e.data.result);
      delete callbacksRef.current[e.data.id];
    };
    return () => workerRef.current?.terminate();
  }, []);

  const run = useCallback((data) => {
    return new Promise(resolve => {
      const id = Date.now() + Math.random();
      callbacksRef.current[id] = resolve;
      workerRef.current?.postMessage({ ...data, id });
    });
  }, []);

  return { run };
}

// 使用
function Calculator() {
  const { run } = useWorker('../workers/fib.worker.js');
  const handleCalc = async () => { const res = await run({ n: 40 }); };
  return <button onClick={handleCalc}>计算 Fib(40)</button>;
}

// Comlink 简化版：让 Worker 通信像普通函数调用
// const worker = Comlink.wrap(new Worker('./worker.js'));
// const result = await worker.heavyCalculate(data);
```

---

### 96. 多环境配置管理

```
// ===== 目录结构 =====
// config/
// ├── env.development.js    → API_BASE: 'http://localhost:8080'
// ├── env.staging.js        → API_BASE: 'https://staging-api.example.com'
// └── env.production.js     → API_BASE: 'https://api.example.com'

// config/index.js
const envConfigs = {
  development: require('./env.development'),
  staging: require('./env.staging'),
  production: require('./env.production'),
};
export const config = envConfigs[import.meta.env.VITE_ENV || 'development'];

// Vite .env 文件
// .env:              VITE_APP_NAME=MyApp
// .env.development:  VITE_API_BASE=http://localhost:3000
// .env.production:   VITE_API_BASE=https://api.myapp.com

// CI/CD 注入（Docker）
// docker build --build-arg VITE_API_BASE=https://api.example.com
// 运行时通过环境变量动态注入
```

---

### 97. 错误兜底与降级策略

```jsx
// ===== 多层错误边界 =====
<ErrorBoundary fallback={<RootErrorFallback />} onError={reportToSentry}>
  <Layout>
    <ErrorBoundary fallback={<WidgetErrorFallback />}>
      <WeatherWidget /> {/* Widget 崩溃不影响页面其他部分 */}
    </ErrorBoundary>
    <NewsFeed />
  </Layout>
</ErrorBoundary>

// ===== API 降级 =====
async function fetchWithFallback(url, fallbackData) {
  try {
    const res = await fetch(url);
    return await res.json();
  } catch (error) {
    monitor.reportError({ type: 'api_fallback', url, error });
    return fallbackData;
  }
}

// ===== 渐进式功能加载 =====
function FeatureWrapper({ children, fallback = null }) {
  const [error, setError] = useState(false);
  if (error) return fallback;
  return <ErrorBoundary fallback={fallback} onError={() => setError(true)}>{children}</ErrorBoundary>;
}

<FeatureWrapper fallback={<SimpleChart />}>
  <ComplexInteractiveChart data={data} />
</FeatureWrapper>
```

---

### 98. Canvas / Three.js + React

```jsx
// ===== react-three-fiber（Three.js 的 React 渲染器）=====
import { Canvas, useFrame } from '@react-three/fiber';
import { OrbitControls } from '@react-three/drei';

function RotatingBox() {
  const meshRef = useRef();
  useFrame((state, delta) => {
    meshRef.current.rotation.x += delta * 0.5;
    meshRef.current.rotation.y += delta * 0.3;
  });

  return (
    <mesh ref={meshRef}>
      <boxGeometry args={[2, 2, 2]} />
      <meshStandardMaterial color="hotpink" />
    </mesh>
  );
}

function Scene() {
  return (
    <Canvas camera={{ position: [0, 0, 5] }} style={{ height: '100vh' }}>
      <ambientLight intensity={0.5} />
      <pointLight position={[10, 10, 10]} />
      <RotatingBox />
      <OrbitControls />
    </Canvas>
  );
}

// ===== React + Canvas 2D 高 DPI 适配 =====
function Chart({ data }) {
  const canvasRef = useRef();
  useEffect(() => {
    const ctx = canvasRef.current.getContext('2d');
    const dpr = window.devicePixelRatio || 1;
    canvasRef.current.width = 600 * dpr;
    canvasRef.current.height = 400 * dpr;
    ctx.scale(dpr, dpr);
    // 绘制逻辑...
  }, [data]);
  return <canvas ref={canvasRef} style={{ width: 600, height: 400 }} />;
}
```

---

### 99. 前端全链路性能优化清单

```
1. 网络层
   □ HTTP/2 or HTTP/3
   □ CDN 加速静态资源
   □ Gzip/Brotli 压缩
   □ 资源预加载：<link rel="preload" />
   □ DNS 预解析：<link rel="dns-prefetch" />

2. 构建层
   □ Code Splitting（路由级 + 组件级）
   □ Tree Shaking（ESM + sideEffects: false）
   □ 图片压缩 + WebP/AVIF
   □ Bundle 分析（@next/bundle-analyzer）

3. 渲染层
   □ 虚拟滚动（长列表）
   □ React.memo + useMemo + useCallback（按需）
   □ 避免渲染中的匿名函数/对象
   □ useDeferredValue / useTransition 降级非紧急更新

4. 数据层
   □ API 响应缓存（React Query staleTime）
   □ 请求去重 + 合并
   □ 乐观更新（Optimistic UI）
   □ 预加载（prefetchQuery / Intersection Observer）

5. 运行时
   □ Web Worker 处理密集计算
   □ 脚本异步加载：<script defer/async>
   □ 事件节流/防抖
   □ 大对象避免频繁序列化

6. 监控
   □ Web Vitals（LCP、INP、CLS）
   □ Lighthouse CI
   □ 自定义性能埋点
   □ Long Task 监控
```

---

### 100. 2025-2026 React 技术趋势与面试收官总结

**React 技术趋势 TOP 10：**

```
1. React Server Components → 新项目默认选择
2. AI 集成 → Vercel AI SDK + React 标准组合
3. Signals 思想渗透 → 精准渲染取代粗粒度重渲染
4. Island Architecture → Astro + React 混合渲染
5. React Native 新架构 → JSI + Fabric 性能飞跃
6. Edge-first → 边缘计算 + Streaming SSR
7. TypeScript → React 项目 100% TS 覆盖
8. Bun / Biome → 替代 Node.js / ESLint 新工具链
9. CSS 零运行时 → Tailwind v4 / Panda CSS 主流
10. 状态管理精简 → React Query + Zustand 标配组合
```

**面试高频 TOP 10（2025-2026）：**

| 排名 | 考点 | 为什么重要 |
|------|------|-----------|
| 1 | Hooks（useState/useEffect/useMemo） | 日常使用最多 |
| 2 | RSC / Server Components | 新范式，大厂必考 |
| 3 | 性能优化（memo/lazy/virtualize） | 区分初中高级 |
| 4 | Next.js App Router | 主流框架标配 |
| 5 | 状态管理选型 | 架构能力 |
| 6 | React 18/19 新特性 | 社区关注度 |
| 7 | Fiber / Concurrent 原理 | 源码深度 |
| 8 | TypeScript + React | 编码规范 |
| 9 | 设计模式（HOC / Compound） | 代码质量 |
| 10 | Suspense / Streaming | 新技术应用 |

---

## 总结

### 全部 100 题模块总览

| # | 模块 | 题号 | 覆盖主题 |
|----|------|------|----------|
| 1 | 基础概念 | 1-6 | Virtual DOM、Diff、类/函数组件、setState、受控非受控、合成事件 |
| 2 | Hooks 深度 | 7-13 | useState/useEffect 原理、useRef、memo/callback、useReducer、自定义 Hook、闭包 |
| 3 | 状态管理 | 14-16 | Context API、Redux（RTK/中间件）、方案选型 |
| 4 | 性能优化 | 17-20 | React.memo 系列、代码分割、虚拟滚动、重渲染排查 |
| 5 | 原理源码 | 21-24 | Fiber、Concurrent Mode、Scheduler、Hooks 源码 |
| 6 | 路由与数据流 | 25-26 | React Router v6、CSR vs SSR vs SSG |
| 7 | 实战场景 | 27-30 | 关键词高亮、Modal、防抖搜索、项目目录设计 |
| 8 | TS + React | 31-32 | 常用类型、泛型组件、错误处理 |
| 9 | 设计模式 | 33-37 | HOC、Render Props、Compound、Provider、展示/容器 |
| 10 | 测试策略 | 38-39 | 测试金字塔、RTL 全场景 |
| 11 | React 19 | 40-41 | RSC、Actions、useOptimistic、use()、18vs19 |
| 12 | 陷阱高阶 | 42-47 | key 原理、依赖陷阱、XSS、StrictMode、惰性初始化、Keep-Alive |
| 13 | 系统设计 | 48-50 | 动态表单、RBAC 路由、撤销/重做编辑器 |
| 14 | 工程化实践 | 51-55 | Webpack/Vite、CI/CD、CSS 方案、微前端、监控 |
| 15 | 生态对比 | 56-60 | React Query、Zustand/Jotai、框架对比、Signals、跨端 |
| 16 | 安全实践 | 61-63 | XSS/CSRF 防护、代码规范、项目搭建决策 |
| 17 | Next.js Router | 64-68 | App vs Pages、文件约定、缓存、Actions、认证 |
| 18 | RSC + 渲染 | 69-72 | RSC 原理、Suspense/Streaming、渲染法则、虚拟滚动 |
| 19 | 架构进阶 | 73-77 | Monorepo、组件库、i18n、GraphQL、状态管理指南 |
| 20 | 高级 Hooks | 78-83 | useLayoutEffect、useImperativeHandle、useSyncExternalStore、deferredValue、批处理、insertionEffect |
| 21 | 实战进阶 | 84-90 | Hook Form、动画、WebSocket、PWA、a11y、Portal、Hydration |
| 22 | 工程化趋势 | 91-100 | 日志系统、A/B、图片优化、移动端、Worker、环境配置、降级、3D、性能清单、趋势 |

### 100 题难度分布

| 难度 | 题号 | 说明 |
|------|------|------|
| ⭐ 初级 | 1, 3, 5, 7, 25 | 基础概念，必会 |
| ⭐⭐ 中级 | 2, 4, 8-12, 14-18, 26-29, 84, 88 | 高频使用 |
| ⭐⭐⭐ 高级 | 6, 13, 19-24, 30-32, 38-39, 78-83 | 原理和优化 |
| ⭐⭐⭐⭐ 专家 | 33-37, 42-47, 64-66, 85-87, 89-90 | 设计模式/陷阱 |
| ⭐⭐⭐⭐⭐ 架构 | 40-41, 48-63, 67-77, 91-100 | 系统/工程化/趋势 |

---

> **面试不是背诵，是展示理解深度。** 把 100 题的答案变成你自己的语言，结合真实项目经历输出——你就是面试官眼中"值得给 offer"的人。
>
> **学习路径**：
> ```
> 基础(1-6) → Hooks(7-13) → 状态(14-16) → 性能(17-20)
>   → 原理(21-24) → 路由(25-26) → 实战(27-30) → TS(31-32)
>     → 设计模式(33-37) → 测试(38-39) → React19(40-41) → 陷阱(42-47)
>       → 系统设计(48-50) → 工程化(51-55) → 生态(56-60) → 安全(61-63)
>         → Next.js(64-68) → RSC(69-72) → 架构(73-77) → 高级Hooks(78-83)
>           → 实战进阶(84-90) → 趋势(91-100)
> ```
