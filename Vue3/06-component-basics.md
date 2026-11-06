# 06 - 组件基础：props 与 emit

## 1. 定义组件与注册

```vue
<!-- src/components/MyButton.vue -->
<script setup lang="ts">
defineProps<{ label: string }>()
const emit = defineEmits<{ (e: 'click', count: number): void }>()
</script>
<template>
  <button @click="emit('click', 1)">{{ label }}</button>
</template>
```

`<script setup>` 下**导入即注册**：

```vue
<script setup lang="ts">
import MyButton from './MyButton.vue'
</script>
<template><MyButton label="点我" /></template>
```

## 2. props 详解

### 2.1 声明与类型

```ts
// 运行时声明
const props = defineProps({
  title: String,
  size: { type: Number, default: 16, validator: (v: number) => v > 0 },
  required: { type: Boolean, required: true },
})

// 类型声明（推荐，编译期即可生成运行时校验）
const props = defineProps<{
  title: string
  size?: number
  items: Item[]
}>()

// 带默认值的类型声明
withDefaults(defineProps<{
  size?: number
  tags?: string[]
}>(), {
  size: 16,
  tags: () => [],       // 对象/数组默认值必须用工厂函数
})
```

### 2.2 命名规范

| 场景 | 规范 | 示例 |
|---|---|---|
| props 声明 | camelCase | `maxCount` |
| 模板中使用 | kebab-case | `:max-count="10"` |
| emit 事件 | kebab-case | `emit('item-click')` |

### 2.3 单向数据流

```
父 → props → 子（只读）
子 → emit → 父 → 更新 props
```

铁律：
- **子组件绝不修改 props**（含对象内部深改——能改但破坏契约，dev 有浅层警告）
- 需要「改」时：局部副本 / computed 包装 / emit 请求父改

```ts
// 局部副本模式
const local = ref(props.initial)
watch(() => props.initial, v => local.value = v)   // 父变化时同步

// computed 包装（配合 emit）
const checked = computed({
  get: () => props.modelValue,
  set: v => emit('update:modelValue', v),
})
```

## 3. emit 详解

```ts
// 校验式
const emit = defineEmits(['change', 'delete'])

// 类型式（推荐）
const emit = defineEmits<{
  change: [value: string]              // 3.3+ 元组语法
  'item-click': [id: number, e: MouseEvent]
}>()
emit('change', 'ok')
```

事件校验：`defineEmits({ submit: (payload) => !!payload.title })` 返回 false 仅警告不阻断。

## 4. attribute 透传

未在 props 中声明的属性（class/style/data-* 等）自动落到根元素：

```vue
<!-- 父：-->
<MyButton class="big" data-test="btn" />
<!-- 子根元素自动继承 class="big" data-test="btn" -->
```

关闭/接管：

```ts
defineOptions({ inheritAttrs: false })   // 3.3+
// 模板中手动放置
<span v-bind="$attrs">…</span>
```

多根节点组件**不会自动透传**，必须显式 `$attrs` 或警告。

## 5. 常见组件契约模式

```vue
<!-- 父组件使用 -->
<UserCard
  v-model:name="name"           <!-- 等价 :name + @update:name -->
  v-model:visible="show"
  :user="user"
  @confirm="onConfirm"
>
  <template #footer>…</template>
</UserCard>
```

## 6. props 校验进阶

```ts
defineProps({
  status: {
    type: String as PropType<'active' | 'disabled'>,  // 字面量联合
    required: true,
  },
})
```

运行时校验只对 dev 有警告，**不要把 validator 当业务校验**；关键契约放 TS。

## 7. defineExpose 与模板 ref

```ts
// 子
defineExpose({ focus: () => input.value?.focus() })
// 父
const child = ref<InstanceType<typeof Child>>()
child.value?.focus()
```

`<script setup>` 组件默认对外封闭（不暴露内部），必须显式 expose——这是对 Vue2 `$refs` 全开放的收紧，防止脆弱耦合。详见 `10-attrs-ref-expose.md`。

## 常见坑

1. **子组件直接改 props 对象**（`props.user.name = 'x'`）：不报错但破坏单向流，状态不可控。
2. **默认值不用工厂函数**：`default: []` 在多实例间共享引用（TS 声明式写法不存在此问题）。
3. **事件命名大小写**：DOM 模板中 `@myEvent` 被转小写监听不到；SFC 无此问题，但团队统一 kebab-case 最稳。
4. **布尔属性**：`:disabled="false"` OK，但 `disabled="false"` 是字符串 truthy → 禁用。
5. **多根组件忘记 inheritAttrs: false**：控制台警告且样式类丢失。
6. **props 解构**（`const { title } = props`）：脱响应，用 `toRefs(props)`。

## Checklist

- [ ] props/emit 全部 TS 类型声明
- [ ] 无子组件直接修改 props
- [ ] 默认值用工厂函数（对象/数组）
- [ ] 多根组件处理了 attrs 透传
- [ ] 对外命令式能力显式 defineExpose
