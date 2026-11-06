# 31 - 虚拟 DOM 与 diff

## 1. VNode 结构

```ts
interface VNode {
  type: string | Component | typeof Fragment | typeof Teleport | typeof Suspense
  props: object | null
  children: string | VNode[] | Slot
  key: string | number | null        // diff 的身份标识
  shapeFlag: number                  // 位运算标记节点类型（ELEMENT/COMPONENT/TEXT…）
  patchFlag: PatchFlags              // 编译期优化标记（见 32）
  dynamicChildren: VNode[] | null    // 编译期收集的动态子节点
}
```

shapeFlag 用位运算组合（ELEMENT | ARRAY_CHILDREN 等），判断分支零成本。

## 2. 更新流程全景

```
状态变更 → 触发组件渲染 effect → 生成新 VNode 树
→ patch(oldVNode, newVNode) → 递归对比 → 最小化 DOM 操作
```

patch 分派逻辑（按 type）：
- type 不同 → 直接卸载旧的、挂载新的
- 同 TEXT → 更新 textContent
- 同 ELEMENT → patchElement（props + children）
- 同 COMPONENT → 更新 props 触发子组件重渲染（子有自己的渲染 effect）

## 3. children diff 三种形态

| children 形态 | 处理 |
|---|---|
| 文本 | 直接替换 |
| 数组 ↔ 文本 | 卸载/挂载整体 |
| 数组 ↔ 数组 | keyed diff 算法 |

## 4. keyed diff 算法（核心）

新老子序列对比，五步走：

```
1. 从头同步：相同 type&key 的节点从头 patch
2. 从尾同步：从尾部对齐 patch
3. 仅新增：老的遍历完，新的剩余 → mount
4. 仅删除：新的遍历完，老的剩余 → unmount
5. 中间乱序 → 核心算法：
   a. 建立新子序列 key→index 映射
   b. 遍历剩余老节点：匹配到 → patch 并记录 newIndex；未匹配 → 卸载
   c. 求最长递增子序列（LIS）——不动的节点
   d. 非 LIS 内的节点按需移动/挂载
```

**最长递增子序列**是 Vue3 diff 相对 Vue2 双端比对的关键优化：尽量少的 DOM move。

例：`旧 [a b c d e] → 新 [a d e b c]`：头尾同步后剩余 `[b c] vs [d e b c]`，LIS 为 `[d e]`（下标 0,1 的映射递增），只有 b、c 需要移动。

## 5. key 的作用原理

- key 是 diff 中节点身份：同 key 同 type → 复用 DOM 更新；无 key → 只能就地更新或全量重建
- index 作 key：节点移动时 key 对不上内容，导致**状态错位复用**（输入框串位）
- key 稳定性要求：与内容对应关系不随排序/删除变化

## 6. 组件级 diff

```ts
// 父组件重渲染时遇到子组件 VNode：
shouldUpdateComponent(prevProps, nextProps)
// 优化路径：
// 1. 子组件是编译优化过的（PatchFlag 动态 props 明确）→ 只比较动态部分
// 2. 否则全量 props 对比
// 3. children 有 slot（动态）→ 强制更新
```

组件更新 ≠ 重渲染子树：更新 props 后，子组件是否重渲染由它自己的响应式依赖决定。

## 7. 卸载与优化细节

- unmount 递归处理组件/指令/Transition 钩子
- 带有 dynamicChildren 的区块跳过全量 children 遍历，只 diff 动态节点（配合 32）
- Fragment（多根/组件数组子）有稳定 Fragment 标记加速定位

## 8. 静态提升与树结构的关系

```
静态节点编译时提升到 render 外（hoisted）：
- 不参与每次 VNode 创建
- patch 时跳过（patchFlag = NO_PATCH / HOISTED）
```

静态内容越多，运行时创建与 diff 成本越低——Vue3 模板性能优势的来源之一（JSX 不享受此优化）。

## 9. 实践启示

| 启示 | 说明 |
|---|---|
| key 稳定唯一 | diff 正确性根基 |
| 避免深层组件树无意义嵌套 | 每层都有 patch 成本 |
| 大列表虚拟滚动 | 减少参与 diff 的节点数 |
| v-if/v-show 选型影响 diff | v-show 不销毁节点，diff 成本固定 |
| v-memo | 缩短参与 diff 的子树 |

## 常见坑（原理层）

1. **key 不稳定**（index/随机数）：LIS 判断错乱，非必要 DOM 移动 + 状态串位。
2. **同标签无 key 切换**：浏览器就地复用，过渡动画失效。
3. **数组顺序打乱 + 组件状态**：组件复用错位，props 与内部状态不一致。
4. **deep watch + 大数组**：trigger 频繁，diff 压力倍增。

## Checklist

- [ ] 能手推 LIS diff 的一例
- [ ] 明白 key 身份机制并落地到所有 v-for
- [ ] 理解组件更新与子树重渲染的区别
- [ ] 知道 dynamicChildren/patchFlag 的运行时收益来源
