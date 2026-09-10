# 实现思路

## 一、React 下载一个游戏 App

### 实现思路

做一个游戏下载类 App（类似游戏商店/下载站），核心是：游戏列表展示 + 点击下载 + 下载进度显示 + 下载状态管理。

```
页面结构
├── 游戏列表页  展示游戏封面、名称、大小、下载按钮
└── 游戏详情页  展示详情、下载地址、下载进度

核心流程
获取游戏列表 → 点击下载 → 流式下载（带进度）→ 显示进度条 → 完成
```

### 技术要点

```
1. 用 axios 流式下载（responseType: 'blob'），用 onDownloadProgress 拿进度
2. 用 <a download> 触发浏览器下载
3. 用 state 管理每个游戏的下载状态（未下载/下载中/已完成）
```

### 代码示例

```
// 1. 游戏列表数据（通常从后端接口获取）
const games = [
    { id: 1, name: '王者荣耀', size: '1.2GB', cover: '/img/wzry.jpg', url: '/api/download/1' },
    { id: 2, name: '原神', size: '2.5GB', cover: '/img/ys.jpg', url: '/api/download/2' }
]
```

```
// 2. 下载函数：带进度
import axios from 'axios'

async function downloadGame(game, onProgress) {
    // responseType: 'blob' 表示按二进制流接收
    const res = await axios.get(game.url, {
        responseType: 'blob',
        onDownloadProgress: (e) => {
            // e.loaded 已下载字节，e.total 总字节
            const percent = Math.round((e.loaded / e.total) * 100)
            onProgress(percent)     // 回调更新进度
        }
    })

    // 把 blob 转成对象 URL，触发浏览器下载
    const blobUrl = URL.createObjectURL(res.data)
    const a = document.createElement('a')
    a.href = blobUrl
    a.download = game.name + '.apk'   // 下载文件名
    a.click()
    URL.revokeObjectURL(blobUrl)      // 释放内存
}
```

```
// 3. 游戏列表组件：管理每个游戏的下载状态
import { useState } from 'react'

function GameList() {
    // 记录每个游戏的下载进度（id -> 百分比），-1 表示未下载
    const [progressMap, setProgressMap] = useState({})

    const handleDownload = (game) => {
        // 开始时进度置 0
        setProgressMap(prev => ({ ...prev, [game.id]: 0 }))

        downloadGame(game, (percent) => {
            setProgressMap(prev => ({ ...prev, [game.id]: percent }))
        })
    }

    return (
        <div>
            {games.map(game => {
                const progress = progressMap[game.id]   // undefined 表示还没下载

                return (
                    <div key={game.id}>
                        <img src={game.cover} />
                        <p>{game.name}（{game.size}）</p>
                        {/* 下载中显示进度条，否则显示下载按钮 */}
                        {progress !== undefined && progress < 100 ? (
                            <progress value={progress} max="100" />
                        ) : (
                            <button onClick={() => handleDownload(game)}>
                                {progress === 100 ? '已完成' : '下载'}
                            </button>
                        )}
                    </div>
                )
            })}
        </div>
    )
}
```

### 剪切板代理 ID 处理（保证不丢失）

下载时需要一个「代理 ID」（从剪切板获取），核心是：读取 → 持久化 → 下载携带 → 提供手动兜底，确保代理 ID 刷新、重进、多次下载都不丢失。

```
处理流程
1. 优先读 localStorage（已保存过就直接用，不丢失）
2. localStorage 没有 → 从剪切板读（navigator.clipboard.readText）
3. 读到后立即存 localStorage，持久化
4. 下载时把代理 ID 放进请求头，后端据此校验/转发
5. 剪切板读不到时，提供手动输入 + 粘贴按钮兜底
```

```
// 1. 从剪切板读取代理 ID
async function readProxyIdFromClipboard() {
    try {
        // navigator.clipboard 需要 HTTPS 或 localhost 环境
        const text = await navigator.clipboard.readText()
        return text.trim()      // 去掉首尾空格/换行
    } catch (err) {
        console.error('读取剪切板失败', err)
        return null             // 权限被拒或浏览器不支持时返回 null
    }
}
```

```
// 2. 持久化代理 ID（localStorage），保证刷新/重进不丢失
function saveProxyId(proxyId) {
    localStorage.setItem('proxyId', proxyId)
}

function getProxyId() {
    return localStorage.getItem('proxyId') || ''
}
```

```
// 3. 初始化代理 ID：优先读 localStorage，没有则从剪切板读并立即保存
async function initProxyId() {
    let proxyId = getProxyId()

    if (!proxyId) {
        // localStorage 没有，尝试从剪切板读
        proxyId = await readProxyIdFromClipboard()
        if (proxyId) {
            saveProxyId(proxyId)   // 读到了立即保存，防止丢失
        }
    }

    return proxyId
}
```

