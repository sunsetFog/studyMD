# 34 - 组合式设计模式

## 1. 模式总览

Vue3 Composition API 支持一批经典模式的现代化实现。本文聚焦「Vue3 里怎么做得对」，而非泛讲设计模式。

| 模式 | Vue3 实现 |
|---|---|
| 容器/展示分离 | 容器页 + props 驱动子组件 |
| 提供者/消费者 | provide/inject（readonly） |
| 状态机 | 状态机 composable |
| 策略模式 | 校验规则/映射表 |
| 组合优于继承 | composables 组合 |
| 控制反转 | 依赖注入 + 接口抽象 |

## 2. 容器 / 展示分离

```
容器（views/xxx.vue）：取数、状态编排、路由 —— 不写样式
展示（components/）：纯 props 入、emit 出 —— 不发请求
```

收益：展示组件可测可复用；数据逻辑集中在容器，来源单一。

变体：**容器逻辑抽 composable**（useOrderListPage），视图层薄到只剩模板——大型列表页的标准形态：

```ts
// views/order/useOrderListPage.ts
export function useOrderListPage() {
  const filter = ref<OrderFilter>(defaultFilter)
  const { data, loading, reload } = useOrderList(filter)
  const onPageChange = (p: number) => { filter.value.page = p }
  return { filter, data, loading, reload, onPageChange }
}
```

## 3. 状态机 composable

异步流程状态不可枚举时（idle/loading/success/error），状态机避免布尔变量爆炸：

```ts
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error }

function useAsyncTask<T>(fn: () => Promise<T>) {
  const state = shallowRef<AsyncState<T>>({ status: 'idle' })
  let seq = 0
  async function run() {
    const id = ++seq
    state.value = { status: 'loading' }
    try {
      const data = await fn()
      if (id === seq) state.value = { status: 'success', data }
    } catch (e) {
      if (id === seq) state.value = { status: 'error', error: e as Error }
    }
  }
  return { state, run }
}

// 使用：模板里 switch status 渲染，互斥状态不可能同时为真
```

对比 `loading/error/data` 三个独立布尔：状态机**不可能出现 loading=true 且 error 有值**的非法态。

## 4. 策略模式：校验/渲染映射

```ts
// 校验策略表
const validators = {
  required: (v: unknown) => v != null && v !== '' || '必填',
  phone: (v: string) => /^1\d{10}$/.test(v) || '手机号格式错误',
} as const

type Rule = { [K in keyof typeof validators]?: Parameters<(typeof validators)[K]>[0] }

const rules: Rule[] = [{ required: 'name' }, { phone: 'mobile' }]
```

配置驱动的表单、报表、权限映射都是此模式；配合 `17-render-jsx.md` 的 schema 渲染器可覆盖大多数「运营可配置」需求。

## 5. 依赖注入式架构（可替换实现）

```ts
// 接口抽象
interface StorageAdapter {
  get<T>(key: string): T | null
  set(key: string, value: unknown): void
}

// 实现 A：localStorage；实现 B：加密存储；实现 C：测试内存实现
export const STORAGE_KEY: InjectionKey<StorageAdapter> = Symbol('storage')

// 组件只依赖接口
const storage = inject(STORAGE_KEY)!
storage.set('draft', form)
```

收益：测试注入内存实现、需求变更换实现，业务代码零改动。

## 6. 组合模式：composable 层叠

```ts
// 底层原语
const { debounced } = useDebounce(query, 300)
// 中层组合
const { data, loading } = useSearch(debounced)
// 上层编排
function useSearchPage() {
  const { debounced } = useDebounce(keyword, 300)
  const { data, loading, reload } = useSearch(debounced)
  const { track } = usePageLog('search')
  watch(debounced, () => track('search', debounced.value))
  return { keyword, debounced, data, loading, reload }
}
```

原则：**每层职责单一、依赖显式**；不要在底层原语里 import 业务 store。

## 7. 插槽型扩展点（模板级策略）

```vue
<!-- 列表组件把"每行怎么渲染"开放为插槽 -->
<ProList :items="items">
  <template #item="{ item }"><slot name="item" v-bind="{ item }" /></template>
  <template #empty>自定义空态</template>
</ProList>
```

组件库设计核心：**通过插槽预留「策略注入点」**，避免为每种定制做一个 prop 开关。

## 8. 反模式：继承思维残留

| 反模式 | 替代 |
|---|---|
| 大而全 BaseComponent + extends | composables 组合 |
| mixin 变相复用（setup 里调 super） | 显式 composable |
| 一个组件 20 个 boolean prop 组合爆炸 | 插槽 / 状态机 / 配置对象 |

## 常见坑

1. **容器组件里写样式、展示组件里发请求**：职责反向。
2. **布尔组合表达状态**：`loading && !error && hasData` 判断分支爆炸，改状态机。
3. **策略表没类型约束**：key 拼错静默失效，satisfies/Record 收紧。
4. **依赖注入过度**：三层 inject 传递一个配置——直接 props。
5. **composable 层级混乱**：底层 import 顶层 store 造成循环依赖。

## Checklist

- [ ] 页面有容器/展示或 composable 分层
- [ ] 多态异步状态用状态机而非布尔组合
- [ ] 配置驱动逻辑有类型约束
- [ ] 可替换实现依赖注入
- [ ] 组件库扩展点优先插槽
