# 11 - 渲染周期与事件系统

## 1. 组件「生命周期」真相

函数组件没有生命周期钩子，但有清晰的**渲染周期阶段**：

```
1. Trigger（触发）：setState / 父渲染 / forceUpdate / useSyncExternalStore 变化
2. Render（渲染）：执行组件函数，产出虚拟 DOM（可中断，见 28/31）
3. Commit（提交）：把 diff 结果应用到真实 DOM，同步执行
4. Layout Effects：useLayoutEffect 同步执行（绘制前）
5. 浏览器绘制
6. Passive Effects：useEffect 异步执行
```

class 组件对应关系（对照理解）：

| 阶段 | class 钩子 | 函数组件等价 |
|---|---|---|
| 挂载前 | constructor/getDerivedStateFromProps | useState 初始化 |
| 渲染 | render | 函数体本身 |
| 提交后（布局） | componentDidMount / componentDidUpdate | useLayoutEffect |
| 提交后（异步） | — | useEffect |
| 卸载 | componentWillUnmount | useEffect cleanup |

## 2. 渲染触发来源

```
1. 自身 setState / reducer dispatch
2. 父组件重渲染（props 引用变化与否不影响触发——父渲染则子默认重渲染，除非 memo）
3. Context value 变化
4. 外部 store（useSyncExternalStore）
5. useReducer/useState 由其他机制调度
```

重要认知：**父渲染 → 所有子组件默认重渲染**（与子组件是否用 props 无关）。这就是 memo/组件切分优化的由来（见 `25-performance.md`）。

## 3. 渲染必须是纯的、可重入的

并发模式下渲染可能：
- 被中断后**从头重跑**
- 在 StrictMode 下**双执行**
- 渲染结果被丢弃（transition 打断）

因此渲染函数体内禁止：
```
❌ 副作用（请求、DOM 操作、外部变量修改）
❌ 非确定性输出（Math.random/Date.now 直接进 JSX）
❌ 读写 ref.current（渲染期）
```

## 4. 事件系统

### 4.1 合成事件与委托

- React 统一包装原生事件为 **SyntheticEvent**（跨浏览器一致性）
- 事件**委托**挂载在 root 容器上（React 17 起，不再挂 document）——多版本 React 共存更安全
- `e.stopPropagation()` 阻止的是 React 合成事件的传播；原生监听器（addEventListener 直接挂 document）可能绕过它

### 4.2 事件时序与状态快照

```tsx
function handleClick() {
  setCount(c => c + 1)
  console.log(count)        // 旧值！state 更新在下一次渲染才可见
}
```

### 4.3 passive 优化

```tsx
useEffect(() => {
  const onScroll = () => …
  window.addEventListener('scroll', onScroll, { passive: true })
  return () => window.removeEventListener('scroll', onScroll)
}, [])
```

## 5. 提交阶段的行为细节

- **Commit 同步且不可中断**：DOM 变更一次完成
- ref 赋值发生在 commit（layout effect 前）
- cleanup 顺序：子先于父（unmount 时）
- effect 执行顺序：**子组件先于父组件**（与 Vue 一致）
  - 影响点：父 useEffect 中读取子 DOM 是安全的；但父在 effect 中改状态触发子 effect 时序要留意

## 6. 批处理与 nextTick 等价物

```tsx
import { flushSync } from 'react-dom'

setA(1); setB(2)            // 自动批处理 → 一次渲染
flushSync(() => setC(3))    // 立即同步渲染（阻塞式）
// 等价 Vue 的 nextTick 场景：
useEffect(() => { /* DOM 已更新 */ }, [dep])   // 或 flushSync + 立即读取
```

「setState 后想拿最新 DOM」的标准答案：**放 useEffect（依赖该状态）**，不要 flushSync 滥用。

## 7. StrictMode 的检测行为

开发期：
1. **组件函数双调用**（检测非纯渲染）
2. **effect 挂载→清理→再挂载**（检测非幂等副作用、忘 cleanup）
3. state updater 双调用（检测 updater 纯度）

生产零影响。**不要为了消除「双请求」关掉 StrictMode**，正确做法是幂等副作用 + cleanup + Query 缓存。

## 8. 渲染周期排错定位表

| 症状 | 定位 |
|---|---|
| 死循环（Maximum update depth） | 渲染期 setState / effect 依赖不稳引用互相触发 |
| effect 每次渲染都跑 | 依赖数组含每次新建的对象/数组/函数 |
| 界面没更新 | state 引用未变（原地修改）/ 条件没走 |
| 输入卡顿 | 渲染昂贵 + 未 transition/deferred |
| 子组件白渲染 | 父渲染连带；memo + 稳定 props |
| 卸载后 setState 警告 | 竞态缺 cleanup |

## 常见坑

1. **渲染期 setState**（含在 JSX 里调用）：无限循环。
2. **认为父渲染不会带动子**：默认会；优化靠 memo/结构拆分。
3. **stopPropagation 想拦截原生全局监听**：委托模型不一致，混用时明确层级。
4. **flushSync 滥用**：掉帧；90% 场景应使用 useEffect。
5. **把 useEffect 当 componentDidMount 用且依赖撒谎**：旧闭包 bug 温床。
6. **StrictMode 双执行当成 bug 修**：修的应是副作用幂等性。

## Checklist

- [ ] 能说出渲染五阶段及 effect 时机
- [ ] 渲染函数纯函数纪律无违规
- [ ] 死循环排错路径熟练（渲染期 setState/不稳依赖）
- [ ] setState 后读取最新值的姿势正确（effect 而非同步读）
- [ ] StrictMode 保持开启
