# 32 - 表单与校验

## 1. 方案选型

| 方案 | 适用 |
|---|---|
| 受控组件（useState/useReducer） | 简单表单（≤5 字段） |
| **React Hook Form** | 复杂表单主流选择（非受控内核，性能好） |
| TanStack Form | 类型推导最强（新锐） |
| formik | 存量项目 |
| Server Action 表单 | Next.js 全栈 |

React Hook Form 的性能卖点：**非受控注册**，输入不触发整表单重渲染；校验/错误订阅字段级。

## 2. React Hook Form 标准用法

```tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  email: z.string().email('邮箱格式错误'),
  age: z.coerce.number().min(0).max(120),
  role: z.enum(['admin', 'member']),
})
type FormValues = z.infer<typeof schema>     // 类型与校验同源！

function Signup() {
  const { register, handleSubmit, formState: { errors, isSubmitting }, reset } = useForm<FormValues>({
    resolver: zodResolver(schema),
    defaultValues: { email: '', age: undefined, role: 'member' },
  })

  return (
    <form onSubmit={handleSubmit(async (values) => {
      await api.signup(values)      // values 已通过校验且类型正确
      reset()
    })}>
      <input {...register('email')} />
      {errors.email && <p>{errors.email.message}</p>}
      <button disabled={isSubmitting}>提交</button>
    </form>
  )
}
```

要点：
- **zod schema = 校验 + 类型单一来源**（`z.infer` 推导 FormValues）
- `register('field')` 展开到 input（ref + name + onBlur 等）
- `isSubmitting` 内建防重

## 3. 校验时机策略

| 时机 | mode | 体验 |
|---|---|---|
| 提交时 | `onSubmit`（默认） | 最不打扰 |
| 失焦 | `onBlur` | 常见平衡点 |
| 每输入 | `onChange` | 即时但打扰 |
| 全部 | `all` | — |

推荐：**onBlur 校验 + 提交全量**（与 vue3/19 一致）；提交失败后变 onChange 即时纠错。

## 4. 手写轻量表单（无依赖场景）

```tsx
type FormState = { values: Record<string, string>; errors: Record<string, string> }
type Action = { type: 'change'; field: string; value: string } | { type: 'blur'; field: string } | { type: 'submitFailed'; errors: Record<string, string> } | { type: 'reset' }

function reducer(s: FormState, a: Action): FormState {
  switch (a.type) {
    case 'change': {
      const values = { ...s.values, [a.field]: a.value }
      const errors = { ...s.errors }; delete errors[a.field]
      return { values, errors }
    }
    case 'blur': return { ...s, values: s.values }   // touched 逻辑见 08
    case 'submitFailed': return { ...s, errors: a.errors }
    case 'reset': return { values: {}, errors: {} }
  }
}
```

配合校验规则表（策略模式，见 vue3/34 §4）：

```ts
const validators = {
  required: (v: string) => v.trim() !== '' || '必填',
  phone: (v: string) => /^1\d{10}$/.test(v) || '手机号格式错误',
}
```

## 5. 服务端校验联动

```tsx
// 提交失败：服务端错误写回字段
const onError = async (e: ApiError) => {
  if (e.status === 422 && e.fields) {
    Object.entries(e.fields).forEach(([field, msg]) => setError(field as keyof FormValues, { message: msg }))
  }
}
```

原则：**前端校验是体验，后端校验是安全**（同 vue3/37 双保险原则）。

## 6. 动态表单

```tsx
const { fields, append, remove } = useFieldArray({ control, name: 'items' })
{fields.map((f, i) => (
  <div key={f.id}>                    {/* f.id 稳定 uid，React Hook Form 自动生成 */}
    <input {...register(`items.${i}.name`)} />
    <button onClick={() => remove(i)}>删</button>
  </div>
))}
<button onClick={() => append({ name: '' })}>加行</button>
```

## 7. 提交防重与 normalize

```tsx
onSubmit={handleSubmit(async (v) => {
  await api.submit(normalize(v))     // trim/number/格式化
}, onError)}
```

- 提交按钮 `disabled={isSubmitting}`（RHF 内建）
- 成功后 reset / 跳转明确
- 表单草稿持久化：watch + useLocalStorage（防误关）

## 常见坑

1. **受控大表单每键入全表重渲染**：RHF 非受控解决。
2. **zod schema 与 TS 类型两套维护**：z.infer 单源。
3. **`register` 用在受控组件上**：冲突；第三方受控组件用 Controller。
4. **动态行 key 用 index**：RHF fields.id 解决。
5. **服务端错误不回写字段**：用户不知道哪里错。
6. **弹窗复用表单脏数据**：打开时 reset(defaultValues)。
7. **number 输入空串**：z.coerce.number 处理或空串分支。

## Checklist

- [ ] 复杂表单用 RHF + zod 单一类型源
- [ ] onBlur + 提交全量校验策略
- [ ] 服务端字段错误回写
- [ ] 动态行 useFieldArray + 稳定 key
- [ ] 提交防重 + normalize + 成功处理
- [ ] 前后端校验双保险
