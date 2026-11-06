# 30 - 响应式原理

## 1. Vue2 vs Vue3 响应式

| 维度 | Vue2 (Object.defineProperty) | Vue3 (Proxy) |
|---|---|---|
| 新增/删除属性 | 需 $set/$delete | 原生支持 |
| 数组索引/length | 需重写数组方法 | 原生支持 |
| Map/Set | 不支持 | 支持 |
| 嵌套对象 | 初始化递归（昂贵） | 访问时惰性代理 |
| 监听范围 | getter/setter 拦截 | 13 种 trap 全拦截 |

## 2. 核心三件套：track / trigger / effect

```
副作用（effect）执行 → 读取响应式数据 → track 收集（谁在读）
数据变化 → trigger 触发 → 通知所有依赖它的 effect 重新执行
```

- effect：一切响应式更新的载体——组件渲染函数、computed getter、watch 回调本质都是 effect
- track 发生在 **get / has / iterate** trap 中
- trigger 发生在 **set / deleteProperty / add** trap 中

## 3. 依赖收集数据结构

```
targetMap: WeakMap<target, Map<key, Set<effect>>>
      ↓ 全局 WeakMap（target 被回收则自动清理）
target → key → Set(ReactiveEffect)
```

嵌套结构：每个对象属性的依赖是一个 effect Set；effect 执行时建立「属性 → effect」双向关联（effect.deps 反向记录，便于清理）。

## 4. Proxy 实现要点

```ts
// 简化版 reactive
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key, receiver) {
      track(target, key)                       // 依赖收集
      const res = Reflect.get(target, key, receiver)
      if (isObject(res)) return reactive(res)  // 惰性深层代理（访问到才代理）
      return res
    },
    set(target, key, value, receiver) {
      const old = target[key]
      const res = Reflect.set(target, key, value, receiver)
      if (old !== value && (hasChanged)) trigger(target, key)  // 触发更新
      return res
    },
  })
}
```

关键设计：
- **Reflect + receiver**：保证 this 指向代理对象（继承/getter 场景正确）
- **惰性代理**：嵌套对象访问到才 reactive，初始化 O(1)
- **同一对象只代理一次**（WeakMap 缓存代理映射，`isReactive` 判断依据）

## 5. effect 调度与异步批处理

trigger 通知 effect 时不是立即执行，而是经过 **scheduler**：

```
组件更新 effect → 排入微任务队列（Promise.then / queueJob）
→ 同一 tick 内多次修改只触发一次更新（去重 + 按 id 排序父子顺序）
```

这就是「Vue3 更新是异步批处理」的机制来源，也是 `nextTick` 存在的原因。

## 6. computed 实现（dirty 标记）

```
computed 内部维护 dirty 标记 + 自己的 effect：
1. 读取 computed.value → dirty 则重新计算（内部 effect 依赖其依赖）
2. 依赖变化 → 不立即重算，仅置 dirty = true 并通知订阅方
3. 订阅方（渲染 effect）收到通知 → 读取 computed → dirty 才重算
```

两层脏检查保证：依赖变了但没人读 computed 就不白算——这是「computed 惰性」的原理。

## 7. watch 实现

watch = 包装 effect + job 调度：

```
watch(getter, cb)：
- 创建 effect(run 时只做依赖收集，不做回调)
- 依赖变化 → effect 入调度队列（flush: pre/post/sync 决定时机）
- 执行 cb(newVal, oldVal)，oldVal 通过 getter 的旧值快照获得
```

flush: 'pre' 在组件更新前、'post' 在更新后，通过 scheduler 与渲染 job 的排序控制。

## 8. ref 实现

```ts
function ref(value) {
  return {
    get value() { track(refObj, 'value'); return rawValue },
    set value(v) { if (changed) { rawValue = v; trigger(refObj, 'value') } },
  }
}
// 类属性访问器实现依赖收集；对象值内部再 reactive() 转换
```

## 9. 为什么 WeakMap

targetMap 用 WeakMap：target（原始对象）无引用时整个依赖映射被 GC，无泄漏。手写响应式时用 Map 会泄漏。

## 10. 副作用清理

effect 重跑前遍历自身 deps，从各属性依赖集合中**清除自己**，再重新收集——解决条件分支导致的「死依赖」：

```ts
effect(() => {
  if (state.a) trackRead(state.b)   // a 为 false 时 b 不应成为依赖
})
```

## 常见坑（原理层解释）

| 现象 | 原理 |
|---|---|
| 解构 reactive 丢响应 | 解构发生时是普通值读取，后续访问不经过 trap |
| await 后 watchEffect 不追踪 | 依赖收集只在同步执行段（activeEffect 存续期） |
| 数组 length 触发两次 | length 与索引双向依赖（iterate + get） |
| shallowRef 深改不更新 | 深层对象未被代理，无 trap 无 track |
| $attrs 修改不响应 | 非 Proxy，无依赖收集 |

## Checklist

- [ ] 能画出 targetMap 结构与 track/trigger 流程
- [ ] 理解调度队列与 nextTick 关系
- [ ] 能解释 computed 的 dirty 两层机制
- [ ] 能用原理解释日常响应式丢失问题
