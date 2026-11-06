# 17 - 渲染函数与 JSX

## 1. 模板 vs 渲染函数 vs JSX

| 维度 | SFC 模板 | h() 渲染函数 | JSX |
|---|---|---|---|
| 表达力 | 受限（受编译器优化） | 完全 | 完全 |
| 编译优化 | ✅（PatchFlag 静态提升） | ❌ 运行时全量 diff | ❌ |
| 动态结构（v-for 套 v-if 复杂逻辑） | 繁琐 | 直接 | 直接 |
| 类型支持 | 好 | 一般 | 好（@vitejs/plugin-vue-jsx） |
| 适用 | 95% 业务组件 | 底层库 | 高动态组件/库组件 |

原则：**模板优先**；只有在模板无法表达（高度动态结构、递归渲染、程序化生成）时才下沉。

## 2. h() 基础

```ts
import { h } from 'vue'

const MyTitle = defineComponent({
  setup(_, { slots }) {
    return () => h('h1', { class: 'title', onClick: handler }, '标题')
    //          h(标签|组件, props, 子节点)
  },
})
```

参数规则：
- 第 1 参：字符串标签 / 组件对象 / 异步组件
- 第 2 参：props 对象（含事件 `onClick`，属性 camelCase）
- 第 3 参：字符串 / VNode 数组 / 插槽对象

## 3. VNode 与插槽

```ts
return () => h('div', [
  h('p', null, '文本'),
  h(ChildComp, { size: 'lg' }),                 // 组件
  slots.default?.(),                            // 透传默认插槽
  h('span', null, `计数 ${props.count.value}`), // setup 返回的 ref 需手动 .value（渲染函数不自动解包）
])
```

要点：渲染函数中 **ref 不自动解包**，必须 `.value`；模板函数返回的是一个 render 副作用（每帧调用）。

## 4. JSX 写法

```bash
npm i -D @vitejs/plugin-vue-jsx
# vite.config.ts
import vueJsx from '@vitejs/plugin-vue-jsx'
plugins: [vue(), vueJsx()]
```

```tsx
// App.tsx
import { defineComponent, ref } from 'vue'

export default defineComponent({
  props: { level: { type: Number, default: 1 } },
  setup(props, { slots }) {
    const count = ref(0)
    return () => (
      <props.level>                                  {/* ❌ 动态标签写法见下 */}
      </>
    )
  },
})

// 动态标签
const Tag = `h${props.level}` as any
return () => <Tag>{slots.default?.()}</Tag>
```

JSX 特殊约定：

| 模板 | JSX |
|---|---|
| `@click` | `onClick` |
| `:key` | `key` |
| `v-if` | 三元 / `&&` |
| `v-for` | `array.map(x => <li key={x.id}>)` |
| `v-model` | 手动 `value={x} onUpdate:modelValue={v => …}` |
| `<template #footer>` | `slots.footer?.()`（接收端）；传递端用对象形式 |

## 5. 典型场景：动态表单渲染器

```tsx
// SchemaForm.tsx —— 模板很难优雅表达的递归动态结构
export default defineComponent({
  props: { schema: { type: Object, required: true } },
  setup(props) {
    return () => renderNode(props.schema)
    function renderNode(node: SchemaNode): any {
      switch (node.type) {
        case 'group':
          return <fieldset>{node.children.map((c, i) => <div key={i}>{renderNode(c)}</div>)}</fieldset>
        case 'input':
          return <ElInput modelValue={node.value} onUpdate:modelValue={v => (node.value = v)} />
        case 'select':
          return <ElSelect modelValue={node.value} onUpdate:modelValue={v => (node.value = v)}
                            v-slots={{ default: () => node.options.map(o => <ElOption key={o.value} {...o} />) }} />
        default:
          return null
      }
    }
  },
})
```

递归、动态组件映射、程序化结构——模板写 `component :is` 套 v-for 也能做，但类型与逻辑表达明显劣化。

## 6. 函数式组件

```ts
// 无状态的轻量组件
const Badge = (props: { type: string }, { slots }) => h('span', { class: `badge-${props.type}` }, slots.default?.())

// JSX 形式
const Badge = ({ type, children }: any) => <span class={`badge-${type}`}>{children}</span>
```

适用：纯展示映射（无响应式状态）。有状态请用 defineComponent。

## 7. 性能注意

- 渲染函数放弃了编译优化，整个 render 每次 patch 全量生成 VNode——**只用于必要的动态区域**
- 大列表循环在 render 内生成 VNode 时，注意 key 稳定性
- 混合策略：外壳用模板（享受优化），动态核用函数式子组件

## 常见坑

1. **渲染函数中 ref 忘 .value**：显示 `[object Object]`。
2. **JSX 动态标签直接 `<props.level>`**：需要先取变量并 as any。
3. **事件名 camelCase**（onClick）而非 kebab-case。
4. **v-slots 传递**：JSX 中插槽用 `v-slots={{…}}` 或第三个参数对象，不能像模板那样写在 children。
5. **defineComponent 的 props 运行时声明**：JSX 场景类型声明用 `setup(props: { level: number })` 或泛型 defineComponent。
6. **渲染函数每次返回新数组导致 diff 开销**：稳定结构可静态提取。

## Checklist

- [ ] 模板能表达的不用渲染函数
- [ ] render 内 ref 全部 .value
- [ ] JSX 项目已装 plugin-vue-jsx 且 TS 配置 jsx: preserve
- [ ] 动态结构渲染器有类型定义（SchemaNode 联合类型）
- [ ] 高频动态区域评估过是否需要人工优化
