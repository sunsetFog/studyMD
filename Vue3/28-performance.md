# 28 - 性能优化

## 1. 优化流程（先测量后动手）

```
1. 定位指标：LCP / INP / CLS（Web Vitals）+ 首屏接口耗时
2. 工具：Lighthouse、Performance 面板、bundle analyze
3. 瓶颈分类：加载（网络/包体）→ 渲染（组件树/重渲染）→ 交互（长任务）
4. 逐项修复后回归测量
```

反模式：无 profile 直接套「优化技巧」。

## 2. 加载性能

### 2.1 包体

| 手段 | 说明 |
|---|---|
| 路由懒加载 | 页面级分包（默认必做） |
| 重组件异步化 | 图表/编辑器 defineAsyncComponent |
| 按需引入 | element-plus（unplugin-vue-components 自动）、echarts 按需注册 |
| 换轻量库 | moment→dayjs、lodash→lodash-es（tree-shaking） |
| vendor 分包 | manualChunks 稳定库长缓存（见 25） |
| 压缩 | nginx/brotli 或 vite-plugin-compression |

### 2.2 资源

- 图片：webp/avif、尺寸合适、懒加载 `loading="lazy"`、大图 CDN 裁剪参数
- 字体：subset、`font-display: swap`
- 预加载：首屏关键接口 `preload`，路由 prerender 关键页

### 2.3 首屏

- 首屏数据并行请求，避免瀑布（async setup 内 Promise.all）
- 骨架屏（Suspense fallback）而非空白
- 关键 CSS 内联（Vite 默认提取）

## 3. 渲染性能

### 3.1 减少不必要的响应式开销

```ts
// 大数据/第三方实例
const bigData = shallowRef(list)          // 拒绝深层代理
const chart = markRaw(new ECharts(el))

// 组件对象引用
const currentTab = shallowRef(TabA)       // 组件对象不做深代理
```

### 3.2 列表优化

```vue
<!-- 万级列表：虚拟滚动 -->
<RecycleScroller :items="bigList" :item-size="56" key-field="id" />

<!-- 局部更新记忆化（精确场景） -->
<div v-memo="[item.id, item.selected]">…</div>
```

长列表三板斧：分页（首选）→ 虚拟滚动（体验要求高）→ v-memo（局部热点）。

### 3.3 减少组件数与深度

- 长列表项内避免大量深层嵌套组件（每个组件都是 diff/实例成本）
- 纯静态片段可 `v-once`（3.x 中静态内容编译期已提升，多数无需手写）

### 3.4 computed 缓存而非方法调用

见 `04-computed-watch.md`——模板中方法调用每次渲染执行，computed 有缓存。

## 4. 交互性能

| 问题 | 手段 |
|---|---|
| 输入触发高频计算 | watchDebounced / refDebounced |
| 滚动监听 | useEventListener + passive + throttle |
| 大计算阻塞 | Web Worker（vite 支持 new Worker） |
| 长任务拆分 | requestIdleCallback / 分批渲染 |
| 动画掉帧 | CSS transform/opacity（合成层），避免布局属性动画 |

## 5. KeepAlive 与路由级缓存

见 `12-transition-keepalive.md`：列表页缓存避免重复请求与重建；max 限制内存。

## 6. 按需响应式：数据量大时的取舍

```
展示型大数据（日志、表格原始行）→ shallowRef + 局部 ref 字段
频繁细粒度编辑（表单）→ 默认深层响应式
```

## 7. 内存泄漏排查

泄漏高发点（详见 `05-lifecycle.md` 清单）：
- 全局事件/定时器未清理
- KeepAlive 缓存页持有大 DOM/图表实例
-闭包持有大数组（watch 回调）
- ECharts/editor 实例未 dispose

工具：Memory 面板 heap snapshot 对比、DevTools Performance Monitor。

## 8. Web Vitals 目标参考

| 指标 | 目标 | 手段 |
|---|---|---|
| LCP < 2.5s | 首屏 | 分包/预加载/骨架 |
| INP < 200ms | 交互 | 降长任务/防抖/虚拟滚动 |
| CLS < 0.1 | 布局稳定 | 占位尺寸/字体 swap |

## 常见坑

1. **v-for 大列表无虚拟滚动**：万级节点 DOM 崩溃。
2. **deep watch 大数组**：每次变更全量遍历。
3. **echarts 实例被响应式代理**：卡顿 + 诡异行为，markRaw。
4. **首屏接口串行瀑布**：async setup 里顺序 await。
5. **图片未压缩/尺寸失控**：LCP 劣化主因。
6. **优化无测量**：改完不知道是否有效，甚至劣化。
7. **keepAlive 缓存无上限**：内存增长。

## Checklist

- [ ] 有一次完整的 profile 与 bundle 分析记录
- [ ] 路由/重组件懒加载就绪
- [ ] 万级列表有虚拟滚动或分页
- [ ] 第三方实例 markRaw + dispose
- [ ] 高频输入有防抖
- [ ] 首屏请求并行 + 骨架屏
- [ ] Web Vitals 有基线数据与目标
