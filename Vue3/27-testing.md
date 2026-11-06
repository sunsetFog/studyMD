# 27 - 单元测试

## 1. 测试栈

```
Vitest          测试框架（Vite 同源，速度与配置复用）
@vue/test-utils 组件挂载与交互
jsdom/happy-dom DOM 环境
msw             网络层 mock（拦截 fetch/xhr）
Playwright      E2E（单独任务）
```

```bash
npm i -D vitest @vue/test-utils happy-dom
# vite.config.ts 或 vitest.config.ts
test: { environment: 'happy-dom', globals: true }
```

## 2. 测试金字塔（前端）

| 层 | 占比 | 对象 |
|---|---|---|
| 单元（utils/composables/store） | 60% | 纯逻辑，快而稳 |
| 组件测试 | 30% | 关键交互组件 |
| E2E | 10% | 主链路（登录/下单） |

原则：**优先测逻辑与 composables**（不需要挂载），组件测试只覆盖复杂交互。

## 3. 工具函数测试

```ts
// utils/price.test.ts
import { describe, it, expect } from 'vitest'
import { formatPrice } from './price'

describe('formatPrice', () => {
  it('千分位与两位小数', () => {
    expect(formatPrice(1234567.891)).toBe('1,234,567.89')
  })
  it('负数', () => { expect(formatPrice(-0.5)).toBe('-0.50') })
  it('非法输入', () => { expect(formatPrice(NaN)).toBe('0.00') })
})
```

## 4. composables 测试

```ts
import { ref } from 'vue'

// 不依赖组件上下文的 composable 直接测
it('useFiltered 基础过滤', () => {
  const list = ref([{ name: 'apple' }, { name: 'banana' }])
  const kw = ref('ap')
  const { filtered } = useFiltered(list, kw)
  expect(filtered.value).toHaveLength(1)
  kw.value = 'na'
  expect(filtered.value).toHaveLength(1)   // 响应式联动验证
})
```

依赖生命周期（onMounted）的 composable 需要 `withSetup` 辅助：

```ts
function withSetup<T>(composable: () => T): { result: T; app: App } {
  let result!: T
  const app = createApp({ setup() { result = composable(); return () => null } })
  app.mount(document.createElement('div'))
  return { result, app }
}
```

## 5. 组件测试

### 5.1 挂载与断言

```ts
import { mount } from '@vue/test-utils'
import Counter from './Counter.vue'

it('点击计数', async () => {
  const wrapper = mount(Counter, { props: { step: 2 } })
  expect(wrapper.text()).toContain('0')
  await wrapper.find('button').trigger('click')
  expect(wrapper.text()).toContain('2')
})
```

### 5.2 props/emit 验证

```ts
const wrapper = mount(MyInput, { props: { modelValue: '' } })
await wrapper.find('input').setValue('abc')
expect(wrapper.emitted('update:modelValue')?.[0]).toEqual(['abc'])
```

### 5.3 mock 依赖

```ts
import { vi } from 'vitest'

vi.mock('@/stores/user', () => ({ useUserStore: () => ({ name: 'test' }) }))
// 或 stub 组件
mount(Page, { global: { stubs: { HeavyChart: true } } })
```

- store：直接给假 store 或 mock 模块
- 子组件：stubs（只验证交互不渲染内部）
- 网络：msw 拦截（推荐，比 mock axios 更真实）

## 6. 测试什么 / 不测什么

| ✅ 测 | ❌ 不测 |
|---|---|
| 工具函数边界值 | 第三方库 |
| 校验规则、格式化 | 样式 |
| store action 状态流转 | 纯静态模板 |
| 表单校验与提交流程 | UI 库内部 |
| 权限守卫逻辑 | 简单透传组件 |
| 路由跳转条件 | 框架行为 |

## 7. Pinia 测试

```ts
import { setActivePinia, createPinia } from 'pinia'

beforeEach(() => { setActivePinia(createPinia()) })

it('cart addItem', async () => {
  const cart = useCartStore()
  await cart.addItem({ id: 1, qty: 2 })
  expect(cart.items).toHaveLength(1)
  expect(cart.totalCount).toBe(2)
})
```

## 8. 异步与时序

```ts
it('加载完成后渲染列表', async () => {
  vi.useFakeTimers()
  const wrapper = mount(List)
  await flushPromises()                 // 等待所有 microtask/ promise
  expect(wrapper.findAll('li')).toHaveLength(3)
  vi.useRealTimers()
})
```

`flushPromises` 是异步组件测试的关键（nextTick + 宏任务清空）。

## 9. CI 集成

```json
"scripts": {
  "test": "vitest run",
  "coverage": "vitest run --coverage"
}
```

- CI 中 vitest run（非 watch）
- coverage 阈值：utils 80%+，组件不做硬性门槛
- 测试文件命名统一 `*.test.ts` / `*.spec.ts`，放源码旁

## 常见坑

1. **组件测试过细**：断言内部实现（class/结构），重构即碎；断言「用户可见行为」。
2. **忘记 setActivePinia**：store 测试互相污染。
3. **fake timer 与异步库冲突**：注意 restore。
4. **测试依赖网络**：一律 msw/mock，CI 无网环境必挂。
5. **快照滥用**：snapshot diff 难审查，仅用于稳定静态结构。
6. **挂载大页面**：依赖链全拉起测试巨慢；stub 子树聚焦单元。

## Checklist

- [ ] Vitest + test-utils 环境就绪，CI 跑 test run
- [ ] utils/composables 覆盖优先
- [ ] 组件测试断言行为而非结构
- [ ] store 测试 setActivePinia
- [ ] 网络一律 mock（msw）
- [ ] 无依赖网络的确定性测试
