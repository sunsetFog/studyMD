# 27 - 数据请求模式

## 1. 请求层分层

```
组件层        → 只调 hooks（useOrders/useUpdateOrder）
Query hooks 层 → useQuery/useMutation + key 工厂（见 19）
api 层        → http 实例 + 按域接口函数 + 类型
HTTP 客户端   → fetch 包装 / axios（拦截器：token/错误/重试）
```

组件不直接 fetch——所有请求经 api 层 + Query hooks 封装。

## 2. HTTP 客户端封装

```tsx
// api/http.ts（fetch 包装示例）
import { getToken, clearAuth } from '@/stores/auth'

export class ApiError extends Error {
  constructor(public status: number, public code: string, message: string) { super(message) }
}

export async function http<T>(url: string, init?: RequestInit & { silent?: boolean }): Promise<T> {
  const res = await fetch(url, {
    ...init,
    headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${getToken()}`, ...init?.headers },
    signal: init?.signal,
  })
  if (res.status === 401) { clearAuth(); location.assign('/login'); throw new ApiError(401, '', '未登录') }
  if (!res.ok) {
    const body = await res.json().catch(() => ({}))
    throw new ApiError(res.status, body.code, body.message ?? '请求失败')
  }
  return res.json()
}
```

要点：
- 统一错误类型 `ApiError`（status/code/message），上层据 status 分流
- 401 统一处理 + 并发去重（多个 401 只跳一次：模块级 flag）
- axios 等价能力：interceptors

## 3. 请求场景矩阵

| 场景 | 方案 |
|---|---|
| 页面数据（缓存） | useQuery |
| 进页面必须等数据 | useSuspenseQuery / Router loader |
| 交互触发写操作 | useMutation + invalidate |
| 表单提交 | mutation / Server Action |
| 轮询 | refetchInterval |
| 无限流 | useInfiniteQuery |
| 一次性无缓存 | mutation 或直接 http() |
| 搜索联想 | 防抖 + keepPreviousData |

## 4. 请求竞态的三道防线

```tsx
// 1. Query 层：queryKey 含全部输入 → 自动按 key 隔离 + 仅最新渲染的 key 生效
useQuery({ queryKey: ['search', kw], queryFn: () => search(kw) })

// 2. HTTP 层：AbortController
const ctrl = new AbortController()
http('/api/search', { signal: ctrl.signal, body })
// 组件卸载/新请求时 abort

// 3. 序号防线（手写时）
let seq = 0
async function doSearch(kw: string) {
  const id = ++seq
  const r = await search(kw)
  if (id === seq) setResult(r)
}
```

## 5. 加载与错误 UI 策略

```tsx
// 三态统一渲染模板
const { data, isPending, error } = useOrders(filter)

if (isPending) return <ListSkeleton />
if (error) return <ErrorState onRetry={refetch} />      // 错误可重试
if (!data.length) return <EmptyState />
return <OrderTable orders={data} />
```

规范：
- 骨架屏（结构占位）优于 spinner（全局转圈）
- 错误态必有重试入口
- 空态与错误态分开（不是一回事）
- 部分失败容忍：主数据失败才整页错误，次要区块独立处理

## 6. 乐观更新模式

见 `19-tanstack-query.md` §5：onMutate 写缓存 → onError 回滚 → onSettled 失效。适用：点赞/收藏/拖拽排序等高频低风险操作；**支付/下单类高风险操作不用乐观**（等真实响应）。

## 7. 大文件上传 / 下载

```tsx
// 上传：XMLHttpRequest 才有 progress（fetch 无）
const xhr = new XMLHttpRequest()
xhr.upload.onprogress = (e) => setProgress(e.loaded / e.total)

// 分片 + 并发 + 失败重传（大文件）
// 下载：a[download] + blob；鉴权接口走 http() 取 blob
```

## 8. 轮询与实时

```tsx
// 轮询：Query 内建
useQuery({ queryKey: ['stats'], queryFn, refetchInterval: (q) => q.state.data?.alert ? 5000 : 30000 })

// WebSocket → 自定义 hook + useSyncExternalStore 订阅（见 05 §8）
// SSE：EventSource + 同样模式
```

实时数据**不要**轮询到 Context——Zustand store + 精确订阅。

## 9. 离线与重试策略

- retry：默认 3 次指数退避；4xx 不重试（`retry: (n, err) => err.status >= 500`）
- 弱网：请求超时设置（AbortSignal.timeout(10_000)）
- 离线：`networkMode: 'offlineFirst'` + Query 缓存兜底

## 常见坑

1. **组件直接 fetch**：绕过封装，错误处理各自为政。
2. **401 并发重复跳转**：模块级 flag 去重。
3. **吞错误**（catch 后不 throw）：Query 拿不到 error 态。
4. **下载/上传无进度无取消**：体验差。
5. **POST 重试**：非幂等请求重试造成重复下单——retry 只对 GET，写操作靠幂等键（requestId）。
6. **高优接口被低优排队**：请求并发控制/优先级队列（极少场景）。

## Checklist

- [ ] 请求分四层，组件不直接 fetch
- [ ] 统一 ApiError + 401 去重
- [ ] 竞态三防线按场景启用
- [ ] 加载/错误/空三态 UI 统一模板
- [ ] 写操作幂等键，GET 才自动重试
- [ ] 大文件有进度/取消/分片
- [ ] 实时数据经 store 订阅不进 Context
