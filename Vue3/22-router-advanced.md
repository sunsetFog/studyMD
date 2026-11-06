# 22 - 路由进阶：守卫与权限

## 1. 守卫全景

| 守卫 | 位置 | 典型用途 |
|---|---|---|
| `beforeEach` | 全局前置 | 登录校验、权限路由、埋点 |
| `beforeResolve` | 全局解析后（组件内守卫后） | 最后一次校验、SEO 设置 |
| `afterEach` | 全局后置 | 页面标题、上报 PV、滚动恢复 |
| `beforeEnter` | 路由独享 | 单路由特殊校验 |
| `beforeRouteEnter/Update/Leave` | 组件内 | 离开确认（未保存提示） |

执行顺序：

```
beforeEach → 路由 beforeEnter → 组件 beforeRouteEnter
→ beforeResolve → 导航确认 → afterEach → DOM 更新 → beforeRouteEnter 的 next 回调
```

## 2. 全局守卫标准实现

```ts
// router/guards.ts
import { useUserStore } from '@/stores/user'

const WHITE_LIST = ['/login', '/register', '/404']

router.beforeEach(async (to, from) => {
  const user = useUserStore()

  // 1. 白名单直接放行
  if (WHITE_LIST.includes(to.path)) return true

  // 2. 未登录 → 登录页，带 redirect
  if (!user.token) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }

  // 3. 权限路由未注册 → 动态注册（幂等标记）
  if (!user.routesLoaded) {
    await user.loadDynamicRoutes()      // 内部调 router.addRoute
    return to.fullPath                  // 重新进入一次，确保新路由可匹配
  }

  // 4. 页面级权限
  if (to.meta.requiresAdmin && !user.isAdmin) {
    return { name: 'forbidden' }
  }
  return true
})

router.afterEach((to, from) => {
  document.title = to.meta.title ? `${to.meta.title} · 应用` : '应用'
  trackPageView(to.fullPath)
})
```

关键细节：
- 守卫内**先注册动态路由再 return to.fullPath** 重入——直接放行会匹配失败
- redirect 回跳防循环：登录成功后 `router.push(route.query.redirect as string)`，注意校验 redirect 防开放重定向（只允许站内路径）

## 3. 动态权限路由完整方案

```ts
// stores/user.ts
async function loadDynamicRoutes() {
  const perms = await api.getPermissions()
  const accessible = filterRoutesByPerms(asyncRoutes, perms)   // 按后端权限码过滤
  accessible.forEach(r => router.addRoute(r))
  routesLoaded.value = true
}

// 静态路由 = 公共页面（login/404/首页）
// 动态路由 = 权限页面（完整定义，运行时按权限过滤注册）
```

方案对比：

| 方案 | 说明 | 适用 |
|---|---|---|
| 前端全量 + 过滤注册 | 路由表在前端，按权限码过滤 addRoute | 中后台常规方案 |
| 后端返回路由表 | 后端下发路由配置，前端映射组件 | 权限粒度极细/运营可配 |
| 前端全量 + meta 控制显示 | 不拦截导航只控制菜单 | 简单场景 |

注意：**菜单和路由权限必须同源**（同一份权限数据），避免「菜单可点但路由拦截」的割裂体验。

## 4. 组件内守卫

```ts
// 未保存离开确认
onBeforeRouteLeave((to, from) => {
  if (formDirty.value) {
    return window.confirm('有未保存修改，确定离开？')
  }
})

// 参数变化
onBeforeRouteUpdate((to) => {
  fetchDetail(to.params.id as string)
})
```

`beforeRouteEnter` 特殊：组件实例还不存在，next(vm => …) 才能访问实例（少用，用 onMounted 替代）。

## 5. 登录流程闭环

```
访问受保护页 → beforeEach 拦 → /login?redirect=/target
登录成功 → await user.login() → router.replace(redirect || '/')
退出登录 → 清 store/token → router.replace('/login') → 动态路由要 removeRoute 或 reload
```

退出后动态路由残留问题：addRoute 的路由在登出后仍可访问——登出时**移除动态路由**（记录已注册 name 列表逐个 removeRoute）或直接 `location.reload()` 重置（粗暴但可靠，看场景取舍）。

## 6. 页面标题、埋点、滚动

```ts
router.afterEach(() => { document.title = …; track(…) })
router.scrollBehavior = (to, from, saved) => saved ?? (to.meta.scrollToTop === false ? {} : { top: 0 })
```

## 7. 过渡与守卫协同

beforeEach 中 await 接口（权限获取）会阻塞导航——**loading 态**要给用户反馈：

```ts
router.beforeEach(async (to, from) => {
  if (needLoadPerms.value) {
    loadingBar.start()      // nprogress / nprogress-like
    await loadPerms()
    loadingBar.finish()
  }
})
```

## 8. 路由懒加载失败兜底

```ts
router.onError((error, to) => {
  if (error.message.includes('Failed to fetch dynamically imported module') ||
      error.message.includes('Loading chunk')) {
    // 发版后旧 chunk 失效 → 强制刷新一次
    window.location.assign(to.fullPath)
  }
})
```

## 常见坑

1. **守卫中死循环**：未登录跳 login，login 又被拦截——白名单必须含 login。
2. **动态路由注册后直接放行**：当前导航匹配不到新路由，404；必须 return fullPath 重入。
3. **redirect 开放重定向**：`redirect` query 未校验协议/域名，登录后被跳到外站（钓鱼）。
4. **登出未清动态路由**：权限路由登出后仍可直接访问。
5. **beforeEach 里每跳都拉权限**：加 routesLoaded 幂等标记。
6. **title/埋点放 beforeEach**：导航可能被取消，PV 虚高；放 afterEach。
7. **keepAlive 页面守卫与 onActivated 双重刷新**：数据逻辑入口要唯一。

## Checklist

- [ ] 白名单含 login/404 等公共页
- [ ] 动态路由注册后 return fullPath 重入
- [ ] redirect 只接受站内路径
- [ ] 登出清理动态路由
- [ ] 权限加载有幂等标记与 loading 反馈
- [ ] afterEach 处理 title/PV
- [ ] router.onError 兜底 chunk 失效