```
// 4. 下载时携带代理 ID（改造下载函数，加 proxyId 参数）
async function downloadGame(game, proxyId, onProgress) {
    const res = await axios.get(game.url, {
        responseType: 'blob',
        headers: {
            'X-Proxy-Id': proxyId   // 代理 ID 放请求头，后端据此校验/转发
        },
        onDownloadProgress: (e) => {
            const percent = Math.round((e.loaded / e.total) * 100)
            onProgress(percent)
        }
    })

    // 触发浏览器下载
    const blobUrl = URL.createObjectURL(res.data)
    const a = document.createElement('a')
    a.href = blobUrl
    a.download = game.name + '.apk'
    a.click()
    URL.revokeObjectURL(blobUrl)
}
```

```
// 5. 代理 ID 设置组件：展示 + 粘贴 + 手动保存（兜底，保证不丢失）
import { useState, useEffect } from 'react'

function ProxyIdSetting({ onReady }) {
    const [proxyId, setProxyId] = useState(getProxyId())   // 初始值从 localStorage 读

    // 挂载时若还没有代理 ID，自动从剪切板读
    useEffect(() => {
        ;(async () => {
            if (!proxyId) {
                const text = await readProxyIdFromClipboard()
                if (text) {
                    setProxyId(text)
                    saveProxyId(text)   // 读到了保存，防止丢失
                    onReady?.(text)
                }
            }
        })()
    }, [])

    const handleSave = () => {
        saveProxyId(proxyId)          // 手动保存，保证不丢失
        onReady?.(proxyId)            // 通知父组件代理 ID 已就绪
    }

    const handlePaste = async () => {
        const text = await readProxyIdFromClipboard()
        if (text) {
            setProxyId(text)
            saveProxyId(text)          // 粘贴后立即保存
        }
    }

    return (
        <div>
            <input
                value={proxyId}
                onChange={e => setProxyId(e.target.value)}
                placeholder="代理 ID"
            />
            <button onClick={handlePaste}>从剪切板粘贴</button>
            <button onClick={handleSave}>保存</button>
        </div>
    )
}
```

```
// 6. 游戏列表组件整合代理 ID
function GameList() {
    const [proxyId, setProxyId] = useState(getProxyId())
    const [progressMap, setProgressMap] = useState({})

    const handleDownload = (game) => {
        // 下载前校验代理 ID 是否存在，防止丢失导致下载失败
        if (!proxyId) {
            alert('请先设置代理 ID')
            return
        }
        setProgressMap(prev => ({ ...prev, [game.id]: 0 }))
        downloadGame(game, proxyId, (percent) => {
            setProgressMap(prev => ({ ...prev, [game.id]: percent }))
        })
    }

    return (
        <div>
            <ProxyIdSetting onReady={setProxyId} />
            {/* ...游戏列表渲染（同前面示例） */}
        </div>
    )
}
```

---

## 二、登录

### 实现思路

实现完整的登录功能，核心链路：表单校验 → 调用登录接口 → 存储 token → 路由守卫 → 请求携带 token → token 失效处理。

```
登录流程
1. 用户填写账号密码
2. 前端校验（非空、格式）
3. 调用登录接口，后端返回 token
4. 前端把 token 存到 localStorage
5. 后续请求在 header 携带 token
6. 未登录访问受保护页面 → 重定向到登录页
```

### 技术要点

```
1. token 用 localStorage 存储（简单），或用 cookie（安全，可配 httpOnly）
2. axios 请求拦截器统一加 Authorization header
3. 响应拦截器统一处理 401（token 失效）
4. 路由守卫：未登录跳登录页
```

### 代码示例

```
// 1. 登录接口调用 + token 存储
import axios from 'axios'

async function login(username, password) {
    // 调用后端登录接口
    const res = await axios.post('/api/login', { username, password })
    // 后端返回 token，存到 localStorage
    const { token } = res.data
    localStorage.setItem('token', token)
    return token
}
```

```
// 2. axios 拦截器：请求自动携带 token，响应统一处理 401
// 请求拦截器
axios.interceptors.request.use(config => {
    const token = localStorage.getItem('token')
    if (token) {
        // 在请求头带上 token
        config.headers.Authorization = `Bearer ${token}`
    }
    return config
})

// 响应拦截器
axios.interceptors.response.use(
    res => res,
    err => {
        if (err.response?.status === 401) {
            // token 失效，清除并跳转登录页
            localStorage.removeItem('token')
            window.location.href = '/login'
        }
        return Promise.reject(err)
    }
)
```

```
// 3. 登录表单组件（含校验）
import { useState } from 'react'

function Login() {
    const [username, setUsername] = useState('')
    const [password, setPassword] = useState('')
    const [error, setError] = useState('')

    const handleSubmit = async (e) => {
        e.preventDefault()

        // 前端校验
        if (!username || !password) {
            setError('账号和密码不能为空')
            return
        }

        try {
            await login(username, password)
            window.location.href = '/home'   // 登录成功跳转
        } catch (err) {
            setError('登录失败，请检查账号密码')
        }
    }

    return (
        <form onSubmit={handleSubmit}>
            <input value={username} onChange={e => setUsername(e.target.value)} placeholder="账号" />
            <input value={password} onChange={e => setPassword(e.target.value)} type="password" placeholder="密码" />
            {error && <p style={{ color: 'red' }}>{error}</p>}
            <button type="submit">登录</button>
        </form>
    )
}
```

