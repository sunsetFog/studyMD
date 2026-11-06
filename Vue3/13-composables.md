# 13 - Composables 设计规范

## 1. 什么是 Composable

利用 Composition API 封装**可复用的有状态逻辑**的函数。命名 `useXxx`。与工具函数的区别：**内部含响应式状态**。

```ts
// composables/useMouse.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)
  const update = (e: MouseEvent) => { x.value = e.clientX; y.value = e.clientY }
  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))
  return { x, y }
}
```

## 2. 设计原则

| 原则 | 说明 |
|---|---|
| 输入响应式兼容 | 参数用 `MaybeRefOrGetter`，内部 `toValue()` 归一化 |
| 返回 refs | 返回 ref 集合而非普通值，保持响应性 |
| 自动清理 | 内部监听/定时器在 onUnmounted 自动清理 |
| 无副作用泄漏 | 请求可 abort，暴露 cleanup |
| SSR 安全 | 无 window 直接访问，用生命周期守卫 |

### 2.1 接收响应式输入的标准写法

```ts
import { type MaybeRefOrGetter, toValue, watchEffect } from 'vue'

export function useFiltered(list: MaybeRefOrGetter<Item[]>, keyword: MaybeRefOrGetter<string>) {
  const filtered = computed(() => {
    const l = toValue(list)        // 兼容 ref / getter / 普通值
    const k = toValue(keyword).trim().toLowerCase()
    return k ? l.filter(i => i.name.toLowerCase().includes(k)) : l
  })
  return { filtered }
}
```

这样 `useFiltered(listRef, 'abc')` 和 `useFiltered(() => props.list, keywordRef)` 都可用。

## 3. 异步数据 composable 模板

```ts
// composables/useFetch.ts
import { ref, shallowRef, watchEffect } from 'vue'

export function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = shallowRef<T | null>(null)
  const error = ref<Error | null>(null)
  const loading = ref(false)
  let seq = 0

  watchEffect(async () => {
    const id = ++seq
    const u = toValue(url)
    loading.value = true
    error.value = null
    try {
      const res = await fetch(u)
      if (id === seq) data.value = await res.json()   // 竞态保护
    } catch (e) {
      if (id === seq) error.value = e as Error
    } finally {
      if (id === seq) loading.value = false
    }
  })

  return { data, error, loading }   // 返回 refs 保持响应
}
```

要点：`shallowRef` 存接口大对象（避免深层代理开销）；seq 竞态；返回三件套。

## 4. 分层结构

```
composables/
├── useXxx.ts          # 业务无关的通用能力（事件、存储、防抖）
├── features/          # 业务组合逻辑（useOrderSubmit）
└── api/               # 接口层（若不用自动生成）
```

| 层 | 例子 | 特点 |
|---|---|---|
| 通用 | useDebounce / useLocalStorage / useMouse | 无业务概念，可跨项目复用 |
| 业务 | useOrderList / useAuthGuard | 依赖接口与 store |
| 页面 glue | 少量 | 就写在页面里，不强行抽 |

抽取时机判断：**同一逻辑出现在 ≥2 个组件**，或一个组件内一段逻辑自成闭环且超 50 行。

## 5. 组合嵌套

composable 内部可调用其他 composable：

```ts
export function useOrderList(query) {
  const { data, loading } = useFetch(() => api.getOrderList(toValue(query)))
  const { debounced } = useDebounce(query, 300)   // 嵌套复用
  // …
}
```

## 6. vs Mixins vs Vue2 时代方案

| 维度 | Composables | Mixins |
|---|---|---|
| 来源清晰度 | 显式 import，IDE 可跳转 | 隐式合并，来源不明 |
| 命名冲突 | 无（局部作用域） | 高发 |
| 类型推导 | 完整 | 差 |
| 逻辑内聚 | 按功能聚合 | 被选项打散 |

Vue3 中 mixin 仅作 Vue2 迁移兼容，新代码禁用。

## 7. 常见内置模式清单

```
useDebounce / useThrottle
useEventListener（自动清理的全局事件）
useLocalStorage / useSessionStorage
useIntersectionObserver / useResizeObserver
useTitle / useFavicon
useMediaQuery / usePreferredColorScheme
useVirtualList
```

自建之前先查 VueUse（见 `15-vueuse.md`），避免重复造轮子。

## 常见坑

1. **参数不接 MaybeRefOrGetter**：只能传死值，复用性减半。
2. **返回普通值而非 ref**：调用方拿到快照，不随源更新。
3. **忘记清理**：每个 useXxx 内部的 addEventListener/setInterval 必须 onUnmounted 配对。
4. **在 composable 中假设组件上下文**：onMounted 等必须在 setup 同步调用链中执行，异步后再调会警告失效。
5. **composable 过度抽取**：单次使用的 30 行逻辑抽成 useXxx，反而增加跳转成本。
6. **返回对象属性名冲突**：多个 composable 返回 `{ data }` 时解构覆盖，返回时重命名或在调用处解构重命名。

## Checklist

- [ ] 通用 composables 参数为 MaybeRefOrGetter 并用 toValue
- [ ] 返回值全部是 ref/computed
- [ ] 每个监听/定时器有清理
- [ ] 异步逻辑有竞态保护
- [ ] 优先复用 VueUse，不重复造轮子
