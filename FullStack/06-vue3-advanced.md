# 06 - Vue3 进阶原理

## 1. 响应式原理：Proxy（必考⭐⭐）

```js
// Vue3 核心：Proxy 拦截 + 依赖收集(trigger/track)
const proxy = new Proxy(target, {
  get(t, key, receiver) {
    track(t, key);                       // 收集：哪个 effect 用了我
    const res = Reflect.get(t, key, receiver);
    return isObject(res) ? reactive(res) : res;   // 惰性深层代理
  },
  set(t, key, val, receiver) {
    const old = t[key];
    const res = Reflect.set(t, key, val, receiver);
    if (old !== val) trigger(t, key);    // 触发：更新用我的 effect
    return res;
  }
});
```

- 三大结构：`reactiveEffect`（副作用）、`targetMap: WeakMap<target, Map<key, Set<effect>>>`、调度器
- 为什么换掉 Object.defineProperty：可监听新增/删除属性、数组索引与 length、Map/Set；无需递归初始化（惰性）
- 弱点仍存：解构失联（toRefs 补救）、`Map/Set` 需要特殊处理、非对象无法代理（ref 包装解决）

## 2. 依赖收集细节

- 嵌套 effect：栈结构（activeEffectStack），收集当前活跃 effect
- cleanup：每次执行 effect 前移除旧依赖，解决条件分支导致的脏依赖
- computed：dirty 标记 + 惰性，作为特殊 effect 嵌入依赖图

## 3. diff 算法（必考⭐⭐）

Vue3 对比新旧子节点数组（带 key）：

```
1. 预处理：从头同步、从尾同步（处理常见首尾追加场景）
2. 中间乱序部分：
   a. 新建 key→index 映射
   b. 遍历旧中间节点：有 key 匹配→patch 并标记；无匹配→卸载
   c. 求最长递增子序列（LIS）：不需要移动的节点，其余节点最小化 DOM 移动
```

- 为什么用 LIS：DOM 移动开销大，LIS 保住最多"不动"的节点
- key 的作用：节点身份识别；用 index 作 key → 移动/删除时状态错位（输入框串行、动画异常）
- patch 标志位（PatchFlags）：编译时标记动态节点类型，运行时跳过静态内容——**编译优化**是 Vue3 diff 快的真正原因

## 4. 编译优化三件套

1. 静态提升（hoistStatic）：静态节点提到 render 外，只创建一次
2. PatchFlags + 动态节点收集（block tree）：diff 只走动态节点数组，跳过静态子树
3. 事件缓存（cacheHandler）：内联事件缓存，避免子组件无谓更新

## 5. KeepAlive 原理

- 缓存 vnode 到 Map（LRU，max 控制淘汰），命中时从缓存取、不重新创建
- 生命周期钩子：`onActivated / onDeactivated`
- include/exclude 按组件 name 匹配

## 6. 组件更新流程

```
数据变化 → trigger → 调度器入队 → 微任务批量 flush（nextTick 后）
→ componentUpdateFn → render 生成新 vnode → patch(prev, next)
→ 组件类型不同：卸载旧子树，挂载新子树
```

- `nextTick`：等 DOM 批量更新完成的 Promise
- 异步更新队列保证同一 tick 内多次修改只渲染一次

## 7. 性能优化清单（Vue 项目）

1. `v-if` vs `v-show` 按切换频率选
2. 大列表：虚拟滚动、`v-memo`、分页
3. 路由/组件懒加载（动态 import + 代码分割）
4. `shallowRef` 处理大对象、避免深层响应式开销
5. computed 缓存代替模板内复杂表达式
6. KeepAlive 缓存 tab 页
7. 图片懒加载、骨架屏、SSR/预渲染（见 10）

## 常见追问

1. Vue3 为什么不需要 `$set`？——Proxy 原生监听新增/删除
2. watch 监听 reactive 对象为什么隐式 deep？——遍历代理所有 key
3. Vue2 数组为什么用方法劫持？——defineProperty 无法监听索引，Vue2 重写 7 个数组方法

## Checklist

- [ ] 能画 targetMap 依赖结构
- [ ] 能完整讲 diff 五步（含 LIS 为什么）
- [ ] 说得出编译时优化如何配合运行时 diff
