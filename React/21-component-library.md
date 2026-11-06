# 21 - 组件库与设计系统

## 1. 决策：用现成的 vs 自建

| 场景 | 策略 |
|---|---|
| 中后台管理 | antd / Arco / MUI 直接用 + 业务组件包装 |
| 面向 C 端定制设计 | Headless（radix-ui / shadcn 思路）+ 自己的样式层 |
| 多项目复用业务组件 | 内部组件库 monorepo |

**React 组件库分层**：Headless 逻辑层（无样式无 UI）→ 样式层（Tailwind/主题）→ 业务层（带业务语义）。

## 2. Headless UI 路线（radix-ui / shadcn/ui）

```tsx
// radix-ui：只管行为与可访问性，样式全自己来
import * as Dialog from '@radix-ui/react-dialog'

<Dialog.Root>
  <Dialog.Trigger asChild><Button>打开</Button></Dialog.Trigger>
  <Dialog.Portal>
    <Dialog.Overlay className="overlay" />
    <Dialog.Content className="content">
      <Dialog.Title>标题</Dialog.Title>
      <Dialog.Close asChild><Button>关闭</Button></Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

shadcn/ui 思路：**组件源码直接拷进项目**（不是 npm 依赖），基于 radix + tailwind——可改可控不锁死，近年最流行的中庸路线。

## 3. 自建业务组件库结构（monorepo）

```
design-system/
├── packages/
│   ├── tokens/          # 设计 token（JSON/CSS 变量生成）
│   ├── icons/
│   ├── ui/              # 基础组件（Button/Modal/…）
│   │   └── src/button/
│   │       ├── Button.tsx
│   │       ├── button.cva.ts      # 变体定义
│   │       ├── index.ts
│   │       └── Button.test.tsx
│   └── business/        # 业务组件（UserPicker/AmountText）
├── apps/docs/           # Storybook
└── package.json（workspaces）
```

构建导出（同 vue3/35 原则）：

```json
{
  "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } },
  "peerDependencies": { "react": "^18.0.0 || ^19.0.0", "react-dom": "^18.0.0 || ^19.0.0" }
}
```

React 专属要点：**react 必须 external**（双 React 实例 → hooks 报错「Invalid hook call」）；peerDependencies 声明版本范围；侧效应标记 `"sideEffects": ["**/*.css"]` 保 tree-shaking。

## 4. 组件 API 规范（React 专属）

```tsx
type ButtonProps = {
  variant?: 'primary' | 'secondary' | 'danger'
  size?: 'sm' | 'md' | 'lg'
  loading?: boolean
} & Omit<React.ButtonHTMLAttributes<HTMLButtonElement>, 'type'>   // 继承原生属性！

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(   // React 18；19 可 ref-as-prop
  ({ variant = 'primary', size = 'md', loading, className, children, disabled, ...rest }, ref) => (
    <button
      ref={ref}
      className={cn(buttonVariants({ variant, size }), className)}
      disabled={disabled || loading}
      {...rest}                          // 原生属性透传（onClick/style/aria…）
    >
      {loading && <Spinner />}
      {children}
    </button>
  ),
)
```

规范：
- 继承原生属性类型（`ButtonHTMLAttributes`），rest 透传——组件 API 与原生一致
- className 合并（cn）允许调用方追加样式
- forwardRef 支持（表单聚焦场景）；React 19 直接 ref prop
- children 类型 `React.ReactNode`

## 5. 主题系统

```tsx
// ThemeProvider + CSS 变量 token
<ConfigProvider theme={{ token: { colorPrimary: '#1677ff', borderRadius: 6 } }}>
  <App />
</ConfigProvider>

// token 层级：原始变量 → 语义变量（--c-primary） → 组件消费
```

暗色模式：`data-theme` 切换变量（见 20 §5）；antd 用 ConfigProvider theme.darkAlgorithm。

## 6. 文档与验收（Storybook）

```tsx
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react'

const meta: Meta<typeof Button> = { component: Button, args: { children: '按钮' } }
export default meta

export const Primary: StoryObj = { args: { variant: 'primary' } }
export const Danger: StoryObj = { args: { variant: 'danger' } }
```

- 每组件：用法示例、props 表、可访问性检查（键盘导航/aria）、交互测试
- **文档即验收**：写不出 story 的组件不进库

## 7. 版本与治理

| 实践 | 说明 |
|---|---|
| semver + changesets | breaking 有迁移指南 |
| peerDependencies 宽范围 | react ^18 || ^19 |
| 弃用流程 | @deprecated 标注 + 两个大版本周期 |
| 消费方升级自动化 | renovate/dependabot |

## 8. 可访问性底线（组件库必备）

- 键盘可达：Tab 焦点顺序、Escape 关浮层
- aria 属性：role/aria-expanded/aria-disabled
- 焦点管理：Modal 打开聚焦、关闭归还焦点、焦点圈定（radix 已内建）

## 常见坑

1. **react 未 external**：双实例 hooks 崩溃。
2. **不继承原生属性**：onClick/style 传不进去，调用方包裹 div 兜底——API 失败。
3. **forwardRef 缺失**：调用方拿不到 DOM（React 19 后用 ref prop 统一）。
4. **CSS 依赖注入顺序问题**：组件样式按需导入与打包顺序；测试 token 化。
5. **Storybook 与构建配置漂移**：storybook 用同一 vite 配置。
6. **copy-paste 组件库（shadcn 式）无版本治理**：组件散落各项目，规范靠约定；至少留源码目录统一切片。

## Checklist

- [ ] react/ReactDOM peerDependencies + external
- [ ] 组件继承原生属性类型 + rest 透传 + className 合并
- [ ] ref 支持完整（18 forwardRef / 19 ref prop）
- [ ] Storybook 文档齐全
- [ ] 可访问性（键盘/aria/焦点）达标
- [ ] semver + changesets 发布流程