```
// 4. 路由守卫：未登录不能访问受保护页面
import { Navigate } from 'react-router-dom'

function RequireAuth({ children }) {
    const token = localStorage.getItem('token')

    // 没有 token 则重定向到登录页
    if (!token) {
        return <Navigate to="/login" replace />
    }

    return children
}

// 使用
<Route path="/home" element={
    <RequireAuth>
        <Home />
    </RequireAuth>
} />
```

```
// 5. 退出登录：清除 token
function logout() {
    localStorage.removeItem('token')
    window.location.href = '/login'
}
```

---

## 三、WebSocket

### 实现思路

WebSocket 用于实时通信（聊天、实时推送、游戏状态同步）。核心：建立连接 → 收发消息 → 心跳保活 → 断线重连 → 封装成 Hook。

```
连接流程
new WebSocket(url) → onopen 连接成功 → onmessage 收消息
→ onerror 出错 → onclose 关闭 → 定时重连
```

### 技术要点

```
1. 原生 WebSocket API：onopen / onmessage / onerror / onclose / send
2. 心跳：定时发 ping 包，判断连接是否存活
3. 断线重连：onclose 后定时重连，避免消息丢失
4. 用 useRef 保存实例，避免重渲染时重复创建
5. 组件卸载时关闭连接（useEffect 返回清理函数）
```

### 代码示例

```
// 1. 封装 WebSocket Hook
import { useEffect, useRef, useState } from 'react'

function useWebSocket(url) {
    const wsRef = useRef(null)              // 保存 WebSocket 实例
    const [message, setMessage] = useState(null)   // 最新收到的消息
    const [status, setStatus] = useState('connecting')  // 连接状态
    const reconnectTimer = useRef(null)     // 重连定时器

    const connect = () => {
        const ws = new WebSocket(url)        // 建立连接
        wsRef.current = ws

        ws.onopen = () => {
            setStatus('connected')           // 连接成功
            clearTimeout(reconnectTimer.current)
        }

        ws.onmessage = (e) => {
            setMessage(e.data)               // 收到消息
        }

        ws.onerror = () => {
            setStatus('error')               // 连接出错
        }

        ws.onclose = () => {
            setStatus('closed')
            // 断线后 3 秒自动重连
            reconnectTimer.current = setTimeout(connect, 3000)
        }
    }

    // 发送消息
    const send = (data) => {
        if (wsRef.current?.readyState === WebSocket.OPEN) {
            wsRef.current.send(JSON.stringify(data))   // 发送（通常发 JSON 字符串）
        }
    }

    useEffect(() => {
        connect()
        // 组件卸载时关闭连接并清理定时器
        return () => {
            clearTimeout(reconnectTimer.current)
            wsRef.current?.close()
        }
    }, [url])

    return { message, status, send }
}
```

```
// 2. 心跳保活：定时发 ping 判断连接是否存活
// 在 connect 函数里加心跳定时器
const heartbeat = setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) {
        ws.send('ping')     // 发心跳包，服务端应回 pong
    }
}, 10000)   // 每 10 秒一次

// 在 onclose 里清除
clearInterval(heartbeat)
```

```
// 3. 聊天组件：使用 useWebSocket
import { useState } from 'react'

function Chat() {
    const [list, setList] = useState([])     // 消息列表
    const [input, setInput] = useState('')

    const { message, status, send } = useWebSocket('ws://localhost:8080/chat')

    // 收到新消息时追加到列表（这里简化处理）
    // 实际应在 message 变化时用 useEffect 更新 list

    const handleSend = () => {
        send({ type: 'chat', content: input })   // 发送消息
        setInput('')
    }

    return (
        <div>
            <p>连接状态：{status}</p>
            <div>
                {list.map((msg, i) => <p key={i}>{msg}</p>)}
            </div>
            <input value={input} onChange={e => setInput(e.target.value)} />
            <button onClick={handleSend}>发送</button>
        </div>
    )
}
```

```
// 4. 服务端示例（Node.js + ws 库）
const WebSocket = require('ws')
const wss = new WebSocket.Server({ port: 8080 })

wss.on('connection', (ws) => {
    console.log('新连接')

    ws.on('message', (data) => {
        console.log('收到:', data.toString())
        // 广播给所有客户端
        wss.clients.forEach(client => {
            if (client.readyState === WebSocket.OPEN) {
                client.send(data)
            }
        })
    })

    ws.on('close', () => console.log('连接关闭'))
})
```

---

### 三个主题总结

| 主题 | 核心 | 关键技术 |
|------|------|----------|
| 游戏下载 App | 下载 + 进度管理 | axios blob 流式下载、onDownloadProgress、a[download] |
| 登录 | 认证 + 鉴权 | token 存储、axios 拦截器、路由守卫 |
| WebSocket | 实时通信 | onmessage/send、心跳、断线重连、Hook 封装 |
