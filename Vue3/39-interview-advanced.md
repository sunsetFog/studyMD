# 39 - 面试题精讲（二）：组件、工程化与原理

> 每题给出：考察点 → 参考答案（要点式）→ 加分项。

## Q1：v-model 在自定义组件上如何实现？3.4+ 有什么新写法？

**考察点**：组件契约理解。

**答案要点**：
- 编译为 `:model-value` + `@update:model-value`；子组件 props 接收 + emit 通知
- 多 v-model 用参数（`v-model:title`）；自定义修饰符经 `modelModifiers` props
- 3.4+ `defineModel()` 一行完成声明 + 双向 ref

**加分项**：能写出 defineModel 编译产物；能说出与 Vue2 `.sync` 的关系与差异。

## Q2：组件间通信方式全景与选型？

**答案要点**：props/emit、v-model、ref+expose、provide/inject、Pinia、事件总线（受控）。选型：层级浅 props/emit，跨层 provide/inject，全局 Pinia。

**加分项**：provide 必须 readonly + Symbol InjectionKey；指出 $parent 链式访问是反模式。

## Q3：Suspense 的工作机制与坑？

**答案要点**：
- 协调子树 async setup：全部 resolve 才渲染，否则 fallback；结构上可嵌套
- 不处理错误（onErrorCaptured）；路由变化需 `:key`
- 陷阱：最慢接口阻塞整页 → 只放「渲染前必须就绪」的依赖

**加分项**：能说出 fallback 防闪烁、async setup 并行请求（Promise.all）实践。

## Q4：Teleport 是什么？解决什么问题？

**答案要点**：DOM 传送到指定容器（body），组件上下文不变；解决浮层受父级层叠上下文/overflow 限制的问题。注意目标存在性（defer）、SSR 首包不含、z-index 全局规范。

## Q5：key 的作用？为什么不能用 index？

**答案要点**：
- key 是 diff 身份标识：同 key 同 type 复用更新
- index 作 key：插入/删除/排序时节点内容与 key 错位 → 状态串位（输入框）、非必要 DOM 移动
- diff 中 key 参与「最长递增子序列」计算，稳定 key 才能最小化移动

**加分项**：能手推一个 LIS diff 例子（见 `31-vdom-diff.md`）。

## Q6：Vue3 编译器做了哪些优化？

**参考**：`32-compiler-optimizations.md`。

**答案要点**：静态提升、PatchFlag（运行时只 diff 标记部分）、Block Tree（dynamicChildren 扁平化动态节点）、事件缓存、SSR 字符串拼接。JSX/h() 不享受这些优化。

**加分项**：能说出 PatchFlag 具体枚举含义（TEXT/CLASS/PROPS…）；能举一个让优化失效的写法（v-html）。

## Q7：虚拟 DOM diff 流程？

**答案要点**：
- 状态变 → 渲染 effect → 新 VNode → patch 对比
- 分派：type 不同直接换；同元素 patch props+children
- children 数组 keyed diff：头尾同步 → 仅增/仅删 → 中间乱序走 LIS 最小移动

**加分项**：组件更新的粒度（props 更新不必然触发子组件重渲染，取决于子组件响应式依赖）。

## Q8：Pinia 为什么取代 Vuex？storeToRefs 是什么？

**答案要点**：无 mutation、TS 友好、扁平多 store、组合式写法。storeToRefs 解构 state/getter 保响应（action 可直接解构）。

**加分项**：能解释「直接解构 store 丢响应」的原理（store 是 reactive 代理，解构即取值）；SSR 每请求新建 pinia。

## Q9：Vite 为什么快？与 webpack 区别？

**答案要点**：
- dev：esbuild 预构建 + 原生 ESM 按需编译，无 bundle 等待
- build：rollup
- 与 webpack 全量 bundle-dev 的本质区别；缓存（预构建产物缓存）

**加分项**：知道 dev/prod 差异带来的坑（CJS 依赖、动态 import 路径）。

## Q10：Vue 项目如何做性能优化？

**答案要点**（分层作答）：
- 加载：路由/组件懒加载、按需引入、分包缓存、图片优化
- 渲染：shallowRef/markRaw、虚拟滚动、computed 缓存、v-memo
- 交互：防抖节流、Web Worker、合成层动画
- 强调先测量（Lighthouse/analyze）再优化

**加分项**：给出 Web Vitals 目标与对应手段映射。

## Q11：Vue 项目的错误处理体系怎么搭？

**答案要点**：api 拦截统一处理 → onErrorCaptured 边界（路由级）→ app.config.errorHandler 全局 → unhandledrejection 兜底 → 统一上报（含版本/sourcemap）。

**加分项**：captured return false 防重复上报；重试指数退避。

## Q12：如何设计一个可复用的 composables？

**参考**：`13-composables.md`。

**答案要点**：MaybeRefOrGetter 入参 + toValue；返回 refs；自动清理；竞态保护；SSR 安全。

**加分项**：现场手写 useFetch/useStateMachine 展示。

## Q13：SSR 原理与水合错误？

**答案要点**：
- 服务端 renderToString + 数据预取进 payload；客户端 hydrate 接管
- 水合不一致原因：时间戳/随机/localStorage 首屏/时区
- 解法：ClientOnly、import.meta.client 分支、统一序列化

**加分项**：Nuxt routeRules 混合渲染模式；payload 复用避免重复请求。

## Q14：组件库怎么设计发布？

**答案要点**：业务语义分层（不重造基础 UI）、withInstall、exports/peerDependencies（vue external）、样式隔离与 CSS 变量主题、文档（Storybook）、semver。

**加分项**：说出「双 Vue 实例」这一典型事故与成因。

---

## 延伸练习

1. 手写 `withInstall` 并解释为什么组件库必须 external vue
2. 设计一个带缓存失效的列表页方案（URL query + TanStack Query + keepAlive）
3. 给 Q6 的回答准备一段「查看编译产物」的现场演示
