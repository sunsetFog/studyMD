# 07 - useContext 与 Context 设计

## 1. Context 解决什么问题

**props 逐层透传**（prop drilling）的解法：跨层级共享「相对全局」的数据。

```tsx
// 创建
import { createContext, useContext, useState } from 'react'

type Theme = 'light' | 'dark'
const ThemeContext = createContext<Theme>('light')

// 提供（Provider）
export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light')
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}

// 消费（任意层级）
function ThemeButton() {
  const { theme, setTheme } = useContext(ThemeContext)
  return <button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}>{theme}</button>
}
```

Provider 找组件树中**最近**的 Provider；找不到时使用 createContext 的默认值。

## 2. 自定义 Hook 封装（强制规范）

直接暴露 Context 对象会导致「忘了包 Provider」时静默用默认值、且换实现困难：

```tsx
// context/auth.tsx —— 标准 Context 模块模板
const AuthContext = createContext<AuthContextValue | null>(null)

export function AuthProvider({ children }: Props) {
  const [user, setUser] = useState<User | null>(null)
  const login = useCallback(async (p: LoginPayload) => { … }, [])
  const logout = useCallback(() => { … }, [])
  const value = useMemo(() => ({ user, login, logout }), [user, login, logout])
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>
}

export function useAuth() {
  const ctx = useContext(AuthContext)
  if (!ctx) throw new Error('useAuth 必须在 <AuthProvider> 内使用')   // fail fast
  return ctx
}

// 使用处只见 hook，不见 Context 对象
const { user, login } = useAuth()
```

规范四条：
1. Context 对象**不导出**，只导出 Provider 与 useXxx
2. value 用 useMemo 稳定（否则每次渲染新对象 → 全树消费者重渲染）
3. 消费 hook 做空值断言 fail fast
4. 文件名 `context/xxx.tsx`（含 JSX 需要）

## 3. Context 的重渲染问题（核心权衡）

**Provider value 变化 → 其所有子树（消费者与非消费者都）重渲染**：

```
<ThemeProvider>            ← value 变
  <Layout>                 ← 整个子树重渲染（React 无法跳过中间层）
    <SideBar />            ← 不消费 theme 也重渲染！
    <Main><ThemeBadge /></Main>
```

缓解手段：

```tsx
// 1. value 引用稳定（useMemo）
const value = useMemo(() => ({ theme, setTheme }), [theme])

// 2. 拆分 Context：变化频率不同的状态分箱
<ThemeDataProvider>…<ThemeActionsProvider>…   // 高频数据 / 低频动作分离

// 3. 子组件提前 memo 化（配合 06）
// 4. 用状态库（Zustand 等基于 useSyncExternalStore 精确订阅，见 17）
```

**明确边界**：Context 适合「低频变化的全局值」（主题、登录态、i18n、配置）；**高频更新（输入框、鼠标、动画）严禁 Context**，走组件 state 或状态库。

## 4. 组合 Provider（嵌套地狱治理）

```tsx
// compose 辅助
function ComposeProviders({ providers, children }: { providers: React.ComponentType<{children: ReactNode}>[]; children: ReactNode }) {
  return providers.reduceRight((acc, P) => <P>{acc}</P>, children)
}

<ComposeProviders providers={[ThemeProvider, AuthProvider, QueryProvider]}>
  <App />
</ComposeProviders>
```

## 5. Context vs 状态库 vs props 决策

| 场景 | 方案 |
|---|---|
| 父子 1-2 层 | props |
| 主题/语言/当前用户/配置 | Context |
| 高频交互状态跨层共享 | Zustand/Jotai |
| 服务端数据缓存 | TanStack Query |
| 表单联动 | 组件内 state / 表单库 |

经验：**Context 数量 > 5 个或出现「每加一个全局状态就建 Context」时，该上状态库了**。

## 6. React 19 的 use() 与 Context

```tsx
// React 19：use() 可在条件语句中读 Context（useContext 不能）
function Component({ needsTheme }: { needsTheme: boolean }) {
  if (needsTheme) {
    const theme = use(ThemeContext)   // ✅ 19 允许
  }
}
```

详见 `10-react-18-19.md`。

## 常见坑

1. **value 不 useMemo**：每次渲染新对象引用，全部消费者重渲染。
2. **忘了 Provider**：拿默认值静默错误；用 hook 内断言 fail fast。
3. **高频状态放 Context**：输入/动画卡顿。
4. **Context 存复杂对象但更新粒度粗**：一个字段变全部通知；拆分 Context 或上状态库。
5. **默认值承载真实逻辑**：默认值只该是「未包裹时的安全占位」，不该被业务依赖。
6. **Provider 层级过深顺序错误**：依赖顺序（如 Query 依赖 Auth token）要理清。

## Checklist

- [ ] Context 对象不导出，只暴露 Provider + useXxx
- [ ] value 经 useMemo
- [ ] 消费 hook 内空值断言
- [ ] Context 只放低频全局数据
- [ ] 变化频率不同的状态已拆分 Context
- [ ] Provider 组合有统一入口
