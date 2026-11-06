# 43 - 术语表与速查卡

## 1. 术语表

| 术语 | 含义 |
|---|---|
| SFC | Single-File Component，.vue 单文件组件 |
| Composition API | 基于 setup/composables 的组合式 API |
| Options API | 按 data/methods/computed 选项组织的 API 风格 |
| Composable | useXxx 命名的可复用有状态逻辑函数 |
| ref | 响应式容器，JS 中 `.value` 访问 |
| reactive | 基于 Proxy 的深层响应式对象 |
| shallowRef | 只追踪 `.value` 整体替换的浅响应 |
| markRaw | 标记对象永不做响应式代理 |
| toRefs / toRef | reactive 属性转 ref，解构保响应 |
| toValue | 归一化 ref/getter/普通值 |
| track / trigger | 依赖收集 / 触发更新（响应式内核） |
| effect | 响应式副作用（渲染/computed/watch 的底层） |
| dirty | computed 的脏标记（惰性重算机制） |
| VNode | 虚拟节点，渲染输出的描述对象 |
| patchFlag | 编译期打的动态类型标记 |
| Block / dynamicChildren | 编译期收集的动态节点集合，加速 diff |
| LIS | diff 中最长递增子序列优化 |
| flush | watch 回调时机（pre/post/sync） |
| hydration | SSR 后客户端接管已有 DOM |
| defineModel | 3.4+ 简化 v-model 的编译宏 |
| defineExpose | script setup 显式暴露实例成员 |
| inheritAttrs | 控制未声明 attrs 是否自动透传根元素 |
| Pinia / storeToRefs | 状态管理 / 解构 store 保响应 |
| NavigationFailure | router.push 被取消/重定向的返回结果 |
| Suspense | 协调异步子树加载的内置组件 |
| KeepAlive | 缓存组件实例的内置组件 |
| Teleport | 将 DOM 传送到其他容器的内置组件 |
| withInstall | 组件库插件化注册的辅助函数 |

## 2. 常用 API 速查

### 响应式
```
ref(x)  reactive(x)  shallowRef(x)  shallowReactive(x)
toRef(s,k)  toRefs(s)  toValue(x)  unref(x)
readonly(x)  shallowReadonly(x)  customRef(fn)  markRaw(x)
effectScope()  triggerRef(r)  isRef/isReactive/isProxy/isReadonly
```

### 副作用
```
computed(getter|{get,set})
watch(src, cb, {immediate, deep, flush, once})
watchEffect(fn(onCleanup))  watchPostEffect  watchSyncEffect
nextTick(fn)
```

### 生命周期
```
onBeforeMount  onMounted  onBeforeUpdate  onUpdated
onBeforeUnmount  onUnmounted
onErrorCaptured  onActivated  onDeactivated
```

### 组件宏
```
defineProps  defineEmits  defineModel  defineExpose
defineOptions  withDefaults  useSlots  useAttrs  useTemplateRef
```

### 编译宏不导入（编译器内置）；其余从 'vue' 导入。

## 3. 指令速查

```
v-if/v-else-if/v-else  v-show  v-for(:key)  v-model(.lazy/.number/.trim)
v-bind(:)  v-on(@)  v-slot(#)  v-html  v-text
v-once  v-memo([...])  v-pre  v-cloak
```

## 4. 事件修饰符速查

```
.stop .prevent .capture .self .once .passive
按键：.enter .esc .tab .delete …
系统键：.ctrl .alt .shift .meta .exact
```

## 5. 常见代码模式速记

```ts
// 双向绑定（3.4+）
const value = defineModel<string>({ default: '' })

// 解构 props 保响应
const { title } = toRefs(props)

// composable 通用入参
function useX(input: MaybeRefOrGetter<string>) { const v = toValue(input) }

// 竞态保护
let seq = 0
watch(src, async () => { const id = ++seq; const r = await api(); if (id === seq) out.value = r })

// markRaw 第三方实例
chart.value = markRaw(new ChartLib(el))

// store 解构
const { count } = storeToRefs(useCounterStore())
const { increment } = useCounterStore()

// 路由参数响应式
const id = computed(() => route.params.id)
watch(id, fetch)

// URL 状态
router.replace({ query: { ...route.query, page: String(p) } })
```

## 6. 一页速查：决策口诀

```
状态放哪？  刷新要留 → query / localStorage 白名单
            多页共享 → Pinia / 缓存 composable
            否则     → 组件本地

ref or reactive？  默认 ref；集合整体替换用 ref；局部深层用 reactive
watch or computed？ 派生值 computed；副作用 watch
v-if or v-show？   少切换 v-if；高频切换 v-show
指令 or 组件？     单元素行为增强指令；有模板结构组件
provide or Pinia？ 主题/上下文注入 provide；业务共享状态 Pinia
SSR or CSR？       要 SEO/首屏 → SSR(Nuxt)；中后台 → CSR
```

## 7. 高频坑位 TOP10 速记

1. 解构丢响应（reactive/props/store）
2. v-if + v-for 混用（优先级 Vue3 反转）
3. key 用 index
4. watch reactive 属性忘 getter
5. 子组件改 props
6. 第三方实例忘 markRaw
7. 异步 watch 无竞态保护
8. build 不跑 vue-tsc
9. token 进 localStorage
10. KeepAlive 页面 onMounted 刷新数据

---

全系列完（00-43）。建议结合 `42-best-practices.md` 在实际项目中落地，面试冲刺看 `38/39`。
