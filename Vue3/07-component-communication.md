# 07 - 组件通信全景

## 1. 通信方式总表

| 场景 | 方案 | 层级 |
|---|---|---|
| 父 → 子 | props | 一层 |
| 子 → 父 | emit | 一层 |
| 父 ↔ 子 | v-model | 一层 |
| 父调子方法 | ref + expose | 一层 |
| 跨层级注入 | provide / inject | 任意 |
| 全局状态 | Pinia | 任意 |
| 事件总线 | mitt（慎用） | 任意 |
| 临时 DOM 通信 | $attrs / 插槽传参 | — |

选型原则：**层级浅用 props/emit，跨层级用 provide/inject，全局共享用 Pinia**。

## 2. props / emit（默认首选）

见 `06-component-basics.md`，此处略。

## 3. v-model 全解

### 3.1 基本形态

```vue
<!-- 父 -->
<input v-model="text" />

<!-- 编译为 -->
<input :model-value="text" @update:model-value="text = $event" />
```

### 3.2 自定义组件实现 v-model

```vue
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [string] }>()
</script>
<template>
  <input :value="modelValue" @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)" />
</template>
```

### 3.3 多 v-model 与参数重命名

```vue
<UserDialog v-model:title="t" v-model:visible="v" />
<!-- 子组件 props: title / visible；emit: update:title / update:visible -->
```

### 3.4 自定义修饰符

```vue
<MyInput v-model.capitalize="text" />
<!-- 子组件 props.modelModifiers = { capitalize: true } -->
```

## 4. provide / inject

```ts
// 祖先
import { provide, readonly, ref } from 'vue'
const user = ref<User | null>(null)
provide('user', readonly(user))           // 只读保护
provide('setUser', (u: User) => user.value = u)  // 提供修改方法

// 后代
import { inject } from 'vue'
const user = inject<Ref<User | null>>('user', null)
const setUser = inject<(u: User) => void>('setUser')!
```

规范要点：
1. **inject 值设为 readonly**，写操作通过父层提供的方法——单向数据流不破
2. **Symbol 作 key** 防命名冲突：

```ts
// keys.ts
export const USER_KEY: InjectionKey<Ref<User | null>> = Symbol('user')
provide(USER_KEY, user)
inject(USER_KEY, null)   // 自动类型推导
```

3. 默认值 `inject(key, default)`，required 场景 `inject(key)!` 并在开发期断言
4. provide 必须在 setup 同步阶段调用

## 5. ref + expose（命令式）

见 `10-attrs-ref-expose.md`。适合：表单 validate、组件 focus、播放器 play/pause 等命令式 API。

## 6. Pinia（全局状态）

```ts
const userStore = useUserStore()
userStore.name           // 直接读（自动解包 storeToRefs 场景）
userStore.setName('x')   // action
```

跨组件/跨页面的共享状态一律进 Pinia，见 `20-pinia.md`。

## 7. 事件总线（mitt）——为何慎用

```ts
import mitt from 'mitt'
const emitter = mitt()
emitter.on('refresh', fn)
emitter.emit('refresh', payload)
```

问题：依赖隐式、类型弱、时序不可控、忘 off 泄漏。

**仅允许的场景**：无父子的底层基础设施（如多 tab 同步、全局埋点），且必须封装为显式 API，禁止业务组件直接互相 emit。

## 8. 兄弟组件通信

兄弟间不直接通信，三种正途：
1. **共同父组件**中转（props ↓ emit ↑）
2. **Pinia** 共享状态
3. provide/inject（兄弟分属不同分支时）

## 9. 通信反模式清单

| 反模式 | 危害 | 替代 |
|---|---|---|
| bus 泛滥 | 关系不可追溯 | Pinia / 父组件中转 |
| 祖先 deep provide 大对象 | 子组件改坏全局 | readonly + action |
| $parent / $children 链式取值 | 强耦合结构 | emit / provide |
| 通过 DOM 查询兄弟 | 脆弱 | ref / 状态 |
| props 传函数当回调泛滥 | 契约模糊 | 明确 emit 事件 |

## 常见坑

1. **provide 的对象未 readonly**：任意后代直接改，全局状态被悄悄污染。
2. **inject 在异步/钩子中调用**：必须在 setup 同步阶段，否则拿不到。
3. **字符串 key 冲突**：大型项目必须 Symbol key。
4. **v-model 组件内部直接改 props.modelValue**：只改了本地副本，父层不知道。
5. **mitt 监听未 off**：组件卸载后回调还在执行。

## Checklist

- [ ] 层级 ≤2 用 props/emit，深层才用 provide/inject
- [ ] provide 的可变状态一律 readonly + 显式 action
- [ ] InjectionKey 用 Symbol 且类型化
- [ ] 业务组件间无直接事件总线通信
- [ ] 全局共享状态已收敛到 Pinia
