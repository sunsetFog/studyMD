# 03 - ref 与 reactive

## 1. 两种响应式方案

| 维度 | ref | reactive |
|---|---|---|
| 包装对象 | 任意类型（含基本类型） | 仅对象/数组/Map/Set |
| 访问方式 | `.value` | 直接访问属性 |
| 解构/传递 | 保持响应式 | 解构后丢失响应式 |
| 替换整体 | `x.value = newObj` 可行 | 直接替换引用会丢失响应 |
| 推荐度 | **默认首选** | 局部状态、需要深层代理时 |

## 2. ref 详解

```ts
import { ref } from 'vue'

const count = ref(0)          // 基本类型
const user = ref({ name: 'a' }) // 对象内部会自动转 reactive
count.value++
user.value.name = 'b'         // ref 包对象时 .value 是 Proxy

// TS 泛型
const list = ref<string[]>([])
const data = ref<User | null>(null)
```

关键规则：
- 在 **JS 中必须 `.value`**，在 **模板中自动解包**（仅限顶层属性）
- `ref` 包装对象时，内部使用 `reactive` 转换深层属性

### 2.1 解包的边界（高频坑）

```ts
const obj = { foo: ref(1) }
// 模板中：{{ obj.foo }} ❌ 不解包（非顶层）
// 模板中：{{ obj.foo + 1 }} 也不行，需要 {{ obj.foo.value }}

// 解构会丢失顶层解包资格
const { foo } = obj
```

口诀：**只有 `setup` 顶层直接 return / script setup 顶层声明的 ref，在模板里才自动解包**。

### 2.2 reactive 数组解包

```ts
const arr = reactive([ref(1), ref(2)])
arr[0] // 模板/JS 中都会自动解包为 1（集合类元素自动解包）
```

## 3. reactive 详解

```ts
import { reactive } from 'vue'

const state = reactive({
  user: { name: 'a', tags: ['x'] },
  list: [1, 2, 3],
  map: new Map([['k', 1]]),   // Vue3 Proxy 原生支持 Map/Set
})

state.user.name = 'b'   // 深层响应式
state.list.push(4)
state.map.set('k2', 2)
```

### 3.1 reactive 的限制

```ts
let state = reactive({ n: 1 })
state = reactive({ n: 2 })  // ❌ 替换引用后，旧订阅全部失效

const { n } = state         // ❌ 解构即脱响应
state.count = 1             // ✅ Vue3 无 Vue2 的 $set 问题，新增属性天然响应
delete state.count          // ✅ 删除也响应
```

## 4. 响应式丢失的场景与解法

| 场景 | 问题 | 解法 |
|---|---|---|
| 解构 reactive | 属性脱钩 | `toRefs(state)` 再解构 |
| 传参给函数 | 值拷贝 | 传整个对象 / 传函数 getter |
| ref 混在普通对象属性 | 模板不解包 | 保持顶层声明 |
| props 解构 | 脱钩 | `toRefs(props)` |

```ts
import { toRefs, toRef } from 'vue'

const state = reactive({ a: 1, b: 2 })
const { a, b } = toRefs(state)   // a/b 都是 ref，.value 可写且回写 state
const c = toRef(state, 'c')      // 单属性转 ref，state 上无 c 也可（作为链接）
```

## 5. readonly 与 isXXX

```ts
import { readonly, isRef, isReactive, isProxy, isReadonly } from 'vue'

const ro = readonly(state)  // 深只读代理，写入时 dev 警告
isRef(count)    // true
isReactive(state) // true
isProxy(state)  // true（reactive 或 readonly 都算）
```

用途：**把暴露给外部的状态设为 readonly，只允许内部方法修改**——单向数据流在 JS 层的强制实现。

## 6. 响应式替换：数组与对象

```ts
// reactive 数组整体替换
const list = reactive(['a'])
// ❌ list = ['b']  语法错误（const）
// ✅ splice / length
list.splice(0, list.length, ...['b'])
// 或用 ref 包装数组：list.value = ['b']
```

实践建议：**集合类数据用 `ref` 包装**，替换、清空、重新赋值都自然。

## 7. 选型口诀

```
单值 / 需要整体替换 / 需要跨模块传递 → ref
深层嵌套的局部状态、不想写 .value     → reactive
暴露给子组件/外部的状态              → ref 或 readonly(reactive)
需要解构                            → toRefs
```

## 常见坑

1. **忘记 `.value`**：`if (count) {…}` 永真——ref 对象本身总是 truthy。开启 TS + ESLint 能减轻。
2. **reactive 解构丢响应**：老项目混写时最常见 bug。
3. **reactive 整体替换引用**：视图不再更新，且难以排查。
4. **ref 嵌套在非响应式对象里**：修改 `.value` 不触发视图（外层对象不是 Proxy）。
5. **Map/Set 用 reactive 可以**，但 watch 时注意 `deep: true`（默认对 reactive 已深层）。
6. **props 上直接赋值**：`props.title = 'x'` 只改本地不通知父层，还会 dev 警告。

## Checklist

- [ ] 默认使用 ref，团队统一 `.value` 风格
- [ ] 无对 reactive 的解构/重赋值操作
- [ ] 对外暴露的状态用 readonly 保护
- [ ] 集合类型（数组）统一用 ref
- [ ] props 从不直接赋值，需要时用局部副本
