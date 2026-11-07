# 01 - HTTP 与网络

## 1. HTTP 各版本演进（高频）

| 版本 | 关键特性 | 解决的问题 |
|---|---|---|
| 1.0 | 短连接 | - |
| 1.1 | 长连接（keep-alive）、管线化、Host、Chunked | 重复握手开销 |
| 2 | 二进制分帧、多路复用、头部压缩 HPACK、服务端推送 | 1.1 队头阻塞（HTTP 层） |
| 3 | QUIC（UDP）、0-RTT、连接迁移 | TCP 层队头阻塞 |

答题要点：1.1 的队头阻塞在 HTTP 层，2 多路复用解决；TCP 层的丢包阻塞由 3 的 QUIC 解决。

## 2. HTTPS 握手（必考）

```
1. ClientHello：随机数 + 支持的加密套件
2. ServerHello：随机数 + 证书（含公钥）
3. 客户端验证证书 → 生成预主密钥，用公钥加密发送
4. 双方用三个随机数算出会话密钥，之后对称加密通信
```

- TLS 1.3：1-RTT 握手，删弱套件
- 证书验证链：根 CA → 中间 CA → 站点证书

## 3. TCP 三次握手 / 四次挥手（必考）

```
握手：SYN → SYN+ACK → ACK
   三次原因：双方确认彼此收发能力都正常 + 防止历史连接误建
挥手：FIN → ACK → FIN → ACK，TIME_WAIT 等待 2MSL
   TIME_WAIT 原因：确保最后 ACK 达到 + 让旧报文消亡
```

前端关联：`ERR_CONNECTION_RESET/TIMED_OUT`、浏览器同域名 6 个 TCP 连接限制。

## 4. 状态码（背全常用）

| 码 | 含义 | 面试追问点 |
|---|---|---|
| 200/201/204 | 成功/已创建/无内容 | RESTful POST 返回 201 |
| 301/302 | 永久/临时重定向 | SEO 影响；307 保持方法 |
| 304 | Not Modified | 配合缓存协商 |
| 400/401/403/404 | 参数/未认证/无权限/不存在 | 401 vs 403 区分 |
| 429 | Too Many Requests | 限流响应 |
| 500/502/503/504 | 内部错误/网关错误/不可用/网关超时 | 502=后端挂，504=后端超时 |

## 5. HTTP 缓存（高频）

```
强缓存：Cache-Control: max-age=3600 / no-cache / no-store / immutable
协商缓存：Last-Modified + If-Modified-Since
        ETag + If-None-Match（优先级更高，解决秒级修改问题）
命中协商 → 304 不传 body
```

实践策略：`index.html` 用 no-cache（协商），带 hash 的静态资源用 `max-age=31536000, immutable`。

## 6. 从输入 URL 到页面显示（万能串联题）

```
DNS 解析（缓存链：浏览器→hosts→本地 DNS→迭代查询）
→ TCP 握手 / TLS 握手
→ 发送 HTTP 请求
→ 服务端处理（Nginx → 应用）
→ 响应回来 → 解析 HTML → DOM/CSSOM → 渲染树 → 布局 → 绘制
→ JS 执行（defer/async 差异）
```

每个环节都能展开（DNS 劫持、CDN、白屏优化）——按面试方向选深度。

## 7. CDN 与跨域预备

- CDN 原理：DNS CNAME 到边缘节点 + 内容分发 + 回源；缓存刷新与预热
- 跨域是浏览器行为，解决方案见 04/20（CORS 为主）

## 常见追问

1. 为什么 HTTPS 同时用对称和非对称？——非对称交换密钥（安全），对称加密数据（性能）
2. HTTP2 还慢在哪？——TCP 队头阻塞、握手 RTT
3. ETag 怎么生成的？——内容哈希/文件 mtime+size，实现相关

## Checklist

- [ ] 能画 HTTPS 握手时序
- [ ] 能讲清 1.1/2/3 队头阻塞的区别
- [ ] 强缓存 vs 协商缓存能给出生产策略
