# Vue 3 知识点总结

## 1. Vue 3 简介

Vue 3 是 Vue 框架的重大升级版本（2020 年发布），采用 Composition API，底层使用 Proxy 实现响应式，性能更好、打包更小，并引入了 `<script setup>` 语法糖。

特点：
- Composition API（组合式 API）
- 基于 Proxy 的响应式系统
- TypeScript 支持更好
- Tree-shaking，打包体积更小
- 更好的性能（更快、更轻量）

## 2. 创建应用

Vue 3 使用 `createApp` 创建应用实例。

```
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)
app.mount('#app')
```

## 3. 组合式 API：setup

`setup` 是组件逻辑的入口，替代了 Vue 2 的 data/methods/computed 等选项。

```
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)
    const add = () => count.value++

    return { count, add }
  }
}
```

## 4. script setup 语法糖

`<script setup>` 更简洁，无需手动 return，是官方推荐的写法。

```
<script setup>
import { ref } from 'vue'

const count = ref(0)
const add = () => count.value++
</script>

<template>
  <button @click="add">{{ count }}</button>
</template>
```

## 5. ref 响应式引用

`ref` 用于包装基本类型，访问和修改需要 `.value`。模板中会自动解包。

```
<script setup>
import { ref } from 'vue'

const msg = ref('hello')
const num = ref(1)

console.log(msg.value)     // hello
msg.value = 'world'
</script>
```

## 6. reactive 响应式对象

`reactive` 用于包装对象，直接访问属性，无需 `.value`。

```
<script setup>
import { reactive } from 'vue'

const user = reactive({
  name: 'zhangsan',
  age: 20
})

user.age++               // 直接修改，响应式生效
</script>
```

## 7. ref 与 reactive 对比

| | ref | reactive |
|--|-----|----------|
| 适用类型 | 基本类型、对象 | 对象/数组 |
| 访问方式 | `.value` | 直接访问 |
| 重新赋值 | 可以 | 不可以（会丢失响应式） |

```
// reactive 不能整体重新赋值，会丢失响应式
const state = reactive({ a: 1 })
// state = { a: 2 }   // 错误，响应式丢失
state.a = 2           // 正确

// ref 可以整体赋值
const obj = ref({ a: 1 })
obj.value = { a: 2 }  // 正确
```

## 8. computed 计算属性

基于响应式数据派生新值，有缓存，依赖不变不会重复计算。

```
<script setup>
import { ref, computed } from 'vue'

const num = ref(10)
const double = computed(() => num.value * 2)

// 可读可写
const count = ref(1)
const doubleCount = computed({
  get: () => count.value * 2,
  set: (val) => (count.value = val / 2)
})
</script>
```

## 9. watch 侦听器

监听数据变化执行副作用，可监听 ref、reactive、getter 或数组。

```
<script setup>
import { ref, watch } from 'vue'

const count = ref(0)

watch(count, (newVal, oldVal) => {
  console.log(newVal, oldVal)
})

// 监听多个
watch([count, msg], ([n, m]) => {
  console.log(n, m)
})

// 深度监听对象
watch(state, (val) => {
  console.log(val)
}, { deep: true })
</script>
```

## 10. watchEffect 自动追踪

自动追踪内部用到的响应式依赖，无需手动指定监听源。

```
<script setup>
import { ref, watchEffect } from 'vue'

const count = ref(0)
const double = ref(0)

watchEffect(() => {
  double.value = count.value * 2   // 自动追踪 count
})
</script>
```

## 11. 生命周期钩子

组合式 API 下通过 `onXxx` 函数注册生命周期。

```
<script setup>
import { onMounted, onUpdated, onUnmounted } from 'vue'

onMounted(() => {
  console.log('组件挂载完成')
})
onUpdated(() => {
  console.log('组件更新')
})
onUnmounted(() => {
  console.log('组件卸载')
})
</script>
```

对应关系：beforeCreate/setup → setup、created → setup、beforeMount → onBeforeMount、mounted → onMounted。

## 12. 模板语法

插值、指令、事件绑定。

```
<template>
  <p>{{ msg }}</p>
  <p v-text="msg"></p>
  <p v-html="html"></p>
  <a :href="url">链接</a>          <!-- 动态属性 -->
  <button @click="handle">点击</button>
  <input v-model="name" />
</template>

<script setup>
import { ref } from 'vue'
const msg = ref('hello')
const html = ref('<b>加粗</b>')
const url = ref('https://vuejs.org')
const name = ref('')
const handle = () => console.log('clicked')
</script>
```

