# Vue.js 面试题清单（100 题 · 详细答案）

> 覆盖 Vue 2/Vue 3 核心原理、Composition API、Vue Router、Pinia、SSR/Nuxt、工程化等全部高频考点。
> 每道题均附带**详细代码示例**和**面试要点**，建议按模块循序渐进学习。

---

## 目录

| 模块 | 内容 | 题号 |
|------|------|------|
| 一 | Reactivity 响应式原理 | 1 - 6 |
| 二 | Composition API vs Options API | 7 - 11 |
| 三 | 组件通信与设计 | 12 - 17 |
| 四 | 生命周期 | 18 - 21 |
| 五 | 内置指令与自定义指令 | 22 - 26 |
| 六 | 插槽 Slots | 27 - 29 |
| 七 | Computed 与 Watch | 30 - 33 |
| 八 | Vue Router | 34 - 39 |
| 九 | Pinia 状态管理 | 40 - 44 |
| 十 | 组合式函数 Composables | 45 - 48 |
| 十一 | KeepAlive / Teleport / Suspense | 49 - 52 |
| 十二 | Transition 与动画 | 53 - 55 |
| 十三 | TypeScript 与 Vue | 56 - 59 |
| 十四 | 性能优化 | 60 - 64 |
| 十五 | Vue 3 新特性升级指南 | 65 - 69 |
| 十六 | 虚拟 DOM 与 Diff 算法 | 70 - 73 |
| 十七 | SSR 与 Nuxt.js | 74 - 79 |
| 十八 | 测试 | 80 - 83 |
| 十九 | Vite / 构建工具 | 84 - 87 |
| 二十 | 高级模式与实践 | 88 - 93 |
| 二十一 | 安全与权限 | 94 - 96 |
| 二十二 | 综合与趋势 | 97 - 100 |

---

## 一、Reactivity 响应式原理

### 1. Vue 2 和 Vue 3 的响应式原理有什么区别？

**⭐⭐⭐⭐⭐ · 必问**

#### Vue 2：Object.defineProperty

```javascript
// Vue 2 响应式核心 —— 对象属性劫持
function defineReactive(obj, key, val) {
  observe(val)
  const dep = new Dep()
  Object.defineProperty(obj, key, {
    get() {
      if (Dep.target) { dep.depend() }
      return val
    },
    set(newVal) {
      if (newVal === val) return
      val = newVal
      observe(newVal)
      dep.notify()
    }
  })
}
```

**Vue 2 的缺陷：**
1. **无法检测对象属性的添加/删除** — 需用 `Vue.set` / `Vue.delete`
2. **无法检测数组索引赋值** — `arr[0] = 'x'` 不触发更新
3. **无法检测数组 length 变化** — `arr.length = 0` 不触发
4. **递归遍历所有属性** — 初始化性能开销大

```javascript
// Vue 2 中不触发响应式更新的操作
vm.obj.b = 2        // ❌ 新增属性
delete vm.obj.a     // ❌ 删除属性
vm.arr[0] = 99      // ❌ 索引赋值
vm.arr.length = 0   // ❌ length
// 替代方案
Vue.set(vm.obj, 'b', 2)
vm.$set(vm.arr, 0, 99)
```

#### Vue 3：Proxy

```javascript
// Vue 3 响应式核心 —— Proxy
function reactive(target) {
  if (!isObject(target)) return target
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key)
      const result = Reflect.get(target, key, receiver)
      return isObject(result) ? reactive(result) : result // 懒递归
    },
    set(target, key, value, receiver) {
      const oldValue = target[key]
      const result = Reflect.set(target, key, value, receiver)
      if (oldValue !== value) { trigger(target, key) }
      return result
    },
    deleteProperty(target, key) {
      const hadKey = Object.prototype.hasOwnProperty.call(target, key)
      const result = Reflect.deleteProperty(target, key)
      if (hadKey && result) { trigger(target, key) }
      return result
    }
  })
}
```

**Vue 3 Proxy 的优势：**

| 对比维度 | Vue 2 (defineProperty) | Vue 3 (Proxy) |
|----------|------------------------|---------------|
| 对象新增属性 | ❌ 需 Vue.set | ✅ 直接赋值 |
| 对象删除属性 | ❌ 需 Vue.delete | ✅ 直接 delete |
| 数组索引赋值 | ❌ 不触发 | ✅ 触发 |
| 数组 length | ❌ 不触发 | ✅ 触发 |
| Map/Set 类型 | ❌ 不支持 | ✅ 支持 |
| 初始化性能 | 递归遍历所有属性 | 懒代理，按需递归 |

---

### 2. ref 和 reactive 的区别？何时使用哪一个？

**⭐⭐⭐⭐⭐ · 必问**

| 特性 | ref | reactive |
|------|-----|----------|
| 支持类型 | 任意（基本类型 + 对象） | 仅对象类型 |
| 访问方式 | `.value`（模板自动解包） | 直接访问 |
| 解构保持响应式 | ❌ 需 toRefs | ❌ 需 toRefs |
| 替换整个值 | ✅ 可重新赋值 | ❌ 重新赋值丢失响应式 |
| watch 默认 | 不深度监听（对象需 deep） | 默认深度监听 |

```javascript
// ❌ reactive 不能重新赋值整个对象
let state = reactive({ count: 0 })
state = reactive({ count: 1 }) // 原代理丢失！

// ✅ ref 可以重新赋值
const stateRef = ref({ count: 0 })
stateRef.value = { count: 1 }  // 正常工作

// ❌ reactive 解构失去响应式
const { count } = reactive({ count: 0 }) // count 是普通数字

// ✅ 用 toRefs 保持解构响应式
const state = reactive({ count: 0, name: 'vue' })
const { count, name } = toRefs(state)
```

#### 推荐实践

```javascript
// 规则 1：单值用 ref，集合可用 reactive
const isLoading = ref(false)
const items = reactive([])

// 规则 2：API 请求数据用 ref（方便整体替换）
const userData = ref(null)

// 规则 3：表单数据用 reactive
const form = reactive({ username: '', password: '', remember: false })

// 规则 4：组合式函数返回用 ref / toRefs
function useMouse() {
  const x = ref(0), y = ref(0)
  return { x, y }
}
```

---

### 3. 说一下 Vue 3 的 effectScope 是什么？

**⭐⭐⭐**

```javascript
import { effectScope, watch, onScopeDispose } from 'vue'

// 不使用 effectScope —— 需要手动停止每一个
function useFeatureWithoutScope() {
  const stop1 = watch(source1, () => {})
  const stop2 = watchEffect(() => {})
  onUnmounted(() => { stop1(); stop2() }) // 容易遗漏
}

// 使用 effectScope —— 一键停止所有
function useFeatureWithScope() {
  const scope = effectScope()
  scope.run(() => {
    watch(source1, () => {})
    watchEffect(() => {})
  })
  onUnmounted(() => scope.stop()) // 一键停止
}

// 实际场景：可暂停的轮询
function useInterval(fn, interval) {
  let scope = null
  const start = () => {
    scope = effectScope()
    scope.run(() => {
      const timer = setInterval(fn, interval)
      onScopeDispose(() => clearInterval(timer))
    })
  }
  const stop = () => { scope?.stop(); scope = null }
  return { start, stop }
}
```

---

### 4. computed 和 watch/watchEffect 的实现原理？

**⭐⭐⭐⭐**

| 特性 | computed | watch | watchEffect |
|------|----------|-------|-------------|
| 返回值 | 返回计算后的值 | 不返回 | 不返回 |
| 缓存 | ✅ 缓存结果 | ❌ | ❌ |
| 立即执行 | ❌ 惰性求值 | ✅ (immediate) | ✅ |
| 获取旧值 | ❌ | ✅ | ❌ |
| deep 选项 | ❌ | ✅ | ❌ |

```javascript
// computed 核心：脏检查 + 惰性求值
class ComputedRefImpl {
  private _dirty = true
  get value() {
    track(this, 'value')
    if (this._dirty) {
      this._value = this._effect.run() // 重新计算
      this._dirty = false
    }
    return this._value // 返回缓存
  }
}

// watch 核心：依赖变化后执行回调
// watchEffect 核心：立即执行 + 自动追踪 + 依赖变化自动重跑
```

```javascript
// 场景选择指南
// computed：派生状态
const fullName = computed(() => `${user.first} ${user.last}`)

// watch：需要 oldValue 或特定时机
watch(() => props.id, (newId, oldId) => fetchData(newId))

// watchEffect：自动追踪 + 不需要 oldValue
watchEffect(() => fetchData(props.id, store.filters))
```

---

### 5. shallowRef 和 shallowReactive 的用途？

**⭐⭐⭐**

```javascript
// shallowRef — 只有 .value 替换触发更新
const config = shallowRef(largeConfigObject)
config.value = newConfig  // ✅ 触发

// shallowReactive — 只有第一层属性响应
const state = shallowReactive({ a: 1, nested: { b: 2 } })
state.a = 2          // ✅ 响应
state.nested.b = 10  // ❌ 不响应

// 典型场景：大型只读数据、第三方实例（ECharts/地图）、不可变数据
const chartInstance = shallowRef(null)
chartInstance.value = echarts.init(container) // ✅ 不会被深层代理
```

---

### 6. toRef 和 toRefs 的作用？

**⭐⭐⭐**

```javascript
// toRef — 为响应式对象的单个属性创建 ref（保持连接）
const nameRef = toRef(state, 'name')
nameRef.value = '李四'
console.log(state.name) // '李四' — 双向同步

// toRefs — 将整个响应式对象转为普通对象，属性全部是 ref
const { name, age } = toRefs(reactive({ name: '张三', age: 25 }))
name.value = '新名字'  // 同步回原对象

// 核心用途：解构 reactive 对象保持响应式
function useUser() {
  return toRefs(reactive({ name: '张三', age: 25 }))
}
```

---

## 二、Composition API vs Options API

### 7. Composition API 解决了 Options API 的什么问题？

**⭐⭐⭐⭐⭐ · 必问**

#### 问题：同一功能逻辑分散在不同选项中

```javascript
// Options API — 搜索逻辑分散在 data/computed/watch/methods/mounted
export default {
  data() { return { searchQuery: '', searchResults: [], isSearching: false } },
  watch: { searchQuery: { handler(q) { this.performSearch(q) }, immediate: true } },
  methods: { async performSearch(q) { /* ... */ } },
  mounted() { /* 初始化 */ }
}

// Composition API — 搜索逻辑内聚在一个函数中
function useSearch() {
  const searchQuery = ref(''), searchResults = ref([]), isSearching = ref(false)
  watch(searchQuery, async (q) => {
    if (!q) { searchResults.value = []; return }
    isSearching.value = true
    try { searchResults.value = await api.search(q) }
    finally { isSearching.value = false }
  })
  return { searchQuery, searchResults, isSearching }
}
```

#### 对比

| 维度 | Options API | Composition API |
|------|-------------|-----------------|
| 代码组织 | 按选项类型分散 | 按功能逻辑内聚 |
| 逻辑复用 | Mixin（来源不清） | Composable（来源清晰） |
| TypeScript 支持 | 弱 | 强 |
| Tree Shaking | 难以 | 方便 |

**面试要点：** 不要说"Composition API 更好"，而要说"两者各有优势，大型项目 Composition API 更有优势"。

---

### 8. `<script setup>` 语法糖做了什么？

**⭐⭐⭐⭐**

```vue
<script setup>
// 1. 无需 return — 顶层绑定自动暴露给模板
const count = ref(0)

// 2. 编译时宏 — 无需导入
const props = defineProps<{ title: string }>()
const emit = defineEmits<{ update: [v: number] }>()

// 3. 显式暴露
defineExpose({ reset: () => { count.value = 0 } })

// 编译后等价于：
// export default defineComponent({
//   setup(props, { emit }) {
//     const count = ref(0)
//     return { count, reset }
//   }
// })
</script>
```

---

### 9. 如何替代 mixin 实现逻辑复用？

**⭐⭐⭐**

| 维度 | Mixin | Composable |
|------|-------|------------|
| 来源可追溯 | ❌ | ✅ |
| 命名冲突 | ❌ | ✅ 可重命名 |
| 类型安全 | ❌ | ✅ |
| 数据隔离 | ❌ | ✅ 每次创建新作用域 |

---

### 10. 如何在 `<script setup>` 中定义组件 name？

**⭐⭐**

```vue
<script setup>
defineOptions({ name: 'MyComponent', inheritAttrs: false })
</script>
<!-- 或额外 <script> 块 -->
<!-- 或使用 vite-plugin-vue-setup-extend 插件 -->
```

