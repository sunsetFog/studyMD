# 25 - 性能优化

## 1. 优化流程（先测量）

```
1. 指标：LCP/INP/CLS + React Profiler 火焰图
2. 工具：React DevTools Profiler（记录渲染时长/次数）、Lighthouse、bundle analyze
3. 分类：加载（包体/网络）→ 渲染（重渲染风暴）→ 交互（长任务）
4. 改完回归测量
```

React DevTools Profiler 使用：录制交互 → 看「每次渲染耗时 + 渲染原因（Why did this render?）」→ 定位重渲染风暴源头。

## 2. 加载性能

| 手段 | 说明 |
|---|---|
| 路由级懒加载 | React.lazy + Suspense（非首页全 lazy） |
| 重组件异步化 | 图表/编辑器 lazy |
| vendor 分包 | manualChunks 稳定库长缓存（见 22） |
| 按需引入 | antd tree-shaking（ESM 天然）；echarts 按需注册 |
| 图片 | webp、`loading="lazy"`、CDN 裁剪 |
| 预加载 | `<link rel="preload">`、router loader 预取 |

## 3. 渲染性能：重渲染风暴治理

### 3.1 为什么会全树重渲染

```
父渲染 → 所有子组件重渲染（不看 props 是否变化）
Context value 变 → 所有子树重渲染
```

### 3.2 治理手段（按优先级）

```
1. 状态下沉：把变化的状态放到最小子树（最便宜最有效）
   ❌ 顶层 useState(kw) 全页重渲染
   ✅ 搜索框组件内自治 + onChange 回调上抛结果
2. 组件切分：渲染贵的子树用 memo 边界隔开
3. memo + useCallback/useMemo 三件套（props 引用稳定）
4. Context 拆分：高频/低频分箱（见 07）
5. 状态库精确订阅（Zustand selector / Jotai 原子）
```

```tsx
// 状态下沉示例
// ❌
const [kw, setKw] = useState('')
return <><SearchBar kw={kw} onChange={setKw} /><HugeList kw={kw} /></>
// ✅ SearchBar 内自治，仅提交时上抛
<SearchBar onSubmit={setKw} /><HugeList kw={kw} />   // 只在提交时影响列表
```

### 3.3 memo 的正确打开方式

见 `06-useref-usememo-usecallback.md`：memo 白包 = 白增加比较成本；三件套配套；React Compiler 未来自动化（见 10）。

## 4. 列表优化

```tsx
// 万级列表：虚拟滚动
import { useVirtualizer } from '@tanstack/react-virtual'
const rowVirtualizer = useVirtualizer({ count: items.length, getScrollElement: () => scrollerRef.current, estimateSize: () => 48, overscan: 10 })
// 只渲染可视窗口 ± overscan 的行

// 行组件 memo + 稳定 props
const Row = memo(RowInner)
```

长列表三板斧：分页（首选）→ 虚拟滚动（体验要求）→ 无限加载（feed 流）。

## 5. 交互性能

| 问题 | 手段 |
|---|---|
| 输入触发昂贵过滤 | useDeferredValue / startTransition（见 10） |
| 高频事件 | 防抖节流 + passive |
| 大计算 | Web Worker（Vite `new Worker`） |
| 重过渡 | startTransition 包切换，防 fallback 闪烁 |

```tsx
// 输入联动大列表的黄金组合
const [kw, setKw] = useState('')
const deferredKw = useDeferredValue(kw)
const isStale = kw !== deferredKw          // 可给列表加降透明度提示
const list = useMemo(() => bigFilter(deferredKw), [deferredKw])
```

## 6. 服务端数据性能（Query 层）

- staleTime 调优：默认 0 导致每次挂载重取；管理后台 30-60s
- `placeholderData: keepPreviousData`：翻页不闪 loading
- prefetch：路由 hover/可见时 `queryClient.prefetchQuery`
- devtools 确认请求去重与缓存命中

## 7. 内存泄漏排查

泄漏点（同 vue3/28 + React 特有）：
- effect 无 cleanup（订阅/定时器/Abort）
- ref 持有大对象跨渲染累积
- Portal 容器忘移除
- Zustand store 订阅（`store.subscribe` 手动订阅忘退订）

工具：Chrome Memory heap snapshot 对比、React Profiler 长录制剂崩溃点。

## 8. Web Vitals 目标

| 指标 | 目标 | React 手段 |
|---|---|---|
| LCP < 2.5s | 首屏 | lazy/分包/预加载/骨架 |
| INP < 200ms | 交互 | 状态下沉/transition/虚拟滚动 |
| CLS < 0.1 | 稳定 | 尺寸占位/字体 swap |

## 常见坑

1. **无 profile 就乱加 memo**：负优化（比较成本 + 代码噪音）。
2. **状态放太顶层**：一输入全页渲染；下沉优先于 memo。
3. **Context 高频状态**：见 07；换 Zustand。
4. **虚拟列表忘了 key/行高估计偏差**：滚动跳变。
5. **deferredValue 用在普通输入**：过度设计。
6. **Query staleTime 全 0**：请求翻倍。
7. **keepPreviousData 与 key 混淆**：理解占位数据语义。

## Checklist

- [ ] Profiler 录制过关键路径，有渲染原因分析
- [ ] 状态归属最小子树
- [ ] memo 三件套配套使用且有实测收益
- [ ] 万级列表虚拟滚动
- [ ] 输入联动有 transition/deferred
- [ ] Query staleTime 调优 + prefetch 关键路由
- [ ] Web Vitals 有基线
