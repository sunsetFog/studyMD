# 35 - 业务组件库建设

## 1. 什么时候需要组件库

| 信号 | 说明 |
|---|---|
| ≥2 个项目复用同一批业务组件 | 提取收益 > 维护成本 |
| 团队 >5 人、多业务线 | 统一交互规范 |
| 有明确 UI 规范/设计系统 | 有契约才有一致性 |

单人小项目直接放 `components/business/`（见 `24-sfc-project-structure.md`），不要过早建库。

## 2. 分层结构

```
packages/
├── ui/               # 基础组件（与 element-plus 等配合，只做业务包装）
├── business/         # 业务组件（UserPicker / OrderCard / AmountText）
├── composables/      # 通用逻辑
├── utils/
└── styles/           # 设计 token（变量/主题）
```

原则：**基础 UI 交给成熟库，自建组件库专注「业务语义」**。从零手写 Button/Select 是负资产。

## 3. 业务组件设计规范

### 3.1 契约先行

```ts
// UserPicker.vue
defineProps<{
  modelValue: number | null
  deptId?: number
  multiple?: boolean
  disabled?: boolean
}>()
defineEmits<{ 'update:modelValue': [number[] | number | null] }>()
defineExpose({ clear: () => {}, focus: () => {} })
```

规则：
- 一个概念一个 prop（`multiple` 而非 `mode: 'single' | 'multi' | 'tags'` 的组合爆炸）
- 命名带业务语义（`AmountText` 而非 `MoneyDiv`）
- 所有文本可覆盖（默认插槽/props 提供默认值）

### 3.2 v-model 与表单集成

表单类组件统一 v-model + blur/change 事件（见 `19-forms-vmodel.md` 契约表），才能接入任意表单校验方案。

### 3.3 样式隔离与主题

```vue
<style scoped>  /* BEM 命名：.biz-user-picker__input */
```

设计 token 走 CSS 变量：

```css
:root { --biz-primary: #1677ff; --biz-radius: 8px; }
/* 组件内 */
button { background: var(--biz-primary); border-radius: var(--biz-radius); }
```

暗色主题 = 换一套变量，组件零改动。

## 4. 工程结构（monorepo）

```
my-biz-ui/
├── packages/ui/
│   ├── src/
│   │   ├── user-picker/
│   │   │   ├── index.ts          # 导出 + withInstall
│   │   │   └── UserPicker.vue
│   │   └── index.ts              # 全量导出
├── package.json（exports 字段）
└── vite.config.ts（lib 模式构建）
```

```ts
// withInstall 标准模式（兼容 app.use 与按需引入）
import type { App, Plugin } from 'vue'

export const withInstall = <T>(comp: T) => {
  const plugin = comp as T & Plugin
  plugin.install = (app: App) => {
    app.component((comp as any).name!, plugin as any)
  }
  return plugin
}
```

```ts
// vite lib 构建
build: {
  lib: { entry: 'src/index.ts', formats: ['es', 'cjs'] },
  rollupOptions: { external: ['vue'] },        // vue 必须 external
}
```

```json
// package.json exports
{
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js", "require": "./dist/index.cjs" } },
  "peerDependencies": { "vue": "^3.4.0" }
}
```

## 5. 按需引入支持

方案：全量包 + ESM 天然 tree-shaking（组件独立文件、无副作用标记）+ 提供 unplugin-vue-components resolver。

```ts
// resolver
export function BizUIResolver() {
  return {
    type: 'component',
    resolve: (name: string) => {
      if (name.startsWith('Biz')) return { name, from: 'biz-ui' }
    },
  }
}
```

## 6. 文档与演示

- **Storybook / Histoire**：组件隔离演示 + props 文档自动生成（defineProps 类型提取）
- 每个组件：用法示例、props/emits/expose 表格、边界说明
- 文档即验收标准：写不出演示的组件不要发布

## 7. 版本与发布

| 实践 | 说明 |
|---|---|
| semver 严格 | breaking 变更 major + 迁移指南 |
| changelog 自动生成 | conventional commits + release-please |
| peerDependencies 声明 vue/element-plus | 不捆绑宿主依赖 |
| dist 类型检查 | vue-tsc 生成 d.ts 保证 TS 体验 |

## 8. 治理机制

- 新组件进库流程：提案（场景/契约）→ review → 实现 + 文档 + 测试 → 发布
- 双向同步：业务项目里临时组件晋升流程明确
- 弃用策略：`@deprecated` 标记 + 两个大版本周期

## 常见坑

1. **vue 未 external**：打进产物，双 Vue 实例一切诡异问题（context 断裂、响应式失效）。
2. **忘 peerDependencies**：宿主 lock 里多份 vue。
3. **样式未隔离**：全局类名冲突；scoped + 前缀/BEM。
4. **发布含 .vue 源文件**：宿主构建器需处理；发布构建产物或用 sfc 转译。
5. **契约随意变更**：minor 里改 props 语义，下游静默出错。
6. **只发包不发文档**：组件库变考古现场。

## Checklist

- [ ] 业务语义组件分层清晰（不重造基础 UI）
- [ ] withInstall + exports + peerDependencies 规范
- [ ] 样式 scoped/BEM + CSS 变量主题
- [ ] 每组件有文档与可运行示例
- [ ] 版本语义化 + changelog
- [ ] ESM tree-shaking 与按需 resolver 可用
