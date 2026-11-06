# 16 - 自定义指令

## 1. 指令适用边界

指令 = **对 DOM 的底层直接操作**（无法或不便用响应式数据表达时）。

✅ 适合：v-focus、v-lazy（图片懒加载）、v-copy（点击复制）、v-permission（权限隐藏）、v-intersect（曝光埋点）。
❌ 不适合：逻辑计算、状态管理、能被响应式数据驱动的 UI——这些用组件/composables。

## 2. 指令生命周期钩子

```ts
const vFocus: Directive = {
  mounted(el) { el.focus() },
}
// 钩子全集（与组件生命周期对应）：
// created / beforeMount / mounted / beforeUpdate / updated
// beforeUnmount / unmounted
```

钩子参数：

```ts
interface DirectiveBinding<T = any> {
  el: HTMLElement        // 指令挂载的 DOM
  value: T               // v-my="value" 的值
  oldValue: T            // 更新前的值（仅 updated 可用）
  arg: string            // v-my:foo 的 foo
  modifiers: Record<string, boolean>   // v-my.foo.bar → { foo: true, bar: true }
  instance: ComponentPublicInstance
  dir: Directive
}
```

## 3. 注册方式

```ts
// 局部：<script setup> 中 v 开头的变量自动可用
const vFocus: Directive<HTMLElement, number> = { mounted(el, { value }) { setTimeout(() => el.focus(), value ?? 0) } }

// 全局：main.ts
app.directive('focus', vFocus)

// 模块化组织
// directives/index.ts
import type { Directive } from 'vue'
export const vPermission: Directive<HTMLElement, string> = { /* … */ }
// main.ts
Object.entries(directives).forEach(([name, dir]) => app.directive(name.replace(/^v/, '').toLowerCase(), dir))
```

## 4. 实战示例

### 4.1 v-permission

```ts
import { useAuthStore } from '@/stores/auth'

export const vPermission: Directive<HTMLElement, string | string[]> = {
  mounted(el, { value }) {
    const auth = useAuthStore()   // 指令内可用 store
    const codes = Array.isArray(value) ? value : [value]
    if (!codes.some(c => auth.hasPermission(c))) {
      el.style.display = 'none'   // 或 el.remove()
    }
  },
}
```

注意：权限变化不会自动响应（mounted 时机固定），需要动态权限则用组件包装或 watch。

### 4.2 v-copy

```ts
export const vCopy: Directive<HTMLElement, string> = {
  mounted(el, { value }) {
    el.addEventListener('click', async () => {
      await navigator.clipboard.writeText(value)
      showToast('已复制')
    })
  },
  unmounted(el) { /* click 监听随元素销毁，无需手动移除 */ },
}
```

### 4.3 v-lazy（IntersectionObserver）

```ts
export const vLazy: Directive<HTMLImageElement, string> = {
  mounted(el, { value }) {
    el.dataset.src = value
    const io = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        el.src = el.dataset.src!
        io.disconnect()
      }
    })
    io.observe(el)
    el._io = io     // 存引用以便清理
  },
  unmounted(el) { el._io?.disconnect() },
}
```

## 5. SSR 注意

指令钩子只在客户端触发，SSR 下 `mounted` 等不执行——若指令承担关键功能（如权限隐藏），SSR 场景要在服务端逻辑里同步处理，或改用组件方案。

## 6. 指令 vs 组件选型

| 场景 | 指令 | 组件 |
|---|---|---|
| 单元素行为增强（复制/聚焦/懒加载） | ✅ | 过重 |
| 带模板结构的 UI | ❌ | ✅ |
| 需要响应式深度参与 | ❌ | ✅ |
| 埋点/行为注入 | ✅ | 过重 |

## 常见坑

1. **unmounted 忘记清理**：IntersectionObserver、第三方实例泄漏。
2. **指令值更新不响应**：mounted 一次性绑定；需要响应 updated 里处理 oldValue。
3. **指令内直接改状态**：指令应无状态；需要状态用闭包存 el 上或改组件。
4. **TypeScript 类型**：`Directive<El, Value>` 双泛型写全，避免 any 传染。
5. **同名指令全局覆盖局部**：全局注册后局部同名被静默覆盖。
6. **SSR 关键逻辑依赖指令**：首屏权限闪现（先显示后被隐藏）。

## Checklist

- [ ] 指令只做 DOM 级行为增强
- [ ] 每个 IO/第三方实例在 unmounted 清理
- [ ] Directive 类型带双泛型
- [ ] 指令集中注册（directives/index.ts）
- [ ] SSR 关键能力不依赖指令钩子
