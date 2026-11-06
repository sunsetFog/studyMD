# 19 - 表单与 v-model

## 1. v-model 修饰符

| 修饰符 | 作用 | 备注 |
|---|---|---|
| `.lazy` | change 事件同步（失焦才更新） | 搜索框/重组件 |
| `.number` | 转数字（parseFloat，无法转换保留原值） | 数值输入 |
| `.trim` | 去首尾空格 | 文本输入 |

```vue
<input v-model.lazy="query" />
<input v-model.number="age" type="number" />
<textarea v-model.trim="bio" />
```

注意 `.number`：空字符串 → 保留原值；配合 `type="number"` 使用仍可能得到字符串（越界等），提交前应显式校验。

## 2. 各原生控件的绑定差异

| 控件 | v-model 绑定 | 多选 |
|---|---|---|
| text/textarea | string | — |
| checkbox 单个 | boolean | — |
| checkbox 多个 | array | ✅ `v-model="ids"` value 匹配 |
| radio | string（value） | — |
| select 单选 | string | — |
| select 多选 | array | ✅ multiple |

```vue
<input type="checkbox" v-model="agree" true-value="yes" false-value="no" />
<select v-model="city"><option disabled value="">请选择</option>…</select>
```

select 初始 value 不匹配任何 option 时显示空白——提供 disabled 占位 option 是标准做法。

## 3. 表单校验

### 3.1 手写轻量校验（小表单）

```ts
const form = reactive({ name: '', age: null as number | null })
const errors = computed(() => {
  const e: Record<string, string> = {}
  if (!form.name.trim()) e.name = '必填'
  if (form.age == null || form.age < 0) e.age = '年龄非法'
  return e
})
const valid = computed(() => Object.keys(errors.value).length === 0)
```

### 3.2 组合式校验函数（可复用）

```ts
// composables/useFormValidation.ts
export function useFormValidation<T extends Record<string, any>>(
  form: T,
  rules: { [K in keyof T]?: Array<(v: T[K]) => string | true> }
) {
  const touched = reactive({}) as Record<keyof T, boolean>
  const errors = computed(() => {
    const e: Partial<Record<keyof T, string>> = {}
    for (const k in rules) {
      for (const rule of rules[k]!) {
        const r = rule(form[k])
        if (r !== true) { e[k] = r; break }
      }
    }
    return e
  })
  const blurField = (k: keyof T) => (touched[k] = true)
  const showError = (k: keyof T) => touched[k] && !!errors.value[k]
  const validateAll = () => { Object.keys(form).forEach(k => (touched[k] = true)); return Object.keys(errors.value).length === 0 }
  return { errors, blurField, showError, validateAll }
}
```

### 3.3 库方案

- **Element Plus / Ant Design Vue / Naive UI**：自带 Form 组件 + async-validator 规则
- **VeeValidate**（生态最完整）：schema 校验（yup/zod）、字段级状态、提交管理
- **zod + 自研绑定**：类型与校验一体，TS 体验最佳

选型：管理后台用 UI 库自带；复杂多步表单/动态 schema 用 VeeValidate/zod。

## 4. 复杂表单模式

### 4.1 动态表单项（增删行）

```ts
const items = reactive([{ name: '', qty: 1 }])
const addRow = () => items.push({ name: '', qty: 1 })
const delRow = (i: number) => items.splice(i, 1)
```

```vue
<div v-for="(item, i) in items" :key="i">
  <!-- ⚠️ 可增删的行 key 用 index 有坑：删除中间行会串状态 -->
</div>
```

行内含输入类状态时，为行生成 uid 作为 key（`crypto.randomUUID()` 或自增 id），删除/插入不串行。

### 4.2 深层对象表单

```ts
const form = reactive({
  address: { province: '', city: '' },
})
form.address.province = '浙江'   // v-model="form.address.province" 直接可用
```

### 4.3 大表单拆分组件

```vue
<!-- 子组件用 defineModel（3.4+）简化 v-model 实现 -->
<script setup lang="ts">
const name = defineModel<string>('name', { required: true })
</script>
<template><input v-model="name" /></template>
```

`defineModel` 是 3.4+ 的标准写法：自动声明 prop + emit + 双向 ref。

## 5. 提交模式

```ts
const submitting = ref(false)
async function onSubmit() {
  if (!validateAll()) return
  submitting.value = true
  try {
    await api.submit(normalize(form))
    showToast('成功')
    resetForm()
  } catch (e) {
    showError(e)
  } finally {
    submitting.value = false   // 按钮解除禁用
  }
}
```

规范：
- 提交按钮 `:disabled="submitting || !valid"`（或 loading 防重复）
- 提交前 normalize（trim、number、格式化）
- 成功后的重定向/reset 明确

## 6. v-model 组件封装要点

见 `06-component-basics.md`。补充表单类组件通用契约：

```
props: modelValue, disabled, placeholder, rules?
emits: update:modelValue, change, blur, focus
expose: focus(), clear(), validate()?
```

## 常见坑

1. **动态行 key 用 index**：删除中间行后输入内容错位。
2. **`.number` 后直接当数字用**：空输入得到字符串，NaN 比较恒 false。
3. **v-model 直接绑 props 的属性**：改的是 props 对象内部，单向流被破坏；用 defineModel 或局部副本。
4. **校验时机**：只在 submit 校验（体验差）或 input 每键校验（打扰）；标准是 blur 校验 + submit 全量。
5. **checkbox 多选绑 undefined**：初始必须数组。
6. **select 忘记占位 option**：初始渲染空白无提示。
7. **表单状态未重置**：弹窗复用组件脏数据，打开时 reset 或使用 `v-if` 重建。

## Checklist

- [ ] 表单初始值完整（含数组类型）
- [ ] 动态行 key 非索引
- [ ] blur 校验 + submit 全量校验
- [ ] 提交防重（loading/disabled）+ normalize
- [ ] 弹窗/路由复用表单有 reset 机制
- [ ] 3.4+ 项目统一 defineModel 写法