用途：DevTools 调试、keep-alive include/exclude、递归组件、错误追踪。

---

### 11. setup 为什么不能访问 this？

**⭐⭐**

执行时序：`beforeCreate → setup() → created → ... → mounted`
setup 执行时实例未完全创建，同时 Composition API 设计上不依赖 this。

---

## 三、组件通信与设计

### 12. Vue 组件间通信的方式有哪些？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// 1. Props / Emits — 父子（首选）
// 2. v-model / defineModel — 双向绑定
// 3. provide / inject — 祖先-后代跨层级
// 4. 模板 ref + defineExpose — 调用子组件方法
// 5. Pinia / Vuex — 全局状态
// 6. EventBus (mitt) — 任意组件（不推荐，最后手段）
```

#### 选择决策树
```
父子？ → Props ↓ / Emits ↑ / v-model ↔ / ref 调用
祖先-后代？ → provide / inject
全局？ → Pinia Store
跨页面？ → 路由参数 / query / Pinia
最后手段 → EventBus
```

```javascript
// provide/inject 防止子组件修改
provide('count', readonly(count))
provide('increment', () => { count.value++ }) // 提供方法控制修改
```

---

### 13. provide/inject 的响应式如何实现？

**⭐⭐⭐**

```javascript
// provide 传递的是 ref/reactive 本身，inject 拿到同一引用
// 响应式自动生效

// 简化实现：沿原型链查找
function provide(key, value) {
  const instance = getCurrentInstance()
  let provides = instance.parent?.provides
  if (provides === instance.provides) {
    provides = instance.provides = Object.create(provides || null)
  }
  provides[key] = value
}

function inject(key) {
  const instance = getCurrentInstance()
  return instance.parent?.provides[key]
}
// Object.create 原型链 → 跨多层也能取到
```

---

### 14. Fragment、Teleport 和 Suspense 是什么？

**⭐⭐⭐⭐ · 必问**

```vue
<!-- Fragment：多根元素，不产生额外 DOM -->
<template>
  <h1>Title</h1>
  <p>Content</p>
</template>

<!-- Teleport：渲染到指定 DOM 节点 -->
<Teleport to="body">
  <Modal v-if="show" /> <!-- 解决 z-index / overflow 问题 -->
</Teleport>
<Teleport to="body" :disabled="isMobile"> <!-- 条件启用 -->
</Teleport>

<!-- Suspense：异步组件加载状态 -->
<Suspense>
  <template #default><AsyncDashboard /></template>
  <template #fallback><Skeleton /></template>
</Suspense>
```

---

### 15. 如何设计高复用的 Vue 组件？

**⭐⭐⭐⭐**

```vue
<!-- 六大原则 -->
<!-- 1. 单一职责 — 一个组件做一件事 -->
<!-- 2. Props 向下，Events 向上 — 清晰数据流 -->
<!-- 3. 合理默认值 — withDefaults -->
<!-- 4. 透传 $attrs — v-bind="$attrs" -->
<!-- 5. 插槽灵活定制 — 默认/具名/作用域插槽 -->
<!-- 6. v-model 支持 — defineModel -->

<script setup>
const props = withDefaults(defineProps<{
  size?: 'small' | 'medium' | 'large'
  disabled?: boolean
}>(), {
  size: 'medium', disabled: false
})

const modelValue = defineModel()
const visible = defineModel('visible', { default: false })
</script>
```

---

### 16. 父组件如何调用子组件方法？

**⭐⭐⭐⭐**

```vue
<!-- 父组件 -->
<ChildForm ref="childRef" />
<script setup>
const childRef = ref()
await childRef.value?.validate() // 调用子组件方法
</script>

<!-- 子组件 — 必须 defineExpose -->
<script setup>
function validate() { /* ... */ }
defineExpose({ validate }) // ⚠️ 不暴露则无法访问
</script>
```

`<script setup>` 默认封闭，比 Options API（默认全部暴露）更安全。

---

### 17. 跨组件事件总线替代方案？

**⭐⭐**

```javascript
// 方案 1：provide/inject 传事件函数
// 方案 2：mitt — 轻量事件库
import mitt from 'mitt'
export const emitter = mitt()
emitter.emit('refresh', data)
emitter.on('refresh', handler)
onUnmounted(() => emitter.off('refresh', handler))

// 方案 3：Pinia Store（推荐 — 状态驱动）
const useEventStore = defineStore('events', () => {
  const refreshFlag = ref(0)
  const triggerRefresh = () => refreshFlag.value++
  return { refreshFlag, triggerRefresh }
})

// 方案 4：浏览器 CustomEvent
window.dispatchEvent(new CustomEvent('app:refresh', { detail: data }))
```

---

## 四、生命周期

### 18. Vue 3 生命周期与 Vue 2 对比

**⭐⭐⭐⭐**

| Vue 2 | Vue 3 (Options) | Vue 3 (Composition) |
|-------|-----------------|---------------------|
| beforeCreate | beforeCreate | setup() 直接代替 |
| created | created | setup() 直接代替 |
| beforeMount | beforeMount | onBeforeMount |
| mounted | mounted | onMounted |
| beforeUpdate | beforeUpdate | onBeforeUpdate |
| updated | updated | onUpdated |
| beforeDestroy | beforeUnmount | onBeforeUnmount |
| destroyed | unmounted | onUnmounted |
| - | - | onRenderTracked / onRenderTriggered（新增调试） |

```javascript
// 父子组件生命周期顺序
// 父 beforeMount → 子 beforeMount → 子 mounted → 父 mounted
// 父 beforeUpdate → 子 beforeUpdate → 子 updated → 父 updated
// 父 beforeUnmount → 子 beforeUnmount → 子 unmounted → 父 unmounted
```

---

### 19. onMounted 和 nextTick 的区别？

**⭐⭐⭐**

```javascript
// onMounted — 组件初始化完成，DOM 已存在
onMounted(() => {
  new Chart(canvasRef.value) // 初始化第三方库
})

// nextTick — 响应式数据变化后，等待 DOM 更新
count.value++
await nextTick()
console.log(myRef.value?.textContent) // 最新值

// 典型场景：数据变化后操作 DOM、v-if 渲染后执行
watch(showChild, async (val) => {
  if (val) {
    await nextTick()
    childRef.value?.focus()
  }
})
```

---

### 20. keep-alive 生命周期 onActivated / onDeactivated

**⭐⭐⭐**

```javascript
// 首次创建 → onMounted → onActivated
// 再次进入 → onActivated（不触发 onMounted）
// 离开     → onDeactivated

onActivated(() => {
  startPolling()  // 恢复数据轮询
  refreshData()   // 刷新过期数据
})

onDeactivated(() => {
  stopPolling()   // 暂停轮询
  saveScrollPosition()
})
```

---

### 21. onBeforeUnmount 常见清理操作

**⭐⭐**

```javascript
onBeforeUnmount(() => {
  clearInterval(timer)                                 // 定时器
  window.removeEventListener('resize', handler)        // 事件
  abortController.abort()                              // 请求
  ws.close()                                           // WebSocket
  chartInstance?.dispose()                             // 第三方实例
  observer.disconnect()                                // IntersectionObserver
  emitter.off('event', handler)                        // EventBus
})
```

---

## 五、内置指令与自定义指令

### 22. v-if 和 v-show 的区别？v-if 和 v-for 优先级？

**⭐⭐⭐⭐**

| 特性 | v-if | v-show |
|------|------|--------|
| 渲染 | 条件为真才渲染 | 始终渲染 |
| 切换开销 | 销毁/创建 DOM | 切换 display |
| 初始开销 | 条件为假时为 0 | 始终渲染 |
| 适用场景 | 不频繁切换 | 频繁切换 |

```javascript
// ❌ v-if 和 v-for 不要放同一元素上
// ✅ 用 computed 先过滤
const activeItems = computed(() => items.filter(item => item.active))
<div v-for="item in activeItems" :key="item.id">{{ item.name }}</div>
```

---

### 23. 自定义指令有哪些钩子？

**⭐⭐⭐**

```javascript
const myDirective = {
  created(el, binding, vnode) {},
  beforeMount(el, binding, vnode) {},
  mounted(el, binding, vnode) {},    // 常用：操作 DOM
  beforeUpdate(el, binding, vnode) {},
  updated(el, binding, vnode) {},
  beforeUnmount(el, binding, vnode) {},
  unmounted(el, binding, vnode) {}   // 常用：清理资源
}

// 实战：v-permission 权限指令
const vPermission = {
  mounted(el, binding) {
    if (!hasPermission(binding.value)) {
      el.parentNode?.removeChild(el)
    }
  }
}

// 函数简写（仅 mounted + updated）
app.directive('highlight', (el, binding) => {
  el.style.backgroundColor = binding.value
})
```

---

### 24. v-model 的实现原理？

**⭐⭐⭐⭐**

```javascript
// <input v-model="text" />
// 等价于：
// <input :value="text" @input="text = $event.target.value" />

// 组件 v-model 本质：props + emit
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [v: string] }>()
emit('update:modelValue', newValue)

// Vue 3.4+ 更简洁：defineModel
const modelValue = defineModel<string>()
modelValue.value = 'new' // 直接赋值
```

---

### 25. v-once 和 v-memo 的使用场景？

**⭐⭐⭐**

```vue
<!-- v-once：只渲染一次，之后永不更新 -->
<div v-once>{{ serverData }}</div>
<li v-for="item in hugeStaticList" :key="item.id" v-once>

<!-- v-memo：依赖不变时跳过渲染 -->
<div v-for="item in list" :key="item.id" v-memo="[item.id === selectedId]">
  <!-- 只有 selectedId 相关的 item 才重渲染 -->
</div>
```

---

### 26. v-cloak、v-pre、动态参数

**⭐⭐**

```vue
<!-- v-cloak：隐藏未编译模板，防闪烁 -->
<style>[v-cloak] { display: none }</style>
<div v-cloak>{{ message }}</div>

<!-- v-pre：跳过编译 -->
<div v-pre>{{ 原样输出 }}</div>

<!-- 动态参数 -->
<div :[attrName]="value" />
<button @[eventName]="handler" />
```

---

## 六、插槽 Slots

### 27. 插槽类型及作用域插槽原理

**⭐⭐⭐⭐**

```vue
<!-- 1. 默认插槽 -->    <slot>默认内容</slot>
<!-- 2. 具名插槽 -->    <slot name="header" />
<!-- 3. 作用域插槽 -->  <slot name="item" :item="item" :index="index" />

<!-- 作用域插槽原理：slot 是函数 -->
<!-- 子组件：this.$slots.item({ item, index }) -->
<!-- 父组件编译为 { item: (scope) => h('div', scope.item.name) } -->
```

---

### 28. $slots 和 $attrs

**⭐⭐⭐**

```javascript
// Composition
const slots = useSlots()
const hasHeader = computed(() => !!slots.header)

const attrs = useAttrs()
// attrs 包含未被 props 声明的所有 attribute
// <input v-bind="attrs" /> 手动透传
```

---

### 29. 用插槽实现高复用表格组件

**⭐⭐⭐**

```vue
<SmartTable :columns="columns" :data="items" selectable>
  <template #cell-price="{ value }">¥{{ value.toFixed(2) }}</template>
  <template #cell-status="{ value }"><Tag :type="value" /></template>
  <template #actions="{ row }">
    <button @click="edit(row)">编辑</button>
    <button @click="remove(row.id)">删除</button>
  </template>
  <template #empty><EmptyState /></template>
</SmartTable>
```

---

## 七、Computed 与 Watch

### 30. computed 的缓存机制

**⭐⭐⭐⭐**

```javascript
// computed 缓存原理：脏检查 + 惰性求值
// 1. 内部维护 _dirty 标记
// 2. 依赖不变 → _dirty = false → 返回缓存
// 3. 依赖变化 → _dirty = true → 下次访问时重算
// 4. 没有被访问 → 即使依赖变了也不计算

const fullName = computed(() => {
  console.log('执行了') // 只在依赖变化且被访问时打印
  return firstName.value + lastName.value
})
fullName.value // "执行了" → "张三"
fullName.value // 直接返回缓存 "张三"
```

---

### 31. watch 的 deep、immediate、flush

**⭐⭐⭐⭐**

```javascript
// deep — 深度监听
watch(() => state.user, callback, { deep: true })
// 注意：deep: true 时 oldVal === newVal（同一代理对象）
// 注意：reactive 对象默认 deep: true，ref 对象需显式设置

// immediate — 立即执行
watch(source, callback, { immediate: true })
// 首次 oldVal 为 undefined