## 13. v-if 与 v-show

- v-if：条件为真才渲染，有销毁/创建开销，适合切换不频繁的场景。
- v-show：通过 display 控制显隐，始终渲染，适合频繁切换。

```
<template>
  <p v-if="visible">v-if 控制</p>
  <p v-show="visible">v-show 控制</p>
</template>

<script setup>
import { ref } from 'vue'
const visible = ref(true)
</script>
```

## 14. v-for 列表渲染

遍历数组，必须绑定唯一 key。

```
<template>
  <ul>
    <li v-for="item in list" :key="item.id">{{ item.name }}</li>
  </ul>
</template>

<script setup>
import { reactive } from 'vue'
const list = reactive([
  { id: 1, name: 'a' },
  { id: 2, name: 'b' }
])
</script>
```

## 15. 事件与修饰符

修饰符用于简化事件处理，如阻止冒泡、阻止默认、按键触发、只触发一次等。

```
<template>
  <button @click.stop="doThis">阻止冒泡</button>   <!-- .stop 阻止事件冒泡 -->
  <a @click.prevent="doThat">阻止默认</a>          <!-- .prevent 阻止默认行为 -->
  <input @keyup.enter="submit" />                  <!-- .enter 回车时触发 -->
  <div @click.once="once">只触发一次</div>         <!-- .once 只触发一次 -->
</template>
```

## 16. props 父传子

父组件向子组件传值，子组件用 `defineProps` 声明。

```
<!-- 父组件 -->
<Child :title="msg" :count="10" />

<!-- 子组件 Child.vue -->
<script setup>
const props = defineProps({
  title: String,
  count: {
    type: Number,
    default: 0
  }
})
</script>

<template>
  <p>{{ title }} - {{ count }}</p>
</template>
```

## 17. emit 子传父

子组件通过 `defineEmits` 触发事件，父组件监听。

```
<!-- 子组件 -->
<script setup>
const emit = defineEmits(['update'])
const send = () => emit('update', 'hello')
</script>

<template>
  <button @click="send">发送</button>
</template>

<!-- 父组件 -->
<Child @update="handle" />
```

## 18. 组件 v-model

组件上使用 v-model 实现双向绑定，对应 `modelValue` prop 和 `update:modelValue` 事件。

```
<!-- 子组件 -->
<script setup>
defineProps(['modelValue'])
defineEmits(['update:modelValue'])
</script>

<template>
  <input
    :value="modelValue"
    @input="$emit('update:modelValue', $event.target.value)"
  />
</template>

<!-- 父组件 -->
<MyInput v-model="text" />
```

## 19. provide / inject 跨层级传值

祖先组件 provide 数据，任意后代组件 inject 获取，无需层层传递。

```
<!-- 祖先组件 -->
<script setup>
import { provide, ref } from 'vue'
const theme = ref('dark')
provide('theme', theme)
</script>

<!-- 后代组件 -->
<script setup>
import { inject } from 'vue'
const theme = inject('theme')
</script>
```

## 20. 模板 ref 获取 DOM

通过 `ref` 获取 DOM 元素或组件实例。

```
<template>
  <input ref="inputRef" />
  <button @click="focus">聚焦</button>
</template>

<script setup>
import { ref, onMounted } from 'vue'
const inputRef = ref(null)

const focus = () => inputRef.value.focus()

onMounted(() => {
  inputRef.value.focus()
})
</script>
```

## 21. nextTick

DOM 更新是异步的，`nextTick` 在 DOM 更新完成后执行回调。

```
<script setup>
import { ref, nextTick } from 'vue'

const count = ref(0)
const el = ref(null)

const update = async () => {
  count.value++
  await nextTick()
  console.log(el.value.textContent)   // DOM 已更新
}
</script>
```

## 22. 自定义指令

`v-custom` 形式的指令，通过 `directives` 选项或命名约定注册。

```
<script setup>
// 局部指令，命名需 vXxx 形式
const vFocus = {
  mounted(el) {
    el.focus()
  }
}
</script>

<template>
  <input v-focus />
</template>
```

## 23. 全局组件与插件

