# 21 - Vue Router 基础

## 1. 安装与初始化

```bash
npm i vue-router@4
```

```ts
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),          // 或 createWebHashHistory()
  routes,
  scrollBehavior(to, from, saved) {
    return saved ?? { top: 0 }
  },
})
app.use(router)
```

| history 模式 | URL | 服务器要求 |
|---|---|---|
| createWebHistory | /user/1（推荐） | 需 fallback 到 index.html |
| createWebHashHistory | /#/user/1 | 无要求（静态托管友好） |

## 2. 路由定义

```ts
const routes = [
  { path: '/', name: 'home', component: HomeView },
  { path: '/user/:id(\\d+)', name: 'user-detail', component: () => import('@/views/UserDetail.vue') },
  { path: '/admin', component: AdminLayout, children: [
      { path: '', name: 'admin-home', component: AdminHome },
      { path: 'settings', name: 'admin-settings', component: SettingsView },
  ]},
  { path: '/:pathMatch(.*)*', name: 'not-found', component: NotFound },  // 404
]
```

规范：
- **命名路由**（name 必填）：跳转/参数变更不写死 path
- **懒加载**：除首页外 component 一律 `() => import()`
- **404 兜底**放最后，pathMatch 正则
- 嵌套路由父组件需 `<RouterView />`

## 3. 动态路由参数

```vue
<script setup lang="ts">
import { useRoute, useRouter } from 'vue-router'

const route = useRoute()
const id = computed(() => Number(route.params.id))   // params 始终 string，需转换

// 参数变化组件复用（同路由不同参数）→ 监听或 key
watch(() => route.params.id, fetchUser)
// 或者模板 <RouterView :key="route.fullPath" /> 强制重建
</script>
```

params vs query：

| | params | query |
|---|---|---|
| 位置 | path 段 `/user/:id` | `?tab=a&page=2` |
| 刷新保留 | 保留（在 path 中） | 保留 |
| 可选性 | 必须匹配 | 任意 |
| 适用 | 资源标识 | 视图状态（筛选、分页） |

**视图状态（tab/筛选/滚动）放 query**，刷新/分享后还原——这是产品级最佳实践。

## 4. 编程式导航

```ts
const router = useRouter()

router.push({ name: 'user-detail', params: { id: 1 } })
router.push({ name: 'list', query: { page: 2, kw: 'x' } })
router.replace('/login')          // 不留历史
router.back() / router.forward() / router.go(-1)
```

返回结果：`router.push` 返回 Promise（NavigationFailure | undefined）：

```ts
const failure = await router.push('/x')
if (failure) {
  if (failure.type === 4) logger.warn('导航被守卫取消')  // NavigationFailureType.aborted
}
```

## 5. RouterView 与插槽

```vue
<RouterView v-slot="{ Component, route }">
  <Transition name="fade" mode="out-in">
    <KeepAlive :max="10">
      <component :is="Component" :key="route.fullPath" />
    </KeepAlive>
  </Transition>
</RouterView>
```

## 6. 路由 meta

```ts
{ path: '/profile', meta: { requiresAuth: true, title: '个人中心', keepAlive: true } }
// TS 类型增强
declare module 'vue-router' {
  interface RouteMeta {
    requiresAuth?: boolean
    title?: string
    keepAlive?: boolean
  }
}
```

meta 类型增强写在 `router/index.ts` 或全局 d.ts，meta 使用全程类型提示。

## 7. 链接与激活态

```vue
<RouterLink to="/list" class="nav-item" active-class="active" exact-active-class="exact-active">列表</RouterLink>
<RouterLink :to="{ name: 'detail', params: { id } }">详情</RouterLink>
```

嵌套路由的激活判断：父链接用 `active-class`（子路由也激活），当前精确匹配用 `exact-active-class`。

## 8. 动态添加路由（权限路由基础）

```ts
router.addRoute({ path: '/dynamic', name: 'dynamic', component: () => import('…') })
router.addRoute('admin', { … })        // 添加为某路由的子路由
router.removeRoute('dynamic')
router.hasRoute('dynamic')
router.getRoutes()                      // 全量路由表
```

异步权限路由的完整方案见 `22-router-advanced.md`。

## 常见坑

1. **params 传对象未序列化**：params 只支持 string/number；复杂数据用 query（序列化）或 state。
2. **同路由参数变化组件不更新**：watch params 或 key 强制重建。
3. **history 模式部署后刷新 404**：服务器未配 fallback（nginx `try_files $uri /index.html`）。
4. **push 重复路由报错未处理**：`router.push` 返回 Promise，捕获 NavigationFailure。
5. **path 与 name 混用**：push 带 path 时 params 会被忽略——要么全 name，要么 params 拼进 path。
6. **404 路由放前面**：正则兜底必须在最后。
7. **Transition + KeepAlive 无 key**：不同页面组件复用动画失效。

## Checklist

- [ ] 全部路由有 name，跳转优先 name
- [ ] 非首页组件懒加载
- [ ] 404 兜底路由存在
- [ ] history 模式部署 fallback 已配置
- [ ] 视图状态（筛选/分页）写 query
- [ ] meta 类型已 declare module 增强