// flush
// 'pre'（默认）— 渲染前执行
// 'post' — 渲染后执行（可访问更新后 DOM）
// 'sync' — 同步执行（慎用）
watch(source, callback, { flush: 'post' })

// once (Vue 3.4+) — 只监听一次
watch(source, callback, { once: true })
```

---

### 32. watchEffect vs watch

**⭐⭐⭐**

```javascript
// watchEffect — 自动追踪，立即执行，不需要 oldValue
watchEffect(() => {
  localStorage.setItem('draft', JSON.stringify(formState))
})

// watch — 显式指定依赖，可获取 oldValue
watch(() => state.a + state.b, (sum, oldSum) => {})

// 选择：
// 需要 oldValue？→ watch
// 需要懒执行？→ watch
// 自动追踪更简洁？→ watchEffect
```

---

### 33. 多个依赖合并 / 手动停止

**⭐⭐**

```javascript
// 合并多个依赖
watch([() => props.id, () => state.page], ([id, page], [oldId, oldPage]) => {})

// 手动停止
const stop = watch(source, (val) => { if (val > 10) stop() })

// 竞态处理 (Vue 3.5+)
watch(userId, async (newId, oldId, onCleanup) => {
  let cancelled = false
  onCleanup(() => { cancelled = true })
  const data = await fetch(`/api/user/${newId}`)
  if (!cancelled) userData.value = data
})
```

---

## 八、Vue Router

### 34. hash 模式和 history 模式区别

**⭐⭐⭐⭐⭐ · 必问**

| 特性 | hash | history |
|------|------|---------|
| URL 形式 | `/#/user` | `/user` |
| 美观 | ❌ 有 # | ✅ 干净 |
| SEO | ❌ 不友好 | ✅ 配合 SSR |
| 服务器配置 | 不需要 | 需要 try_files |
| 原理 | hashchange 事件 | History API (pushState) |
| 锚点 | ❌ # 被占用 | ✅ 可使用 |

```nginx
# Nginx history 模式配置
location / {
  try_files $uri $uri/ /index.html;
}
```

---

### 35. 路由守卫及执行顺序

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// 完整导航解析流程：
// 1. 导航触发
// 2. 失活组件 beforeRouteLeave
// 3. 全局 beforeEach
// 4. 重用组件 beforeRouteUpdate
// 5. 路由独享 beforeEnter
// 6. 解析异步路由组件
// 7. 激活组件 beforeRouteEnter
// 8. 全局 beforeResolve
// 9. 导航确认
// 10. 全局 afterEach
// 11. DOM 更新
// 12. beforeRouteEnter next 回调

// 全局守卫
router.beforeEach((to, from, next) => {
  if (to.meta.requiresAuth && !isLoggedIn) next('/login')
  else next()
})
router.afterEach((to) => { document.title = to.meta.title })

// 路由独享
{ path: '/admin', beforeEnter: (to, from, next) => {} }

// 组件内（Composition API）
onBeforeRouteUpdate((to, from) => { fetchData(to.params.id) })
onBeforeRouteLeave((to, from) => {
  if (hasUnsavedChanges.value) return false // 阻止离开
})
```

---

### 36. 动态路由和路由懒加载

**⭐⭐⭐⭐**

```javascript
// 动态路由 — 根据权限添加
const router = createRouter({ routes: staticRoutes })
async function addDynamicRoutes() {
  const menus = await api.getUserMenus()
  const routes = generateRoutes(menus)
  routes.forEach(r => router.addRoute(r))
  router.addRoute({ path: '/:pathMatch(.*)*', redirect: '/404' }) // 兜底
}

router.beforeEach(async (to, from, next) => {
  if (isLoggedIn && !hasRoutes) {
    await addDynamicRoutes()
    hasRoutes = true
    return next({ ...to, replace: true }) // 必须重新导航
  }
  next()
})

// 路由懒加载
const UserList = () => import('@/views/UserList.vue')
// webpack: /* webpackChunkName: "user" */
```

---

### 37. router.push vs replace / 参数传递

**⭐⭐⭐**

```javascript
// push：添加历史记录（可后退）
router.push({ name: 'User', params: { id: 1 } })
router.push({ path: '/search', query: { keyword: 'vue' } })

// replace：替换当前记录（不能后退，常用于登录后跳转）
router.replace({ name: 'Dashboard' })

// go / back / forward
router.go(-1); router.back(); router.forward()

// params 必须配 name，query 可配 path 或 name
// router.push({ path: '/user', params: { id: 1 } }) // ❌ params 被忽略
```

---

### 38. 路由懒加载原理及代码分割

**⭐⭐⭐**

```javascript
// import() 返回 Promise，路由匹配时才执行
// 编译产物：index.js（主包）+ UserList.hash.js（chunk）+ ...

// Vite 分包策略
// vite.config.ts
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        'vue-vendor': ['vue', 'vue-router', 'pinia'],
        'ui-lib': ['element-plus']
      }
    }
  }
}
```

---

### 39. 路由 meta 实际应用

**⭐⭐⭐**

```javascript
const routes = [{
  path: '/dashboard',
  meta: {
    title: '仪表盘',          // 页面标题
    requiresAuth: true,       // 需要登录
    permissions: ['admin'],   // 需要权限
    keepAlive: true,          // 是否缓存
    hidden: false,            // 菜单中隐藏
    sort: 1,                  // 菜单排序
    icon: 'dashboard',        // 图标
    breadcrumb: ['首页', '仪表盘']  // 面包屑
  }
}]
```

---

## 九、Pinia 状态管理

### 40. Pinia 和 Vuex 的区别

**⭐⭐⭐⭐⭐ · 必问**

| 特性 | Vuex 4 | Pinia |
|------|--------|-------|
| mutations | ✅ 必须 | ❌ 没有，直接修改 |
| 模块嵌套 | ✅ 嵌套 + 命名空间 | ❌ 扁平 store（可组合） |
| TypeScript | 需额外类型声明 | 原生完美支持 |
| 代码量 | 多（4 个部分） | 少（3 个部分） |
| 体积 | ~10KB | ~5KB |

```javascript
// Vuex 4 — 啰嗦
this.$store.commit('user/SET_NAME', '张三')
this.$store.dispatch('user/login', credentials)

// Pinia — 简洁
const store = useUserStore()
store.name = '张三'       // 直接修改！
await store.login(credentials)
```

---

### 41. Options Store vs Setup Store

**⭐⭐⭐**

```javascript
// Options Store（类似 Vuex 风格）
export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0 }),
  getters: { double: (s) => s.count * 2 },
  actions: { increment() { this.count++ } }
})

// Setup Store（Composition API 风格，推荐）
export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const double = computed(() => count.value * 2)
  function increment() { count.value++ }
  return { count, double, increment }
})
// Setup Store 优势：可用的组合式函数、私有变量、watch
```

---

### 42. $subscribe / $onAction / $patch

**⭐⭐⭐**

```javascript
// $subscribe — 监听 state 变化
store.$subscribe((mutation, state) => {
  localStorage.setItem('store', JSON.stringify(state))
}, { detached: true })

// $onAction — 监听 action 调用
store.$onAction(({ name, after, onError }) => {
  after(() => ElMessage.success(`${name} 成功`))
  onError((e) => ElMessage.error(e.message))
})

// $patch — 批量更新
store.$patch({ name: '新名', age: 30 })           // 对象形式
store.$patch((state) => { state.name = '新名' })  // 函数形式（推荐）

// $reset — Options Store 支持
store.$reset()
```

---

### 43. store 间互相引用

**⭐⭐**

```javascript
// 在函数内部调用，避免循环依赖
async function login(credentials) {
  user.value = await api.login(credentials)
  const cartStore = useCartStore() // 函数内调用
  await cartStore.fetchCart()
}
```

---

### 44. Pinia 插件开发

**⭐⭐**

```javascript
// 持久化插件
function createPersistedState({ key = 'pinia', storage = localStorage } = {}) {
  return ({ store }) => {
    const saved = storage.getItem(`${key}-${store.$id}`)
    if (saved) store.$patch(JSON.parse(saved))
    store.$subscribe((_, state) => {
      storage.setItem(`${key}-${store.$id}`, JSON.stringify(state))
    }, { detached: true })
  }
}

pinia.use(createPersistedState())
// 推荐三方插件：pinia-plugin-persistedstate
```

---

## 十、组合式函数 Composables

### 45. 好的 Composable 设计原则

**⭐⭐⭐⭐**

```javascript
// 1. 单一职责 — 每个 composable 只做一件事
// 2. 明确输入 — 用 MaybeRefOrGetter 兼容 ref/getter/普通值
import { toValue } from 'vue'

function useFetch(url: MaybeRefOrGetter<string>) {
  const data = ref(null)
  watchEffect(() => {
    const resolvedUrl = toValue(url) // 统一处理三种输入
    fetch(resolvedUrl).then(r => r.json()).then(d => data.value = d)
  })
  return { data }
}

// 3. 返回 ref，可安全解构
// 4. 副作用自动清理
// 5. 提供语义化的生命周期方法
```

---

### 46. 封装带防抖的搜索 Composable

**⭐⭐⭐**

```javascript
export function useSearch<T>(options: {
  debounceMs?: number
  minLength?: number
  onSearch: (query: string) => Promise<T[]>
}) {
  const { debounceMs = 300, minLength = 1, onSearch } = options
  const query = ref('')
  const results = ref<T[]>([])
  const loading = ref(false)
  let timer: ReturnType<typeof setTimeout> | null = null
  let abortController: AbortController | null = null

  const debouncedSearch = () => {
    if (timer) clearTimeout(timer)
    abortController?.abort()
    if (query.value.length < minLength) { results.value = []; return }
    timer = setTimeout(async () => {
      abortController = new AbortController()
      loading.value = true
      try { results.value = await onSearch(query.value) }
      catch (e: any) { if (e.name !== 'AbortError') throw e }
      finally { loading.value = false }
    }, debounceMs)
  }

  watch(query, debouncedSearch)
  onBeforeUnmount(() => { if (timer) clearTimeout(timer); abortController?.abort() })

  return { query, results, loading }
}
```

---

### 47. effectScope 管理副作用

**⭐⭐⭐**

```javascript
// 可暂停的轮询 composable
export function usePolling(fn: () => Promise<void>, interval: number) {
  const isActive = ref(false)
  let scope = null

  const start = () => {
    if (isActive.value) return
    isActive.value = true
    scope = effectScope()
    scope.run(() => {
      const timer = setInterval(fn, interval)
      fn() // 立即执行一次
      onScopeDispose(() => clearInterval(timer))
    })
  }

  const stop = () => { isActive.value = false; scope?.stop(); scope = null }
  onBeforeUnmount(stop)
  return { isActive, start, stop }
}
```

---

### 48. VueUse 常用 API

**⭐⭐⭐**

```javascript
// useStorage — localStorage 响应式绑定
const token = useStorage('auth-token', '')

// useEventListener — 自动清理的事件监听
useEventListener(window, 'resize', () => {})

// useDebounceFn / useThrottleFn
const search = useDebounceFn((q) => api.search(q), 500)

// useInfiniteScroll — 无限滚动
useInfiniteScroll(el, loadMore, { distance: 100 })

// useTitle — 页面标题
const title = useTitle('默认')

// useClipboard — 剪贴板
const { copy, copied } = useClipboard()

// useOnline — 网络状态
const online = useOnline()

// useIntersectionObserver — 元素可见性
useIntersectionObserver(target, ([{ isIntersecting }]) => {})

// useVModel — 简化 v-model
const value = useVModel(props, 'modelValue', emit)
```


---

## 十一、KeepAlive / Teleport / Suspense

### 49. KeepAlive 缓存机制

**⭐⭐⭐⭐**

```javascript
// 核心原理：
// 1. 内部维护 cache Map（key → VNode）
// 2. 根据 include/exclude/max 控制
// 3. 切换时 VNode 缓存，DOM 移出文档流但保留在内存
// 4. 切回时从 cache 取出，重新插入 DOM
// 5. 达到 max 时 LRU 淘汰最早缓存

