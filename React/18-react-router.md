# 18 - React Router

## 1. 版本与模式

```bash
npm i react-router-dom     # v6/v7
```

- **v6/v7 数据路由**：`createBrowserRouter` + Loader/Action（推荐）
- v7 可作为框架模式（配合 Remix 思路）或库模式；本文以库模式 + v7 API 为准

## 2. 基础路由定义

```tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom'

const router = createBrowserRouter([
  { path: '/', element: <Home /> },
  { path: '/user/:id', element: <UserDetail /> },
  {
    path: '/admin',
    element: <AdminLayout />,
    children: [
      { index: true, element: <AdminHome /> },
      { path: 'settings', element: <Settings /> },
    ],
  },
  { path: '*', element: <NotFound /> },
])

createRoot(root).render(<RouterProvider router={router} />)
```

规范：`element` 直接写组件；`*` 404 兜底；嵌套子路由父组件放 `<Outlet />`。

## 3. 导航

```tsx
import { Link, NavLink, useNavigate } from 'react-router-dom'

<Link to="/user/1">详情</Link>
<NavLink to="/list" className={({ isActive }) => isActive ? 'active' : ''}>列表</NavLink>

const navigate = useNavigate()
navigate('/login')
navigate(-1)                                  // 后退
navigate('/list', { replace: true })          // 替换历史
navigate({ pathname: '/list', search: '?page=2' })
```

## 4. 参数与查询

```tsx
import { useParams, useSearchParams, useLocation } from 'react-router-dom'

// 路径参数
const { id } = useParams()          // string，需转换
const userId = Number(id)

// 查询参数（受控双向）
const [searchParams, setSearchParams] = useSearchParams()
const page = Number(searchParams.get('page') ?? 1)
setSearchParams({ page: String(p + 1), kw })   // 写入 URL（可分享/刷新还原）

// location 对象
const location = useLocation()      // pathname/search/state/key
```

**视图状态（筛选/分页/tab）放 searchParams**——刷新与分享还原，配合 Query 的 queryKey 联动（见 19）。

## 5. 嵌套布局与 Outlet

```tsx
function AdminLayout() {
  return (
    <div>
      <AdminNav />
      <Outlet />                {/* 子路由出口 */}
      <Outlet context={userData} />   {/* 传 context 给子路由 */}
    </div>
  )
}
// 子路由中
const data = useOutletContext()
```

## 6. 数据路由：Loader / Action（v6.4+/v7）

```tsx
{
  path: '/user/:id',
  loader: async ({ params, signal }) => {
    const res = await fetch(`/api/user/${params.id}`, { signal })
    if (res.status === 404) throw new Response('Not Found', { status: 404 })
    return res.json()
  },
  element: <UserDetail />,
  errorElement: <ErrorPage />,      // loader/渲染错误兜底
}

// 组件内
const user = useLoaderData<User>()
```

- **导航前预取**：loader 在路由切换时先执行，数据就绪才渲染（避免瀑布）
- `errorElement`：路由级错误边界（React Router 内建）
- Action 处理 form 提交（`useFetcher` / `<Form>`）

与 TanStack Query 的关系：Loader 适合「进页面必须等数据」的路由预取；也可以 Query-only（组件内 useQuery + Suspense）。团队统一一种即可。

## 7. 守卫（权限路由）

React Router 无内建守卫，标准做法是**布局级拦截组件**：

```tsx
function RequireAuth({ children }: { children: JSX.Element }) {
  const { user } = useAuth()
  const location = useLocation()
  if (!user) return <Navigate to="/login" state={{ from: location }} replace />
  return children
}

// 用法
{ path: '/admin', element: <RequireAuth><AdminLayout /></RequireAuth> }
```

登录后回跳：

```tsx
const location = useLocation()
const from = location.state?.from?.pathname ?? '/'
navigate(from, { replace: true })
```

安全注意：**前端拦截只是体验层**，接口鉴权在后端（同 vue3/22）；redirect 校验防开放重定向。

## 8. 动态路由与懒加载

```tsx
const router = createBrowserRouter([
  { path: '/admin', lazy: async () => { const m = await import('@/pages/Admin'); return { Component: m.default } } },
])

// 或传统 lazy + Suspense
{ path: '/chart', element: <Suspense fallback={<Skeleton />}><ChartPage /></Suspense> }
```

权限动态注册：router 创建后 `router.routes` 可变，或用 `createMemoryRouter`/受控方案；常规做法是**静态全量注册 + RequireAuth 层拦截**（比动态 addRoute 简单可靠）。

## 9. 过渡动画与滚动恢复

```tsx
// v7 内建 scrollRestoration（数据路由）
import { ScrollRestoration } from 'react-router-dom'
// 布局组件中：<ScrollRestoration />

// 路由过渡：framer-motion AnimatePresence + useLocation key
<AnimatePresence mode="wait">
  <Routes location={location} key={location.pathname}>…
```

## 10. 懒加载失败与 404 兜底

```tsx
{ path: '*', element: <NotFound /> }                       // 必须最后
// errorElement 兜底 loader/渲染错误，内含 retry + reload
```

## 常见坑

1. **useParams 是 string**：Number 转换 + NaN 防御。
2. **Navigate 循环**：login 页也要放行，RequireAuth 别包 login。
3. **searchParams 数字为 null**：`?? 1` 兜底。
4. **loader 里异常未抛 Response**：errorElement 拿不到状态码。
5. **RequireAuth 忘 replace**：登录后 history 堆积 login。
6. **URL 状态直接塞对象**：searchParams 只能 string，复杂对象序列化。
7. **404 `*` 放中间**：兜底必须最后。

## Checklist

- [ ] 404 与 errorElement 兜底齐全
- [ ] 筛选/分页状态在 searchParams
- [ ] RequireAuth 布局级拦截 + from 回跳 + redirect 校验
- [ ] 路由组件懒加载
- [ ] 数据获取策略统一（loader 或 Query，不混用无章法）
- [ ] 接口鉴权在后端（前端拦截仅体验层）
