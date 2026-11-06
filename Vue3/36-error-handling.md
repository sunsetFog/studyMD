# 36 - 错误处理体系

## 1. 错误分类与处理层

| 错误来源 | 处理层 | 策略 |
|---|---|---|
| 网络请求失败 | api 层统一拦截 | 重试/降级/提示 |
| 业务校验失败 | 表单/提交处 | 内联提示 |
| 渲染/运行时错误 | 组件 + 全局捕获 | 边界兜底 + 上报 |
| Promise 未处理 | 全局 onunhandledrejection | 兜底上报 |
| 资源加载失败 | window error | 上报 |

原则：**可预期错误就地处理，不可预期错误逐层上报兜底**。

## 2. api 层统一拦截（第一道防线）

```ts
// api/http.ts
import axios, { AxiosError } from 'axios'

http.interceptors.response.use(
  (res) => res.data,
  async (err: AxiosError<ApiError>) => {
    const status = err.response?.status
    const code = err.response?.data?.code

    if (status === 401) { await relogin(); return }
    if (isRetryable(status) && retried(err.config) < 3) return retry(err)  // 5xx/超时自动重试
    if (shouldSilent(err.config)) throw err      // 静默请求由调用方处理
    showToast(getFriendlyMessage(code, status))  // 默认统一提示
    throw err
  },
)
```

规范：
- 统一错误码 → 文案映射表（`constants/error-messages.ts`）
- 请求 config 上标记 `silent`（不打扰型轮询请求）
- 401 统一处理（去重并发多个 401 只弹一次）

## 3. 组件级错误边界

Vue3 无官方 ErrorBoundary 组件，用 `onErrorCaptured` 组合：

```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const hasError = ref(false)
const emit = defineEmits<{ error: [Error] }>()

onErrorCaptured((err, instance, info) => {
  hasError.value = true
  emit('error', err)
  reportError(err, { info, component: instance?.$options.name })
  return false   // 阻止向上扩散
})
function reset() { hasError.value = false }   // 配合 :key 重挂载子树
</script>

<template>
  <slot v-if="!hasError" />
  <div v-else class="error-boundary">
    <p>页面出错了</p>
    <button @click="reset">重试</button>
  </div>
</template>
```

部署粒度：**路由级**（每个页面包一层）比全 app 一个更利于隔离。

## 4. 全局捕获（最后防线）

```ts
// main.ts
import { captureError } from '@/utils/error-report'

app.config.errorHandler = (err, instance, info) => {
  captureError(err as Error, { info, component: instance?.$options.name })
  console.error(err)
}

window.addEventListener('unhandledrejection', (e) => {
  captureError(new Error(String(e.reason)), { type: 'unhandledrejection' })
})

window.addEventListener('error', (e) => {
  if (e.target instanceof HTMLScriptElement || e.target instanceof HTMLImageElement) {
    captureError(new Error(`资源加载失败: ${(e.target as any).src}`), { type: 'resource' })
  }
}, true)   // 捕获阶段才拿得到资源错误
```

errorHandler 与 onErrorCaptured 的关系：captured 返回 false 则不再冒泡到全局；未捕获才到全局 handler。

## 5. async setup 与 Suspense 错误

见 `18-suspense-async.md`：async setup 抛错由 onErrorCaptured 捕获，Suspense 自身不处理错误。页面级组合：

```vue
<ErrorBoundary>
  <Suspense :key="route.fullPath">
    <PageView />
    <template #fallback><PageSkeleton /></template>
  </Suspense>
</ErrorBoundary>
```

## 6. 用户友好的错误呈现

| 场景 | 呈现 |
|---|---|
| 表单校验 | 字段内联提示 |
| 提交失败 | 轻提示（可重试） |
| 页面级失败 | 错误边界 + 重试按钮 |
| 静默后台任务 | 不打扰，状态页显示 |

文案规范：**说人话、给出路**。`code: 50012` 不直接展示；`「加载失败，点击重试」` 而非 `undefined`。

## 7. 错误上报设计

```ts
// utils/error-report.ts
export function captureError(err: Error, ctx: Record<string, unknown>) {
  // 1. 聚合指纹（防刷屏）
  const fingerprint = `${err.name}:${err.message.slice(0, 100)}`
  // 2. 采样/去重（同指纹 5 分钟一次）
  // 3. 携带上下文：路由、用户 ID、版本号、UA
  reporter.send({ fingerprint, stack: err.stack, ctx, version: __APP_VERSION__, url: location.href })
}
```

必带字段：stack、版本号、路由、组件名/生命周期阶段、release 对应 sourcemap。

## 8. 重试与降级

```ts
async function withRetry<T>(fn: () => Promise<T>, times = 3, delay = 500): Promise<T> {
  for (let i = 0; i < times; i++) {
    try { return await fn() } catch (e) {
      if (i === times - 1) throw e
      await new Promise(r => setTimeout(r, delay * 2 ** i))   // 指数退避
    }
  }
  throw new Error('unreachable')
}
```

降级：接口失败时缓存兜底（上次成功数据）、功能开关关闭新特性。

## 常见坑

1. **errorHandler 吞掉所有错误**：不上报只 console，线上无感知。
2. **onErrorCaptured 忘 return false**：同错误重复上报。
3. **Promise 链缺 catch**：unhandledrejection 噪音淹没真错误。
4. **重试无指数退避/上限**：雪崩放大。
5. **401 并发重复弹窗**：拦截器去重。
6. **sourcemap 未关联版本**：stack 不可读，等于没监控。
7. **error boundary 粒度过粗**：一个角落报错整 app 白屏。

## Checklist

- [ ] api 拦截器统一处理错误码/401/重试
- [ ] 路由级 ErrorBoundary + 重试
- [ ] 全局 errorHandler + unhandledrejection + 资源 error 三路捕获
- [ ] 上报带版本与 sourcemap 体系
- [ ] 错误文案映射表，不裸抛 code
- [ ] 重试指数退避 + 上限
