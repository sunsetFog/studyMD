# 08 - 插槽系统

## 1. 三类插槽

```vue
<!-- 子组件 -->
<template>
  <header><slot name="header">默认标题</slot></header>  <!-- 具名 + 后备内容 -->
  <main><slot /></main>                                  <!-- 默认插槽 -->
  <footer><slot name="footer" /></footer>
</template>

<!-- 父组件 -->
<BaseLayout>
  <template #header>页头</template>
  默认内容（无 template 包裹的进默认插槽）
  <template #footer>页脚</template>
</BaseLayout>
```

## 2. 作用域插槽（核心能力）

插槽内容在**父组件作用域**编译，但默认插槽无法访问子组件内部数据——作用域插槽把子组件数据传出来：

```vue
<!-- 子 -->
<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      <slot :item="item" :index="i">
        {{ item.name }}   <!-- 后备内容可访问子作用域的 item -->
      </slot>
    </li>
  </ul>
</template>

<script setup lang="ts">
defineProps<{ items: Item[] }>()
</script>

<!-- 父 -->
<MyList :items="list">
  <template #default="{ item, index }">
    <span :class="{ active: item.id === currentId }">{{ index }}. {{ item.name }}</span>
  </template>
</MyList>
```

TS 类型化 slot props（子组件）：

```vue
<script setup lang="ts">
defineSlots<{
  default?(props: { item: Item; index: number }): any
  header?(props: {}): any
}>()
</script>
```

## 3. 具名插槽传给子组件数据的命名

具名作用域插槽：

```vue
<!-- 子 -->
<slot name="item" :item="item" />
<!-- 父 -->
<template #item="{ item }">…</template>
```

## 4. useSlots 动态插槽

```ts
import { useSlots } from 'vue'
const slots = useSlots()
slots.default?.({ item: someItem })   // 手动调用渲染函数形式的插槽（少见）
```

判断父是否传了某插槽：`!!slots.footer` → 条件渲染包装容器（如无 footer 就不渲染卡片底部）。

## 5. 高级模式

### 5.1 无渲染组件（renderless）

```vue
<!-- FetchData.vue：只提供数据，不渲染 -->
<script setup lang="ts">
const props = defineProps<{ url: string }>()
const data = ref(null)
onMounted(load)
defineExpose({ data, loading })
</script>
<template>
  <slot :data="data" :loading="loading" :reload="load" />
</template>

<!-- 使用 -->
<FetchData url="/api/users" v-slot="{ data, loading, reload }">
  <div v-if="loading">…</div>
  <UserTable v-else :users="data" />
  <button @click="reload">刷新</button>
</FetchData>
```

实际项目中更推荐 composables（见 `13-composables.md`），无渲染组件适合「插槽内容强依赖模板上下文」的场景。

### 5.2 列表项自定义（表格/列表通用模式）

```vue
<ProTable :columns="columns" :data="rows">
  <template #cell-status="{ row }">
    <el-tag :type="row.status === 1 ? 'success' : 'danger'">{{ row.statusText }}</el-tag>
  </template>
</ProTable>
```

### 5.3 透传插槽（包装组件）

```vue
<!-- Wrapper.vue 把所有插槽原样传给内层组件 -->
<template>
  <InnerComp>
    <template v-for="(_, name) in $slots" :key="name" #[name]="slotProps">
      <slot :name="name" v-bind="slotProps" />
    </template>
  </InnerComp>
</template>
```

## 6. 作用域与编译边界

- 插槽内容始终在**父作用域**执行：能访问父的数据，访问不到子组件的 state（除非通过 slot props）
- 插槽内容作为函数传入子组件，**惰性求值**：子不渲染就不执行

## 7. 插槽 vs props 选型

| 场景 | 用插槽 | 用 props |
|---|---|---|
| 传模板/结构（DOM 片段） | ✅ | ❌ |
| 传数据 | 可（slot props 场景反过来） | ✅ |
| 子渲染逻辑由父决定 | ✅ | 勉强（传 render 函数） |
| 简单文本/配置 | 后备插槽也行 | ✅ 更直接 |

## 常见坑

1. **作用域插槽内容里访问子组件数据**：必须通过 slot props，直接写子组件变量报 undefined。
2. **具名插槽忘记 `#name` 包 template**：内容进了默认插槽。
3. **动态插槽名大小写**：`#[dynamicName]` 中表达式结果须与子组件 name 完全一致。
4. **v-for 内的具名插槽**：每项都传同名插槽会互相覆盖，只渲染最后一个。
5. **默认插槽与后备内容混淆**：后备内容（slot 标签内部）只有父没传时才渲染，且后备内容在子作用域编译。
6. **无渲染组件过度使用**：简单场景 composables 更清晰。

## Checklist

- [ ] 插槽全部有 TS 类型（defineSlots）
- [ ] 需要条件的包装 UI 先判 slots 存在再渲染
- [ ] 包装组件正确透传插槽（v-for $slots 模式）
- [ ] 无渲染组件与 composables 使用边界明确
