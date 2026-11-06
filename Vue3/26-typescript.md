# 26 - TypeScript 集成

## 1. 基础配置

```json
// tsconfig.json（Vue 官方 create-vue 生成为准）
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "preserve",
    "types": ["vite/client"],
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src/**/*.ts", "src/**/*.vue", "env.d.ts"]
}
```

```bash
npm i -D vue-tsc
# package.json
"build": "vue-tsc -b && vite build"    # 类型检查参与构建
```

`vue-tsc` 是 Vue 版 tsc，能检查 .vue 文件内类型。

## 2. 组件类型化

### 2.1 props

```ts
// 泛型声明（推荐）
const props = defineProps<{
  title: string
  size?: 'sm' | 'md' | 'lg'
  items: OrderItem[]
  onSuccess?: (id: number) => void
}>()

// 带默认值
const props = withDefaults(defineProps<{
  size?: 'sm' | 'md' | 'lg'
}>(), { size: 'md' })

// 3.5+ 响应式解构默认值（替代 withDefaults 的简化）
const { size = 'md' } = defineProps<{ size?: 'sm' | 'md' | 'lg' }>()
```

### 2.2 emits

```ts
const emit = defineEmits<{
  change: [value: string]
  submit: [payload: FormPayload]
}>()
emit('submit', payload)     // 参数类型强约束
```

### 2.3 defineModel

```ts
const visible = defineModel<boolean>('visible', { required: true })
const keyword = defineModel<string>({ default: '' })
```

### 2.4 expose 与模板 ref

```ts
defineExpose({ validate: () => true })

// 父组件
const form = useTemplateRef<FormExpose>('form')   // 3.5+
// 或声明式接口
interface FormExpose { validate(): boolean }
```

## 3. 响应式类型

```ts
import { ref, computed, type Ref, type ComputedRef } from 'vue'

const count = ref(0)                        // Ref<number> 自动推导
const list = ref<Order[]>([])               // 数组
const maybe = ref<Order | null>(null)       // 联合

// ref 包装复杂类型时的 UnwrapRef 陷阱
const state = ref<{ cb: () => void }>()     // 函数类型会被 UnwrapRef 处理异常
// 解法：shallowRef 保持类型原样
const state = shallowRef<{ cb: () => void }>()
```

复杂/函数类型状态用 `shallowRef` 既保性能又保类型。

## 4. 通用类型工具

```ts
// 组件实例类型
import type { ComponentPublicInstance } from 'vue'
let el: ComponentPublicInstance | null = null

// 从组件定义反推 props 类型（封装透传组件时）
import type { ComponentProps } from 'vue-component-type-helpers'  // 或手写
type BtnProps = InstanceType<typeof ElButton>['$props']
```

## 5. 全局声明

```ts
// env.d.ts
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_BASE: string
}

// 全局组件类型（unplugin-vue-components 生成或手写 components.d.ts）
declare module 'vue' {
  interface GlobalComponents {
    ElButton: typeof import('element-plus/es')['ElButton']
  }
}
```

## 6. api 层类型化

```ts
// api/order.ts
import type { PageResult } from '@/types/common'

export interface OrderQuery { page: number; status?: OrderStatus }
export interface OrderItem { id: number; amount: number; status: OrderStatus }

export function getOrders(q: OrderQuery) {
  return http.get<PageResult<OrderItem>>('/orders', { params: q })
}
// 调用方
const { data } = await getOrders({ page: 1 })
data.records   // OrderItem[] 完整推导
```

规范：接口的入参/出参类型**定义在 api 模块内并导出**，组件 import type 使用——类型跟着接口走。

## 7. 常见类型场景

```ts
// 枚举
export const ORDER_STATUS = { pending: 0, paid: 1 } as const
export type OrderStatus = keyof typeof ORDER_STATUS        // 'pending' | 'paid'

// Record 键约束
const text: Record<OrderStatus, string> = { pending: '待付款', paid: '已付款' }

// 事件处理器
const onChange = (e: Event) => { const v = (e.target as HTMLInputElement).value }

// DOM ref
const input = useTemplateRef<HTMLInputElement>('kw')
```

## 8. ESLint 与 TS 联动

```js
// eslint flat config 要点
import tseslint from 'typescript-eslint'
export default tseslint.config(
  ...tseslint.configs.recommended,
  { rules: { '@typescript-eslint/no-explicit-any': 'warn', '@typescript-eslint/explicit-function-return-type': 'off' } }
)
```

策略：`any` 警告不禁止（渐进），`@typescript-eslint/no-unused-vars` 开启，公共 API 显式返回类型可选开启。

## 常见坑

1. **vue 文件不检查**：只用 tsc 漏掉全部 SFC 类型错误，必须 vue-tsc。
2. **UnwrapRef 吞函数类型**：复杂类型用 shallowRef。
3. **组件 props 用运行时声明 + as PropType 混搭**：统一泛型声明。
4. **`import type` 忘写**：值导入造成运行时循环依赖；isolatedModules 开启后报错提醒。
5. **第三方库无类型**：临时 `declare module 'lib'` 的 any shims 入库扩散——及时补 d.ts 或换有类型的库。
6. **build 不跑 vue-tsc**：类型错误上线；CI 强制 `vue-tsc -b`。

## Checklist

- [ ] build 命令含 vue-tsc 类型检查
- [ ] props/emits/expose/model 全类型化
- [ ] api 层类型定义并导出
- [ ] 复杂/函数类型状态用 shallowRef
- [ ] 无散落的 declare any shim
- [ ] strict 开启且 any 有 ESLint 约束
