# 01 - 快速上手与环境搭建

## 1. Vue3 是什么

Vue3 是渐进式 JavaScript 框架，核心特征：

| 特征 | 说明 |
|---|---|
| Composition API | 逻辑组织与复用的新范式，替代 mixin |
| 响应式重写 | 基于 Proxy，性能更强，支持 Map/Set，可监听动态属性 |
| 更好的 TS 支持 | 全部用 TypeScript 重写 |
| 性能提升 | 编译期优化（静态提升、PatchFlag），diff 更快 |
| Tree-shaking | 未用 API 可被摇掉，包体更小 |

## 2. 环境搭建

### 2.1 创建项目（官方推荐 Vite）

```bash
npm create vue@latest my-app
# 交互式选择：TypeScript / JSX / Router / Pinia / Vitest / E2E / ESLint / Prettier
cd my-app && npm install && npm run dev
```

### 2.2 最小依赖理解

```
vue          核心
vite         构建工具（开发 esbuild + 生产 rollup）
@vitejs/plugin-vue   编译 .vue SFC
vue-tsc      Vue 版 TS 类型检查
```

### 2.3 浏览器直接体验（CDN）

```html
<script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
<div id="app">{{ message }}</div>
<script>
  const { createApp, ref } = Vue
  createApp({ setup: () => ({ message: ref('Hello Vue3') }) }).mount('#app')
</script>
```

## 3. 第一个组件

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const double = () => count.value * 2
</script>

<template>
  <button @click="count++">点击了 {{ count }} 次，双倍是 {{ double() }}</button>
</template>
```

`<script setup>` 是 Vue3 的默认推荐写法：顶层变量/函数/导入的组件自动暴露给模板。

## 4. 应用实例

```ts
// main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)
app.use(router).use(pinia).mount('#app')
```

要点：
- `createApp` 每次调用返回**全新独立**的应用实例，多实例互不污染（Vue2 全局 API 污染问题已解决）
- 全局配置挂到实例上：`app.component / app.directive / app.provide / app.config.globalProperties`
- `mount` 必须最后调用

## 5. 两种 API 风格对比

| 维度 | Options API | Composition API |
|---|---|---|
| 组织方式 | 按选项分组（data/methods/computed） | 按逻辑功能分组 |
| 逻辑复用 | mixins（来源不清晰） | composables（显式导入） |
| 类型推导 | 较弱 | 天然友好 |
| 适用 | 简单项目、老项目 | 中大型项目、新项目 |
| Vue3 支持 | 完全支持 | 默认推荐 |

同一个逻辑两种写法的本质区别：**Options 按「API 类型」切代码，Composition 按「业务功能」切代码**。

## 6. 开发工具链

```bash
# 调试
npm i -D vite-plugin-vue-devtools   # Vite 内嵌 DevTools
# VSCode 必装插件
# Vue - Official (Volar)：语法高亮、类型提示、双栏预览
```

注意：Vue2 的 `Vetur` 与 Vue3 的 `Vue - Official` 冲突，只能装一个。

## 7. SFC 单文件组件结构

```vue
<!-- 布局块 -->
<template>…</template>

<!-- 逻辑块（可多个，普通 script 用于运行时配置） -->
<script setup lang="ts">…</script>

<!-- 样式块：scoped 局部作用域 / module CSS Modules -->
<style scoped lang="scss">…</style>
```

## 常见坑

1. **`mount` 必须在 `use` 之后**：先 mount 再 use 不会生效。
2. **`createApp` 不要放在循环里**：每次调用创建新实例，组件/指令需要重复注册。
3. **CDN 引入生产版要用 `vue.global.prod.js`**，否则体积大且有开发期警告。
4. **`<script setup>` 中导入即注册**，不需要 `components: {}`；写法迁移时最常忘的就是删掉 components 选项。

## Checklist

- [ ] 使用 `npm create vue@latest` 初始化，开启 TS + ESLint
- [ ] 安装 Vue - Official 插件并卸载 Vetur
- [ ] 明确团队统一使用 Composition API
- [ ] main.ts 中 use → mount 顺序正确
