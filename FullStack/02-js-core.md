# 02 - JavaScript 核心

## 1. 数据类型与判断

- 7 种原始类型（string/number/boolean/null/undefined/symbol/bigint）+ Object
- 类型判断：`typeof`（null→object、函数→function）、`instanceof`（原型链）、`Object.prototype.toString.call()`（最准）
- == vs ===：隐式转换规则（null==undefined、NaN 不等自身）——面试考 `[ ]==false`、`[]+{}`
- 深拷贝：`structuredClone()`（原生）/ 递归（处理循环引用用 WeakMap）

## 2. 作用域与闭包（必考）

```js
function counter() {
  let n = 0;
  return () => ++n;   // 闭包：函数 + 定义时的词法环境
}
```

- 考点：词法作用域 vs 动态作用域、var 的函数级作用域与循环陷阱、IIFE
- 应用：模块封装、防抖节流、柯里化、缓存记忆化
- 缺陷：闭包引用的变量无法被回收 → 内存泄漏

## 3. 原型与原型链（必考）

```
实例.__proto__ === 构造函数.prototype
prototype.constructor === 构造函数
查找链：实例 → 构造函数.prototype → Object.prototype → null
```

- `new` 做了什么：创建空对象 → 绑定原型 → 执行构造函数（this 指向新对象）→ 返回（构造函数返回对象则覆盖）
- 手写 `Object.create`、手写 `new`、`instanceof` 手写是经典手写题
- class 本质：原型语法糖；extends 的 static/super 语义

## 4. this 与执行上下文

| 调用方式 | this 指向 |
|---|---|
| 普通函数调用 | undefined（严格）/ window |
| 方法调用 obj.fn() | obj |
| new 调用 | 新实例 |
| 箭头函数 | 定义时外层（词法） |
| call/apply/bind | 显式指定 |

优先级：new > 显式 > 隐式 > 默认。手写 call/bind 是常考手写题。

## 5. 事件循环 Event Loop（⭐⭐必考）

```
执行栈空 → 取一个宏任务（setTimeout/IO/UI事件）→ 清空全部微任务（Promise.then/queueMicrotask/MutationObserver）→ 渲染检查
```

```js
console.log(1);
setTimeout(() => console.log(2));
Promise.resolve().then(() => console.log(3));
console.log(4);            // 1 4 3 2
```

进阶：async/await 即 Promise 语法糖（await 后续相当于 then 微任务）；Node 的 process.nextTick 优先于 Promise 微任务；`requestAnimationFrame` 在渲染前。

## 6. Promise / async-await

- 三状态不可逆：pending → fulfilled / rejected
- 手写 Promise（then 链、值穿透）、Promise.all/race/allSettled/any 区别——常考手写
- `async` 函数总是返回 Promise；错误用 try/catch 或 `.catch` 接
- 经典题：并发限制器（最大并发 3 的请求池）

```js
async function pool(tasks, limit) {
  const ret = [];
  const executing = new Set();
  for (const t of tasks) {
    const p = Promise.resolve(t()).then(v => { executing.delete(p); return v; });
    executing.add(p); ret.push(p);
    if (executing.size >= limit) await Promise.race(executing);
  }
  return Promise.all(ret);
}
```

## 7. 防抖节流（必考手写）

```js
// 防抖：停顿 N ms 执行最后一次（搜索输入）
function debounce(fn, ms) {
  let t;
  return (...args) => { clearTimeout(t); t = setTimeout(() => fn(...args), ms); };
}
// 节流：每 N ms 最多执行一次（滚动/resize）
function throttle(fn, ms) {
  let last = 0;
  return (...args) => {
    const now = Date.now();
    if (now - last >= ms) { last = now; fn(...args); }
  };
}
```

选型：搜索建议用防抖；滚动加载用节流。

## 8. ES6+ 高频语法点

- 解构、展开、可选链 `?.`、空值合并 `??`
- 模板字符串、Map/Set、WeakMap/WeakSet（弱引用防泄漏）
- Generator 与迭代器协议（for...of 的底层）
- Proxy/Reflect（Vue3 响应式的基础，见 06）

## 常见追问

1. `setTimeout(fn,0)` 何时执行？——下轮宏任务，有 4ms 最小嵌套延迟
2. 为什么数组有空位陷阱？——`forEach` 跳过空位、稀疏数组语义
3. 事件委托：利用冒泡，`e.target` 判断，减少监听器

## Checklist

- [ ] 事件循环题手推顺序无错
- [ ] 手写：防抖节流、Promise、new、call、并发池
- [ ] 能讲闭包的应用与泄漏场景
