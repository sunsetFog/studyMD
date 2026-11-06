# 19 - TanStack Query（服务端状态标准答案）

## 1. 定位

```bash
npm i @tanstack/react-query
```

TanStack Query（React Query）管理**服务端状态**：缓存、去重、失效、重试、分页、乐观更新。它不是请求库（fetch 仍自己写/由 axios 承担），是「服务端数据的客户端缓存数据库」。

```
useState/useEffect 手写数据获取的问题：
无缓存、无去重、无失效、竞态易错、重试/Loading 各写各的 → Query 全部内建
```

## 2. 基础：useQuery / useMutation

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

// 入口
const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 30_000, retry: 2, refetchOnWindowFocus: false } },
})
<QueryClientProvider client={queryClient}><App /></QueryClientProvider>

// 查询
const { data, isPending, error, refetch } = useQuery({
  queryKey: ['orders', filter],          // 状态即缓存键（序列化后的数组）
  queryFn: () => api.getOrders(filter),
  staleTime: 30_000,                     // 30s 内视为新鲜不发请求
  enabled: !!filter.deptId,              // 条件启用
})

// 变更
const qc = useQueryClient()
const mutation = useMutation({
  mutationFn: (patch: OrderPatch) => api.updateOrder(patch),
  onSuccess: () => qc.invalidateQueries({ queryKey: ['orders'] }),   // 写后失效
  onError: (e) => toast(getErrMsg(e)),
})
mutation.mutate({ id: 1, patch: { status: 2 } })
```

## 3. queryKey 设计（核心）

```tsx
// 层级数组：从粗到细
['orders']                          // 全部订单缓存域
['orders', { page: 2, kw: 'x' }]    // 具体查询实例
['orders', 'detail', id]            // 详情

// 精确失效
qc.invalidateQueries({ queryKey: ['orders'] })                 // 整域失效
qc.invalidateQueries({ queryKey: ['orders', 'detail', id] })   // 单条
qc.setQueryData(['orders', 'detail', id], updater)             // 直接写缓存
```

规则：
- key 必须包含**所有影响结果的输入**（筛选、分页、权限范围）
- key 可序列化（对象会被稳定序列化）
- key 设计 = 缓存架构，先设计 key 再写 queryFn

## 4. 新鲜度与重取策略

| 参数 | 含义 |
|---|---|
| `staleTime` | 数据多久内算新鲜（不重取）——**核心调优参数** |
| `gcTime`（原 cacheTime） | 无订阅后缓存保留时长 |
| `refetchOnWindowFocus` | 回到 tab 自动重取（后台系统常关） |
| `refetchInterval` | 轮询 |
| `retry` | 失败重试次数（默认 3，指数退避） |
| `enabled` | 条件执行 |

经验配置：管理后台 `staleTime: 60s + refetchOnWindowFocus: false`；看板类保留 focus 重取 + interval 轮询。

## 5. 失效与写入联动（CRUD 闭环）

```tsx
// 创建/更新/删除后 → 失效相关列表与详情
const createOrder = useMutation({
  mutationFn: api.createOrder,
  onSuccess: () => {
    qc.invalidateQueries({ queryKey: ['orders'] })
    toast('创建成功')
  },
})

// 乐观更新（列表即时反馈）
useMutation({
  mutationFn: api.like,
  onMutate: async (id) => {
    await qc.cancelQueries({ queryKey: ['posts'] })         // 取消在途避免覆盖
    const prev = qc.getQueryData<Post[]>(['posts'])
    qc.setQueryData(['posts'], old => old!.map(p => p.id === id ? { ...p, likes: p.likes + 1 } : p))
    return { prev }                                          // 上下文用于回滚
  },
  onError: (_e, _v, ctx) => qc.setQueryData(['posts'], ctx!.prev),   // 失败回滚
  onSettled: () => qc.invalidateQueries({ queryKey: ['posts'] }),
})
```

## 6. 依赖请求与分页

```tsx
// 依赖请求：先拿用户，再拿订单
const { data: user } = useQuery({ queryKey: ['user', id], queryFn: () => api.getUser(id) })
const { data: orders } = useQuery({
  queryKey: ['orders', id],
  queryFn: () => api.getOrders(id),
  enabled: !!user,     // user 就绪才发
})

// 分页 keepPreviousData（翻页不闪 loading）
import { keepPreviousData } from '@tanstack/react-query'
useQuery({ queryKey: ['orders', page], queryFn: () => api.getOrders(page), placeholderData: keepPreviousData })

// 无限加载
useInfiniteQuery({ queryKey: ['feed'], queryFn: ({ pageParam }) => api.feed(pageParam),
  initialPageParam: 0, getNextPageParam: (last) => last.nextCursor })
```

## 7. Suspense 模式

```tsx
import { useSuspenseQuery } from '@tanstack/react-query'

// promise 挂起交给最近 Suspense，错误交给 ErrorBoundary
const { data } = useSuspenseQuery({ queryKey: ['user', id], queryFn: … })
```

配合 `14-portal-boundary-suspense.md` 的页面骨架（Boundary 外层 + Suspense 内层），组件内**零 loading 判断**。

## 8. 请求层封装（项目级标准结构）

```tsx
// queries/orders.ts —— queryKey 工厂 + hook 封装
export const orderKeys = {
  all: ['orders'] as const,
  list: (q: OrderQuery) => [...orderKeys.all, 'list', q] as const,
  detail: (id: number) => [...orderKeys.all, 'detail', id] as const,
}

export function useOrders(q: OrderQuery) {
  return useQuery({ queryKey: orderKeys.list(q), queryFn: () => api.getOrders(q) })
}

export function useUpdateOrder() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: api.updateOrder,
    onSuccess: (_d, vars) => qc.invalidateQueries({ queryKey: orderKeys.detail(vars.id) }),
  })
}

// 组件只见 useOrders / useUpdateOrder，不见 useQuery 细节
```

**key 工厂模式**（tkdodo 推荐）让失效关系显式可维护，是团队级最佳实践。

## 9. Query vs 其他方案

| 场景 | 方案 |
|---|---|
| 服务端数据（绝大多数） | TanStack Query |
| RTK 栈项目 | RTK Query（同栈等价物） |
| Next.js App Router | RSC fetch / Server Actions 优先，客户端交互再 Query |
| 一次性非缓存请求 | 直接 mutation 或普通 fetch 即可 |

## 常见坑

1. **queryKey 漏了输入变量**：不同筛选命中同一缓存 → 数据错。
2. **staleTime 0 默认**：每次挂载都重取（默认 staleTime 0 + 立即重取），性能差；调优 staleTime。
3. **写后忘 invalidate**：界面陈旧。
4. **组件卸载误解**：Query 缓存不随卸载丢（gcTime 才丢）——缓存共享是它的核心价值。
5. **乐观更新忘 cancelQueries/回滚**：在途请求覆盖乐观值。
6. **queryFn 内吞错误**：throw 出去才能进 error 态。
7. **Query 当状态总线**：客户端状态（选中项、弹窗）别塞 queryKey。

## Checklist

- [ ] QueryClientProvider 全局 + 默认参数调优
- [ ] queryKey 工厂模式，包含全部输入
- [ ] 写操作 invalidate 闭环完整
- [ ] staleTime 按 业务调优，focus 重取明确决策
- [ ] 乐观更新含 cancel/回滚/settled 失效
- [ ] 请求 hook 化封装，组件不散用 useQuery
