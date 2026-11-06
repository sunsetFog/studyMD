# 11 - Teleport

## 1. 解决什么问题

组件模板的 DOM 结构受父组件 CSS 上下文限制（transform/overflow/z-index 层叠上下文），弹窗、抽屉、通知等浮层需要「逻辑在组件内、DOM 挂到 body」：

```vue
<Teleport to="body">
  <div class="modal">弹窗内容</div>
</Teleport>
```

- `to` 支持 CSS 选择器 / 元素引用：`to="#app"`、`to="body"`、`:to="elRef"`
- Teleport 内组件保持原有组件上下文（props/inject/父子关系不变），只挪 DOM

## 2. 渲染时机与目标存在性

Teleport 目标必须在挂载时已存在 DOM 中：

```vue
<!-- ❌ 目标在后渲染的兄弟元素里，报错 -->
<Teleport to="#later-target">…</Teleport>
<div id="later-target"></div>

<!-- ✅ 延迟挂载 -->
<Teleport to="#later-target" :defer="true">…</Teleport>  <!-- 3.5+ defer -->
<!-- 或 v-if 控制目标先出现 -->
```

## 3. 弹窗组件标准实现

```vue
<!-- BaseModal.vue -->
<script setup lang="ts">
defineProps<{ visible: boolean; title: string }>()
const emit = defineEmits<{ 'update:visible': [boolean] }>()
</script>

<template>
  <Teleport to="body">
    <Transition name="fade">
      <div v-if="visible" class="modal-mask" @click.self="emit('update:visible', false)">
        <div class="modal-card" role="dialog" aria-modal="true">
          <header>{{ title }}<button @click="emit('update:visible', false)">×</button></header>
          <main><slot /></main>
          <footer v-if="$slots.footer"><slot name="footer" /></footer>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>
```

要点：
- Teleport 包在 Transition **外层**（v-if 放 Teleport 内，确保进出动画作用于被传送内容）
- `@click.self` 防点内容误关
- `v-if="$slots.footer"` 条件渲染

## 4. 多层 Teleport 与嵌套

```vue
<Teleport to="body">
  <A />
  <Teleport to="#inner-portal">   <!-- 嵌套：目标查找基于 DOM，与嵌套层级无关 -->
    <B />
  </Teleport>
</Teleport>
```

多个 Teleport 到同一目标按**挂载顺序**追加。z-index 治理用统一层规范：

```css
/* 全局层级约定 */
:root {
  --z-modal: 1000;
  --z-drawer: 900;
  --z-toast: 1100;
  --z-tooltip: 1200;
}
```

## 5. SSR 注意

SSR 中 Teleport 内容不渲染到 HTML 首包（客户端挂载后传送）。需要 SEO 的内容不要 Teleport。

## 6. disabled 模式（就地渲染）

```vue
<Teleport to="body" :disabled="isMobile">
  <div class="popover">…</div>
</Teleport>
```

`disabled` 时内容留在原位（不传送），适合响应式切换「桌面传送弹窗 / 移动端就地内联」。

## 常见坑

1. **传送内容样式丢失**：scoped 样式仍有效（编译期加 data 属性，跟随 DOM），但**父组件的深层选择器**可能失效——浮层样式应自包含。
2. **Teleport 目标不存在**：确保目标先渲染，或用 defer。
3. **动画方向**：Transition 放 Teleport 外层时，leave 动画可能不生效——把 v-if 放 Teleport 内、Transition 也放内层。
4. **z-index 大战**：传送后脱离父层叠上下文反而更简单，但需全局层级规范。
5. **滚动锁定**：弹窗打开锁 body 滚动（overflow hidden），关闭恢复；多弹窗嵌套要计数管理。
6. **Vue devtools 中组件树与 DOM 树不一致**：排查问题时要知道 Teleport 只挪 DOM 不改组件树。

## Checklist

- [ ] 浮层统一走 Teleport + 全局 z-index 规范
- [ ] 目标节点存在性有保障（body 或 defer）
- [ ] 遮罩点击关闭用 @click.self
- [ ] 弹窗打开锁定背景滚动
- [ ] Teleport 内 Transition 的 v-if 位置正确
