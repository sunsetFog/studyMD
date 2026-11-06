# 41 - 反模式清单

> 按「出现频率 × 危害」排序的 Vue3 反模式速查，每条给出症状与修复。

## A. 响应式类

### A1. 解构 reactive 丢响应
```ts
const { name } = state          // ❌
const { name } = toRefs(state)  // ✅
```

### A2. reactive 整体替换
```ts
state = reactive(newObj)    // ❌ 旧订阅全失效
// ✅ 用 ref，或 Object.assign(state, newObj)
```

### A3. watch reactive 属性忘 getter
```ts
watch(state.count, cb)          // ❌ 无响应
watch(() => state.count, cb)    // ✅
```

### A4. watchEffect 中 await 后读依赖
```ts
watchEffect(async () => {
  await fetch()
  doSomething(query.value)   // ❌ query 不再是依赖
})
// ✅ 同步段读取所有依赖
```

### A5. computed 里写状态/副作用
computed 必须纯；副作用进 watch。

## B. 组件类

### B1. 子组件直接修改 props（含对象深改）
单向数据流破坏；用 emit / defineModel / 局部副本。

### B2. props 直接解构
```ts
const { title } = props   // ❌ 脱响应
const { title } = toRefs(props)  // ✅
```

### B3. 大量 boolean props 组合爆炸
`<Modal :large :borderless :noHeader :compact>` → 拆组件或插槽变体。

### B4. expose 整个内部状态
只 expose 行为 + readonly 状态。

### B5. $parent / $children / DOM 查询找组件
显式契约（emit/provide/ref）。

### B6. 事件总线泛滥
业务组件互相 mitt → Pinia / 父中转。

## C. 模板类

### C1. v-if 与 v-for 同节点
Vue3 v-if 优先级更高，item 不可见 → computed 过滤 / template 分离。

### C2. v-for 用 index 作 key
有增删排序的列表必坏；稳定 id。

### C3. v-html 渲染未消毒输入
XSS；DOMPurify 或改插值。

### C4. 模板里写复杂表达式
`{{ list.filter(x => x.a && x.b).map(…).length > 3 && … }}` → computed。

### C5. TransitionGroup 无 key / 无 leave-active 绝对定位
动画跳动失效。

## D. 状态类

### D1. Pinia 当数据库
服务端列表全量塞 store → 请求缓存层（见 23）。

### D2. 组件直接改 store 状态
绕过 action，devtools 无记录 → action 收口。

### D3. 解构 store 不用 storeToRefs
丢响应。

### D4. 持久化敏感/超大字段
localStorage 白名单 + 脱敏。

## E. 性能类

### E1. 第三方实例不 markRaw
echarts/编辑器被深代理 → 卡顿与诡异行为。

### E2. 大列表无虚拟滚动
万级节点 diff 崩溃。

### E3. deep watch 大数组
全量遍历 → 精确 getter。

### E4. 首屏请求串行瀑布
async setup 顺序 await → Promise.all。

### E5. KeepAlive 无 include/max
全缓存内存膨胀。

## F. 工程类

### F1. build 不跑 vue-tsc
类型错误上线。

### F2. 组件库 vue 未 external
双 Vue 实例 → 响应式断裂。

### F3. 测试依赖真实网络
CI 必挂；msw mock。

### F4. 全局样式散落污染
scoped/BEM/CSS 变量。

### F5. 错误只有 console
全局 errorHandler + 上报（见 36）。

## G. 安全类

### G1. token 进 localStorage
XSS 可偷 → 内存 + refresh token。

### G2. 前端隐藏当权限
接口必须独立鉴权。

### G3. redirect 参数不校验
开放重定向。

## 使用方式

- Code Review 时按此清单扫描
- 新人入职第一周通读一遍
- ESLint 能拦截的（key/props 修改/v-if v-for 混用）配好规则兜底
