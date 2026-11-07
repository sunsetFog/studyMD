# 05 - Vue3 核心

## 1. 响应式 API（必考）

```ts
// ref：包装值类型，.value 访问
const count = ref(0); count.value++;

// reactive：对象代理，直接访问；解构失去响应式！
const state = reactive({ name: 'a' });
const { name } = state;          // ❌ 失去响应式（解构的是快照值）
const { name } = toRefs(state);  // ✅ 保持连接

// ref vs reactive 选型：组合式 API 风格下优先 ref（值/对象通吃、可整体替换）
```

- `shallowRef/shallowReactive`：只代理第一层（大对象性能）
- `readonly`：只读代理
- computed：惰性求值 + 缓存，依赖变化才重算，也是响应式的

## 2. 组合式 API 结构

```vue
<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue';
const props = defineProps<{ id: string }>();
const emit = defineEmits<{ (e: 'save', data: unknown): void }>();

const data = ref<User | null>(null);
const name = computed(() => data.value?.name ?? '');

watch(() => props.id, async (id) => {
  data.value = await fetchUser(id);
}, { immediate: true, deep: false });

onMounted(() => { /* DOM 就绪 */ });
</script>
```

- `<script setup>` 是编译糖：顶层变量自动暴露给模板，组件自动注册
- watch vs watchEffect：watch 显式指定源（精确、可拿旧值）；watchEffect 立即执行收集依赖（副作用）
- `watchEffect` 返回 stop 函数；异步场景注意清理（onCleanup）

## 3. 生命周期（setup 对应关系）

```
beforeCreate/created → setup 本身
beforeMount/onBeforeMount
mounted/onMounted            ← DOM 就绪，发请求常用
beforeUpdate/onBeforeUpdate
updated/onUpdated
beforeUnmount/onBeforeUnmount  ← 清定时器/监听
unmounted/onUnmounted
onErrorCaptured（错误边界）
```

组合式函数中 onMounted 可多次注册，按注册顺序执行。

## 4. 组件通信全家桶（高频）

| 场景 | 方案 |
|---|---|
| 父→子 | props |
| 子→父 | emit |
| 父访问子 | ref + `defineExpose` |
| 跨层注入 | provide / inject（默认非响应式，传 ref 即响应式） |
| 全局状态 | Pinia |
| 双向绑定 | defineModel（3.4+）/ v-model 语法糖 |
| 事件总线 | 替代方案：Pinia/mitt（Vue3 移除 $on） |

v-model 本质：`:modelValue` + `@update:modelValue`；多个 v-model：`v-model:title`。

## 5. 模板要点

```vue
<!-- 条件与列表 -->
<div v-if="ok">…</div>          <!-- 真正增删 -->
<div v-show="ok">…</div>        <!-- display 切换，适合高频切换 -->
<li v-for="u in users" :key="u.id">   <!-- key 必须稳定唯一，禁用 index（见 06 diff） -->

<!-- v-if 与 v-for 优先级：v-if 更高（3.x），不建议同标签混用 -->
<!-- 插槽 -->
<slot name="header" :data="row">  <!-- 作用域插槽：子传数据给父的插槽内容 -->
```

- `v-memo`：跳过子树 diff（大列表优化）
- `<Suspense>`：异步组件加载态
- `defineAsyncComponent`：代码分割

## 6. 常用内置组件

- `<Teleport to="body">`：渲染到指定 DOM（弹层脱离父级层级）
- `<KeepAlive include exclude max>`：缓存组件实例（见 06）
- `<Transition>/<TransitionGroup>`：过渡动画钩子

## 常见追问

1. ref 为什么需要 .value？——包装类让值类型也能被 Proxy 追踪
2. reactive 局限？——只能对象、解构失联、整体替换失效（ref 重新赋值即可）
3. computed 和 method 区别？——缓存 vs 每次执行
4. props 校验与单向数据流：子组件不能改 props，需 emit 或本地副本

## Checklist

- [ ] ref/reactive 选型与解构陷阱能讲清
- [ ] watch/watchEffect 能选对
- [ ] 通信方案能按场景列出
