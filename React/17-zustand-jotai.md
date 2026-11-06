# 17 - Zustand 与 Jotai

## 1. 为什么是这两个

现代 React 轻量状态库双雄，共同特征：

```
1. 基于 useSyncExternalStore —— 精确订阅、无 Context 重渲染问题、并发安全
2. 极简 API，无 Provider 包裹（Zustand）/ 按需原子（Jotai）
3. 天然支持 selector 粒度订阅
4. TS 友好，样板代码极少
```

## 2. Zustand

### 2.1 基本用法

```bash
npm i zustand
```

```tsx
import { create } from 'zustand'

type CartState = {
  items: CartItem[]
  addItem: (item: CartItem) => void
  removeItem: (id: number) => void
  clear: () => void
}

export const useCartStore = create<CartState>()((set, get) => ({
  items: [],
  addItem: (item) => set(s => ({
    items: [...s.items.filter(i => i.id !== item.id), item],   // 不可变更新
  })),
  removeItem: (id) => set(s => ({ items: s.items.filter(i => i.id !== id) })),
  clear: () => set({ items: [] }),
}))

// 组件内：selector 订阅
const count = useCartStore(s => s.items.length)
const addItem = useCartStore(s => s.addItem)     // 函数引用稳定
addItem(item)                                     // 组件外也可直接 useCartStore.getState()
```

要点：
- selector 订阅：**只在该字段变化时重渲染**（底层 shallow 比较或 Object.is）
- 无 Provider：模块级单例，任何地方（组件外、工具函数）可 `useCartStore.getState() / setState()`
- `get()` 读最新状态（action 内）

### 2.2 多字段订阅与 shallow

```tsx
import { useShallow } from 'zustand/react/shallow'

// ❌ 一次取多个字段返回新对象 → 每次都变
const { a, b } = useCartStore(s => ({ a: s.a, b: s.b }))
// ✅ useShallow 浅比较
const { a, b } = useCartStore(useShallow(s => ({ a: s.a, b: s.b })))
```

### 2.3 中间件

```tsx
import { persist, devtools } from 'zustand/middleware'

export const useSettingsStore = create<Settings>()(
  devtools(
    persist(
      (set) => ({ theme: 'light', setTheme: (t) => set({ theme: t }) }),
      {
        name: 'app:settings',
        partialize: (s) => ({ theme: s.theme }),    // 白名单持久化
        version: 2,                                  // schema 版本
        migrate: (persisted, version) => version < 2 ? { ...persisted, newField: def } : persisted,
      },
    ),
  ),
)
```

### 2.4 切片组合

```tsx
const createCartSlice = (set, get) => ({ items: [], addItem: … })
const createUserSlice = (set, get) => ({ user: null, login: … })

export const useStore = create<CartSlice & UserSlice>()((...a) => ({
  ...createCartSlice(...a),
  ...createUserSlice(...a),
}))
```

按业务域切片再组合——中大型 Zustand 项目的标准组织。

## 3. Jotai（原子化）

```bash
npm i jotai
```

```tsx
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai'

const countAtom = atom(0)
const doubledAtom = atom((get) => get(countAtom) * 2)         // 派生原子

function Counter() {
  const [count, setCount] = useAtom(countAtom)               // 读写
  const doubled = useAtomValue(doubledAtom)                  // 只读
  const setCountOnly = useSetAtom(countAtom)                 // 只写（稳定引用）
  …
}
```

- **自底向上的依赖图**：组件订阅它用到的原子，某原子变只重渲染依赖它的组件（自动精确）
- 派生原子（计算原子）≈ Redux selector + 自动依赖追踪
- 异步原子：`atom(async (get) => fetch…)` 配 Suspense（可结合 Query）

```tsx
// Provider 可选（默认全局隐式 Provider；SSR/多实例需显式 <Provider>）
```

## 4. Zustand vs Jotai 选型

| 维度 | Zustand | Jotai |
|---|---|---|
| 心智 | 集中式 store + selector | 自底向上原子依赖图 |
| 状态形态 | 相关字段一组的域状态 | 大量独立/派生小状态 |
| 组件外访问 | getState 一等公民 | 需 getDefaultStore |
| 学习/推广成本 | 极低 | 稍高（原子思维） |
| 典型 | 购物车/用户/设置 | 复杂表单联动、画布、大量派生 |

默认推荐 **Zustand**（团队上手最快）；状态间依赖关系复杂、派生密集时 Jotai 更优雅。

## 5. 与 Query/Context 的边界

```
服务端数据      → TanStack Query（不进 Zustand）
主题/用户会话   → Zustand persist 或 Context（低频两者皆可）
高频交互状态    → Zustand/Jotai（不是 Context）
组件私有        → useState
```

## 6. Zustand 之外的可选项速览

- **Valtio**：代理式可变写法（`state.count++`），心智与 Vue 近
- **MobX**：老牌响应式 OOP 风格，存量项目
- **XState**：状态机建模，复杂流程编排
- **React 原生**：useReducer+Context（小规模够用，见 08/07）

## 常见坑

1. **selector 返回新对象无 shallow**：重渲染风暴（useShallow 解）。
2. **action 内直接改 state**（`s.items.push`）：违反不可变；Zustand set 里也要新引用。
3. **persist 存敏感数据/全量状态**：partialize 白名单。
4. **Jotai 原子爆炸无组织**：atom 文件按域归组。
5. **组件外滥用 getState 改状态**：逻辑散落难追溯；改状态收口到 action。
6. **两个 store 互相引用**：action 内互调，不顶层循环 import。

## Checklist

- [ ] selector 最小粒度，多字段用 useShallow
- [ ] 不可变更新纪律与 useState 一致
- [ ] persist 白名单 + version + migrate
- [ ] 服务端数据不进 Zustand/Jotai
- [ ] 状态修改收口到 store action
- [ ] devtools 中间件开启
