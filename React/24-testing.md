# 24 - 单元测试

## 1. 测试栈（Vitest 时代）

```bash
npm i -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

```ts
// vitest.config.ts
test: { environment: 'jsdom', globals: true, setupFiles: './src/test/setup.ts' }

// src/test/setup.ts
import '@testing-library/jest-dom/vitest'
```

哲学（Testing Library）：**测试用户可见行为，而非实现细节**。

## 2. 测试金字塔

| 层 | 占比 | 对象 |
|---|---|---|
| 单元（utils/hooks/reducer） | 60% | 纯逻辑，快 |
| 组件测试 | 30% | 关键交互组件 |
| E2E（Playwright） | 10% | 主链路 |

## 3. 工具函数与 reducer 测试

```tsx
import { describe, it, expect } from 'vitest'

describe('cartReducer', () => {
  it('addItem 已存在则累加数量', () => {
    const s = cartReducer({ items: [{ id: 1, qty: 1 }] }, { type: 'addItem', item: { id: 1, qty: 2 } })
    expect(s.items[0].qty).toBe(3)
  })
})
```

reducer/useReducer 的纯函数特性使其成为**最容易测**的部分——这是用 useReducer 的隐藏收益。

## 4. 自定义 Hook 测试

```tsx
import { renderHook, act } from '@testing-library/react'

it('useCounter 递增', () => {
  const { result } = renderHook(() => useCounter())
  act(() => result.current.increment())
  expect(result.current.count).toBe(1)
})
```

## 5. 组件测试标准模板

```tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'

it('点击按钮计数', async () => {
  render(<Counter />)
  const btn = screen.getByRole('button', { name: /点击/ })
  await userEvent.click(btn)
  expect(screen.getByText('点击了 1 次')).toBeVisible()
})
```

查询优先级（可访问性即选择器）：

```
getByRole > getByLabelText > getByPlaceholderText > getByText > getByTestId（最后手段）
```

- `getBy*` 找不到抛错；`queryBy*` 用于断言不存在
- `findBy*` 用于异步出现（内置 waitFor）
- `userEvent.click/type` 模拟真实交互（比 fireEvent 更接近浏览器行为）

## 6. Mock 策略

```tsx
import { vi } from 'vitest'

// 1. 网络：MSW（推荐，拦截层最真实）
import { setupServer } from 'msw/node'
const server = setupServer(http.get('/api/user/:id', () => HttpResponse.json(mockUser)))
beforeAll(() => server.listen()); afterEach(() => server.resetHandlers()); afterAll(() => server.close())

// 2. 模块 mock
vi.mock('@/api/user', () => ({ getUser: vi.fn().mockResolvedValue(mockUser) }))

// 3. 子组件 stub
render(<Page />, { wrapper: TestProviders })
vi.mock('./HeavyChart', () => ({ default: () => <div data-testid="chart-stub" /> }))

// 4. Provider 包装（Router/Query/Theme）
const TestProviders = ({ children }) => (
  <QueryClientProvider client={new QueryClient()}><MemoryRouter>{children}</MemoryRouter></QueryClientProvider>
)
render(<MyPage />, { wrapper: TestProviders })
```

## 7. 测什么 / 不测什么

| ✅ 测 | ❌ 不测 |
|---|---|
| reducer 状态转移 | 第三方库 |
| 校验规则、格式化 | 样式/类名细节 |
| 关键交互（提交/筛选） | 简单透传组件 |
| hooks 纯逻辑 | 快照式的结构断言 |
| 权限条件渲染 | 集成逻辑（E2E 管） |

## 8. 异步与时序

```tsx
it('加载完成后渲染列表', async () => {
  server.use(http.get('/api/orders', () => HttpResponse.delay(10).then(() => HttpResponse.json(data))))
  render(<OrderList />)
  expect(screen.getByText(/加载中/)).toBeVisible()
  expect(await screen.findByRole('list')).toBeVisible()   // findBy 内建等待
})
```

- fake timers：`vi.useFakeTimers()`（防抖/轮询），注意 `vi.useRealTimers()` 还原
- Query 组件测试：MSW + 真 QueryClient（缓存隔离：每 test 新建）

## 9. E2E 与 CI

```json
"scripts": { "test": "vitest run", "coverage": "vitest run --coverage" }
```

- CI：vitest run + coverage（utils ≥80%，组件不设硬门槛）
- Playwright 管主链路（登录/下单）——与单元测试分工明确
- a11y 测试：jest-dom + axe-core（`@testing-library/jest-dom` toBeVisible 已覆盖部分语义）

## 常见坑

1. **断言实现细节**（class/state）：重构即碎；断言用户可见行为。
2. **fireEvent vs userEvent 混用**：统一 userEvent。
3. **Query 缓存跨测试污染**：每 test 新 QueryClient。
4. **fake timer 忘还原**：后续测试挂起。
5. **mock 泛滥**：优先 MSW 网络层，少 mock 模块。
6. **异步断言不等待**：同步 getBy 拿不到 → findBy/waitFor。

## Checklist

- [ ] Vitest + RTL + jest-dom + userEvent + MSW 就绪
- [ ] reducer/hooks 覆盖优先
- [ ] 组件测试用语义查询（role 优先）
- [ ] 网络一律 MSW
- [ ] Provider 包装统一（TestProviders）
- [ ] CI 跑 test run + coverage
