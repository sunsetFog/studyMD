# 12 - Transition 与 KeepAlive

## 1. Transition 基础

```vue
<Transition name="fade">
  <div v-if="show">内容</div>
</Transition>

<style>
.fade-enter-active, .fade-leave-active { transition: opacity .3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
```

类名周期：`v-enter-from → v-enter-active → v-enter-to`；离开同理 `v-leave-*`。

约束：**单元素**（v-if/v-show 或组件切换）；多元素列表用 TransitionGroup。

## 2. Transition 常用配置

```vue
<Transition
  name="fade"
  appear                <!-- 首次渲染也执行进入动画 -->
  mode="out-in"         <!-- 先离开后进入（避免重叠错位）；in-out 反之 -->
  :duration="{ enter: 300, leave: 200 }"   <!-- 显式时长（嵌套动画时必须） -->
>
```

JS 钩子（动画用库时）：

```vue
<Transition @before-enter="onBeforeEnter" @enter="onEnter" @after-enter="onEnterDone" @leave="onLeave">
```

CSS 与 JS 混用时注意：默认以 CSS 过渡结束为信号，纯 JS 动画要设 `:css="false"` 并在 done 回调里收尾。

## 3. TransitionGroup

```vue
<TransitionGroup name="list" tag="ul">
  <li v-for="item in items" :key="item.id">{{ item.text }}</li>
</TransitionGroup>

<style>
.list-enter-from, .list-leave-to { opacity: 0; transform: translateY(20px); }
.list-leave-active { position: absolute; }   <!-- 移动动画流畅的关键 -->
.list-move { transition: transform .3s; }    <!-- 排序位移动画 -->
</style>
```

铁律：**每个子元素必须有 key**；leave-active 设 absolute 让移动平滑。

## 4. KeepAlive

```vue
<RouterView v-slot="{ Component }">
  <KeepAlive :include="['UserList', 'OrderList']" :max="10">
    <component :is="Component" />
  </KeepAlive>
</RouterView>
```

| 属性 | 说明 |
|---|---|
| `include` | 组件 name 白名单（须与组件显式 name 匹配，script setup 需 defineOptions） |
| `exclude` | 黑名单 |
| `max` | LRU 缓存上限（防内存膨胀，必设） |

组件 name 匹配注意：`<script setup>` 组件默认从文件名推断 name，但推断 name **不保证**参与 include 匹配（dev 可用，build 场景不稳）——显式声明：

```ts
defineOptions({ name: 'UserList' })
```

## 5. 缓存生命周期

```ts
onActivated(() => { /* 每次显示：刷新数据、恢复轮询 */ })
onDeactivated(() => { /* 每次隐藏：暂停轮询、清理临时态 */ })
```

典型策略：
- 进页面刷新数据的逻辑：`onMounted`（首次）+ `onActivated`（后续）
- 或者只在 `onActivated` 里做「非首次也刷新」判断

## 6. KeepAlive 与异步组件/Teleport

- KeepAlive 内放异步组件：首次激活加载，之后走缓存（无需 v-if 重建）
- KeepAlive 不能与 Teleport 同节点冲突，Teleport 内容不走缓存

## 7. 路由页面缓存实践

```ts
// meta 控制缓存
{ path: '/list', meta: { keepAlive: true } }
```

```vue
<RouterView v-slot="{ Component, route }">
  <KeepAlive :max="15">
    <component :is="Component" v-if="route.meta.keepAlive" :key="route.fullPath" />
    <component :is="Component" v-else :key="route.fullPath" />
  </KeepAlive>
</RouterView>
```

缓存 + key 的取舍：需要「列表页保留筛选状态，详情页每次重建」时，用 include 白名单比 meta+v-if 更清晰。

## 8. 动画与缓存组合

KeepAlive 包裹的组件切换同样可以套 Transition：

```vue
<RouterView v-slot="{ Component }">
  <Transition name="page" mode="out-in">
    <KeepAlive :max="10">
      <component :is="Component" />
    </KeepAlive>
  </Transition>
</RouterView>
```

## 常见坑

1. **Transition 内两个元素需要 key** 且不同（相同标签元素切换时浏览器复用，动画失效）。
2. **TransitionGroup 忘记 leave-active: absolute**：删除项时其余元素跳动。
3. **KeepAlive include 匹配不上**：script setup 组件没显式 defineOptions name。
4. **缓存无限增长**：内存泄漏，max 必设。
5. **缓存的表单页数据陈旧**：onActivated 中判断刷新或主动 invalidate。
6. **mode="out-in" 下嵌套 Transition 时长叠加**：显式 duration 或统一 CSS。
7. **纯 JS 动画未设 :css="false"**：Vue 等 CSS 事件超时导致卡顿。

## Checklist

- [ ] Transition 元素/组件 key 唯一且不同
- [ ] TransitionGroup 子项有 key 且 leave-active 绝对定位
- [ ] KeepAlive 设置 include/exclude + max
- [ ] script setup 组件需要缓存时显式 defineOptions name
- [ ] 缓存页数据新鲜度策略明确（onActivated 刷新）
- [ ] 列表页返回保留筛选、详情页重建的诉求已验证
