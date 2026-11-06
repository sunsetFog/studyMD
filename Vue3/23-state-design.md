# 23 - 大型应用状态设计

## 1. 状态分类学

动手设计前先分类，**不同类别状态放不同层**：

| 类别 | 示例 | 归属 |
|---|---|---|
| 服务端状态 | 列表数据、详情、用户资料 | 请求缓存层（composable/库），不进 store |
| 客户端全局 | token、主题、语言、侧栏折叠 | Pinia |
| 页面/组件本地 | 表单草稿、弹窗开关 | 组件 ref/reactive |
| URL 状态 | 筛选、分页、tab | 路由 query |

最大误区：**把所有数据都塞 Pinia**。服务端数据的「缓存、失效、重取」逻辑与全局状态完全不同，混放导致 store 巨大且陈旧数据满天飞。

## 2. 服务端状态策略

### 2.1 轻量方案：useFetch composable + 组件级缓存

见 `13-composables.md` 的 useFetch。适合中小项目。

### 2.2 中型方案：模块级请求缓存 composable

```ts
// composables/useDict.ts —— 字典类数据：全局只拉一次
const dictCache = new Map<string, AsyncState<DictItem[]>>()

export function useDict(type: string) {
  if (!dictCache.has(type)) {
    const state = reactive<AsyncState<DictItem[]>>({ data: null, loading: true })
    dictCache.set(type, state)
    api.getDict(type).then(d => { state.data = d; state.loading = false })
  }
  return dictCache.get(type)!
}
```

字典、配置类数据适合缓存共享；列表、详情按需拉取。

### 2.3 大型方案：引入 TanStack Query（Vue Query）

```ts
npm i @tanstack/vue-query
```

```ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/vue-query'

const { data, isPending, error } = useQuery({
  queryKey: ['orders', filter],        // 状态即键：筛选变了自动重取
  queryFn: () => api.getOrders(filter.value),
  staleTime: 30_000,                   // 30s 内视为新鲜，不重取
})

const qc = useQueryClient()
const mutation = useMutation({
  mutationFn: api.updateOrder,
  onSuccess: () => qc.invalidateQueries({ queryKey: ['orders'] }),   // 写后自动失效
})
```

引入信号：数据间存在「写后失效」「多组件共享同源数据」「重取/重试/缓存窗口」需求时。

## 3. Pinia store 设计规范

### 3.1 store 拆分粒度

```
✅ user（会话）  cart（购物车）  appSettings（偏好）  permissions（权限）
❌ appStore（把上面全塞一起，变成 2000 行巨物）
```

经验粒度：一个 store 一个业务域、100-300 行以内。超过即拆。

### 3.2 store 内结构模板

```ts
export const useOrderStore = defineStore('order', () => {
  // ---- state ----
  const orders = shallowRef<Order[]>([])
  const loading = ref(false)
  const filter = ref<OrderFilter>(defaultFilter)

  // ---- getters ----
  const pendingCount = computed(() => orders.value.filter(o => o.status === 'pending').length)

  // ---- actions（服务端交互 + 派生修改）----
  async function load() { … }
  async function submit(order: Order) { … }

  // ---- 内部辅助（不导出）----
  function normalize(o: RawOrder): Order { … }   // 不 return，保持私有

  return { orders, loading, filter, pendingCount, load, submit }
})
```

规范：
- **state 扁平、可序列化**（利于持久化与调试）
- **派生一律 getter**，不冗余存
- 私有辅助不 return
- 大集合用 shallowRef

### 3.3 跨 store 协作

```ts
// cart store 内引用 user store：函数体内调用，避免顶层循环依赖
async function checkout() {
  const user = useUserStore()      // ✓ 在 action 内
  await api.checkout(user.token)
}
```

## 4. URL 状态规范

```ts
// 筛选状态与路由 query 双向同步
const route = useRoute()
const filter = computed({
  get: () => parseFilter(route.query),
  set: (v) => router.replace({ query: serializeFilter(v) }),   // replace 不污染历史
})
```

收益：刷新/分享/回退状态完整还原；列表页返回时筛选保留（配合 keepAlive）。

## 5. 状态分层图（最终答案）

```
组件内 ref/reactive          ← 弹窗开关、输入草稿
    ↑ props/emit
页面级 composable            ← 页面数据（useOrderList）
    ↑
URL query                   ← 筛选/分页/tab
    ↑
Pinia                       ← 会话/偏好/权限/购物车等全局
    ↑
localStorage                 ← 仅持久化白名单（由 Pinia 插件落）
```

判定三问：
1. 刷新页面还要在吗？→ query 或 localStorage（白名单）
2. 多个页面/组件共享吗？→ Pinia 或缓存 composable
3. 都不是？→ 组件本地状态

## 6. 不可变性约定

Pinia + Vue3 响应式**允许直接改**（`store.count++`），但团队规范：
- 组件层不改 store 状态，一律调 action（可追溯、devtools 有记录）
- store 内修改集中在 action，state 定义处不散改

## 常见坑

1. **Pinia 当数据库**：列表全量塞 store、层层嵌套原始响应——改用请求缓存层。
2. **store 之间状态复制**：userStore.name 拷进 orderStore.form——单源，用时引用。
3. **服务端数据无失效策略**：staleTime/invalidate 缺失，陈旧数据；写操作后必须失效相关 query。
4. **query 状态复杂对象直接 JSON.parse**：加版本字段与解析兜底（try/catch 默认值）。
5. **全局 store 滥用 shallowRef 后手动管理**：除非 profiling 出问题，默认深层。
6. **模块级 Map 缓存未清理**：长会话内存增长；缓存加上限或 TTL。

## Checklist

- [ ] 服务端状态与客户端状态分层，Pinia 不当数据库
- [ ] 列表筛选/分页状态在 URL query
- [ ] store 按业务域拆分且行数可控
- [ ] 组件只通过 action 改 store
- [ ] 写操作后失效相关缓存/数据
- [ ] 持久化白名单 + 无敏感字段
- [ ] 跨 store 引用在 action 内调用