const KeepAliveImpl = {
  setup(props, { slots }) {
    const cache = new Map()
    const keys = new Set()

    return () => {
      const vnode = slots.default()[0]
      const key = vnode.key ?? vnode.type.name

      // include/exclude 过滤
      if (props.include && !props.include.includes(key)) return vnode
      if (props.exclude && props.exclude.includes(key)) return vnode

      if (cache.has(key)) {
        // 命中缓存 — 复用组件实例
        vnode.component = cache.get(key).component
        keys.delete(key); keys.add(key) // 更新 LRU
      } else {
        keys.add(key)
        // LRU 淘汰
        if (props.max && keys.size > props.max) {
          const first = keys.values().next().value
          keys.delete(first); cache.delete(first)
        }
      }
      return vnode
    }
  }
}
```

**缓存生命周期：**

| 阶段 | 操作 |
|------|------|
| 首次进入 | 创建实例 → mounted → activated → 渲染 DOM |
| 离开 | deactivated → DOM 移出 → VNode 存缓存 |
| 再次进入 | 取缓存 VNode → activated → DOM 重新插入 |
| 达到 max | LRU 淘汰 → 调用 unmounted |

---

### 50. Teleport 解决的问题

**⭐⭐⭐**

```vue
<!-- Teleport 解决三个问题： -->
<!-- 1. z-index 层级：弹窗被 overflow:hidden 裁剪 -->
<!-- 2. 定位上下文：fixed 相对于最近的 transform 祖先 -->
<!-- 3. 语义混乱：弹窗逻辑在组件内但视觉上独立 -->

<!-- 基础用法 -->
<Teleport to="body">
  <Modal v-if="show" @close="show = false" />
</Teleport>

<!-- 条件禁用 -->
<Teleport to="body" :disabled="isMobile">
  <Dropdown :items="menuItems" />
</Teleport>

<!-- 同一目标多次传送（按顺序追加） -->
<Teleport to="#notifications"><Notification :msg="msg1" /></Teleport>
<Teleport to="#notifications"><Notification :msg="msg2" /></Teleport>

<!-- 典型应用：Modal/Dialog、Toast、全屏 Loading、下拉菜单 -->
```

---

### 51. Suspense 异步处理及注意事项

**⭐⭐⭐**

```vue
<Suspense @pending="onPending" @resolve="onResolve" @fallback="onFallback">
  <template #default>
    <AsyncDashboard /> <!-- 组件内有 async setup 或顶层 await -->
  </template>
  <template #fallback>
    <SkeletonLoader />
  </template>
</Suspense>

<!-- AsyncDashboard.vue -->
<script setup>
// 顶层 await — Suspense 等待完成后才渲染 #default
const data = await fetch('/api/dashboard')
</script>
```

注意事项：
- ⚠️ 目前仍是实验性功能
- 错误处理需配合 onErrorCaptured 或 ErrorBoundary
- 嵌套 Suspense：父 pending → 子 pending → 子 resolve → 父 resolve
- SSR 水合：服务端渲染 fallback，客户端替换为 #default

---

### 52. 自己实现简单的 KeepAlive

**⭐⭐⭐**

核心思路：维护 cache Map → 首次缓存 VNode → 离开保留实例 → 激活时复用。
注意需要处理 activated/deactivated 调用和 LRU 淘汰逻辑。

---

## 十二、Transition 与动画

### 53. Transition 和 TransitionGroup 区别

**⭐⭐⭐**

| 特性 | Transition | TransitionGroup |
|------|-----------|-----------------|
| 适用 | 单个元素/组件 | 列表元素 |
| 渲染标签 | 不渲染额外元素 | 默认 span，tag 指定 |
| :key | 不需要 | 必须 |
| v-move 类 | ❌ | ✅ |
| mode 属性 | ✅ out-in/in-out | ❌ |

```vue
<!-- Transition -->
<Transition name="fade">
  <p v-if="show">Hello</p>
</Transition>

<!-- TransitionGroup -->
<TransitionGroup name="list" tag="ul">
  <li v-for="item in items" :key="item.id">{{ item.name }}</li>
</TransitionGroup>

<style>
/* v-move — TransitionGroup 专有（元素移动动画） */
.list-move { transition: transform 0.5s ease; }
.list-leave-active { position: absolute; } /* 离开元素脱离文档流 */
</style>
```

---

### 54. Transition JS 钩子 + 第三方动画库

**⭐⭐**

```vue
<Transition
  @before-enter="onBeforeEnter"
  @enter="onEnter"
  @after-enter="onAfterEnter"
  @before-leave="onBeforeLeave"
  @leave="onLeave"
  @after-leave="onAfterLeave"
>
  <div v-if="show">Content</div>
</Transition>

<script setup>
import gsap from 'gsap'

function onEnter(el, done) {
  gsap.to(el, { opacity: 1, scale: 1, duration: 0.6, onComplete: done })
}
// ⚠️ JS 钩子必须调用 done() 通知 Vue 动画完成

// CSS animation 库（animate.css）
// enter-active-class="animate__animated animate__bounceIn"
// leave-active-class="animate__animated animate__bounceOut"
</script>
```

---

### 55. 页面切换动画

**⭐⭐**

```vue
<RouterView v-slot="{ Component, route }">
  <Transition name="page" mode="out-in">
    <component :is="Component" :key="route.path" />
  </Transition>
</RouterView>

<style>
.page-enter-active, .page-leave-active { transition: all 0.3s ease; }
.page-enter-from { opacity: 0; transform: translateX(30px); }
.page-leave-to { opacity: 0; transform: translateX(-30px); }
</style>
```

---

## 十三、TypeScript 与 Vue

### 56. Vue 3 中如何用好 TypeScript？

**⭐⭐⭐⭐**

```vue
<script setup lang="ts">
// 1. Props 类型声明
interface Props {
  title: string
  count?: number
  items: { id: number; name: string }[]
  status: 'active' | 'inactive' | 'pending'
}
const props = withDefaults(defineProps<Props>(), {
  count: 0,
  items: () => []
})

// 2. Emits 类型
const emit = defineEmits<{
  'update:modelValue': [value: string]
  submit: [data: FormData]
  delete: [id: number]
}>()

// 3. 模板 ref 类型
const inputRef = ref<HTMLInputElement>()
const childRef = ref<InstanceType<typeof ChildComponent>>()

// 4. provide/inject 类型安全
import type { InjectionKey, Ref } from 'vue'
interface User { name: string; role: string }
export const USER_KEY: InjectionKey<Ref<User>> = Symbol('user')
provide(USER_KEY, userRef)
const user = inject(USER_KEY) // 类型自动推断，不需要泛型

// 5. 组合式函数返回类型
function useCounter(initial = 0) {
  const count = ref(initial)
  const increment = () => count.value++
  return { count, increment } as const // as const 保持字面量类型
}

// 6. 事件处理
function handleInput(e: Event) {
  const target = e.target as HTMLInputElement
  console.log(target.value)
}

// 7. defineModel 类型 (Vue 3.4+)
const modelValue = defineModel<string>({ required: true })
const visible = defineModel<boolean>('visible', { default: false })
</script>
```

---

### 57. Vue 3 的泛型组件怎么写？

**⭐⭐⭐**

```vue
<!-- GenericList.vue -->
<script setup lang="ts" generic="T extends { id: string | number }">
const props = defineProps<{
  items: T[]
  columns: { key: keyof T; title: string }[]
}>()

const emit = defineEmits<{
  select: [item: T]
}>()
</script>

<template>
  <table>
    <tr v-for="item in items" :key="item.id" @click="emit('select', item)">
      <td v-for="col in columns" :key="String(col.key)">
        {{ item[col.key] }}
      </td>
    </tr>
  </table>
</template>

<!-- 使用 — 自动推断 T 类型 -->
<GenericList
  :items="users"
  :columns="[{ key: 'name', title: '姓名' }, { key: 'age', title: '年龄' }]"
  @select="(user) => { /* user 类型为 User */ }"
/>
```

---

### 58. 如何给 Vue 3 全局属性添加类型？

**⭐⭐**

```typescript
// 全局属性：app.config.globalProperties.$http = axios

// 1. 声明类型扩展
// types/global.d.ts
import type { AxiosInstance } from 'axios'

declare module 'vue' {
  interface ComponentCustomProperties {
    $http: AxiosInstance
    $formatDate: (date: Date, format?: string) => string
  }
}

// 2. 在组合式 API 中使用
import { getCurrentInstance } from 'vue'

export function useHttp() {
  const instance = getCurrentInstance()
  return instance?.appContext.config.globalProperties.$http!
}

// 3. 更推荐的方式：直接导出工具函数
// utils/http.ts
export const http = axios.create({ baseURL: '/api' })
// 在 composable 或 store 中直接 import
```

---

### 59. 常见 TS 错误及解决

**⭐⭐**

```typescript
// 错误 1：ref 值作为 prop 传递类型不匹配
// ❌ <Child :count="count" />  // count 是 Ref<number>
// ✅ 模板中 ref 自动解包，实际传递的是 number

// 错误 2：reactive 对象直接赋值
const state = reactive<{ name: string }>({ name: 'vue' })
// ❌ state = { name: 'new' }  // 会丢失响应式

// 错误 3：toRefs 后的可选属性
const state = reactive<{ name?: string }>({})
const { name } = toRefs(state)
// name 类型：ToRef<string | undefined>，使用需判空

// 错误 4：defineProps 默认值类型
interface Props { count?: number }
// ❌ withDefaults(defineProps<Props>(), { count: 0 }) // 不对应
// ✅ default value type must match
withDefaults(defineProps<Props>(), { count: 0 })
```

---

## 十四、性能优化

### 60. 大列表渲染优化方案

**⭐⭐⭐⭐**

```vue
<template>
  <!-- 1. 虚拟滚动（核心方案） -->
  <RecycleScroller
    :items="largeList"
    :item-size="50"
    key-field="id"
    v-slot="{ item }"
  >
    <ListItem :data="item" />
  </RecycleScroller>

  <!-- 2. 分页 / 无限滚动 -->
  <!-- 3. v-memo 跳过不变项 -->
  <div v-for="item in list" :key="item.id" v-memo="[item.selected]">
    {{ item.name }}
  </div>

  <!-- 4. 惰性渲染（可视区域内渲染） -->
</template>

<script setup>
// 虚拟滚动库：vue-virtual-scroller, vxe-table, @tanstack/vue-virtual
</script>
```

---

### 61. 组件级别的优化策略

**⭐⭐⭐⭐**

```javascript
// 1. 合理拆分组件 — 局部更新代替全量更新
//    ❌ 大组件：一个数据变了全量 re-render
//    ✅ 细粒度组件：只渲染变化的部分

// 2. v-once — 纯静态内容
<div v-once>{{ staticContent }}</div>

// 3. KeepAlive — 缓存不常变的页面
<KeepAlive :include="['Home', 'List']" :max="5">

// 4. shallowRef / shallowReactive — 大型数据避免深层代理
const bigData = shallowRef(hugeObject)

// 5. computed 代替模板内复杂表达式
const processedList = computed(() => items.filter(/* ... */).map(/* ... */))

// 6. v-if + v-show 组合
//    频繁切换用 v-show，首屏不渲染用 v-if

// 7. 冻结数据 — 不需要响应式的数据用 Object.freeze
const staticOptions = Object.freeze(largeConstantArray)

// 8. 异步组件 + defineAsyncComponent
const HeavyChart = defineAsyncComponent(() => import('./HeavyChart.vue'))
```

---

### 62. Vue 3 编译优化做了哪些事？

**⭐⭐⭐⭐**

```javascript
// 1. PatchFlags（动态标记）— 精准更新
//    只对比有动态绑定的部分，跳过静态内容
//    PatchFlags.TEXT = 1       // 动态文本
//    PatchFlags.CLASS = 2      // 动态 class
//    PatchFlags.STYLE = 4      // 动态 style
//    PatchFlags.PROPS = 8      // 动态属性
//    PatchFlags.FULL_PROPS = 16 // 动态 key

// 编译前模板：
// <div><span>静态</span><span>{{ dynamic }}</span></div>

// 编译后（简化）：
// createBlock('div', null, [
//   createVNode('span', null, '静态'),          // 静态 — 不追踪
//   createVNode('span', null, ctx.dynamic, 1)   // 动态 — PatchFlags=1
// ])

// 2. 静态提升（HoistStatic）
//    不会变化的 VNode 提升到渲染函数外部，只创建一次
// const _hoisted_1 = createVNode('span', null, '静态文本')

// 3. 事件缓存（CacheHandler）
//    内联事件处理函数缓存，避免子组件不必要的更新
//    @click="foo" → onClick: _cache[0] || (_cache[0] = $event => foo($event))

// 4. Block Tree — 只比对动态节点的差异
//    扁平化存储动态节点，O(n) 变 O(k)，k 是动态节点数

// 5. 预字符串化（Static Hoisting 进阶）
//    连续静态节点直接编译为字符串，innerHTML 设置
```

---

### 63. 如何定位和排查 Vue 性能问题？

**⭐⭐⭐**

```javascript
// 1. Vue DevTools Performance 标签
//    查看组件渲染时间、操作次数

