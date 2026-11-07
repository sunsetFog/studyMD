# 21 - 实时通信与文件处理（全栈整合）

## 1. 实时通信方案对比（必考⭐）

| 方案 | 原理 | 特点 |
|---|---|---|
| 短轮询 | 定时请求 | 实现简单、延迟高、大量无效请求 |
| 长轮询 | 请求挂起直到有数据/超时 | 兼容性好、占连接 |
| SSE | 服务端单向流（text/event-stream） | 简单、自动重连、只能服务器→客户端（AI 流式输出标配） |
| **WebSocket** | HTTP 升级后全双工 | 实时双向、需处理心跳/重连 |

选型：只需服务器推送（通知/进度/流式 AI）→ SSE；双向交互（聊天/协同）→ WebSocket。

## 2. WebSocket 实战（前后端）

```java
// SpringBoot
@ServerEndpoint("/ws/chat/{userId}")          // 或 STOMP：@MessageMapping
public void onMessage(String msg) { ... }
// 集群问题：用户连在节点 A，消息却发给节点 B → Redis Pub/Sub 或 MQ 广播到所有节点，本地会话表匹配
```

```ts
// 前端心跳与重连（必答细节）
const ws = new WebSocket(url);
const heartbeat = setInterval(() => ws.send('ping'), 30000);
ws.onclose = () => { clearInterval(heartbeat); reconnectWithBackoff(); };
```

- 心跳防代理/NAT 超时断连；指数退避重连
- STOMP：消息语义层（订阅/发布路径），Spring 支持好

## 3. 文件上传（必考⭐）

### 小文件直传

```html
<input type="file" @change="onFile" />
const fd = new FormData(); fd.append('file', file);
axios.post('/api/upload', fd);   // multipart/form-data，后端 MultipartFile
```

- 后端校验：大小、类型（魔数而非后缀）、重命名（UUID/雪花，防路径穿越）、存对象存储
- 存储选型：本地（单机调试）vs MinIO/OSS（生产）；返回 URL/预签名

### 大文件三件套（高频）

```
分片上传：前端 Blob.slice 切片并发上传（控制并发 3~5）
断点续传：已传分片记录（后端查已有分片），续传只补缺
秒传：文件 hash（spark-md5 抽样+全量）→ 服务端已有则直接返回 URL
合并：全部分片到达后按序合并（或 OSS CompleteMultipartUpload）
```

### 大文件直传 OSS（生产首选）

```
后端签发预签名 URL（PutObject）→ 前端直传 OSS → 回调通知后端
优点：应用不扛带宽；需配置 CORS + 回调鉴权
```

## 4. 文件下载与安全

- `Content-Disposition: attachment; filename*=UTF-8''xxx`（中文文件名编码）
- 流式下载：`StreamUtils.copy` / OSS 预签名 GET，防内存爆
- 权限校验：下载链接必须鉴权（预签名短有效期），防 URL 外泄
- 防 zip 炸弹/路径穿越：解压大小限制、`File.getCanonicalPath` 校验

## 5. 进度显示与体验

- axios `onUploadProgress`（本地上传进度）
- 分片场景：已完成分片/总数实时更新 + 服务端合并不确定时的轮询
- 上传失败重试单个分片（非整个文件）

## 常见追问

1. WebSocket 集群怎么广播？——节点注册到 Redis（用户→节点映射）定向推送，或 pub/sub 广播
2. 分片上传怎么保证顺序？——分片号 + 服务端按序合并；或对象存储分片 API 天然管理
3. SSE 断线重连？——`Last-Event-ID` 头恢复，服务端从该 ID 之后继续推

## Checklist

- [ ] 四种实时方案能按场景选
- [ ] 大文件分片/续传/秒传三件套能讲全流程
- [ ] 知道预签名直传是生产首选
