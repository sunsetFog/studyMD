# 28 - Fiber 架构与渲染机制

## 1. Fiber 是什么

React 16+ 的协调引擎重构：把渲染工作拆成**可中断的小单元**（Fiber 节点），实现并发渲染——时间片调度、优先级、可恢复。

每个组件对应一个 Fiber 节点（链表结构）：

```ts
interface Fiber {
  type: any               // 组件类型/标签
  key: string | null
  stateNode: any          // DOM 实例/class 实例
  return: Fiber | null    // 父
  child: Fiber | null     // 第一个子
  sibling: Fiber | null   // 兄弟
  alternate: Fiber | null // 双缓冲对应节点（current 树 ↔ workInProgress 树）
  memoizedState: any      // hooks 链表头
  memoizedProps: any
  lanes: Lanes            // 优先级（并发标记）
  flags: Flags            // 副作用标记（增/删/更新）
}
```

链表结构（return/child/sibling）的意义：**渲染可中断**——遍历是循环而非递归，随时保存当前位置恢复（递归调用栈做不到）。

## 2. 双缓冲树

```
current 树    ：当前屏幕显示的 Fiber 树
workInProgress 树：正在构建的新树（复用 alternate 复用节点）
构建完成 → commit → current = workInProgress（交换指针，复用旧树继续做下下次）
```

好处：渲染期间屏幕保持旧内容（可中断不白屏），DOM 引用/状态在 alternate 间复用（useState 状态保存在 fiber.memoizedState）。

## 3. 渲染两阶段

### 3.1 Render 阶段（可中断）

- 从触发更新的 Fiber 开始**向上找到根**，向下遍历构建 workInProgress
- 执行组件函数 → 计算 hooks 状态 → diff 子节点（协调）
- 纯计算，无 DOM 操作 → 可随时中断/重跑/丢弃
- 优先级：高优（同步，如输入）可打断低优（transition）

### 3.2 Commit 阶段（同步不可中断）

分三个子阶段一次性完成：
1. **before mutation**：getSnapshotBeforeUpdate / 读 DOM 前状态
2. **mutation**：应用 DOM 增删改（按 flags）
3. **layout**：useLayoutEffect 同步执行、ref 赋值

然后浏览器绘制，之后异步执行 **passive effects**（useEffect，由调度器安排）。

```
可中断的是 Render；Commit 必须原子（否则用户看到半成品 DOM）
```

## 4. 调度器（Scheduler）

```
 lane（优先级赛道）：SyncLane > InputContinuousLane > DefaultLane > TransitionLane > IdleLane
 每帧预算 ~5ms：requestIdleCallback 思想的 MessageChannel 实现
 过期任务强制同步执行（防饿死）
```

`startTransition` 标记的更新 = TransitionLane（低优、可中断、可被打断丢弃重算）；输入类 = InputContinuousLane（高优）。

## 5. Hooks 与 Fiber 的关系

- Hooks 状态存在 **fiber.memoizedState**——它是一条按调用顺序的链表（见 `30-hooks-internals.md`）
- useState 的 setState 实际是 dispatch：把 update 挂到 hook 队列，并调度重渲染
- effect 存在 fiber.updateQueue，commit 后由调度器 flush

## 6. 更新触发全链路

```
setState/update
  → 创建 update 对象，标记 fiber.lanes（优先级）
  → ensureRootIsScheduled：调度器安排渲染任务
  → render 阶段（可中断）：从根向下 beginWork / completeWork
  → 完成后进入 commit（同步）
  → layout effects → 浏览器绘制 → passive effects
```

`beginWork`：处理组件（执行函数组件、比较 props 决定复用）；`completeWork`：收集 flags、创建/复用 DOM。

## 7. 为什么渲染必须纯

并发语义下组件函数可能：
- 执行两次（StrictMode / 中断重跑）
- 结果被丢弃（transition 打断）

React 的整个并发设计建立在「渲染 = 纯计算」之上。渲染期副作用 = 并发下不可预测行为。这也是 React Compiler 的自动 memoization 前提（见 10）。

## 8. 与 Vue 响应式的架构差异

| 维度 | React Fiber | Vue 响应式 |
|---|---|---|
| 更新粒度 | 从根开始整树遍历（配合 memo 跳过） | 依赖收集精确到组件 |
| 更新时机 | 调度器优先级队列 | 队列去重 nextTick |
| 中断能力 | render 可中断 | Vue 渲染小，无需中断 |
| 心智 | 「重新算一遍」 | 「谁变了更新谁」 |

## 常见坑（原理层解释）

| 现象 | 原理 |
|---|---|
| StrictMode 双执行 | render 阶段可重跑的检测机制 |
| effect 双挂载 | 挂载→清理→再挂载模拟可丢弃渲染 |
| transition 更新被丢 | 低优 lane 可被高优打断后整体丢弃 |
| setState 后同步读不到 | commit 未发生（render 异步调度中） |
| Hook 不能条件调用 | memoizedState 链表按顺序索引 |

## Checklist

- [ ] 能画出双缓冲树与两阶段流程
- [ ] 理解 Render 可中断 / Commit 原子
- [ ] 知道 lanes 优先级与 transition 的关系
- [ ] 能用纯函数纪律解释并发行为
