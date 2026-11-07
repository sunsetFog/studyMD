# 09 - Vue3 vs React 深度对比（高频开放题）

## 1. 核心差异一览（答题框架）

| 维度 | Vue3 | React |
|---|---|---|
| 响应式 | 依赖追踪（Proxy，精准更新） | 不可变 + setState，组件级重渲染 |
| 更新粒度 | 组件 + 编译期标记的动态节点 | 组件整棵子树 diff（需 memo 拦截） |
| 模板 | SFC 模板（编译优化空间大） | JSX（完全 JS 表达力） |
| 状态逻辑复用 | 组合式函数 composables | 自定义 Hooks |
| 心智模型 | 可变数据，自动收集依赖 | 快照不可变，显式声明依赖 |
| 元框架 | Nuxt | Next.js |

## 2. 响应式实现差异（必展开）

- **Vue**：读时收集依赖，写时触发更新 → 知道"哪个数据被哪个组件用"
- **React**：状态变化 → 该组件及其子树重新执行 render → vdom diff 找差异 → React 不知道"谁依赖了谁"，靠比较
- 引申差异：
  - React 需要 key/memo/useMemo 手动减少比较成本；Vue 编译期 PatchFlags 自动跳过静态
  - React 的 deps 数组、闭包旧值问题在 Vue composables 中不存在（依赖自动追踪）

## 3. 编译时 vs 运行时

- React：运行时为主（JIT 心智），React Compiler 开始补编译优化
- Vue：编译时+运行时结合（模板可静态分析）→ 静态提升/块树/缓存事件
- Svelte 走纯编译极端（无 vdom），可作为知识面加分项

## 4. 生态对照表

| 能力 | Vue | React |
|---|---|---|
| 路由 | Vue Router | React Router / TanStack Router |
| 服务端状态 | TanStack Query | TanStack Query |
| 全局状态 | Pinia（官方） | Zustand/Redux |
| 表单 | VeeValidate | react-hook-form |
| SSR | Nuxt | Next.js |
| 移动端 | uni-app | React Native |

## 5. 选型答题模板

```
1. 团队既有经验与代码资产（最大权重）
2. 业务形态：后台管理 → Vue3 + Element Plus 生态成熟；
   复杂交互/跨端 → React 生态广（RN/桌面）
3. 招聘市场与社区规模
4. 结论先行：两者都能做好业务，工程化差距小于团队因素
```

## 6. 追问弹药库

1. "你说 React 粗粒度，那 memo 不就解决了？"——memo 需要全链路引用稳定（useCallback/useMemo），心智负担；Vue 默认精准
2. "Vue 的自动依赖收集有没有坑？"——解构失联、非响应式来源不追踪（ref 兜底）
3. "JSX vs 模板谁强？"——JSX 表达力强、类型推导好；模板上手快、约束带来编译优化
4. Hooks vs composables 相同点：逻辑复用单位、无 this；差异：依赖声明方式

## Checklist

- [ ] 能从响应式实现推导出所有表层差异
- [ ] 给出结构化选型框架而非站队