// 2. onRenderTracked / onRenderTriggered
onRenderTracked((e) => console.log('依赖追踪', e.deps))
onRenderTriggered((e) => console.log('触发更新', e.key, e.target))

// 3. why-did-you-render 类工具
//    追踪不必要的组件 re-render

// 4. Chrome DevTools Performance
//    录制 → 分析 long task → 定位慢组件

// 5. 关键指标
//    FCP（首次内容绘制）
//    LCP（最大内容绘制）← 关注首屏大组件加载
//    INP（交互到下次绘制）← 关注事件响应速度

// 6. 常见性能问题
//    - 不必要的大对象 reactive
//    - watch deep: true 监听大对象
//    - 模板内复杂计算（应移到 computed）
//    - 未清理的定时器/事件（内存泄漏）
//    - 大列表未使用虚拟滚动
```

---

### 64. 打包优化策略

**⭐⭐⭐**

```javascript
// vite.config.ts
export default defineConfig({
  build: {
    // 1. 代码分割
    rollupOptions: {
      output: {
        manualChunks: {
          vue: ['vue', 'vue-router', 'pinia'],
          ui: ['element-plus'],
          charts: ['echarts']
        }
      }
    },
    // 2. chunk 大小警告阈值
    chunkSizeWarningLimit: 500, // KB
    // 3. 压缩
    minify: 'terser',
    terserOptions: {
      compress: { drop_console: true, drop_debugger: true }
    }
  }
})

// 4. 路由懒加载 — 按需加载
// 5. 图片压缩 + WebP + 懒加载
// 6. Gzip/Brotli 压缩
// 7. CDN 加载第三方库（externals）
// 8. Tree Shaking — 按需引入 UI 库
import { ElButton, ElInput } from 'element-plus'
```

---

## 十五、Vue 3 新特性升级指南

### 65. Vue 2 项目升级到 Vue 3 的注意事项

**⭐⭐⭐⭐**

```javascript
// 1. 全局 API 变化
// Vue 2: Vue.component / Vue.directive / Vue.mixin / Vue.prototype.$xxx
// Vue 3: app.component / app.directive / app.mixin / app.config.globalProperties

// 2. v-model 变化
// Vue 2: v-model = :value + @input（prop 和 event 名固定）
// Vue 3: v-model = :modelValue + @update:modelValue（可自定义）
//        v-model:title = :title + @update:title

// 3. v-if/v-for 优先级变化
// Vue 2: v-for > v-if
// Vue 3: v-if > v-for（不要放同一元素上！）

// 4. $listeners 移除 — 合并到 $attrs
// Vue 2: $attrs 不含 listeners
// Vue 3: $attrs 包含所有（class/style/listeners）

// 5. 过滤器（filters）移除
// Vue 2: {{ date | format }}
// Vue 3: 用 computed 或方法替代

// 6. 事件总线 $on/$off/$once 移除
//     用 mitt 或 Pinia 替代

// 7. key 变化
// Vue 2: v-if 分支使用相同元素时用 key 区分
// Vue 3: v-if v-else 自动生成不同 key，无需手动加

// 8. 渲染函数 API 变化
// Vue 2: h 从 render 参数传入
// Vue 3: import { h } from 'vue'

// 9. 过渡类名变化
// Vue 2: v-enter → Vue 3: v-enter-from
// Vue 2: v-leave → Vue 3: v-leave-from
```

---

### 66. Vue 3 新增的内置组件和 API

**⭐⭐⭐**

```javascript
// 新增组件
// Fragment — 多根元素
// Teleport — 传送门
// Suspense — 异步加载状态

// 新增响应式 API
// ref / reactive / computed / watch / watchEffect — 基础
// readonly — 只读代理
// shallowRef / shallowReactive — 浅层响应
// triggerRef — 手动触发 shallowRef
// toRaw — 获取原始对象
// markRaw — 标记永不代理
// effectScope — 作用域管理
// isRef / isReactive / isReadonly / isProxy — 判断

// 新增组合式 API
// setup / <script setup>
// defineProps / defineEmits / defineExpose / defineOptions / defineModel

// 新增工具函数
// toRef / toRefs — 响应式转换
// toValue (Vue 3.3+) — 统一 ref/getter/普通值
// unref — 等同 val = isRef(val) ? val.value : val

// 新增生命周期
// onRenderTracked / onRenderTriggered — 调试
// onServerPrefetch — SSR
```

---

### 67. Vue 3 的 Tree Shaking 支持

**⭐⭐⭐**

```javascript
// Vue 3 源码层面支持 Tree Shaking
// 1. 使用 ES Module 的 import/export
// 2. 大部分 API 按函数导出，非挂载在对象上

// ✅ Tree Shakable — 不用的不会打包
import { ref, computed, watch, onMounted } from 'vue'

// ❌ Vue 2 中全量引入
// import Vue from 'vue'  // 整个 Vue 都打包了

// 打包结果对比：
// Vue 2 全量：~33KB gzip
// Vue 3 按需：~13KB gzip（基础运行时）

// 额外优化：
// - 将不常用功能拆成独立包（如 @vue/compiler-sfc）
// - import { createRouter } from 'vue-router' 同样 Tree Shakable
```

---

### 68. Vue 3.3 / 3.4 / 3.5 主要新增功能

**⭐⭐⭐**

```javascript
// Vue 3.3
// - defineOptions 宏
// - defineSlots 宏
// - 泛型组件 <script setup generic="T">
// - toRef / toValue 增强
// - defineProps 解构保持响应式（实验性）

// Vue 3.4
// - defineModel 稳定版（v-model 简化）
// - v-bind 同名简写
//   <!-- <div :id :class /> 等价于 <div :id="id" :class="class" /> -->
// - 解析器性能提升（2x 更快）
// - watch 的 once 选项

// Vue 3.5
// - Props 解构响应式（稳定）
//   const { count = 0 } = defineProps<{ count?: number }>()
//   // count 自动是响应式的，不需要 toRef！
// - useTemplateRef 新 API
//   const inputRef = useTemplateRef('input') // 代替 ref 绑定
// - onWatcherCleanup
//   watch(source, (newVal, oldVal, onCleanup) => {
//     onCleanup(() => { /* 清理上次副作用 */ })
//   })
// - 响应式系统内部优化（内存和速度改进）
```

---

### 69. Vue 3 中废弃的功能及替代方案

**⭐⭐**

```
废弃功能                  → 替代方案
─────────────────────────────────────────
filters（过滤器）         → computed / 方法
$on / $off / $once       → mitt / Pinia / provide
Vue.set / Vue.delete     → 直接赋值 / delete
v-on 的 keyCode 修饰符   → 用 e.key（如 e.key === 'Enter'）
$children                → $refs / provide
$listeners               → $attrs（已合并）
.native 修饰符           → emits 中声明
Functional Components    → 普通函数组件
Vue.prototype            → app.config.globalProperties
nextTick 回调形式         → nextTick() 返回 Promise
```

---

## 十六、虚拟 DOM 与 Diff 算法

### 70. 什么是虚拟 DOM？为什么需要它？

**⭐⭐⭐⭐**

```javascript
// 虚拟 DOM 是真实 DOM 的 JavaScript 对象表示
const vnode = {
  type: 'div',           // 标签名
  props: { class: 'container', id: 'app' },
  children: [
    { type: 'h1', props: null, children: 'Hello' },
    { type: 'p', props: null, children: 'World' }
  ],
  el: null,              // 对应真实 DOM（挂载后赋值）
  key: null,
  shapeFlag: 9           // 标记类型（元素/组件/文本等）
}

// 为什么需要虚拟 DOM：
// 1. 跨平台 —— 不直接操作 DOM，可映射到不同平台
// 2. 批量更新 —— 多次数据变化合并为一次 DOM 操作
// 3. Diff 优化 —— 新旧 VNode 对比，最小化 DOM 操作
// 4. 声明式 —— 描述 UI 应该长什么样，而非怎么做

// 注意：虚拟 DOM 不是银弹
// - 对于简单应用，直接操作 DOM 可能更快
// - 优势在于管理复杂 UI 的更新和跨平台能力
```

---

### 71. Vue 3 的 Diff 算法优化

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// Vue 3 Diff 算法核心策略：双端对比 + 最长递增子序列

function patchKeyedChildren(c1, c2, container) {
  let i = 0
  const l2 = c2.length
  let e1 = c1.length - 1 // 旧列表尾索引
  let e2 = l2 - 1        // 新列表尾索引

  // 1. 头头对比（sync from start）
  while (i <= e1 && i <= e2) {
    if (isSameVNodeType(c1[i], c2[i])) {
      patch(c1[i], c2[i], container)
    } else break
    i++
  }

  // 2. 尾尾对比（sync from end）
  while (i <= e1 && i <= e2) {
    if (isSameVNodeType(c1[e1], c2[e2])) {
      patch(c1[e1], c2[e2], container)
    } else break
    e1--; e2--
  }

  // 3. 新增（旧的遍历完，新的还有）
  if (i > e1) {
    while (i <= e2) {
      patch(null, c2[i], container) // 挂载新节点
      i++
    }
  }

  // 4. 删除（新的遍历完，旧的还有）
  else if (i > e2) {
    while (i <= e1) {
      unmount(c1[i]) // 卸载旧节点
      i++
    }
  }

  // 5. 乱序处理 — 最长递增子序列
  else {
    const s1 = i, s2 = i
    const keyToNewIndexMap = new Map()
    for (i = s2; i <= e2; i++) {
      keyToNewIndexMap.set(c2[i].key, i)
    }

    // 新旧索引映射
    const newIndexToOldIndexMap = new Array(e2 - s2 + 1).fill(0)
    for (i = s1; i <= e1; i++) {
      const newIndex = keyToNewIndexMap.get(c1[i].key)
      if (newIndex !== undefined) {
        newIndexToOldIndexMap[newIndex - s2] = i + 1
        patch(c1[i], c2[newIndex], container)
      } else {
        unmount(c1[i]) // key 在新列表中不存在，删除
      }
    }

    // 最长递增子序列 — 确定不需要移动的节点
    const increasing = getSequence(newIndexToOldIndexMap)
    let j = increasing.length - 1
    for (i = e2 - s2; i >= 0; i--) {
      if (newIndexToOldIndexMap[i] === 0) {
        // 新节点 — 挂载
        patch(null, c2[i + s2], container)
      } else if (j < 0 || i !== increasing[j]) {
        // 需要移动
        move(c2[i + s2], container)
      } else {
        j-- // 位置不变
      }
    }
  }
}

// 最长递增子序列（LIS）：找到尽可能多的不需要移动的节点
// 复杂度：O(n log n)
function getSequence(arr) {
  // 返回最长递增子序列的索引数组
  // [2, 3, 1, 5] → [0, 1, 3]（对应值 2, 3, 5）
}
```

#### Vue 2 vs Vue 3 Diff 对比

| 步骤 | Vue 2（双端 Diff） | Vue 3（快速 Diff） |
|------|-------------------|-------------------|
| 预处理 | 四指针（头头/尾尾/头尾/尾头） | 两指针（头头 + 尾尾） |
| 乱序处理 | 逐个 oldKey→newIndex | keyToNewIndexMap + LIS |
| 静态优化 | 无 | PatchFlags 跳过静态 |
| 复杂度 | O(n) | O(n) + 常数优化 |
| 移动次数 | 可能多余 | 最小化（LIS 保证） |

---

### 72. key 的作用是什么？为什么不能用 index 作为 key？

**⭐⭐⭐⭐⭐ · 必问**

```vue
<!-- key 的作用 -->
<!-- 1. Vue 通过 key 判断节点是否可复用（isSameVNodeType） -->
<!-- 2. key 是 VNode 的唯一标识，Diff 的核心依赖 -->
<!-- 3. 没有 key 时 Vue 使用"就地复用"策略 -->

<!-- ❌ 用 index 做 key 的问题 -->
<template>
  <!-- 原始列表 -->
  <li v-for="(item, index) in items" :key="index">{{ item.name }}</li>
  <!-- items = [{id:1,name:'A'}, {id:2,name:'B'}, {id:3,name:'C'}] -->
  <!-- 渲染: key=0 A, key=1 B, key=2 C -->

  <!-- 在头部插入 D -->
  <!-- items = [{id:4,name:'D'}, {id:1,name:'A'}, {id:2,name:'B'}, {id:3,name:'C'}] -->
  <!-- 渲染: key=0 D, key=1 A, key=2 B, key=3 C -->
  <!-- Vue 认为：key=0 从 A 变成 D（更新），key=1 从 B 变成 A（更新）... -->
  <!-- 全部更新一遍！原本可以不更新的节点也被迫更新 -->
</template>

<!-- ✅ 用唯一 id 做 key -->
<li v-for="item in items" :key="item.id">{{ item.name }}</li>
<!-- 插入 D 后只新增 key=4，其余 key=1,2,3 保持不变，复用即可 -->
```

