# 12 - 列表渲染与表单受控

## 1. 列表渲染规范

```tsx
{users.map(u => <UserRow key={u.id} user={u} />)}
```

- key：稳定唯一 id，禁 index（有增删/排序/过滤时状态错位，见 `29-diff.md`）
- key 不用于取值/传参：要数据直接闭包捕获 `onClick={() => select(u.id)}`
- 分组/嵌套列表：外层 Fragment key 用 `<Fragment key={…}>`
- 大列表：虚拟滚动（react-window / TanStack Virtual），见 `25-performance.md`

## 2. 受控组件 vs 非受控组件

| | 受控 | 非受控 |
|---|---|---|
| 数据源 | React state | DOM 自己持有 |
| 取值 | state 即值 | ref / FormData |
| 实时校验/联动 | ✅ | 弱 |
| 性能 | 每键入重渲染 | 轻 |
| 适用 | 大多数表单 | 文件、一次性表单、第三方集成 |

```tsx
// 受控
const [email, setEmail] = useState('')
<input value={email} onChange={e => setEmail(e.target.value)} />

// 非受控
const ref = useRef<HTMLInputElement>(null)
<input defaultValue="初始" ref={ref} />
submit(ref.current!.value)

// FormData 整体提取（非受控利器）
function onSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault()
  const fd = new FormData(e.currentTarget)
  const email = fd.get('email')
}
```

**value 没有 onChange = 只读输入框**（React 警告）：受控就必须给 onChange。

## 3. 各控件绑定差异

| 控件 | 受控绑定 | 多选 |
|---|---|---|
| text/textarea | string | — |
| checkbox 单 | boolean | — |
| checkbox 多 | array | `checked={ids.includes(v)}` + onChange 维护数组 |
| radio | string（checked 比较） | — |
| select | string；`value=""` 占位 | multiple → array |

```tsx
// checkbox 多选标准实现
const [ids, setIds] = useState<number[]>([])
<input
  type="checkbox"
  checked={ids.includes(u.id)}
  onChange={e => setIds(cur => e.target.checked ? [...cur, u.id] : cur.filter(i => i !== u.id))}
/>

// select 占位
<select value={city} onChange={…}>
  <option value="" disabled>请选择</option>
  {cities.map(c => <option key={c} value={c}>{c}</option>)}
</select>
```

## 4. 受控的性能：输入防抖的正确位置

```tsx
// ❌ 每键入 setState 全列表重过滤重渲染
// ✅ 输入用受控（轻），派生计算用 deferred / debounce
const [kw, setKw] = useState('')
const deferredKw = useDeferredValue(kw)
const list = useMemo(() => bigFilter(deferredKw), [deferredKw])
```

搜索提交场景可用非受控 + onChange 防抖到外部状态，输入框本身不受控（详见 `25-performance.md`）。

## 5. 提交与防重

```tsx
const [submitting, setSubmitting] = useState(false)

async function onSubmit(e: React.FormEvent) {
  e.preventDefault()
  if (!validate(values)) return
  setSubmitting(true)
  try { await api.submit(values); toast('成功'); reset() }
  catch (e) { toast(getErrMsg(e)) }
  finally { setSubmitting(false) }
}

<form onSubmit={onSubmit}>
  <button disabled={submitting}>提交</button>
</form>
```

React 19 可用 form actions（`useActionState` + `isPending`，见 `10-react-18-19.md`）。

## 6. 数值输入的坑

```tsx
// .number 修饰符不存在（那是 Vue）
const [age, setAge] = useState<number | ''>('')
<input
  type="number"
  value={age}
  onChange={e => {
    const v = e.target.value
    setAge(v === '' ? '' : Number(v))   // 空串单独处理，否则 Number('') = 0
  }}
/>
```

## 7. 受控组件封装（自定义 input 契约）

```tsx
type TextInputProps = {
  value: string
  onChange: (v: string) => void
  placeholder?: string
  disabled?: boolean
}

export function TextInput({ value, onChange, ...rest }: TextInputProps) {
  return <input value={value} onChange={e => onChange(e.target.value)} {...rest} />
}
```

表单库（react-hook-form 等）组件封装遵循同样契约（value/onChange），见 `32-forms.md`。

## 8. 列表 + 表单组合：动态行

```tsx
const [rows, setRows] = useState<Row[]>([{ id: crypto.randomUUID(), name: '' }])

// key 用行 uid（不是 index）：删除中间行不串状态
{rows.map(r => (
  <div key={r.id}>
    <input value={r.name} onChange={e => patch(r.id, { name: e.target.value })} />
    <button onClick={() => setRows(rs => rs.filter(x => x.id !== r.id))}>删</button>
  </div>
))}
```

## 常见坑

1. **受控 input 忘 onChange**：变成只读且警告。
2. **动态行 key 用 index**：删除行输入内容错位。
3. **Number('') === 0**：数值输入空串处理。
4. **checkbox 多选初始 undefined**：必须数组初始化。
5. **每键入全树重渲染**：大列表场景 defer/防抖/局部拆分。
6. **submit 后未防重**：双击重复提交。
7. **e.preventDefault() 忘写**：form 提交整页刷新。

## Checklist

- [ ] 受控组件 value/onChange 成对出现
- [ ] 列表 key 稳定唯一（动态行为 uid）
- [ ] 数值输入空串处理
- [ ] checkbox 多选数组初始化
- [ ] 提交防重 + normalize
- [ ] 大表单/复杂表单评估 react-hook-form（见 32）
