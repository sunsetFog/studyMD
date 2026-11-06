# 01 - 快速上手与环境搭建

## 1. React 是什么

用于构建用户界面的 JavaScript 库（不是框架），核心特征：

| 特征 | 说明 |
|---|---|
| 声明式 | 描述「UI 应该是什么样」，由 React 负责 DOM 更新 |
| 组件化 | UI = 组件的组合，组件即函数 |
| 单向数据流 | props 自上而下，状态提升通信 |
| 学习一次随处编写 | Web（ReactDOM）、Native（React Native）、SSR |
| 虚拟 DOM + Diff | 最小化真实 DOM 操作 |
| 函数组件 + Hooks | 现代开发范式（2019+ 取代 class 组件） |

## 2. 创建项目

### 2.1 Vite（推荐）

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install && npm run dev
```

### 2.2 Next.js（需要 SSR/全栈时）

```bash
npx create-next-app@latest my-app --typescript --app --tailwind
```

选型：纯 SPA / 内嵌到现有系统 → Vite；需要 SEO/SSR/全栈 → Next.js（见 `26-ssr-nextjs.md`）。

### 2.3 CDN 体验

```html
<script type="text/babel">
  const { useState } = React
  function App() {
    const [count, setCount] = useState(0)
    return <button onClick={() => setCount(count + 1)}>点击 {count} 次</button>
  }
  ReactDOM.createRoot(document.getElementById('root')).render(<App />)
</script>
```

## 3. 第一个组件

```tsx
// src/App.tsx
import { useState } from 'react'

export default function App() {
  const [count, setCount] = useState(0)
  return (
    <button onClick={() => setCount(c => c + 1)}>
      点击了 {count} 次
    </button>
  )
}
```

```tsx
// src/main.tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

要点：
- **组件就是返回 JSX 的函数**，首字母必须大写（小写会被当原生标签）
- `createRoot` 是 React 18+ 并发模式入口（替代旧 `ReactDOM.render`）
- `StrictMode` 开发期双重渲染检测副作用——**开发环境组件会渲染两次**，这是特性不是 bug

## 4. 项目依赖理解

```
react            核心（组件/Hooks）
react-dom        Web 渲染器（createRoot/hydrateRoot）
vite             构建（esbuild + rollup）
typescript       类型
@types/react     React 类型定义
```

## 5. React 心智模型（最重要的一节）

| 心智 | 说明 |
|---|---|
| UI = f(state) | 界面是状态的纯函数映射，状态变 → 重新调用函数 → diff 更新 |
| 每次渲染是快照 | 组件函数执行时，props/state 都是当次快照（闭包） |
| 不可变更新 | 不直接修改 state，而是创建新对象/数组 |
| 单向数据流 | 数据从父到子；子改父通过回调 |
| 声明式 | 你不命令 DOM，只描述结果 |

```
setState → 触发重渲染 → 函数重新执行 → 生成新虚拟 DOM → Diff → 更新真实 DOM
```

理解「每次渲染都是一次完整的函数调用」是掌握 Hooks 一切规则的前提（详见 `05-useEffect.md`、`30-hooks-internals.md`）。

## 6. 开发工具链

- **React Developer Tools** 浏览器扩展：组件树、props/state 检查、Profiler 性能分析
- VSCode：ESLint + Prettier
- React 18+ 自带 `StrictMode` 副作用检查

## 7. 与 Vue 的速览差异

| 维度 | React | Vue |
|---|---|---|
| 模板 | JSX（JS 内写 UI） | SFC 模板（指令系统） |
| 响应式 | 显式 setState + 不可变 | 自动依赖追踪（Proxy） |
| 心智 | 重新渲染整个函数 | 精确依赖更新 |
| 生态 | 官方只管核心，生态自由组合 | 官方全家桶（Router/Pinia） |

详细对照见 `37-vue-react-comparison.md`。

## 常见坑

1. **组件名小写**：`function button()` 会被当 HTML 标签渲染成 `<button>` 原生元素。
2. **忘 import React？**：JSX 转换新版（automatic runtime）不需要 `import React`，但需要 Vite/构建器支持（`jsx: "react-jsx"`）。
3. **StrictMode 双渲染误解**：开发期组件执行两次，用于暴露非幂等副作用；生产不受影响。
4. **把 React 当模板引擎**：直接改 DOM / 依赖渲染次数——React 渲染次数不可预测。
5. **createRoot 挂多个元素**：一个 root 一个容器，多入口需多个 createRoot。

## Checklist

- [ ] 使用 Vite react-ts 模板初始化
- [ ] main.tsx 用 createRoot + StrictMode
- [ ] 组件首字母大写、文件名 PascalCase
- [ ] 安装 React Developer Tools
- [ ] 理解 UI = f(state) 与渲染快照模型
