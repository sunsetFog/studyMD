# 14 - 响应式进阶 API

## 1. API 总览

| API | 作用 | 典型场景 |
|---|---|---|
| `toRef` / `toRefs` | reactive 属性 ↔ ref | 解构保响应 |
| `toValue` | 归一化 ref/getter/值 | composable 参数 |
| `shallowRef` | 只追踪 .value 替换 | 大对象/组件引用 |
| `shallowReactive` | 只追踪第一层 | 性能敏感浅状态 |
| `triggerRef` | 手动触发 shallowRef 更新 | 深改后手动通知 |
| `customRef` | 自定义依赖追踪 | 防抖 ref |
| `markRaw` | 标记永不做代理 | 第三方实例 |
| `effectScope` | 收集并统一停止副作用 | composable 作用域 |
| `unref` | `isRef ? v.value : v` | 工具函数 |

## 2. shallowRef 与性能

```ts
const state = shallowRef({ list: [...] })
state.value.list.push(1)      // ❌ 不触发更新（深层变化不追踪）
state.value = { ...state.value, list: [...state.value.list, 1] }  // ✅ 整体替换
triggerRef(state)             // ✅ 或手动触发
```

适用：**大数据表格、图表配置、接口返回的大 JSON、组件/实例引用**。深层响应式对大对象的开销（递归代理 + 深度 watch）不可忽视。

## 3. markRaw：跳过代理

```ts
import { markRaw, shallowRef } from 'vue'

const chart = shallowRef()
onMounted(() => {
  chart.value = markRaw(new ECharts(el))   // echarts 实例禁止代理
})
```

必须 markRaw 的对象：第三方库实例（echarts/editor/map）、复杂 class 实例、大型二进制（Blob 等）。不标记会导致：性能损耗、库内部逻辑异常（Proxy 陷阱）、dev 警告。

反向：`isReactive()` 判断失败时先想是否 markRaw 过。

## 4. customRef

```ts
import { customRef } from 'vue'

export function useDebouncedRef(value: string, delay = 300) {
  let timer: ReturnType<typeof setTimeout>
  return customRef((track, trigger) => ({
    get() { track(); return value },          // 收集依赖
    set(v) {
      clearTimeout(timer)
      timer = setTimeout(() => { value = v; trigger() }, delay)  // 触发更新
    },
  }))
}
```

用途：防抖 v-model、带日志的 ref、跨 worker 的 ref。日常能用 watch 防抖就用 watch，customRef 是「把策略封装进 ref 本身」的手段。

## 5. effectScope

```ts
import { effectScope } from 'vue'

const scope = effectScope()
scope.run(() => {
  watch(a, cb)
  watchEffect(cb2)
  computed(...)
})
scope.stop()   // 一次性停止所有内部副作用
```

用途：**非组件环境**（Pinia store、工具函数、测试）中创建可整体停止的作用域；组件内不需要手动（组件卸载自动 stop）。

注意：`scope.run` 必须同步；嵌套 scope 随父停止。

## 6. toValue 与 unref

```ts
import { toValue, unref, type MaybeRefOrGetter } from 'vue'

unref(x)       // ref → .value；否则原样
toValue(x)     // ref → .value；getter → 调用；普通值原样（3.3+）
```

工具函数通用化模板：

```ts
function useWidth(el: MaybeRefOrGetter<HTMLElement | undefined>) {
  watchEffect(() => {
    const node = toValue(el)
    if (!node) return
    // …
  })
}
```

## 7. shallowReactive / shallowReadonly

```ts
const s = shallowReactive({ a: { b: 1 } })
s.a = { b: 2 }   // ✅ 触发
s.a.b = 3        // ❌ 不触发

const ro = shallowReadonly(state)   // 浅层只读：常见于 props 实现
```

## 8. 深浅选择决策

```
状态会整体替换 / 超大对象 → shallowRef
第三方实例 / class        → markRaw + shallowRef
普通业务状态              → ref / reactive（深层）
表格行内编辑等细粒度场景  → ref（默认）即可，别过度优化
```

原则：**先默认深层响应式，profiling 出问题再降浅**。

## 9. 响应式系统边界

以下操作**不会**触发更新：

```ts
let n = reactive({ a: 1 })
n = reactive({ a: 2 })       // 重赋值引用
const { a } = n              // 解构
arr.length = 0               // reactive 数组直接改 length？（实际 length 是响应的，但 splice 更稳）
Object.assign(state, obj)    // ✅ 这个可以（属性级写入）
```

## 常见坑

1. **shallowRef 深改后视图不更新**：养成「浅则替换」的肌肉记忆或 triggerRef。
2. **忘记 markRaw 第三方实例**：echarts 图表诡异失效、代理导致的内存膨胀。
3. **effectScope.run 中异步注册 watcher**：run 返回后注册的副作用不归 scope 管。
4. **customRef 的 set 忘了 track/trigger**：get 里不 track 则永不更新。
5. **对 Vue 组件对象做深层代理**：组件定义应为 shallowRef 或 markRaw。

## Checklist

- [ ] 第三方实例一律 markRaw
- [ ] 大数据状态评估 shallowRef
- [ ] composable 参数 MaybeRefOrGetter + toValue
- [ ] 非组件环境的副作用用 effectScope 收口
- [ ] 未在深浅之间过度优化（默认深层）
