# 23 - TypeScript 集成

## 1. 基础配置

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true,
    "types": ["vite/client"]
  }
}
```

构建命令：`tsc --noEmit && vite build`（类型检查独立于打包）。

## 2. 组件 props 类型

```tsx
type ButtonProps = {
  variant?: 'primary' | 'danger'
  onClick?: (e: React.MouseEvent<HTMLButtonElement>) => void
}

// 函数组件推荐直接参数解构
export function Button({ variant = 'primary', onClick }: ButtonProps) { … }

// 继承原生属性（透传组件必备）
type InputProps = React.InputHTMLAttributes<HTMLInputElement> & { label: string }

// 泛型组件
type ListProps<T> = {
  items: T[]
  renderItem: (item: T, index: number) => React.ReactNode
  keyOf: (item: T) => string
}
export function List<T>({ items, renderItem, keyOf }: ListProps<T>) {
  return <ul>{items.map((it, i) => <li key={keyOf(it)}>{renderItem(it, i)}</li>)}</ul>
}
// 调用：T 自动从 items 推导
```

## 3. children 与节点类型

```tsx
import type { ReactNode, ReactElement, JSX } from 'react'

type Props = {
  children?: ReactNode                    // 最通用
  icon?: ReactElement                     // 要求 JSX 元素
  title?: string | ReactNode
  render?: (item: T) => ReactNode         // 函数 children
}
```

## 4. Hooks 类型

```tsx
import { useState, useReducer, type Dispatch, type SetStateAction } from 'react'

const [user, setUser] = useState<User | null>(null)
const [list, setList] = useState<string[]>([])
// 显式标注 updater 类型（封装 setter 透传时）
const setCount: Dispatch<SetStateAction<number>> = …

// useReducer
const [state, dispatch] = useReducer(reducer, initialState)   // 类型从 reducer 参数推导

// ref
const inputRef = useRef<HTMLInputElement>(null)
const timerRef = useRef<ReturnType<typeof setInterval>>()     // 定时器 id

// Context
export const ThemeContext = createContext<ThemeCtx | null>(null)
```

## 5. 事件类型速查

| 事件 | 类型 |
|---|---|
| 点击 | `React.MouseEvent<HTMLButtonElement>` |
| 输入 | `React.ChangeEvent<HTMLInputElement>` |
| 表单提交 | `React.FormEvent<HTMLFormElement>` |
| 键盘 | `React.KeyboardEvent<HTMLInputElement>` |
| focus | `React.FocusEvent<HTMLInputElement>` |
| 原生监听 | `window.addEventListener('resize', (e) => …)` 自动推导 |

推导技巧：**先写 handler 再补类型**——`const onChange = (e) => …` 挂到 JSX 上由编辑器推断出完整类型。

## 6. 派生工具类型

```tsx
// 组件实例类型（ref 转发）
type Ref = React.ComponentRef<typeof MyInput>          // 推荐（替代 InstanceType 写法）

// 组件 props 提取
type BtnProps = React.ComponentProps<typeof Button>

// Omit 组合
type SubmitButtonProps = Omit<React.ButtonHTMLAttributes<HTMLButtonElement>, 'disabled'>

// DOM 元素类型
type DivProps = React.HTMLAttributes<HTMLDivElement>
```

## 7. TanStack Query / 状态库类型

```tsx
// Query：类型从 queryFn 返回值自动推导，无需手动标注
const { data } = useQuery({ queryKey: ['user', id], queryFn: () => api.getUser(id) })
// data: User | undefined 自动推导

// Zustand
type Store = { count: number; inc: () => void }
export const useStore = create<Store>()((set) => ({ … }))   // create<Store>()() 双括号模式
```

## 8. API 层类型组织

```ts
// api/user.ts —— 类型与接口同域导出
export interface User { id: number; name: string; role: Role }
export type Role = 'admin' | 'member'

export async function getUser(id: number): Promise<User> {
  const res = await http.get<User>(`/users/${id}`)
  return res.data
}
```

规范：接口出入参类型定义在 api 模块并导出；联合/枚举用 `as const` 派生：

```ts
export const ROLES = { admin: 'admin', member: 'member' } as const
export type Role = keyof typeof ROLES
```

## 9. 常见类型陷阱

| 陷阱 | 解法 |
|---|---|
| `React.FC` 显式标注 | **不用 FC**（children 隐式注入 + 泛型支持差），直接函数参数类型 |
| defaultProps 类型不一致 | 解构默认值 |
| 泛型组件 + forwardRef 复杂 | React 19 ref-as-prop 简化；或类型别名单独导出 |
| `strictFunctionTypes` 下回调逆变报错 | 参数最小化 `()` 或显式类型 |
| any 泛滥 | ESLint 警告 + 未知处 unknown + 收窄 |

## 10. ESLint 协作

```js
import tseslint from 'typescript-eslint'
export default tseslint.config(
  ...tseslint.configs.recommended,
  { rules: {
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/no-floating-promises': 'error',   // 忘 await 的异步
    '@typescript-eslint/no-misused-promises': 'error',    // onClick={asyncFn} 防误用
  } },
)
```

## 常见坑

1. **React.FC 滥用**：children 自动注入与显式意图不符、泛型受限。
2. **事件类型手写错**（MouseEvent 当 ChangeEvent）：用推断或对照表。
3. **Query data 是 `T | undefined`**：直接 `data.xxx` 报错——正确处理 pending 态（这也是它的设计意图）。
4. **组件 props 继承了 children 又传函数 children**：类型冲突，二选一。
5. **枚举（enum）与 as const 混用**：统一 as const（tree-shaking 友好）。
6. **tsc 不在 CI**：类型错误上线。

## Checklist

- [ ] strict + noUncheckedIndexedAccess 开启
- [ ] CI 有 tsc --noEmit
- [ ] 无 React.FC
- [ ] 透传组件继承原生属性类型
- [ ] 泛型组件用在 List/表格渲染器
- [ ] api 类型同域导出，枚举 as const
