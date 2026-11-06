# 29 - Diff 算法与协调

## 1. 协调（Reconciliation）定义

新 JSX（或重渲染结果）与 current Fiber 树的比较过程，产出最小 DOM 操作（flags）。核心假设三条（降低复杂度 O(n³)→O(n)）：

1. **不同类型元素 → 直接重建子树**（`<div>` → `<span>` 不复用）
2. **同类型元素 → 复用 DOM，只更新变化的属性**
3. **同级 children 用 key 标识**跨渲染对应关系

## 2. 单元素 diff

```
type 相同：
  - DOM 元素：比对 props，标记变化的属性（flags: Update）
  - 函数组件：比较 props，是否 re-render（默认 re-render，memo 控制）
type 不同：
  - 卸载旧树（含 cleanup），挂载新树（state 全新）
```

这就是「切换组件类型 = 状态重置」的原理；同类型想强制重建用 **key**：

```tsx
<ProfileForm key={userId} userId={userId} />   // key 变 → 重建 → state 重置
```

## 3. children diff（同层级列表）

React 对 children 数组的 diff 策略（简化模型）：

```
1. 第一轮：从左到右按顺序（index+key）逐个比对，遇到 key/type 不匹配即停
2. 第二轮：剩余老节点建 Map<key, fiber>；遍历剩余新 children：
   - key 匹配 → 复用（标记移动与否）
   - 不匹配 → 新建
3. 老节点中未被复用的 → 删除
4. 移动位置通过 lastPlacedIndex 判断（右移最小化原则）
```

例：`[a b c] → [b a c]`：第一轮 a vs b key 不匹配停；第二轮 b/a 复用并标记移动，c 复用。React 18/19 的实际算法有预优化路径（key 匹配优先），但语义一致：**key 是身份，index 只是兜底**。

## 4. key 规则的原理推导

| 用法 | 后果 |
|---|---|
| 稳定唯一 id | 身份正确，复用 + 最小移动 |
| index | 增删/排序时身份错位：状态（输入框/局部 state）跟错节点 |
| 随机数 | 每次渲染 key 全变 → 全部重建（性能 + 状态丢失） |
| 数组元素本身 | 复杂对象不稳定；用原始 id |

index 作 key 的经典 bug：列表项含非受控输入/内部 state，删除中间项后「后面的输入框内容串位」——因为 React 按 key(index) 复用了错位的 DOM。

## 5. 协调中的 props 比较

```
函数组件：新旧 props Object.is 浅比较（每个 prop），不一致 → re-render
memo 组件：浅比较全通过 → 跳过 re-render
浅比较的含义：props 里的对象/函数/数组每次新引用 → 永远「不相等」→ memo 失效
  → 引出 useCallback/useMemo（见 06）
```

## 6. Context 的协调行为

Provider value 变化 → 传播到子树中**消费 Context 的 Fiber**（React 会跳过不消费的子树吗？——不会跳过渲染，但只对消费者标记更新；中间非消费者依然进入 render 流程，除非 memo 边界）。所以 Context 重渲染问题依旧（见 07）。

## 7. 卸载的细节

- 卸载子树：cleanup effect（子先父后）→ 移除 DOM → 释放引用
- Portal 的卸载：DOM 层级无关，按组件树清理
- ErrorBoundary：子树错误 → 替换 fallback（见 14）

## 8. 与 Vue diff 的对照

| 维度 | React | Vue3 |
|---|---|---|
| 优化方式 | 运行时全量协调 + memo 手动/Compiler 自动 | 编译期 PatchFlag/Block 静态跳过 |
| key diff | 顺序两轮 + lastPlacedIndex | 头尾同步 + 最长递增子序列 |
| 静态内容 | 每次渲染重新创建 JSX 对象（Compiler 可优化） | 编译期静态提升 |
| 心智 | 「重算 + 对比」 | 「精确更新」 |

## 9. 实践启示

| 启示 | 来源 |
|---|---|
| key 稳定唯一 | 身份机制 |
| 不同 type = 状态重置 | 单元素规则 → 刻意用 key/type 切换做重置 |
| memo 配稳定引用 | props 浅比较 |
| 列表外层包裹元素类型稳定 | type 不同整树重建 |
| lazy 组件切换 | Suspense fallback + 重建语义 |

## 常见坑（原理层）

1. **key 随机/漂移**：全量重建或错位复用。
2. **条件渲染改变组件 type**：`{flag ? <A/> : <B/>}` 状态重置是特性；不想重置换 props 或 CSS。
3. **列表项包在不稳定 type 的容器里**（每次渲染不同组件）：全部重建。
4. **数组 children 中布尔/null**：合法占位（不渲染），但注意占位也参与位置计算。
5. **memo 内传新对象字面量**：`<Row config={{ a: 1 }} />` 每次新引用。

## Checklist

- [ ] 能解释三条 diff 假设
- [ ] key 稳定唯一，明白 index 兜底的错位机制
- [ ] 理解 type 切换 = 状态重置并加以利用（key 重置模式）
- [ ] memo 与引用稳定的因果关系清晰
