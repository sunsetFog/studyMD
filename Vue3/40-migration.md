# 40 - 迁移指南：Vue2 → Vue3

## 1. 迁移策略选择

| 策略 | 适用 | 说明 |
|---|---|---|
| @vue/compat 渐进迁移 | 中大型存量项目 | Vue2 API 兼容层，逐步关 flag |
| 整体重写 | 小项目/技术债严重 | 按新规范重写，不背旧包袱 |
| 新老并存（微前端/双版本） | 业务停不了 | 极少数场景 |

**不建议**长期停留在 Vue2.7（已 EOL 2023-12），安全补丁停更。

## 2. 全局 API 变化

```ts
// Vue2                          // Vue3
import Vue from 'vue'             import { createApp } from 'vue'
Vue.use(plugin)                   app.use(plugin)
Vue.component('x', X)             app.component('x', X)
Vue.directive('d', dir)           app.directive('d', dir)
Vue.prototype.$http = http        app.config.globalProperties.$http = http（推荐 provide）
new Vue({ render: h => h(App) })  createApp(App).mount('#app')
```

变化本质：全局配置 → 应用实例隔离（多实例不再互相污染）。

## 3. 语法变化清单

### 3.1 移除的特性

| 移除项 | 替代 |
|---|---|
| `$on / $off / $once`（事件总线） | mitt / 状态管理 |
| filter（管道） | computed / 方法 |
| `$children` | ref / provide |
| `slot-scope` 语法 | `v-slot`（2.6 已支持） |
| `functional` 组件 | 函数式组件写法 |
| `inline-template` | 默认插槽 |
| `$set / $delete` | 直接赋值/删除（Proxy 原生响应） |
| `v-on.native` | 直接 emit / $attrs 事件透传 |
| keyCode 修饰符（`.enter` 保留） | `@keyup.enter` 可用；数字 keyCode 移除 |
| 双向 `.sync` | `v-model:xxx` |

### 3.2 生命周期改名

```
beforeCreate/created → setup（合并）
beforeDestroy        → onBeforeUnmount
destroyed            → onUnmounted
```

### 3.3 异步组件写法

```ts
// Vue2: () => import('./x.vue')
// Vue3 高级配置必须 defineAsyncComponent({...})
```

### 3.4 v-model 变化

```
Vue2: value + @input（组件默认 model 选项可改）
Vue3: modelValue + @update:modelValue（多 v-model:xxx）
```

## 4. data/mixin 相关

- data 必须是函数（Vue2 全局 Vue 里可以对象，Vue3 统一函数）
- mixins 继续可用但**迁移期就应替换为 composables**
- 全局 mixin → app.mixin（基本应消失）

## 5. 响应式行为变化（隐形坑大户）

| 变化 | 影响 |
|---|---|
| 数组/对象直接操作 | `$set/$splice 特判代码可删，但依赖重写数组的 hack 失效 |
| Proxy 不代理原始对象 | 传入组件前的「快照引用」不再可变通知 |
| `this.$refs` 类型 | `<script setup>` 默认封闭，需 defineExpose |

## 6. 渲染函数变化

```ts
// Vue2: h('div', { attrs: {...}, on: { click }, class, style, props: {...} }, children)
// Vue3: h('div', { class, style, onClick, ...props }, children)  // 扁平化
```

- `attrs/on/props` 分组取消，全部平铺；事件 camelCase `onClick`
- children 全部进数组；字符串文本直接传
- slot 变为函数 `slots.default?.()`

## 7. v-for 与 v-if 优先级反转

Vue2：v-for 高；Vue3：v-if 高。迁移时**全部重查**混用写法。

## 8. 迁移 checklist（compat 流程）

```
1. 依赖升级：vue@3 + @vue/compat（阶段一）
2. 跑 lint（vue-compat 插件）收集不兼容清单
3. 按文件批量修：全局 API → 实例 API；移除 $on/filters/.sync
4. 逐个关闭 compat flag：RENDER_FUNCTION → COMPONENT_V_MODEL → ... → GLOBAL
5. 移除 @vue/compat，切换 build 别名
6. TS：逐步 .js → .ts，props 类型化
7. 回归测试 + 编译产物检查
```

## 9. 迁移中的典型事故

1. **事件总线依赖 $on**：Vue3 直接报错，需提前用 mitt 替换并盘点全部事件。
2. **第三方库仍是 Vue2 版**：查生态矩阵（element-ui → element-plus；vuex → pinia；vue-router 3 → 4）。
3. **UI 库全量替换工作量**：element-ui → element-plus 建议混合期共存（按路由分批替换）而非一次性切换。
4. **render 函数组件第三方库**（旧版图表包装）：签名全变，需升级或包一层。
5. **测速回归**：迁移后性能基线对比，Proxy 行为差异可能引发隐性重渲染。

## 10. 生态版本对照表

| Vue2 | Vue3 |
|---|---|
| vue-router 3 | vue-router 4 |
| vuex 3/4 | pinia |
| element-ui | element-plus |
| vue-cli | vite |
| vue-test-utils v1 | v2 |
| vue-property-decorator | `<script setup lang="ts">` |

## Checklist

- [ ] 迁移策略已选定且排期
- [ ] 生态依赖升级矩阵确认
- [ ] 事件总线/过滤器/.sync 清单盘点完毕
- [ ] v-if + v-for 混用全查
- [ ] compat flag 逐个关闭直至移除
- [ ] 迁移后性能与功能回归通过
