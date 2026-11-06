# 36 - 面试题精讲（二）：原理与工程化

## Q1：Fiber 架构解决什么问题？两阶段是什么？

**要点**：
- 解决同步渲染阻塞长任务；把渲染拆为可中断单元（链表 return/child/sibling）
- Render 阶段（可中断/可丢弃，纯计算）+ Commit 阶段（同步原子：DOM 变更 + layout effects）
- 双缓冲树（current/workInProgress + alternate）

**加分**：lanes 优先级；commit 三子阶段；「渲染纯函数是并发前提」。

## Q2：Hooks 的实现原理？为什么不能条件调用？

**要点**：hooks 存在 fiber.memoizedState 按调用顺序的链表；render 时按顺序遍历取状态，顺序变化即错位。useState 的 update 队列在下次 render 应用（批处理本质）。

**加分**：能画出链表图；解释惰性初始化只在 mount 生效。

## Q3：Diff 算法的策略？

**要点**：三条假设（type 不同重建、同类型复用更新、key 标识列表）；children 两轮 diff（顺序比对 + key Map）；lastPlacedIndex 移动最小化。

**加分**：index 作 key 的错位案例推导；与 Vue3 编译优化的架构差异（运行时 vs 编译时）。

## Q4：React 并发渲染与 StrictMode 双执行？

**要点**：并发 = 可中断/可恢复/可丢弃；渲染可能多次执行（StrictMode 检测 + 低优作废重跑）→ 渲染必须纯。StrictMode 双挂载 effect 检测副作用幂等性。

## Q5：状态管理方案怎么选？

**要点**：分类驱动——服务端数据 → TanStack Query；低频全局 → Context；高频共享 → Zustand/Jotai；强规范集中式 → RTK；URL 状态 → searchParams。核心认知：服务端状态 ≠ 客户端状态。

**加分**：Zustand 基于 useSyncExternalStore 的精确订阅原理；RTK createSelector 记忆化。

## Q6：TanStack Query 解决什么问题？staleTime 与 gcTime？

**要点**：服务端状态缓存（缓存/去重/失效/重试/竞态内建）；staleTime 新鲜窗口（不重取），gcTime 无订阅保留时长；queryKey 包含全部输入；写后 invalidate 闭环；乐观更新（onMutate/onError 回滚/onSettled 失效）。

**加分**：key 工厂模式；keepPreviousData 翻页体验。

## Q7：性能优化方法论？

**要点**：先测量（Profiler「why render」）→ 加载（lazy/分包/图片）→ 渲染（状态下沉 > memo 三件套 > Context 拆分/状态库）→ 交互（transition/deferred/虚拟滚动）。强调「状态下沉最便宜有效」。

**加分**：虚拟滚动实现原理（可视窗口 + overscan）。

## Q8：SSR/RSC 的心智模型？

**要点**：App Router 默认 Server Component（服务端执行、直连数据、不进 bundle）；'use client' 边界最小化下沉；RSC 流式 + Suspense 注水；Server Actions 处理 mutations（内建 pending/渐进增强）。

**加分**：水合错误成因与修复；fetch 缓存语义版本差异。

## Q9：错误处理体系怎么搭？

**要点**：api 层统一 ApiError → Query 错误态组件化 → ErrorBoundary 路由级/区块级（只捕渲染错误）→ showBoundary 转交异步错误 → 全局 unhandledrejection 兜底 → 上报（指纹去重 + 版本 + sourcemap）。

## Q10：测试策略？

**要点**：Testing Library 哲学（测行为不测实现）；查询优先级 getByRole；userEvent > fireEvent；MSW 网络层；reducer/hooks 单元覆盖优先；CI vitest run。

## Q11：React 与 Vue 的本质差异？

**要点**：
- 更新模型：React 重算整组件 + 协调（运行时 diff）；Vue 依赖收集精确更新 + 编译期优化
- 模板：JSX（JS 全能力）vs SFC 指令（受约束但可编译优化）
- 生态：React 官方小核心生态自选 vs Vue 官方全家桶
- 双向绑定：React 无 v-model 语法糖（value+onChange）

**加分**：并发 vs 响应式两种「性能哲学」；React Compiler 与 Vue 编译优化殊途同归。

## Q12：项目工程化检查清单？

**要点**：TS strict + tsc 进 CI、react-hooks lint、目录分层（pages/components/hooks/queries/api）、Query 封装层、错误边界部署、测试金字塔、性能基线。

---

## 延伸练习

1. 手写迷你版 useSyncExternalStore 并解释 tearing
2. 设计一个电商列表页的状态分层方案（URL/Query/Zustand/本地）
3. 从 Fiber 视角解释「为什么渲染期禁止 setState」
