# 16 - Redux Toolkit

## 1. RTK 是什么

Redux 官方标准工具集（**现代 Redux = Redux Toolkit + react-redux**），解决了裸 Redux 的样板代码、不可变更新繁琐、store 拆分问题。

```bash
npm i @reduxjs/toolkit react-redux
```

核心心智：**单一全局 store、action→reducer 纯函数、immer 风格可变写法（内部不可变）**。

## 2. Store 与 Slice

```tsx
// store/index.ts
import { configureStore } from '@reduxjs/toolkit'
import { useDispatch, useSelector, TypedUseSelectorHook } from 'react-redux'
import cartReducer from './slices/cart'
import userReducer from './slices/user'

export const store = configureStore({
  reducer: { cart: cartReducer, user: userReducer },
  middleware: getDefault => getDefault().concat(loggerMiddleware),
})

export type RootState = ReturnType<typeof store.getState>
export type AppDispatch = typeof store.dispatch
export const useAppDispatch: () => AppDispatch = useDispatch
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector
```

```tsx
// store/slices/cart.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit'

interface CartState { items: CartItem[]; status: 'idle' | 'loading' | 'error' }

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [], status: 'idle' } as CartState,
  reducers: {
    addItem(state, action: PayloadAction<CartItem>) {
      // ✅ 直接「改」，内部 immer 转不可变
      const exist = state.items.find(i => i.id === action.payload.id)
      if (exist) exist.qty += action.payload.qty
      else state.items.push(action.payload)
    },
    removeItem(state, action: PayloadAction<number>) {
      state.items = state.items.filter(i => i.id !== action.payload)
    },
    clear(state) { state.items = [] },
  },
  // 处理其他 slice 的 action / 外部 action
  extraReducers: (builder) => {
    builder.addCase(logout.fulfilled, (state) => { state.items = [] })
  },
})

export const { addItem, removeItem, clear } = cartSlice.actions
export default cartSlice.reducer
```

要点：
- reducer 里**允许写可变风格**（immer 代理），禁止在里面做副作用
- action 类型自动生成，无需手写 actionCreator 字符串

## 3. 异步：createAsyncThunk

```tsx
import { createAsyncThunk } from '@reduxjs/toolkit'

export const fetchOrders = createAsyncThunk(
  'orders/fetch',
  async (query: OrderQuery, { rejectWithValue, signal }) => {
    try { return await api.getOrders(query, { signal }) }   // signal 自动取消
    catch (e: any) { return rejectWithValue(e.message) }
  },
  { condition: (q, { getState }) => !getState().orders.loading }   // 防重复
)

// slice 中处理三态
extraReducers: (b) => b
  .addCase(fetchOrders.pending, (s) => { s.status = 'loading' })
  .addCase(fetchOrders.fulfilled, (s, a) => { s.status = 'idle'; s.items = a.payload })
  .addCase(fetchOrders.rejected, (s, a) => { s.status = 'error'; s.error = a.payload as string })
```

**更现代的做法**：服务端数据用 RTK Query（见 §6）或 TanStack Query，thunk 留给「写操作流程」。

## 4. 组件内使用

```tsx
import { useAppSelector, useAppDispatch } from '@/store'

function CartBadge() {
  const count = useAppSelector(s => s.cart.items.length)   // 精确订阅：只取需要的字段
  const dispatch = useAppDispatch()
  return <Badge onClick={() => dispatch(addItem(sample))}>{count}</Badge>
}
```

订阅规范：**selector 取最小粒度**（取字段而非整个 state 对象）；复杂派生用 createSelector 记忆化：

```tsx
import { createSelector } from '@reduxjs/toolkit'

const selectCartTotal = createSelector(
  [(s: RootState) => s.cart.items],
  items => items.reduce((sum, i) => sum + i.price * i.qty, 0),
)
```

## 5. 组织结构（Slice 按业务域）

```
store/
├── index.ts          # configureStore + 类型导出
├── hooks.ts          # 类型化 useAppSelector/useAppDispatch
└── slices/
    ├── user.ts
    ├── cart.ts
    └── settings.ts
```

跨 slice 协作：extraReducers 监听其他 slice 的 action，或 thunk 内 getState。

## 6. RTK Query（RTK 内置的数据获取层）

```tsx
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react'

export const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Orders'],
  endpoints: (build) => ({
    getOrders: build.query<Order[], OrderQuery>({
      query: (q) => ({ url: 'orders', params: q }),
      providesTags: ['Orders'],
    }),
    updateOrder: build.mutation<Order, { id: number; patch: Partial<Order> }>({
      query: ({ id, patch }) => ({ url: `orders/${id}`, method: 'PATCH', body: patch }),
      invalidatesTags: ['Orders'],          // 写后自动失效重取
    }),
  }),
})

// 组件
const { data, isLoading } = api.useGetOrdersQuery(filter)
const [updateOrder] = api.useUpdateOrderMutation()
```

RTK Query vs TanStack Query：功能等价（缓存/失效/乐观更新），**用了 RTK 就用 RTK Query 保持同栈**；纯 Query 团队选 TanStack 更轻。

## 7. 中间件与 devtools

```tsx
configureStore({ reducer, middleware: g => g().concat(customLogger), devTools: process.env.NODE_ENV !== 'production' })
```

Redux DevTools：action 时间旅行、state diff——RTK 团队选它的核心理由之一。

## 8. 什么时候（不）选 Redux

| 选 | 不选 |
|---|---|
| 存量 Redux 项目延续 | 新小项目（Zustand 更轻） |
| 团队需要强模板规范 | 服务端数据为主的 CRUD（Query 足够） |
| 复杂跨模块 action 编排、中间件审计 | 只想共享几个全局值 |
| 大量 non-React 代码读同一状态 | |

## 常见坑

1. **selector 返回新对象/数组**（`s => s.items.map(...)`）→ 每次渲染都变 → 重渲染风暴；用 createSelector。
2. **reducer 内副作用**：发请求进 thunk/RTK Query，不进 reducer。
3. **immer 写法里嵌套解构后修改**：解构出来的是草稿外引用，改了无效。
4. **忘类型化 hooks**：直接 useDispatch/useSelector 无类型；统一 useAppSelector。
5. **state 存服务端缓存**：无失效机制，数据陈旧；交给 Query 层。
6. **dispatch 后同步读 store.getState()**：拿到旧值；监听变化用 subscription/组件订阅。

## Checklist

- [ ] 统一 RTK 写法（createSlice + 类型化 hooks）
- [ ] selector 最小粒度 + createSelector 派生
- [ ] 服务端数据走 RTK Query / TanStack Query，不进 slice
- [ ] reducer 纯函数（可变风格但无副作用）
- [ ] devtools 开启
- [ ] slice 按业务域拆分
