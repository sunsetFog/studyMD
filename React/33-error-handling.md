# 33 - 错误处理体系

## 1. 错误分类与处理层

| 错误来源 | 处理层 | 策略 |
|---|---|---|
| 网络请求失败 | api 层 + Query | 统一错误类型/重试/提示 |
| 业务校验 | 表单层 | 字段回写 |
| 渲染错误 | ErrorBoundary | 边界兜底 + 上报 |
| 异步未捕获 | showBoundary / 全局 | 兜底上报 |
| 事件回调 | 就地 try/catch | 不进边界（边界捕不到） |

关键认知：**ErrorBoundary 只捕获渲染期错误**；事件/promise 错误需手动转交（见 14 §2）。

## 2. api 层统一拦截

见 `27-data-fetching.md` §2：统一 `ApiError(status/code/message)` + 401 去重 + 错误码文案映射表。

```ts
// constants/error-messages.ts
export const MESSAGES: Record<string, string> = {
  'AUTH_EXPIRED': '登录已过期，请重新登录',
  'RATE_LIMIT': '操作过于频繁，请稍后再试',
}
export const getFriendlyMessage = (e: unknown) =>
  e instanceof ApiError ? (MESSAGES[e.code] ?? e.message) : '网络异常，请稍后重试'
```

## 3. 三路全局捕获

```tsx
// main.tsx
window.addEventListener('unhandledrejection', (e) => {
  reportError(new Error(String(e.reason)), { type: 'unhandledrejection' })
})
window.addEventListener('error', (e) => {
  const t = e.target as HTMLElement
  if (t && (t.tagName === 'SCRIPT' || t.tagName === 'IMG' || t.tagName === 'LINK')) {
    reportError(new Error(`资源加载失败: ${(t as any).src}`), { type: 'resource' })
  }
}, true)
```

React 侧：路由级 ErrorBoundary（getDerivedStateFromError + componentDidCatch 上报，见 14）。Next.js 用 error.tsx（内建边界）。

## 4. Query 的错误处理

```tsx
// 组件级
const { error, refetch } = useQuery(…)
if (error) return <ErrorState onRetry={refetch} />

// 全局默认
new QueryClient({ defaultOptions: {
  queries: { retry: (n, e) => (e as ApiError).status >= 500 && n < 2, throwOnError: false },
  mutations: { onError: (e) => toast.error(getFriendlyMessage(e)) },
}})
```

规范：
- 查询错误组件级展示（区块化）；变更错误默认 toast（全局兜底）+ 关键表单就地处理
- retry 只对 5xx/网络错误；4xx 不重试
- 401 在 http 层拦截，不进 Query error

## 5. 错误边界部署策略

```
路由级（必配）：页面崩不拖垮全局
区块级（推荐）：图表/第三方嵌入/富组件单独包
fallback：错误信息 + 重试按钮 + 反馈入口
resetKeys：路由参数变自动重置边界
```

```tsx
<ErrorBoundary FallbackComponent={SectionError} resetKeys={[tabId]}>
  <ChartSection />
</ErrorBoundary>
```

## 6. 用户友好的呈现

| 场景 | 呈现 |
|---|---|
| 表单字段 | 内联提示 |
| 操作失败（mutation） | toast + 可重试 |
| 页面级 | 边界 fallback + 重试 |
| 静默轮询 | 不打扰，状态页反映 |

文案纪律：**说人话、给出路**。不裸抛 `code: 50012`、不显示 `undefined`。

## 7. 上报设计

```tsx
function reportError(err: Error, ctx: Record<string, unknown>) {
  const fingerprint = `${err.name}:${err.message.slice(0, 100)}`
  // 同指纹去重（内存 Map + 时间窗）+ 采样
  reporter.send({ fingerprint, stack: err.stack, ...ctx, version: __APP_VERSION__, url: location.href })
}
```

必带：stack、版本、路由、组件栈（componentStack）、sourcemap 体系对应 release。

## 8. 重试与降级

```tsx
// 指数退避
async function withRetry<T>(fn: () => Promise<T>, times = 3) {
  for (let i = 0; ; i++) {
    try { return await fn() } catch (e) {
      if (i >= times - 1 || (e as ApiError).status! < 500) throw e
      await new Promise(r => setTimeout(r, 500 * 2 ** i))
    }
  }
}
```

降级：接口失败展示上次成功缓存（Query gcTime 内自动兜底）、特性开关关闭。

## 常见坑

1. **以为边界能捕异步/事件错误**：showBoundary / try-catch 转交。
2. **onErrorCaught 忘 return false**（Next/库场景）：重复上报。
3. **吞错误**（catch 空块）：unhandledrejection 噪音或静默失败。
4. **retry 对非幂等 POST**：重复下单；写操作幂等键。
5. **全局 toast 轰炸**：轮询类静默化（silent 配置）。
6. **sourcemap 与版本不对应**：stack 不可读。

## Checklist

- [ ] ApiError 统一类型 + 文案映射表
- [ ] 路由级 ErrorBoundary + resetKeys + 重试
- [ ] 异步错误转交边界（showBoundary）
- [ ] 全局 unhandledrejection + 资源 error 兜底
- [ ] retry 区分幂等性
- [ ] 上报带版本 + sourcemap 体系
- [ ] 错误文案不裸抛 code
