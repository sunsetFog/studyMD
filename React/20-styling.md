# 20 - 样式方案全景

## 1. 方案地图

| 方案 | 隔离性 | 运行时成本 | 适用 |
|---|---|---|---|
| 全局 CSS + BEM | 手动约定 | 无 | 简单项目 |
| CSS Modules | 文件级自动 | 无（构建期 hash） | Vite 默认好选择 |
| Tailwind CSS | 类名即样式 | 无 | 快速开发、设计系统 token |
| CSS-in-JS（styled-components / Emotion） | 组件级 | 运行时开销 | 存量项目 |
| CSS-in-JS 编译期（PandaCSS / vanilla-extract / Linaria） | 组件级 | 零运行时 | 新项目趋势 |
| UI 库（antd/MUI/radix） | — | — | 直接用成熟组件 |

## 2. CSS Modules

```tsx
// Button.module.css
.primary { background: var(--c-primary); }
.btn { composes: base; }          // composes 组合

import styles from './Button.module.css'
<button className={styles.primary}>确定</button>
```

- 类名构建期 hash，天然隔离
- TS 类型：`*.module.d.ts` 或 vite 自动（CSS Modules typed）
- 组合复杂变体时配合 clsx

## 3. Tailwind CSS（当前主流）

```bash
npm i -D tailwindcss @tailwindcss/vite
```

```tsx
// utility-first + CVA 变体（见 13）
<button className="inline-flex h-9 items-center rounded-md bg-blue-600 px-4 text-sm text-white hover:bg-blue-700 disabled:opacity-50">

// 设计 token 在 tailwind.config / CSS 变量
@theme {
  --color-brand: #1677ff;
}
```

要点：
- **设计 token 一致性**是 Tailwind 价值核心（spacing/color/typography 全走配置，不写魔法值）
- `cn()` 工具统一类名合并：

```tsx
import { twMerge } from 'tailwind-merge'
import clsx, { type ClassValue } from 'clsx'
export const cn = (...inputs: ClassValue[]) => twMerge(clsx(inputs))
```

- 伪类/响应式/暗色：`hover:`, `md:`, `dark:` 前缀
- 动态类名陷阱：**不能字符串拼接生成类名**（`bg-${color}-500` 不被扫描）——完整类名映射对象替代

## 4. CSS-in-JS

### 4.1 styled-components / Emotion（运行时）

```tsx
const Button = styled.button<{ variant?: 'primary' | 'danger' }>`
  background: ${p => p.variant === 'danger' ? '#f5222d' : '#1677ff'};
`
```

- 优点：组件级隔离、props 驱动样式、主题 ThemeProvider
- 缺点：运行时序列化开销、SSR 样式抽取复杂、包体
- **新项目一般不再首选**（React 官方对运行时 CSS-in-JS 与并发/RSC 的兼容性存保留态度）

### 4.2 编译期方案（趋势）

```tsx
// vanilla-extract：TS 写样式，构建期产出 CSS 文件
import { style } from '@vanilla-extract/css'
export const button = style({ background: 'var(--c-primary)' })
```

零运行时 + 类型安全 + SSR 友好；代价是样式不在 JSX 内联。

## 5. 主题与暗色模式

```css
/* 设计 token 走 CSS 变量 */
:root { --c-bg: #fff; --c-text: #333; }
[data-theme='dark'] { --c-bg: #141414; --c-text: #eee; }
```

```tsx
// 切换：html data-theme 属性 + localStorage 持久化 + prefers-color-scheme 默认
useEffect(() => {
  document.documentElement.dataset.theme = theme
}, [theme])
```

组件样式只引用变量，主题切换零组件改动。

## 6. 布局与响应式速记

```css
/* Flex 一维 / Grid 二维 */
display: flex; gap: 12px;               /* gap 替代 margin 兄弟间距 */
display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));

/* 响应式 */
@media (min-width: 768px) { … }
/* Tailwind: md:grid-cols-3 */
```

移动端安全区：`env(safe-area-inset-bottom)`。

## 7. 动画

| 场景 | 方案 |
|---|---|
| 简单过渡 | CSS transition（transform/opacity 优先） |
| 进出场 | CSS 动画类切换 / react-transition-group |
| 复杂编排 | framer-motion（motion） |
| FLIP 列表 | framer-motion layout / @formkit/auto-animate |

```tsx
// framer-motion
<motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }} />
```

动画只动 `transform/opacity`（合成层），避免布局属性动画掉帧。

## 8. 选型建议

```
新项目默认组合：Tailwind（页面/布局） + CSS Modules/变量（复杂组件） + antd/radix（基础交互组件）+ motion（动画）
存量 styled-components 项目：维护即可，不迁移
团队无 Tailwind 经验：CSS Modules + BEM 起步完全够用
```

## 常见坑

1. **Tailwind 动态拼接类名**不被扫描生成；用映射对象。
2. **styled-components 每次渲染新组件定义**（styled 组件写在组件函数体内）→ 重建类名；定义放模块顶层。
3. **全局样式污染**：reset 只引入一次；组件样式不裸写全局类。
4. **暗色模式样式双写**：走 CSS 变量而非两套类。
5. **z-index 随手写**：全局层级 token（同 vue3/11 规范）。
6. **CSS Modules 类型缺失**：配 typed-css-modules 或 vite 插件。

## Checklist

- [ ] 样式方案团队统一且有决策记录
- [ ] 设计 token（颜色/间距/字号）集中管理
- [ ] 暗色主题走 CSS 变量
- [ ] 无动态拼接 Tailwind 类
- [ ] z-index 层级规范
- [ ] 动画仅 transform/opacity
