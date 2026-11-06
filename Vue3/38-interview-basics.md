# 38 - 面试题精讲（一）：基础与响应式

> 每题给出：考察点 → 参考答案（要点式）→ 加分项。

## Q1：ref 和 reactive 的区别？为什么默认推荐 ref？

**考察点**：响应式 API 本质理解。

**答案要点**：
- ref 是基于属性访问器的容器（`.value`），支持任意类型；reactive 是 Proxy 代理，仅对象类
- ref 整体替换、解构（toRefs 后）安全；reactive 解构/重赋值丢响应
- ref 统一心智模型（单值/对象/集合一致），代码风格一致；Vue 官方风格指南推荐

**加分项**：能说出 ref 内部对对象值调用 reactive 的实现细节；能解释解包只在顶层 render 上下文发生。

## Q2：Vue3 响应式原理？为什么用 Proxy 替代 defineProperty？

**参考**：见 `30-reactivity-internals.md`。

**答案要点**：
- track/trigger/effect 三机制 + targetMap(WeakMap) 结构
- Proxy 13 种 trap 全拦截：新增/删除属性、数组索引/length、Map/Set；惰性深层代理
- 依赖收集发生在 getter，触发发生在 setter；副作用经调度队列异步批处理

**加分项**：WeakMap 防 GC 泄漏；Reflect+receiver 保 this 正确；条件分支依赖清理机制。

## Q3：computed 和 watch 的区别？何时选谁？

**答案要点**：
- computed：派生值，缓存 + 惰性（dirty 两层机制），必须纯
- watch：副作用，拿到新旧值，flush 控制时机，可清理（onCleanup）
- 原则：能 computed 不 watch；watch 处理请求/DOM/存储等副作用

**加分项**：解释 computed 的 dirty 依赖传播（依赖变→置脏→订阅方读时才重算）；watchEffect 的 await 边界陷阱。

## Q4：v-if 和 v-show 的区别？

**答案要点**：v-if 真实增删（触发生命周期、初始为假不渲染）；v-show display 切换（初始总渲染）。切换频繁用 v-show，条件渲染用 v-if。

**加分项**：两者与 diff/KeepAlive 组合的行为差异；v-if 与 v-for 优先级 Vue3 中反转（v-if 高）。

## Q5：Vue3 的 keepAlive 与 activated/deactivated？

**答案要点**：缓存组件实例（不销毁），切换触发 activated/deactivated；include/exclude/max（LRU）；典型坑是缓存页数据刷新应放 onActivated。

## Q6：watchEffect 和 watch 的区别？

**答案要点**：watchEffect 立即执行、自动收集同步段依赖、无旧值；watch 显式源、旧值、更精确。异步副作用务必注意 onCleanup 与 await 后不追踪。

## Q7：nextTick 的原理与使用场景？

**答案要点**：
- Vue 更新是异步批处理：修改状态后 DOM 未更新，nextTick 回调在 DOM 更新后执行
- 实现：微任务（Promise.then）队列，与渲染 effect 同一调度体系
- 场景：修改后立即读 DOM（测量、滚动、第三方库 refresh）

**加分项**：能说明「同一 tick 多次修改状态只渲染一次」由 job 去重实现。

## Q8：Vue2 到 Vue3 的重大变化？

**答案要点**：
- Composition API；响应式 Proxy 重写
- 性能：编译优化（patchFlag/静态提升/block）、包体 tree-shaking
- Fragment/Teleport/Suspense
- 生命周期改名；全局 API 改 createApp 实例化；移除 filter/$on 等旧特性
- 更完整的 TS 支持

## Q9：为什么 Vue3 中 v-for 与 v-if 同标签连用是坑？

**答案要点**：Vue3 中 v-if 优先级**更高**，其作用域内拿不到 v-for 的 item（Vue2 反之），导致报错或逻辑错乱。正确做法：computed 过滤 或 template 包裹分离。

## Q10：toRef / toRefs / toValue 各做什么？

**答案要点**：toRefs 把 reactive 每个属性转成 ref（解构保响应）；toRef 单属性（可不存在属性，建立链接）；toValue 归一化 ref/getter/值，是 composable 参数兼容的核心。

## Q11：什么是响应式丢失？列举场景与解法。

**答案要点**：解构 reactive、reactive 重赋值、ref 嵌套于普通对象非顶层、props 直接解构。解法：toRefs、shallowRef+替换、保持顶层声明、toRefs(props)。能引到 Proxy trap 层面解释（解构后访问不经过 trap）为最佳。

## Q12：shallowRef/markRaw 什么时候用？

**答案要点**：大数据/组件引用用 shallowRef（深代理开销大且无必要）；第三方实例（echarts/编辑器）、class 实例 markRaw 防代理。配合 triggerRef 手动通知。

---

## 延伸练习

1. 手写一个 `computed`（dirty 机制，10 行内）
2. 解释 `watch(() => state.x, cb)` 与 `watch(state.x, cb)` 的行为差异
3. 设计一个「防抖 v-model」：比较 customRef 与 watchDebounced 两种实现的取舍