**key 最佳实践：**
- 使用列表数据的唯一标识（如 id）
- 不要用 index（除非列表是静态的、永远不会重新排序/增删）
- v-for 必须绑定 key
- v-if/v-else/v-else-if 分支如果类型相同，需要不同的 key 来区分

---

### 73. 虚拟 DOM 到真实 DOM 的过程（patch 流程）

**⭐⭐⭐**

```javascript
// patch 函数 — 核心入口
function patch(oldVNode, newVNode, container) {
  // 1. 类型不同 → 直接替换
  if (oldVNode.type !== newVNode.type) {
    unmount(oldVNode)
    mount(newVNode, container)
    return
  }

  // 2. 相同类型
  const el = (newVNode.el = oldVNode.el)

  // 2a. 文本节点
  if (newVNode.type === Text) {
    if (newVNode.children !== oldVNode.children) {
      el.textContent = newVNode.children
    }
    return
  }

  // 2b. 普通元素
  // 更新 props（class/style/attrs/events）
  patchProps(el, oldVNode.props, newVNode.props)

  // 更新 children
  patchChildren(oldVNode, newVNode, el)
}

function patchChildren(oldVNode, newVNode, container) {
  const oldChildren = oldVNode.children
  const newChildren = newVNode.children

  if (typeof newChildren === 'string') {
    // 新 children 是文本
    if (Array.isArray(oldChildren)) {
      unmountChildren(oldChildren) // 卸载旧子节点
    }
    if (newChildren !== oldChildren) {
      container.textContent = newChildren
    }
  } else if (Array.isArray(newChildren)) {
    // 新 children 是数组
    if (typeof oldChildren === 'string') {
      container.textContent = ''
      mountChildren(newChildren, container)
    } else {
      // 核心：数组 Diff（见 71 题）
      patchKeyedChildren(oldChildren, newChildren, container)
    }
  }
}
```

---

## 十七、SSR 与 Nuxt.js

### 74. SSR 的原理是什么？与 CSR 相比有什么优缺点？

**⭐⭐⭐⭐⭐ · 必问**

#### SSR 工作流程

```
浏览器请求 /page
  → 服务端执行 Vue，生成完整 HTML 字符串
  → 返回 HTML（首屏内容已包含）
  → 浏览器渲染 HTML（用户看到内容）
  → 加载 JS 文件
  → 客户端 Hydration（水合 — 给静态 HTML 绑定事件和响应式）
  → 应用变为可交互
```

#### CSR 工作流程

```
浏览器请求 /page
  → 返回空 HTML（只有 <div id="app">）
  → 加载 JS 文件
  → JS 执行，构建 DOM
  → 用户看到内容（白屏时间较长）
```

#### 对比

| 维度 | SSR | CSR |
|------|-----|-----|
| 首屏速度 | ✅ 快，HTML 直接含内容 | ❌ 慢，等 JS 加载执行 |
| SEO | ✅ 搜索引擎可抓取内容 | ❌ 抓取空壳 HTML |
| 服务器压力 | ❌ 需服务端渲染，压力大 | ✅ 仅托管静态文件 |
| 开发复杂度 | ❌ 需处理服务端/客户端差异 | ✅ 纯前端逻辑 |
| Time to Interactive | ❌ 水合前不可交互 | 加载完即可交互 |
| 缓存策略 | ✅ 可 CDN 缓存页面 | HTML 不变，缓存简单 |

---

### 75. Nuxt.js 的核心功能和约定

**⭐⭐⭐⭐**

```javascript
// Nuxt 3 核心功能

// 1. 文件系统路由 — pages/ 目录自动生成路由
// pages/index.vue     → /
// pages/about.vue     → /about
// pages/user/[id].vue → /user/:id
// pages/blog/[...slug].vue → /blog/**

// 2. 自动导入 — 无需手动 import
// components/ 下的组件自动可用
// composables/ 下的组合式函数自动可用
// utils/ 下的工具函数自动可用

// 3. 数据获取
// useFetch — 智能请求（SSR + CSR 复用）
<script setup>
const { data, pending, error, refresh } = await useFetch('/api/users')
</script>

// useAsyncData — 复杂数据获取逻辑
const { data: posts } = await useAsyncData('posts', () =>
  queryContent('blog').find()
)

// 4. 渲染模式控制
// 全局：nuxt.config.ts
export default defineNuxtConfig({
  ssr: true, // 全局 SSR
})

// 单页面：routeRules
routeRules: {
  '/admin/**': { ssr: false },                // CSR only
  '/blog/**': { swr: 600 },                   // ISR（10 分钟）
  '/api/**': { cors: true }
}

// 5. 模块系统
// @nuxt/content — 基于文件的 CMS
// @nuxt/image — 图像优化
// @nuxtjs/i18n — 国际化
// @pinia/nuxt — 状态管理
// @nuxtjs/color-mode — 暗色模式

// 6. 服务端 API — server/api/ 目录
// server/api/hello.ts
export default defineEventHandler(() => ({ message: 'Hello' }))
```

---

### 76. useFetch 和 useAsyncData 的区别？

**⭐⭐⭐**

```javascript
// useAsyncData — 更底层，需要手动处理请求
const { data, pending, error, refresh } = await useAsyncData(
  'users',  // 唯一 key（用于缓存和去重）
  () => $fetch('/api/users')  // 异步函数
)

// useFetch — useAsyncData + $fetch 的语法糖
const { data, pending, error, refresh } = await useFetch('/api/users')
// 等价于 useAsyncData(() => $fetch('/api/users'))

// 区别：
// | 特性 | useAsyncData | useFetch |
// |------|-------------|----------|
// | 请求方式 | 任意异步函数 | $fetch（ofetch） |
// | 灵活性 | 更高 | 更简洁 |
// | URL 响应式 | 手动 watch | URL 是 ref 会自动重新请求 |
// | 拦截器 | $fetch.create | $fetch.create |
// | 类型推断 | 需泛型 | URL 推断 |

// 使用建议：
// - 简单 API 请求 → useFetch
// - 复杂逻辑（多请求合并、非 HTTP 数据） → useAsyncData
// - 需要 queryContent → useAsyncData + queryContent
```

---

### 77. Nuxt 3 的服务端中间件和插件

**⭐⭐⭐**

```typescript
// 1. 插件 — plugins/ 目录
// plugins/auth.ts — 注册全局功能
export default defineNuxtPlugin((nuxtApp) => {
  // nuxtApp.vueApp — Vue 应用实例
  // nuxtApp.provide — 注入辅助方法
  // nuxtApp.hook — 生命周期钩子

  return {
    provide: {
      hello: (msg: string) => `Hello ${msg}`
    }
  }
})
// 使用：const { $hello } = useNuxtApp()

// 2. 服务端中间件 — server/middleware/ 目录
// server/middleware/auth.ts
export default defineEventHandler((event) => {
  // 在每个 API 请求前执行
  const token = getHeader(event, 'authorization')
  if (!token) {
    throw createError({ statusCode: 401, message: 'Unauthorized' })
  }
  event.context.auth = { userId: verifyToken(token) }
})

// 3. 路由中间件 — middleware/ 目录
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const user = useUser()
  if (!user.value && to.path !== '/login') {
    return navigateTo('/login')
  }
})

// 在页面中使用
// <script setup>
// definePageMeta({ middleware: 'auth' })
// </script>
```

---

### 78. Nuxt 3 渲染策略（SSR / SSG / ISR / SWR）

**⭐⭐⭐**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // 全局默认：SSR
  ssr: true,

  routeRules: {
    // SSR — 每次请求服务端渲染
    '/': { ssr: true },

    // SPA — 纯客户端渲染
    '/admin/**': { ssr: false },

    // SSG — 构建时生成静态页面（预渲染）
    '/about': { prerender: true },

    // ISR — 增量静态再生（首次渲染后缓存 N 秒）
    '/blog/**': { swr: 600 },  // 10 分钟

    // ISG — 构建时静态生成
    '/docs/**': { static: true },

    // 混合 — API 请求走 CDN 缓存
    '/api/posts': { cache: { maxAge: 60 } },

    // 重定向
    '/old-page': { redirect: '/new-page' }
  }
})
// swr 原理：首次请求 → SSR 渲染 → 缓存到 CDN → N 秒内命中缓存
// N 秒后第一次请求 → 返回旧缓存（stale）+ 后台重新渲染（revalidate）
```

---

### 79. SSR 的坑和注意事项

**⭐⭐⭐**

```javascript
// 1. 浏览器 API 不可用
// ❌ 直接使用 window / document / localStorage
// ✅ 用 onMounted 包裹或条件判断
if (import.meta.client) {
  // 仅在客户端执行的代码
}
// Nuxt 中：if (process.client) { }

// 2. 第三方库兼容性
// 使用 ClientOnly 组件包裹
<ClientOnly>
  <ChartComponent />
  <template #fallback>
    <div>加载中...</div>
  </template>
</ClientOnly>

// Nuxt 中的 .client 后缀
// ChartComponent.client.vue — 自动只在客户端渲染

// 3. 状态污染（State Pollution）
// ❌ 在 setup 顶层创建全局共享状态
const globalState = ref({}) // 多个用户可能共享！

// ✅ 在 setup 内部或 composable 中定义
function useMyFeature() {
  const state = ref({}) // 每个请求独立
  return { state }
}

// 4. 内存泄漏
// 服务端渲染时，每个请求的全局变量可能累积
// 确保在请求结束时清理

// 5. 水合不匹配（Hydration Mismatch）
// 服务端和客户端渲染结果不一致会报警告
// 常见原因：v-if + 浏览器 API 判断、随机值、时区不同
// 解决：用 ClientOnly、onMounted 中处理差异、<template #fallback>
```



---

## 十八、测试

### 80. Vue 项目的测试策略是什么？

**⭐⭐⭐**

```javascript
// 测试金字塔：
//     /\
//    /E2E\      少量端到端测试（关键流程）
//   /------\
//  /集成测试\    适量集成测试（组件交互）
// /----------\
//   单元测试     大量单元测试（函数/composable/组件）
// ------------

// 工具推荐
// 单元/组件：Vitest + @vue/test-utils
// E2E：Playwright / Cypress
// 覆盖率：c8 / istanbul

// 测试关注重点
// - 纯函数/composable：单元测试覆盖所有分支
// - 核心组件：输入输出、事件、插槽、v-model
// - 关键业务流程：E2E 覆盖
// - 边界情况：空数据、loading、error、权限
```

---

### 81. 如何用 Vitest + @vue/test-utils 测试 Vue 组件？

**⭐⭐⭐**

```javascript
// Counter.test.ts
import { mount } from '@vue/test-utils'
import { describe, it, expect } from 'vitest'
import Counter from './Counter.vue'

describe('Counter', () => {
  // 1. 基础渲染
  it('renders count', () => {
    const wrapper = mount(Counter, {
      props: { initialCount: 5 }
    })
    expect(wrapper.text()).toContain('5')
  })

  // 2. 事件测试
  it('increments on button click', async () => {
    const wrapper = mount(Counter)
    await wrapper.find('button').trigger('click')
    expect(wrapper.text()).toContain('1')
    expect(wrapper.emitted('update')[0]).toEqual([1])
  })

  // 3. v-model 测试
  it('supports v-model', async () => {
    const wrapper = mount(MyInput, {
      props: { modelValue: 'hello', 'onUpdate:modelValue': (e) => wrapper.setProps({ modelValue: e }) }
    })
    await wrapper.find('input').setValue('world')
    expect(wrapper.props('modelValue')).toBe('world')
  })

  // 4. 插槽测试
  it('renders slot content', () => {
    const wrapper = mount(Card, {
      slots: { default: '<span class="content">Hello</span>', header: '<h1>Title</h1>' }
    })
    expect(wrapper.find('.content').text()).toBe('Hello')
    expect(wrapper.find('h1').text()).toBe('Title')
  })

  // 5. 异步测试
  it('fetches data', async () => {
    const wrapper = mount(UserList)
    await wrapper.vm.$nextTick()
    await new Promise(resolve => setTimeout(resolve, 100)) // 等待 mock 返回
    expect(wrapper.findAll('li')).toHaveLength(3)
  })

  // 6. stub 子组件
  it('stubs child component', () => {
    const wrapper = mount(Parent, {
      global: { stubs: { HeavyChild: true } }
    })
  })
})
```

---

### 82. 如何测试 Composable？

**⭐⭐⭐**

```javascript
// useCounter.test.ts
import { useCounter } from './useCounter'
import { describe, it, expect } from 'vitest'

