# 32 - 编译优化机制

## 1. Vue3 编译器的定位

Vue3 性能优势的核心来源：**编译时知道模板结构，运行时不需要盲目 diff**。模板是受约束的语言（相比 JSX），编译器可静态分析出「哪些会变、哪些不会」。

```
.vue 文件 → @vitejs/plugin-vue → compiler-sfc → render 函数（带优化标记）
```

## 2. 三大编译优化

### 2.1 静态提升（hoistStatic）

```ts
// 模板
<div><p>静态文字</p><p>{{ msg }}</p></div>

// 编译产物
const _hoisted_1 = /*#__PURE__*/ createElementVNode('p', null, '静态文字', -1 /* HOISTED */)
function render() {
  return createElementVNode('div', null, [_hoisted_1, createElementVNode('p', null, toDisplayString(msg))])
}
```

静态节点提升到 render 外：每次渲染不再重建 VNode，且 patch 时跳过。连续 ≥20 个静态节点还会合并为一个字符串 VNode（`createStaticVNode`，innerHTML 一次写入）。

### 2.2 PatchFlag（补丁标记）

```ts
createElementVNode('p', null, toDisplayString(msg), 1 /* TEXT */)
// PatchFlags 枚举：
// 1 TEXT        仅文本动态
// 2 CLASS       仅 class 动态
// 4 STYLE       仅 style 动态
// 8 PROPS       动态属性（附 dynamicProps 数组指明哪些 key）
// 16 FULL_PROPS 有动态 key 的属性
// 32 HYDRATE_EVENTS
// 64 STABLE_FRAGMENT / 128 KEYED_FRAGMENT / 256 UNKEYED_FRAGMENT
// 512 NEED_PATCH / 1024 DYNAMIC_SLOTS
```

运行时收益：patch 时**只对比标记的部分**。`1 /* TEXT */` 直接跳过 props 对比只更新文本；`8 /* PROPS */` 只比 dynamicProps 列出的 key。

### 2.3 Block Tree（动态节点收集）

```ts
// 编译产物结构
function render() {
  return (_openBlock(), createElementBlock('div', null, [
    /* 静态节点被跳过收集 */
    createElementVNode('p', null, toDisplayString(msg), 1)
  ]))
}
// openBlock 开启收集上下文，每个带 PatchFlag 的节点自动登记进 dynamicChildren
// patch 时不再遍历整棵树，只 diff dynamicChildren（扁平数组）
```

Block 语义：v-if/v-for 等结构指令处产生新的 Block（因为子节点结构不稳定），dynamicChildren 按层级收集「所有动态后代」，diff 成本从 O(树大小) 降到 O(动态节点数)。

**代价**：dynamicChildren 依赖结构稳定——`v-html`、手动动态 slot 等破坏稳定性的地方编译器会标记 `STABLE_FRAGMENT` 失效回退全量 diff。

## 3. 事件缓存（cacheHandlers）

```ts
// 模板 <button @click="onClick">
const _cache = {}
createElementVNode('button', { onClick: _cache[0] || (_cache[0] = (...args) => onClick(...args)) }, 'x')
```

内联箭头函数不再每次渲染新建，避免子组件因 props 函数引用变化而无谓更新。

## 4. SSR 编译优化

SSR 模式编译输出不同：
- 静态内容直接拼接为 HTML 字符串（**跳过 VNode 创建**）
- 动态部分生成 `ssrRender` 函数
- 输出的静态部分不需要水合 diff（`createStaticVNode` 水合跳过）

这就是「SSR 直出快」的编译层原因。

## 5. 优化失效的写法

| 写法 | 后果 |
|---|---|
| `v-html` | NEED_PATCH，子树回退 |
| 动态组件 `<component :is>` | 组件内优化保留，但无静态提升收益于动态切换 |
| JSX / h() | 完全无编译优化，全量 diff |
| 手写 render 函数组件内嵌模板组件 | 子组件仍享受优化（边界在组件） |
| `v-for` 无 key | UNKEYED_FRAGMENT 全量对比 |

## 6. 运行时 vs 编译时 API 对照

```ts
// 编译时（推荐，享受优化）
import { createElementBlock, openBlock, createElementVNode } from 'vue'

// 运行时（无优化，库作者/特殊场景）
import { h, createVNode } from 'vue'
```

`h()` 是运行时通用入口，编译产物用更细的 `createElementVNode/createElementBlock`——两者生成的 VNode 兼容同一 patch 算法。

## 7. 查看编译产物的方法

- `vue - Official` 插件：SFC 分屏显示编译后的 render 代码
- `vite-plugin-inspect`：查看 Vite 转换后的模块
- SFC Playground（官方在线）验证优化标记

调优方法：写完动态结构后**看一眼产物里有没有 PatchFlag 与 hoisted**，没有就说明写法让优化失效了。

## 8. 与手写优化的关系

```
编译优化是自动的：写正常模板就有
手动优化是补充的：v-memo / shallowRef / 虚拟列表 —— 用于编译优化覆盖不到的运行时问题
```

先信任编译器，再考虑手动。

## 常见坑

1. **JSX 项目误以为性能等同模板**：无 patchFlag/block，大组件树注意拆分。
2. **v-html 包裹大段模板内容**：优化回退。
3. **运行时动态生成组件树（递归 h）无优化**：必要时接受成本或改模板递归组件。
4. **v-for 无 key 不报警告但性能劣化**：ESLint 强制 key。
5. **过度手动 v-once/memo 与编译优化叠加**：可读性下降收益微小。

## Checklist

- [ ] 能看懂 SFC 编译产物中的 PatchFlag
- [ ] 知道哪些写法导致优化失效
- [ ] 关键页面用 playground/inspect 验证过编译输出
- [ ] JSX 组件控制在动态渲染核心区
