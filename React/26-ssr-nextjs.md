# 26 - SSR 与 Next.js

## 1. 渲染模式全景

| 模式 | 说明 | 适用 |
|---|---|---|
| CSR | 纯客户端渲染（Vite SPA） | 中后台、工具 |
| SSR | 每请求服务端渲染 | 动态内容 + SEO |
| SSG | 构建期静态生成 | 博客/文档 |
| ISR | 增量静态再生 | 内容站 |
| RSC（React Server Components） | 服务端组件不进客户端 bundle | Next.js App Router 核心 |

Next.js App Router 的默认世界 = **RSC 优先**：组件默认在服务端渲染，需要交互（state/effect/事件）才标 `'use client'`。

## 2. App Router 结构

```
app/
├── layout.tsx            # 根布局（html/body）
├── page.tsx              # /
├── loading.tsx           # 路由级 Suspense fallback（自动）
├── error.tsx             # 路由级 ErrorBoundary（自动）
├── not-found.tsx
├── dashboard/
│   ├── layout.tsx
│   ├── page.tsx
│   └── [id]/page.tsx     # 动态路由
└── api/                  # Route Handlers（服务端 API）
```

约定即路由：文件夹即路径，`layout` 嵌套自动组合，`loading/error/not-found` 是内建的 Suspense/边界。

## 3. Server Components vs Client Components

```tsx
// Server Component（默认）：服务端执行，可直连 DB/密钥，无 hooks/事件
import { db } from '@/lib/db'
export default async function Page() {
  const user = await db.user.findFirst()     // 直接 await！
  return <Profile user={user} />
}

// Client Component：'use client' 顶部声明
'use client'
import { useState } from 'react'
export function Counter() { … }
```

| | Server | Client |
|---|---|---|
| 数据访问 | DB/API/密钥安全 | 仅通过 API/route handler |
| hooks/state/events | ❌ | ✅ |
| bundle | 不进客户端 | 进客户端 |
| 用途 | 数据聚合、静态内容 | 交互、本地状态 |

规则：
- **数据获取尽量在 Server Component**（省一层 API + 无客户端请求瀑布）
- Server → Client 传 props 必须可序列化（不能传函数/类实例/Date 之外的复杂对象注意）
- `'use client'` 边界是「客户端入口」，其子组件默认都是客户端

## 4. 数据获取三件套

```tsx
// 1. RSC 直接 fetch/DB（Server Component 内）
const data = await fetch('https://…', { next: { revalidate: 60 } })   // ISR 缓存

// 2. Server Actions（表单/变更）
'use server'
export async function createOrder(formData: FormData) { … }

// 3. 客户端交互数据：TanStack Query（在 'use client' 组件中）
```

缓存语义（Next 15 趋于显式）：
- `fetch` 默认不再缓存（Next 15）；`{ cache: 'force-cache' }` / `{ next: { revalidate: 60 } }` 显式控制
- Server Action 后 `revalidatePath('/x')` / `revalidateTag('orders')` 失效

## 5. 表单与 Mutations

```tsx
// Server Action + useFormStatus/useActionState（React 19 Actions 生态）
'use client'
import { useActionState } from 'react'
import { createOrder } from './actions'

function OrderForm() {
  const [state, action, pending] = useActionState(createOrder, null)
  return (
    <form action={action}>
      <input name="title" />
      <button disabled={pending}>{pending ? '提交中' : '提交'}</button>
    </form>
  )
}
```

Server Actions 是 Next 全栈开发的 mutation 标准姿势（渐进增强 + 内建 pending）。

## 6. 流式渲染与 Suspense

```tsx
// page.tsx：慢的部分包 Suspense，骨架先出
export default function Page() {
  return (
    <>
      <Header />                              {/* 立即渲染 */}
      <Suspense fallback={<StatsSkeleton />}>
        <SlowStats />                         {/* await 数据，流式补齐 */}
      </Suspense>
    </>
  )
}
```

RSC 流式 = 服务端逐块推 HTML + 注水，首屏 TTFB 显著优于整页 await。

## 7. 水合错误排查

同 vue3/29 的问题集：时间戳/随机值/localStorage 首屏/locale 差异。

```tsx
// 客户端专属渲染
import dynamic from 'next/dynamic'
const Clock = dynamic(() => import('./Clock'), { ssr: false })
```

## 8. 部署与性能

- 托管：Vercel（零配置）/ Node 服务器 / Docker
- 图片：`next/image` 自动 webp/尺寸/懒加载
- 字体：`next/font` 自托管零布局偏移
- Meta：`generateMetadata` / 直接写 `<title>`（React 19）

## 9. 什么时候不用 Next

```
纯中后台（登录后工具）→ Vite SPA 更简单
嵌入现有后端（Java/Go 渲染页面）→ SPA 微前端
需要 SEO/首屏/全栈一体 → Next App Router
```

## 常见坑

1. **Server Component 里用了 hooks/事件**：报错；补 `'use client'` 或下沉交互到子组件。
2. **'use client' 标太顶层**：整页变客户端 bundle；边界尽量下沉。
3. **Server → Client 传不可序列化 props**：函数/类实例报错。
4. **fetch 缓存误解**：Next 15 默认不缓存，需要显式声明（版本差异大，查当前文档）。
5. **Server Action 里忘鉴权**：Action 是公开端点，必须独立鉴权。
6. **水合不一致忽略**：整树重渲染。
7. **metadata 不生效**：在 Server Component/page 中导出（client 组件不行）。

## Checklist

- [ ] 数据获取默认 Server Component，交互才 'use client'
- [ ] 'use client' 边界最小化下沉
- [ ] 慢数据段用 Suspense 流式
- [ ] mutations 走 Server Actions + pending 状态
- [ ] revalidate 策略明确
- [ ] next/image、next/font 采用
- [ ] 水合错误零容忍