// 方式 1：直接测试（简单 composable）
describe('useCounter', () => {
  it('increments count', () => {
    const { count, increment } = useCounter()
    expect(count.value).toBe(0)
    increment()
    expect(count.value).toBe(1)
  })
})

// 方式 2：使用 withSetup 包装（需要生命周期的 composable）
// test-utils/withSetup.ts
import { createApp } from 'vue'

export function withSetup<T>(composable: () => T): T {
  let result: T
  const app = createApp({
    setup() {
      result = composable()
      return () => null
    }
  })
  app.mount(document.createElement('div'))
  // 返回前需要卸载
  // app.unmount()
  return result!
}

// 测试
import { withSetup } from './test-utils/withSetup'
import { useEventListener } from './useEventListener'

describe('useEventListener', () => {
  it('registers event listener', () => {
    const handler = vi.fn()
    withSetup(() => {
      useEventListener(window, 'resize', handler)
    })
    window.dispatchEvent(new Event('resize'))
    expect(handler).toHaveBeenCalled()
  })
})

// 方式 3：使用 @vue/test-utils 挂载一个测试组件
import { mount } from '@vue/test-utils'

describe('useSearch', () => {
  it('debounces search', async () => {
    vi.useFakeTimers()
    const TestComponent = defineComponent({
      setup() {
        const { query, results } = useSearch({ onSearch: vi.fn(async () => ['result']) })
        return { query, results }
      },
      template: '<div />'
    })

    const wrapper = mount(TestComponent)
    wrapper.vm.query = 'test'
    vi.advanceTimersByTime(300)
    await vi.runAllTimers()
    expect(wrapper.vm.results).toEqual(['result'])
    vi.useRealTimers()
  })
})
```

---

### 83. E2E 测试关键流程

**⭐⭐**

```javascript
// Playwright E2E 测试示例
import { test, expect } from '@playwright/test'

test.describe('Login Flow', () => {
  test('user can login and see dashboard', async ({ page }) => {
    // 1. 访问登录页
    await page.goto('/login')

    // 2. 填写表单
    await page.fill('[data-testid="username"]', 'admin')
    await page.fill('[data-testid="password"]', 'password123')
    await page.click('[data-testid="login-btn"]')

    // 3. 验证跳转到仪表盘
    await expect(page).toHaveURL('/dashboard')
    await expect(page.locator('[data-testid="welcome"]')).toContainText('欢迎')

    // 4. 验证 token 存储在 localStorage
    const token = await page.evaluate(() => localStorage.getItem('token'))
    expect(token).toBeTruthy()
  })
})

// E2E 测试覆盖的关键流程：
// - 登录/注册/退出
// - 核心业务 CRUD 操作
// - 表单验证和提交
// - 权限控制（不同角色看到不同内容）
// - 错误页面展示（404/403）
// - 多标签/多窗口场景
```

---

## 十九、Vite / 构建工具

### 84. Vite 为什么快？原理是什么？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// Vite 快的原因：

// 1. 开发服务器 — 基于 ES Module
//    利用浏览器原生 ESM，无需打包即可运行
//    import { ref } from '/node_modules/.vite/deps/vue.js?v=abc123'
//    只编译当前需要的模块，而非整个应用

// 2. 依赖预构建 — esbuild
//    node_modules 的依赖用 esbuild 预打包（Go 编写，极快）
//    将 CommonJS/UMD 转为 ESM
//    合并大量小模块为一个文件，减少 HTTP 请求数

// 3. 热更新（HMR）— 精准失效
//    修改的模块只更新它自身和引用链上的模块
//    不需要刷新整个页面
//    对比 webpack：需要重新构建整个 bundle

// 4. 生产构建 — Rollup
//    Vite 开发用 ES Module，生产用 Rollup 打包
//    Rollup 输出更小、更高效的代码

// 速度对比（大型项目冷启动）：
// webpack 5:  30s - 60s
// Vite:       1s - 3s
// HMR 更新：
// webpack:    1s - 5s（随模块数量增加）
// Vite:       < 50ms（几乎恒定）
```

### 85. Vite 配置中常见的优化项

**⭐⭐⭐**

```javascript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import AutoImport from 'unplugin-auto-import/vite'
import Components from 'unplugin-vue-components/vite'
import { resolve } from 'path'

export default defineConfig({
  // 1. 路径别名
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
      '@comps': resolve(__dirname, 'src/components')
    }
  },

  // 2. 开发服务器配置
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    },
    hmr: { overlay: true } // 错误遮罩
  },

  // 3. 构建优化
  build: {
    target: 'es2015',          // 目标浏览器
    minify: 'terser',          // 压缩
    sourcemap: false,          // 生产环境不需要
    chunkSizeWarningLimit: 500,
    rollupOptions: {
      output: {
        manualChunks: {
          vue: ['vue', 'vue-router', 'pinia'],
          ui: ['element-plus']
        }
      }
    }
  },

  // 4. CSS 配置
  css: {
    preprocessorOptions: {
      scss: {
        additionalData: `@import "@/styles/variables.scss";`
      }
    },
    postcss: {
      plugins: [require('autoprefixer')]
    }
  },

  // 5. 环境变量前缀
  envPrefix: 'VITE_',

  // 6. 插件
  plugins: [
    vue(),
    AutoImport({
      imports: ['vue', 'vue-router', 'pinia'],
      dirs: ['src/composables'],
      dts: 'src/auto-imports.d.ts'
    }),
    Components({
      dirs: ['src/components'],
      dts: 'src/components.d.ts'
    })
  ]
})
```

---

### 86. Vite 的 HMR 原理

**⭐⭐⭐**

```
HMR 流程：
1. 文件变化 → Vite Dev Server 检测到（chokidar 监听）
2. Vite 通过 WebSocket 向客户端发送更新通知
   { type: 'update', updates: [{ type: 'js-update', path: '/src/Comp.vue', acceptedPath: '/src/Comp.vue' }] }
3. 客户端收到通知 → 动态 import 新模块
4. 热更新边界（HMR Boundary）处理：
   - 自身 accept → 直接替换
   - 向上冒泡 → 直到找到 accept 的模块或到达入口
5. 组件更新时 Vue HMR runtime 保留组件状态，仅重新渲染

// 自定义 HMR 处理
if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    // 自定义热更新逻辑
  })
  import.meta.hot.dispose(() => {
    // 清理旧模块的副作用
  })
}
```

---

### 87. Vite 插件机制

**⭐⭐**

```javascript
// Vite 插件基于 Rollup 插件格式 + Vite 特有钩子
// vite-plugin-my-feature.ts
import type { Plugin } from 'vite'

export function myPlugin(): Plugin {
  return {
    name: 'vite-plugin-my-feature',

    // Rollup 钩子（构建时）
    resolveId(id) { /* 解析模块路径 */ },
    load(id) { /* 加载模块内容 */ },
    transform(code, id) { /* 转换代码 */ },

    // Vite 特有钩子
    config(config) { /* 修改 Vite 配置 */ },
    configResolved(config) { /* 配置解析后 */ },
    configureServer(server) { /* 配置开发服务器 */ },
    transformIndexHtml(html) { /* 转换入口 HTML */ },
    handleHotUpdate(ctx) { /* 自定义 HMR */ }
  }
}

// 最常用插件：
// @vitejs/plugin-vue — Vue SFC 编译
// unplugin-auto-import — 自动导入 API
// unplugin-vue-components — 自动注册组件
// vite-plugin-compression — Gzip 压缩
// vite-plugin-imagemin — 图片压缩
// vite-plugin-svg-icons — SVG 雪碧图
```

---

## 二十、高级模式与实践

### 88. 如何实现 Vue 的函数式组件？

**⭐⭐⭐**

```vue
<!-- Vue 3 函数式组件 — 普通函数返回 VNode -->
<script setup lang="ts">
import { h } from 'vue'

interface Props {
  level: 1 | 2 | 3 | 4 | 5 | 6
  label: string
}

const props = defineProps<Props>()
</script>

<!-- 函数式实现 — 无状态、无实例 -->
<script lang="ts">
import { h } from 'vue'

export default function Heading(props: Props, { slots }) {
  return h(`h${props.level}`, { class: 'heading' }, slots.default?.())
}
// 特点：无 this、无响应式开销、渲染更快
// 适用场景：纯展示组件、高阶组件包装
</script>
```

---

### 89. 如何实现 Vue 的递归组件？

**⭐⭐⭐**

```vue
<!-- TreeItem.vue — 递归树组件 -->
<script setup lang="ts">
interface TreeNode {
  id: string
  label: string
  children?: TreeNode[]
}

// 递归组件需要 name
defineOptions({ name: 'TreeItem' })

const props = defineProps<{ node: TreeNode }>()
const expanded = ref(false)
</script>

<template>
  <div class="tree-item">
    <div class="tree-label" @click="expanded = !expanded">
      <span v-if="node.children?.length">{{ expanded ? '▼' : '▶' }}</span>
      {{ node.label }}
    </div>
    <div v-if="expanded && node.children?.length" class="tree-children">
      <!-- 自己调用自己 — 递归 -->
      <TreeItem v-for="child in node.children" :key="child.id" :node="child" />
    </div>
  </div>
</template>
```

---

### 90. 如何实现表单验证和动态表单？

**⭐⭐⭐**

```vue
<script setup lang="ts">
import { useForm } from 'vee-validate'
import * as yup from 'yup'

// 方案 1：vee-validate + yup（推荐）
const schema = yup.object({
  name: yup.string().required('必填').min(2, '至少 2 个字符'),
  email: yup.string().required('必填').email('邮箱格式不正确'),
  age: yup.number().typeError('必须是数字').min(18, '需满 18 岁').required(),
  password: yup.string().required().min(6),
  confirmPassword: yup.string().oneOf([yup.ref('password')], '两次密码不一致')
})

const { handleSubmit, errors, values, setFieldValue } = useForm({
  validationSchema: schema
})

const onSubmit = handleSubmit((values) => {
  console.log('提交:', values)
})
</script>

<!-- 方案 2：动态表单配置驱动 -->
<script setup>
const formConfig = reactive([
  { key: 'name', label: '姓名', type: 'input', required: true },
  { key: 'gender', label: '性别', type: 'select', options: ['男', '女'] },
  { key: 'desc', label: '描述', type: 'textarea', when: (form) => form.gender === '女' }
])
</script>
```

---

### 91. 微前端中如何使用 Vue？（qiankun / Module Federation）

**⭐⭐⭐**

```javascript
// 方案 1：qiankun 微前端
// main.ts — 子应用入口
import { createApp } from 'vue'
import { createRouter, createWebHistory } from 'vue-router'

let app: ReturnType<typeof createApp> | null = null
let router: ReturnType<typeof createRouter> | null = null

function render(props: any = {}) {
  const { container } = props
  router = createRouter({
    history: createWebHistory(window.__POWERED_BY_QIANKUN__ ? '/vue-app' : '/'),
    routes
  })

  app = createApp(App)
  app.use(router)
  app.mount(container ? container.querySelector('#app') : '#app')
}

// 独立运行
if (!window.__POWERED_BY_QIANKUN__) {
  render()
}

// qiankun 生命周期
export async function bootstrap() { console.log('vue app bootstrapped') }
export async function mount(props: any) { render(props) }
export async function unmount() {
  app?.unmount()
  app = null
  router = null
}

// 方案 2：Module Federation（Vite 或 webpack）
// vite.config.ts → @originjs/vite-plugin-federation
// 共享依赖：vue, vue-router, pinia
// 远程模块通过 import('remoteApp/Component') 加载
```

---

### 92. 国际化（i18n）实现方案

**⭐⭐⭐**

