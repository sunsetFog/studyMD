# 10 - attrs / ref / expose

## 1. $attrs 全解

`$attrs` 是**未声明为 props/emits 的所有属性与事件**（Vue3 已含 class/style）。

```vue
<!-- 父 -->
<MyInput class="big" placeholder="请输入" @focus="onFocus" />

<!-- 子 MyInput.vue -->
<script setup lang="ts">
defineProps<{ modelValue: string }>()
defineOptions({ inheritAttrs: false })
</script>
<template>
  <div class="wrap">
    <input v-bind="$attrs" :value="modelValue" />  <!-- 手动指定落点 -->
  </div>
</template>
```

### 1.1 何时关闭自动透传

| 场景 | 设置 |
|---|---|
| 根节点不是接收 attrs 的元素（如 input 在深层） | `inheritAttrs: false` + 手动 v-bind |
| 多根节点（Vue3 允许） | 必须手动处理，否则警告 |
| 需要过滤部分 attrs | computed 挑选后绑定 |

### 1.2 useAttrs

```ts
import { useAttrs } from 'vue'
const attrs = useAttrs()   // 非响应式对象（底层每次渲染取新值），勿解构后缓存
```

注意：`$attrs` 不是响应式 Proxy，**不要**对它做 watch/解构保存。

## 2. useTemplateRef（3.5+）与 ref

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'

const inputEl = useTemplateRef<HTMLInputElement>('myInput')
// 等价老写法：const inputEl = ref<HTMLInputElement>() + template ref="myInput"
onMounted(() => inputEl.value?.focus())
</script>
<template>
  <input ref="myInput" />
</template>
```

v-for 中模板 ref：

```vue
<li v-for="item in list" :key="item.id" :ref="(el) => (itemRefs[item.id] = el)">
```

## 3. defineExpose

`<script setup>` 组件默认**封闭**——父通过 ref 拿到的是空壳。显式暴露：

```vue
<!-- 子：FormPanel.vue -->
<script setup lang="ts">
import { reactive } from 'vue'
const state = reactive({ dirty: false })
function validate(): boolean { /* … */ return true }
function reset() { /* … */ }

defineExpose({
  validate,
  reset,
  state: readonly(state),     // 数据只读，行为开放
})
</script>
```

```vue
<!-- 父 -->
<FormPanel ref="panel" />
<script setup lang="ts">
const panel = ref<InstanceType<typeof FormPanel>>()
const ok = panel.value?.validate()
</script>
```

### 3.1 类型获取

```ts
// 推荐显式接口类型
type FormPanelExpose = { validate(): boolean; reset(): void }
const panel = ref<FormPanelExpose | null>(null)
```

### 3.2 设计原则

| 原则 | 说明 |
|---|---|
| 只暴露行为，不暴露可变状态 | 状态给 readonly，修改走方法 |
| expose 集合保持最小 | 每多一个就是一份维护契约 |
| 文档化 | expose 之外的属性不属于公共 API |

与事件的关系：**连续交互流**（如打开后立刻 focus）用 expose；**离散通知**用 emit。

## 4. ref 透传组合场景

包装原生元素组件需要把 ref 转给内部元素：

```vue
<!-- MyInput.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const inputRef = ref<HTMLInputElement>()
defineExpose({ el: inputRef, focus: () => inputRef.value?.focus() })
</script>
```

## 5. 常见模式：命令式弹窗

```ts
// useDialog：组合 expose + Promise
const dialog = ref<DialogExpose>()
function openConfirm(): Promise<boolean> {
  return new Promise((resolve) => {
    dialog.value?.open((result: boolean) => resolve(result))
  })
}
```

## 常见坑

1. **$attrs 响应式误解**：解构缓存后值不更新；直接在模板/每次渲染取用。
2. **多根组件忘记处理 attrs**：class 等静默丢失。
3. **expose 了 reactive 状态本体**：父组件直接改导致子组件状态被外力破坏；用 readonly。
4. **父在子 mounted 前访问 ref**：`panel.value` 为 null，需 onMounted / nextTick。
5. **InstanceType 泛型组件失效**：泛型组件类型推导有限，显式声明 Expose 接口更稳。
6. **defineOptions 与 defineProps 顺序**：无强制顺序，但团队统一放最前提高可读性。

## Checklist

- [ ] 非根元素接收 attrs 的组件设置 inheritAttrs: false
- [ ] expose 仅暴露方法 + readonly 状态
- [ ] ref 访问全部有空值防护（?.）
- [ ] 未对 $attrs 做解构缓存
