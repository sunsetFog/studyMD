# 04 - computed / watch / watchEffect

## 1. 三者定位

| API | 定位 | 何时用 |
|---|---|---|
| `computed` | 派生值（同步、缓存） | 由已有状态计算出的值 |
| `watch` | 显式监听（可异步、可拿到新旧值） | 状态变化后执行副作用 |
| `watchEffect` | 立即执行并自动追踪依赖 | 依赖多的副作用、简易场景 |

## 2. computed

```ts
const fullName = computed(() => `${firstName.value} ${lastName.value}`)
const count = computed({
  get: () => base.value * 2,
  set: (v) => (base.value = v / 2),   // 可写 computed
})
count.value = 10   // base 变 5
```

核心特性：
- **惰性求值 + 缓存**：依赖不变不重算，这是与方法调用的本质区别
- **脏检查**：只有依赖变化才标记 dirty
- **必须纯函数**：不要在 computed 里改状态、发请求、操作 DOM

### 2.1 computed vs methods

```vue
<p>{{ now() }}</p>       <!-- methods：每次渲染都执行 -->
<p>{{ now }}</p>         <!-- computed：缓存，仅依赖变化时执行 -->
```

渲染重算时 methods 全部重跑；computed 只在依赖变化时重跑。

## 3. watch

```ts
import { watch } from 'vue'

// 单个源
watch(count, (n, o) => {…})

// 多个源（数组）
watch([a, b], ([na, nb], [oa, ob]) => {…})

// getter 函数源：监听 reactive 的属性必须用函数
watch(() => state.user.name, (n) => {…})

// 对象源默认隐式深层
watch(state, (n) => {…})                    // reactive 对象 → 自动深层
watch(() => state, (n) => {…}, { deep: true }) // getter 返回对象 → 需显式 deep
```

### 3.1 options

```ts
watch(src, cb, {
  immediate: true,  // 立即执行一次（初始加载场景）
  deep: true,       // 深层遍历（大对象慎用，性能）
  flush: 'post',    // 回调在 DOM 更新后执行（默认 'pre'）
  once: true,       // 只触发一次（3.4+）
})
```

flush 语义：

| flush | 时机 | 典型场景 |
|---|---|---|
| `pre`（默认） | DOM 更新前 | 状态联动 |
| `post` | DOM 更新后 | 需要访问更新后的 DOM |
| `sync` | 状态变更同步执行 | 极少用，性能杀手 |

### 3.2 拿 DOM：onWatcherCleanup / post

```ts
watch(src, async () => {
  // flush: 'post' 时此处 DOM 已更新
  await nextTick()
}, { flush: 'post' })
```

## 4. watchEffect

```ts
watchEffect(async () => {
  const res = await api.getData(query.value, page.value)
  result.value = res
  // query/page 在同步段被读取 → 自动成为依赖
  // await 之后的代码不参与依赖追踪！
})
```

规则：
- **同步执行段**内读取的响应式数据才是依赖；`await` 之后的读取不算
- 无法拿到旧值；首次立即执行（等价 immediate）
- 适合「依赖多、逻辑内聚」的场景；需要精确控制时用 watch

### 4.1 停止与清理

```ts
const stop = watchEffect((onCleanup) => {
  onCleanup(() => abortController.abort()) // 下次执行前/停止时调用
})
stop() // 手动停止
```

## 5. 竞态处理（异步 watch 必做）

```ts
let seq = 0
watch(query, async (q) => {
  const id = ++seq
  const res = await api.search(q)
  if (id === seq) result.value = res   // 只有最新请求可写入
})
```

替代写法：`onCleanup` 中中止上一次请求（AbortController），语义等价且更优雅。

## 6. watch 依赖的黄金法则

```ts
// ❌ 监听 reactive 的属性（ getter 隐式返回值，实际是解构出的普通值？
// 否——watch(src) 若 src 是普通值则无响应！）
watch(state.count, cb)        // ❌ state.count 是 number，无追踪

// ✅ getter
watch(() => state.count, cb)

// ❌ 监听 ref 数组内嵌套
watch([() => state.a, b], cb) // ✅ getter 与 ref 可混用
```

## 7. 组合场景模式

```ts
// 防抖搜索
import { useDebounce } from './useDebounce'
watch(useDebounce(query, 300), (q) => doSearch(q))

// 派生状态优先于 watch 同步
// ❌ watch(a, v => b.value = v * 2)
const b = computed(() => a.value * 2)   // ✅
```

原则：**能 computed 就不要 watch**；watch 只做副作用（请求、DOM、存储、路由跳转）。

## 8. watchEffect vs watch 速判

| 特征 | watch | watchEffect |
|---|---|---|
| 旧值 | ✅ | ❌ |
| 显式依赖 | ✅ | 自动收集 |
| 首次执行 | immediate 控制 | 默认执行 |
| 复杂异步流控制 | 强 | 弱 |
| 适用 | 精确联动 | 一揽子副作用 |

## 常见坑

1. **watch reactive 属性忘记用 getter**：`watch(state.x, …)` 毫无响应。
2. **await 后的代码不追踪依赖**（watchEffect），首次执行后永远不再触发。
3. **computed 中写状态**：导致无限循环或依赖污染。
4. **组件卸载未清理异步回调**：闭包里 set 了已卸载组件的 ref——用 onCleanup / AbortController。
5. **大对象 deep watch**：每次变化全量遍历，性能黑洞，改用精确 getter。
6. **flush: 'sync' 滥用**：一帧内多次触发回调。

## Checklist

- [ ] 派生值一律 computed，不用 watch 手动同步
- [ ] watch reactive 属性统一 `() => state.x` 写法
- [ ] 异步 watch 有竞态保护（seq 或 cleanup）
- [ ] watchEffect 注意 await 边界
- [ ] 无 deep watch 大对象