```javascript
// vue-i18n v9 配置
import { createI18n } from 'vue-i18n'

const i18n = createI18n({
  legacy: false,  // Composition API 模式
  locale: 'zh-CN',
  fallbackLocale: 'en',
  messages: {
    'zh-CN': {
      hello: '你好',
      user: { name: '姓名', age: '年龄' },
      items: '共 {count} 项'  // 参数插值
    },
    'en': {
      hello: 'Hello',
      user: { name: 'Name', age: 'Age' },
      items: '{count} items'
    }
  }
})

// 使用
// {{ $t('hello') }}
// {{ $t('items', { count: 10 }) }} → "共 10 项"

// Composition API
import { useI18n } from 'vue-i18n'
const { t, locale } = useI18n()
locale.value = 'en'  // 切换语言

// 日期/数字格式化
// {{ $d(new Date(), 'short') }}
// {{ $n(12345.67, 'currency', 'CNY') }}

// 按需加载语言包
const loadLocaleMessages = async (locale: string) => {
  const messages = await import(`./locales/${locale}.json`)
  i18n.global.setLocaleMessage(locale, messages.default)
}
```

---

### 93. 大文件上传、断点续传

**⭐⭐**

```vue
<script setup>
// 核心思路：切片 + 并发上传 + 断点续传

const CHUNK_SIZE = 5 * 1024 * 1024 // 5MB

// 1. 文件切片
function createChunks(file: File) {
  const chunks: Blob[] = []
  let start = 0
  while (start < file.size) {
    chunks.push(file.slice(start, start + CHUNK_SIZE))
    start += CHUNK_SIZE
  }
  return chunks
}

// 2. 计算文件 hash（用于断点续传标识）
async function calculateHash(chunks: Blob[]): Promise<string> {
  // 使用 spark-md5 或 Web Crypto API
  return 'file-unique-hash'
}

// 3. 检查已上传的切片
async function checkUploaded(hash: string): Promise<number[]> {
  const res = await api.verifyUpload(hash)
  return res.uploadedChunks // 已上传的切片索引数组
}

// 4. 并发上传
async function uploadChunks(chunks: Blob[], hash: string, uploaded: number[]) {
  const tasks = chunks.map((chunk, index) => {
    if (uploaded.includes(index)) return Promise.resolve() // 跳过已上传
    const formData = new FormData()
    formData.append('chunk', chunk)
    formData.append('hash', hash)
    formData.append('index', String(index))
    return api.uploadChunk(formData)
  })

  // 控制并发数
  const CONCURRENCY = 3
  for (let i = 0; i < tasks.length; i += CONCURRENCY) {
    await Promise.all(tasks.slice(i, i + CONCURRENCY))
  }
}

// 5. 通知服务端合并切片
async function mergeChunks(hash: string, filename: string) {
  await api.mergeChunks({ hash, filename, chunkSize: CHUNK_SIZE })
}
</script>
```

---

## 二十一、安全与权限

### 94. Vue 中 XSS 防护

**⭐⭐⭐**

```javascript
// XSS 攻击防范要点

// 1. ⚠️ v-html 是最大风险点
// ❌ 永不对用户输入使用 v-html
<div v-html="userInput" /> // 危险！

// ✅ 如需渲染富文本，使用 DOMPurify 过滤
import DOMPurify from 'dompurify'
const sanitized = computed(() => DOMPurify.sanitize(userInput.value))
<div v-html="sanitized" />

// 2. URL 注入防护
// ❌ <a :href="userInput">链接</a>  // 用户输入 javascript:alert(1)
// ✅ 验证 URL 协议
const safeUrl = computed(() => {
  const url = userInput.value
  return /^(https?:)?\/\//i.test(url) ? url : '#'
})

// 3. 样式注入防护
// ❌ <div :style="userStyles" /> // CSS expression() 攻击（老旧 IE）
// ✅ Vue 3 自动过滤危险 CSS 值

// 4. SSR 特有风险 — 服务端渲染输出需转义
// Nuxt/SSR 默认转义 {{ }} 中的内容
// 不要用 v-html 渲染来自 SSR 的用户内容

// 5. CSP（Content Security Policy）配置
// 禁止内联脚本，只允许特定域名脚本
```

---

### 95. 前端权限控制方案

**⭐⭐⭐⭐**

```javascript
// 权限控制三层模型

// 1. 路由权限（页面级）
router.beforeEach((to, from, next) => {
  const store = useUserStore()
  if (to.meta.permissions?.length) {
    const hasAccess = to.meta.permissions.every(p => store.hasPermission(p))
    if (!hasAccess) return next('/403')
  }
  next()
})

// 2. 按钮/功能权限（组件级）
// 指令方式
const vPermission = {
  mounted(el, binding) {
    const store = useUserStore()
    if (!store.hasPermission(binding.value)) {
      el.parentNode?.removeChild(el)
    }
  }
}
// <button v-permission="'user:delete'">删除</button>

// 组件方式
<Permission :code="'user:delete'">
  <button>删除</button>
</Permission>

// 3. 数据权限（API 级）
// 由后端根据用户角色过滤返回数据
// 前端不需要处理数据级权限（不安全）
// 例如：管理员看到所有订单，普通用户只看到自己的

// 权限存储
const usePermissionStore = defineStore('permission', () => {
  const permissions = ref<string[]>([])
  const roles = ref<string[]>([])

  const hasPermission = (code: string) => permissions.value.includes(code)
  const hasRole = (role: string) => roles.value.includes(role)
  const hasAnyPermission = (...codes: string[]) => codes.some(hasPermission)
  const hasAllPermissions = (...codes: string[]) => codes.every(hasPermission)

  return { permissions, roles, hasPermission, hasRole, hasAnyPermission, hasAllPermissions }
})
```

---

### 96. 前端敏感信息保护

**⭐⭐**

```javascript
// 1. 不要在代码中硬编码密钥/Token
// ❌ const API_KEY = 'sk-xxxxxxxx'
// ✅ 环境变量
const API_KEY = import.meta.env.VITE_API_KEY

// 2. Token 存储
// ❌ localStorage（易受 XSS 攻击）
// ✅ httpOnly Cookie（JS 无法访问，防 XSS）
// 或 accessToken 内存 + refreshToken httpOnly Cookie

// 3. 敏感页面水印
function addWatermark(text: string) {
  const canvas = document.createElement('canvas')
  canvas.width = 200
  canvas.height = 100
  const ctx = canvas.getContext('2d')!
  ctx.rotate((-20 * Math.PI) / 180)
  ctx.fillStyle = 'rgba(0,0,0,0.06)'
  ctx.font = '14px Arial'
  ctx.fillText(text, 10, 50)
  const bg = `url(${canvas.toDataURL()})`
  document.body.style.backgroundImage = bg
}

// 4. 日志脱敏
function maskPhone(phone: string) { return phone.replace(/(\d{3})\d{4}(\d{4})/, '$1****$2') }
function maskEmail(email: string) { return email.replace(/(.{3}).*(@.*)/, '$1***$2') }

// 5. 开发环境 mock 数据不要包含真实用户信息
// 6. sourcemap 不应部署到生产环境
//    build: { sourcemap: false }
```

---

## 二十二、综合与趋势

### 97. Vue 项目目录结构最佳实践

**⭐⭐⭐**

```
src/
├── api/              # API 接口层
│   ├── request.ts    # axios 封装（拦截器/错误处理）
│   ├── modules/      # 按业务模块划分
│   │   ├── user.ts
│   │   └── product.ts
├── assets/           # 静态资源（图片/字体/样式）
├── components/       # 公共组件
│   ├── base/         # 基础组件（Button/Input/Modal）
│   └── business/     # 业务组件
├── composables/      # 组合式函数
│   ├── useAuth.ts
│   ├── useRequest.ts
│   └── usePermission.ts
├── directives/       # 自定义指令
├── layouts/          # 布局组件
├── router/           # 路由
│   ├── index.ts
│   ├── routes/       # 路由配置
│   └── guards.ts     # 路由守卫
├── stores/           # Pinia Store
├── styles/           # 全局样式
│   ├── variables.scss
│   ├── mixins.scss
│   └── reset.scss
├── types/            # TypeScript 类型
├── utils/            # 工具函数
├── views/            # 页面组件
│   ├── user/
│   │   ├── UserList.vue
│   │   ├── UserDetail.vue
│   │   └── components/  # 页面私有组件
├── App.vue
└── main.ts
```

---

### 98. 前端监控和错误上报

**⭐⭐⭐**

```javascript
// 错误监控体系

// 1. 全局 Vue 错误
app.config.errorHandler = (err, instance, info) => {
  report({
    type: 'vue-error',
    message: err.message,
    stack: err.stack,
    component: instance?.$options.name,
    info
  })
}

// 2. Promise 错误
window.addEventListener('unhandledrejection', (event) => {
  report({
    type: 'promise-error',
    message: event.reason?.message,
    stack: event.reason?.stack
  })
})

// 3. 全局 JS 错误
window.addEventListener('error', (event) => {
  report({
    type: 'js-error',
    message: event.message,
    filename: event.filename,
    lineno: event.lineno
  })
})

// 4. 性能监控
import { onLCP, onFID, onCLS, onINP } from 'web-vitals'
onLCP(metric => report({ type: 'web-vitals', name: 'LCP', value: metric.value }))
onFID(metric => report({ type: 'web-vitals', name: 'FID', value: metric.value }))
onCLS(metric => report({ type: 'web-vitals', name: 'CLS', value: metric.value }))
onINP(metric => report({ type: 'web-vitals', name: 'INP', value: metric.value }))

// 5. 接口监控
axios.interceptors.response.use(
  res => res,
  error => {
    report({
      type: 'api-error',
      url: error.config?.url,
      status: error.response?.status,
      message: error.message
    })
    return Promise.reject(error)
  }
)

// 6. 上报函数（收集 + 批量发送）
const queue: any[] = []
let timer: NodeJS.Timeout | null = null

function report(data: any) {
  queue.push({ ...data, timestamp: Date.now(), url: location.href })
  if (!timer) {
    timer = setTimeout(() => {
      navigator.sendBeacon('/api/monitor', JSON.stringify(queue))
      queue.length = 0
      timer = null
    }, 2000) // 2 秒合并上报
  }
}
```

---

### 99. Vue 项目 CI/CD 流程

**⭐⭐**

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with: { node-version: 18 }
      - run: npm ci
      - run: npm run lint           # 代码检查
      - run: npm run type-check     # TS 类型检查
      - run: npm run test:unit      # 单元测试
      - run: npm run build          # 构建
      - run: npm run test:e2e       # E2E 测试
      # - 部署到服务器/CDN
      # - 上传 sourcemap 到监控平台
      # - 企业微信/钉钉通知
```

---

### 100. 2025-2026 Vue 生态趋势与面试 TOP 10

**⭐⭐⭐**

#### 生态趋势

| 趋势 | 说明 |
|------|------|
| Vue 3 全面普及 | 新项目默认 Vue 3 + Composition API + `<script setup>` |
| Nuxt 4 | 更强的 SSR/SSG/ISR，Edge 部署 |
| Vapor Mode | 无虚拟 DOM 的编译模式（实验性，性能极致） |
| TypeScript 一体化 | 泛型组件、defineModel 类型、更好的 IDE 支持 |
| Vite 生态 | Vitest 成为测试标配，VitePress 文档站 |
| 跨端方案 | uni-app x / Taro 3.x 对 Vue 3 的完善支持 |
| 状态管理 | Pinia 完全取代 Vuex |
| AI 集成 | AI 辅助编码、AI 生成组件 |

#### 面试 TOP 10 高频题

1. **Vue 2 vs Vue 3 响应式原理** — defineProperty vs Proxy（⭐必问）
2. **ref vs reactive** — 区别、陷阱、最佳实践
3. **Composition API 解决了什么问题** — 逻辑复用、代码组织
4. **Vue Router 路由守卫及执行顺序**
5. **v-if vs v-show / v-if vs v-for 优先级**
6. **组件通信方式** — props/emits/provide/inject/Pinia/EventBus
7. **Vite 为什么快** — ESM + esbuild + HMR
8. **Diff 算法优化** — 双端对比 + LIS + Block Tree
9. **SSR 原理与 Nuxt** — 首屏速度/SEO/水合
10. **性能优化** — 虚拟滚动/KeepAlive/shallowRef/编译优化

#### 学习路径推荐

```
入门：Vue 3 官方文档 → 动手项目 → Composition API
进阶：响应式原理 → 虚拟 DOM/Diff → 编译优化
生态：Vue Router → Pinia → Vite → Nuxt → VueUse
实战：性能优化 → 错误监控 → CI/CD → 微前端
源码：reactivity → runtime-core → compiler-core
```

---

> **总结：** 这 100 题覆盖了 Vue 面试中 95% 以上的考点。建议按模块学习，每道题不仅要知道答案，还要能写出核心代码。面试中常见的手写题（响应式原理、Diff 算法、computed/watch 实现）务必亲自动手写一遍。

---

*最后更新：2026 年 7 月*
