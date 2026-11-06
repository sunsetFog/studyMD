# 20 - Pinia 状态管理

## 1. 为什么是 Pinia（替代 Vuex）

| 维度 | Pinia | Vuex 4 |
|---|---|---|
| mutation | 无（action 直接改） | 必须经 mutation |
| TS 支持 | 完整推导 | 弱 |
| 模块化 | 天然扁平多 store | 嵌套 modules |
| 组合式写法 | ✅ setup store | ❌ |
| 体积 | ~1KB | 更大 |

## 2. 基础用法

```ts
// stores/counter.ts —— Setup Store（推荐）
import { defineStore, acceptHMRUpdate } from 'pinia'
import { ref, computed } from 'vue'

export const useCounterStore = defineStore('counter', () => {
  // state → ref
  const count = ref(0)
  const name = ref('app')
  // getter → computed
  const double = computed(() => count.value * 2)
  // action → 函数
  function increment(step = 1) { count.value += step }
  async function fetchRemote() { count.value = await api.getCount() }

  return { count, name, double, increment, fetchRemote }
})

// HMR（vite 开发体验必备）
if (import.meta.hot) {
  import.meta.hot.accept(acceptHMRUpdate(useCounterStore, import.meta.hot))
}
```

```ts
// Options Store（迁移 Vuex 更顺）
export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0 }),
  getters: { double: (s) => s.count * 2 },
  actions: { increment() { this.count++ } },
})
```

约定：**新项目统一 Setup Store**（类型最顺、写法统一）。

## 3. 组件内使用

```ts
import { storeToRefs } from 'pinia'

const store = useCounterStore()

// ❌ 直接解构 store 丢响应
const { count } = store

// ✅ storeToRefs 解构 state/getter
const { count, double } = storeToRefs(store)
const { increment } = store           // action 直接解构（函数不丢）
count.value++
```

$ 工具方法：`store.$reset()`（Setup store 需自实现或不用）、`store.$patch({ count: 10 })`、`store.$subscribe(cb)`、`store.$onAction(cb)`。

## 4. store 组织

```
stores/
├── index.ts          # 导出所有 useXxxStore
├── user.ts           # 用户/权限
├── cart.ts           # 购物车
└── app.ts            # 全局 UI 状态（主题、侧栏折叠）
```

划分原则：**按业务域一个 store**，不按技术层（不放 utilsStore）。store 间可互相引用：

```ts
const user = useUserStore()
const cart = useCartStore()
// storeA 的 action 里用 storeB —— 直接在函数内调用即可
```

## 5. 状态持久化

```bash
npm i pinia-plugin-persistedstate
```

```ts
// main.ts
pinia.use(createPersistedState())

// store
export const useUserStore = defineStore('user', () => { … }, {
  persist: {
    key: 'app:user',
    storage: localStorage,
    pick: ['token', 'profile'],   // 只持久化白名单字段
  },
})
```

原则：**只持久化必要字段**（token/偏好），敏感信息（手机号、身份证）与临时态（loading）不落盘。

## 6. 异步与错误处理

```ts
async function login(payload: LoginPayload) {
  loading.value = true
  try {
    const { token, profile } = await api.login(payload)
    token.value = token; profile.value = profile
    return { ok: true }
  } catch (e) {
    return { ok: false, message: getErrMsg(e) }   // 返回结果让调用方决定 UI
  } finally {
    loading.value = false
  }
}
```

规范：action 内**不直接操作 UI**（toast/跳转），返回结果由组件层处理——store 保持可测试性。

## 7. 插件机制

```ts
const pinia = createPinia()
pinia.use(({ store }) => {
  store.$subscribe((mutation, state) => {
    // 全局持久化/埋点/日志
  })
})
```

常见官方生态：持久化、`pinia-plugin-devtools` 调试、请求缓存插件。

## 8. devtools 调试

- Vue Devtools → Pinia 面板：state 实时查看、action 时间旅行
- `store.$onAction` 记录 action 日志，排查「谁改了状态」

## 9. SSR 注意

SSR 下 store 必须按请求创建，不能模块级单例（跨请求数据污染）：

```ts
// Nuxt 已自动处理；手写 SSR 时：
const useStore = defineStore('main', () => { … })
// 每次请求新建 pinia 实例传给 app
```

## 常见坑

1. **解构丢响应**：一律 storeToRefs。
2. **Setup store 忘了 return**：属性不在 store 上，undefined 报错。
3. **$reset 在 Setup store 抛错**：自带 $reset 只支持 Options store，Setup 需手写或用 $patch。
4. **持久化大对象/敏感字段**：性能 + 安全问题，pick 白名单。
5. **store 里塞 UI 逻辑**：action 内 toast/路由跳转，破坏复用与测试。
6. **store 循环依赖**：A 顶部 import B、B 顶部 import A；改在 action 函数体内 import/use。
7. **模块级实例化 pinia**（SSR）：数据串请求。

## Checklist

- [ ] 统一 Setup Store 写法
- [ ] 组件内解构用 storeToRefs
- [ ] 持久化白名单字段且无敏感数据
- [ ] action 不直接操作 UI
- [ ] store 按 HMR 配置 acceptHMRUpdate
- [ ] devtools 可用时开启 action 日志
