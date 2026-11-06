# 02 - 模板语法

## 1. 插值与基本绑定

```vue
<template>
  <!-- 文本插值 -->
  <span>{{ message }}</span>
  <!-- JS 表达式（非语句） -->
  <span>{{ count > 10 ? '多' : '少' }}</span>
  <!-- 原始 HTML：数据必须可信，否则 XSS（参见 37） -->
  <div v-html="trustedHtml"></div>
  <!-- 属性绑定 -->
  <img :src="imgUrl" :alt="title" />
  <!-- 布尔属性 -->
  <button :disabled="!canSubmit">提交</button>
  <!-- 动态参数名 -->
  <a :[attrName]="url">链接</a>
  <!-- 动态事件名 -->
  <button @[eventName]="onClick">点击</button>
</template>
```

规则：
- 插值内只能是**表达式**：不能 `if`、`var x = 1`，可以用三元、算术、方法调用
- 动态参数必须是**字符串**（或 null，null 表示移除），且不能含空格和引号

## 2. 指令速查

| 指令 | 作用 | 缩写 |
|---|---|---|
| `v-bind` | 绑定属性 | `:` |
| `v-on` | 绑定事件 | `@` |
| `v-if / v-else-if / v-else` | 条件渲染（真实增删节点） | — |
| `v-show` | 条件显示（CSS display 切换） | — |
| `v-for` | 列表渲染 | — |
| `v-model` | 双向绑定 | — |
| `v-slot` | 插槽 | `#` |
| `v-once` | 只渲染一次 | — |
| `v-memo` | 记忆化渲染 | — |
| `v-pre` | 跳过编译 | — |
| `v-cloak` | 编译完成前隐藏 | — |

## 3. v-if vs v-show 决策

| 维度 | v-if | v-show |
|---|---|---|
| 切换成本 | 高（销毁/重建，触发生命周期） | 低 |
| 初始成本 | 条件为假时不渲染（轻） | 总是渲染（重） |
| 适用场景 | 很少切换 / 权限控制 | 频繁切换（Tab、抽屉） |

经验法则：**默认用 v-if；切换频率高且初始就要显示时用 v-show**。

## 4. 列表渲染

```vue
<li v-for="item in items" :key="item.id">{{ item.name }}</li>
<li v-for="(item, index) in items" :key="item.id">…</li>
<li v-for="(value, key, index) in obj" :key="key">…</li>
<span v-for="n in 10" :key="n">{{ n }}</span>
```

### 4.1 key 的铁律

- **必须用稳定唯一 id**，禁止用 index 作为 key（除非列表纯静态且永不重排）
- index 作 key 在插入/删除/排序时会错位复用节点：输入框内容串位、动画异常、状态错乱

### 4.2 v-for + v-if 的坑

```vue
<!-- ❌ 反模式：v-if 优先级高于 v-for，item 不在作用域 -->
<li v-for="item in list" v-if="item.show" :key="item.id">

<!-- ✅ 正确方式一：computed 过滤 -->
<li v-for="item in visibleList" :key="item.id">

<!-- ✅ 正确方式二：template 包裹 -->
<template v-for="item in list" :key="item.id">
  <li v-if="item.show">{{ item.name }}</li>
</template>
```

Vue3 中 v-if 优先级**高于** v-for（与 Vue2 相反），这是迁移最容易踩的坑之一。

## 5. 事件处理

```vue
<button @click="onClick">方法处理器</button>
<button @click="count++">内联表达式</button>
<button @click="onClick($event)">手动传事件对象</button>

<!-- 事件修饰符（按顺序书写） -->
<form @submit.prevent="onSubmit">…</form>
<a @click.stop.prevent="onLink">…</a>
<div @click.capture="onCapture">…</div>
<div @click.self="onSelf">只有自身点击才触发</div>
<button @click.once="onOnce">只触发一次</button>
<input @keyup.enter="onEnter" />
<input @keyup.enter.exact="onExact">严格组合键</input>

<!-- passive：移动端滚动优化 -->
<div @scroll.passive="onScroll">…</div>
```

注意：`.passive` 与 `.prevent` 不能同时用，passive 告诉浏览器不阻止默认行为。

## 6. class 与 style 绑定

```vue
<!-- 对象语法 -->
<div :class="{ active: isActive, disabled: isDisabled }">
<!-- 数组语法 -->
<div :class="[baseClass, isActive ? 'active' : '', errorClass]">
<!-- 嵌套混合 -->
<div :class="[{ active: isActive }, isBold && 'bold']">

<!-- style 对象语法：camelCase 或 kebab-case 皆可 -->
<div :style="{ color: textColor, fontSize: size + 'px' }">
<!-- style 数组：多来源合并 -->
<div :style="[baseStyle, overrideStyle]">
```

自动前缀：Vue 会为 `transform` 等属性自动加 `-webkit-` 等前缀（值改变时按需探测）。

## 7. v-memo 与渲染优化

```vue
<div v-memo="[item.id, item.selected]" v-for="item in list" :key="item.id">
  <Detail :item="item" />
</div>
```

依赖数组不变时跳过整个子树更新，适合超大列表中大部分节点不常变的场景。**仅在明确 profiling 出瓶颈后使用**，不要过早优化。

## 8. 模板中的全局白名单

模板表达式可访问的全局对象默认白名单：`Math, Date, JSON, Infinity, undefined, NaN, isNaN, parseFloat, parseInt, decodeURI…`。

需要额外全局对象时配置：

```ts
app.config.globalProperties.X = Y
// 或编译配置 compilerOptions.isCustomElement / globals
```

## 常见坑

1. **v-if 与 v-for 同标签连用**（Vue3 优先级反转），100% 出 bug 的写法。
2. **key 用 index**：列表有排序/过滤/删除操作时状态错位。
3. **`@click.self` 和 `.stop` 混淆**：self 只在 e.target 是自身时触发，不是「阻止子元素冒泡」。
4. **v-html 直接渲染用户输入** → XSS，必须先消毒或改用文本插值。
5. **动态参数表达式大小写**：`:[someAttr]` 在 DOM 内模板中会被浏览器转小写，SFC 中无此问题，但避免在 `in-DOM` 模板用大写属性名。
6. **模板里访问 `window`**：默认不可用，要么白名单要么写 computed。

## Checklist

- [ ] 所有 v-for 都有稳定唯一 key
- [ ] 无 v-if 与 v-for 同节点混用
- [ ] 条件渲染按切换频率选择 v-if / v-show
- [ ] v-html 输入来源可信或有消毒
- [ ] 事件修饰符使用恰当（.prevent/.stop/.once），未滥用