通过 `app.component` 注册全局组件，通过 `app.use` 安装插件。

```
// main.js
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

app.component('MyButton', MyButton)  // 全局组件
app.use(router)                      // 插件
app.use(pinia)

app.mount('#app')
```

## 24. 响应式原理（Proxy）

Vue 3 使用 Proxy 拦截对象操作，解决 Vue 2 Object.defineProperty 无法监听新增/删除属性的问题。

```
// 简化实现
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key) {
      track(target, key)          // 依赖收集
      return target[key]
    },
    set(target, key, value) {
      target[key] = value
      trigger(target, key)        // 触发更新
      return true
    }
  })
}
```

## 25. toRef / toRefs

从响应式对象中解构属性并保持响应式。

```
<script setup>
import { reactive, toRef, toRefs } from 'vue'

const user = reactive({ name: 'tom', age: 20 })

const name = toRef(user, 'name')   // 单个属性转 ref

const { name: n, age } = toRefs(user)   // 整个对象转 refs，解构后仍响应式
</script>
```

## 26. 组合式函数 Composables

把可复用的逻辑封装成以 `use` 开头的函数，是组合式 API 的核心复用方式。

```
// useCounter.js
import { ref } from 'vue'

export function useCounter() {
  const count = ref(0)
  const increment = () => count.value++
  const decrement = () => count.value--
  return { count, increment, decrement }
}

// 组件中使用
<script setup>
import { useCounter } from './useCounter'
const { count, increment } = useCounter()
</script>
```

## 27. 插槽 Slots

父组件向子组件传递模板内容。

```
<!-- 子组件 -->
<template>
  <div>
    <slot>默认内容</slot>          <!-- 默认插槽 -->
    <slot name="header"></slot>    <!-- 具名插槽 -->
  </div>
</template>

<!-- 父组件 -->
<Child>
  <template #default>正文</template>
  <template #header>标题</template>
</Child>
```

## 28. 动态组件与 keep-alive

`component :is` 动态切换组件，`keep-alive` 缓存组件状态。

```
<template>
  <button @click="cur = cur === 'A' ? 'B' : 'A'">切换</button>
  <keep-alive>
    <component :is="cur" />
  </keep-alive>
</template>

<script setup>
import { ref } from 'vue'
import A from './A.vue'
import B from './B.vue'
const cur = ref('A')
</script>
```

## 29. Suspense 异步组件

处理异步组件加载状态，配合 `defineAsyncComponent` 或顶层 await。

```
<template>
  <Suspense>
    <template #default>
      <AsyncComp />
    </template>
    <template #fallback>
      <p>加载中...</p>
    </template>
  </Suspense>
</template>

<script setup>
import { defineAsyncComponent } from 'vue'
const AsyncComp = defineAsyncComponent(() => import('./Heavy.vue'))
</script>
```

## 30. Teleport 传送门

把组件渲染到指定 DOM 位置（如 body），常用于弹窗、通知。

```
<template>
  <Teleport to="body">
    <div class="modal">弹窗内容</div>
  </Teleport>
</template>
```

## 31. 状态管理 Pinia

Vue 3 官方推荐的状态管理库，替代 Vuex。

```
// stores/counter.js
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0 }),
  getters: {
    double: (state) => state.count * 2
  },
  actions: {
    increment() {
      this.count++
    }
  }
})

// 组件中使用
<script setup>
import { useCounterStore } from '@/stores/counter'
const store = useCounterStore()
store.increment()
console.log(store.double)
</script>
```

## 32. 路由 Vue Router

通过路由表配置页面与路径的映射，用 useRoute/useRouter 获取路由信息和进行编程式导航。

```
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', component: Home },
    { path: '/user/:id', component: User, props: true }
  ]
})

// 组件中使用
<script setup>
import { useRoute, useRouter } from 'vue-router'
const route = useRoute()       // 获取当前路由信息
const router = useRouter()     // 编程式导航
router.push('/user/1')
</script>
```

## 33. 与 Vue 2 主要区别

- 响应式：Object.defineProperty → Proxy
- API：Options API → Composition API（也可兼容）
- 生命周期：destroyed → unmounted
- v-model：`value`/`input` → `modelValue`/`update:modelValue`
- 新增：Teleport、Suspense、Fragment
- 全局 API：`new Vue()` → `createApp()`
