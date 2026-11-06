# 54 · WebSocket 与 STOMP

## 1. 为什么用 WebSocket

HTTP 是请求-响应、无状态；实时场景（IM、通知、行情、协同编辑）下轮询/长轮询浪费资源。

WebSocket：**一次握手，全双工长连接**，服务端可主动推送。

| 方案 | 说明 |
|---|---|
| 轮询 | 简单，浪费 |
| 长轮询 | 折中，仍有开销 |
| SSE | 单向（服务端→客户端），基于 HTTP，自动重连 |
| **WebSocket** | 双向，最通用 |

## 2. 原生 WebSocket（底层 API）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

```java
@Configuration
@EnableWebSocket
public class WsConfig implements WebSocketConfigurer {
    @Autowired private MyHandler handler;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(handler, "/ws")
                .setAllowedOriginPatterns("*")
                .addInterceptors(new HttpSessionHandshakeInterceptor());
    }
}

@Component
public class MyHandler extends TextWebSocketHandler {
    private static final Map<String, WebSocketSession> SESSIONS = new ConcurrentHashMap<>();

    @Override public void afterConnectionEstablished(WebSocketSession s) { SESSIONS.put(s.getId(), s); }
    @Override protected void handleTextMessage(WebSocketSession s, TextMessage m) throws IOException {
        s.sendMessage(new TextMessage("echo: " + m.getPayload()));
    }
    @Override public void afterConnectionClosed(WebSocketSession s, CloseStatus st) { SESSIONS.remove(s.getId()); }
    @Override public void handleTransportError(WebSocketSession s, Throwable t) { /* ... */ }
}
```

> `WebSocketSession` **不是线程安全的**，并发发送要加锁或用 `ConcurrentWebSocketSessionDecorator`。

## 3. STOMP（推荐，消息语义）

STOMP 是**基于帧的简单消息协议**，在 WebSocket 之上提供发布/订阅语义，Spring 提供完整支持（类似消息中间件）。

```java
@Configuration
@EnableWebSocketMessageBroker
public class StompConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();                 // 降级支持（浏览器不支持 ws 时）
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue");     // 内存 broker（单节点）
        registry.setApplicationDestinationPrefixes("/app");  // 客户端发往服务端的地址前缀
        registry.setUserDestinationPrefix("/user");          // 点对点
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration r) {
        r.interceptors(new AuthChannelInterceptor());        // 握手/订阅鉴权
    }
}
```

### 服务端收发

```java
@Controller
public class ChatController {

    @MessageMapping("/chat.send")           // 客户端发到 /app/chat.send
    @SendTo("/topic/messages")              // 广播给订阅者
    public ChatMessage send(ChatMessage msg) { return msg; }

    // 点对点（给指定用户）
    @Autowired private SimpMessagingTemplate template;
    public void notify(String userId, Notify n) {
        template.convertAndSendToUser(userId, "/queue/notify", n);   // /user/{userId}/queue/notify
    }

    // 订阅事件监听
    @SubscribeMapping("/topic/init")
    public List<Msg> init() { return history; }
}
```

### 客户端（浏览器）

```javascript
const socket = new SockJS('/ws');
const stomp = Stomp.over(socket);
stomp.connect({}, () => {
  stomp.subscribe('/topic/messages', m => console.log(JSON.parse(m.body)));
  stomp.subscribe('/user/queue/notify', m => console.log(JSON.parse(m.body)));
  stomp.send('/app/chat.send', {}, JSON.stringify({content: 'hi'}));
});
```

## 4. 鉴权

```java
public class AuthChannelInterceptor implements ChannelInterceptor {
    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor a = MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
        if (StompCommand.CONNECT.equals(a.getCommand())) {
            String token = a.getFirstNativeHeader("Authorization");
            Principal p = jwtAuth(token);
            if (p == null) throw new MessagingException("未认证");
            a.setUser(p);                      // ★ 后续 @Payload / convertAndSendToUser 用它
        }
        return message;
    }
}
```

也可用 HTTP 握手阶段鉴权（把 token 放 query/header，`HandshakeInterceptor` 设置 `Principal`）。

## 5. 心跳与重连

```java
@Override
public void configureWebSocketTransport(WebSocketTransportRegistration r) {
    r.setMessageSizeLimit(128 * 1024);
    r.setSendTimeLimit(20 * 1000);
    r.setSendBufferSizeLimit(512 * 1024);
}
```
- 服务端心跳：`enableSimpleBroker` 自带；也可配置 `TaskScheduler` 心跳。
- 客户端：`Stomp` 有 `heartbeat.outgoing/incoming`；断线重连需业务实现（指数退避）。

## 6. 集群问题（关键）

`enableSimpleBroker` 是**内存 broker**，多实例时消息只在本节点广播 → 用户 A 连在节点1、用户 B 连在节点2，收不到。

解决：
| 方案 | 说明 |
|---|---|
| **外置 Broker**（RabbitMQ/ActiveMQ） | `registry.enableStompBrokerRelay("/topic","/queue")` |
| Redis 广播转发 | 自定义：收到消息 publish 到 Redis，各节点订阅后本地发送 |
| 用户路由 + 定向转发 | 维护 userId → 节点映射（Redis），跨节点用 MQ 转发 |
| 专业 IM | 用 Netty + 自研/云信/融云 |

```java
// Broker Relay（需 RabbitMQ 开启 STOMP 插件）
registry.enableStompBrokerRelay("/topic", "/queue")
        .setRelayHost("localhost").setRelayPort(61613)
        .setClientLogin("guest").setClientPasscode("guest");
```

## 7. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 集群下消息不通 | 部分用户收不到 | 外置 broker 或 Redis 转发 |
| 连接数上限 | 无法新建连接 | 调服务器参数、水平扩容 |
| 消息丢失 | 断线期间消息没了 | 离线消息存储 + 重连拉取 |
| 心跳缺失 | 假死连接占用资源 | 配心跳 + 服务端超时踢除 |
| `WebSocketSession` 并发发送 | 报错 | 加锁/装饰器 |
| 网关不支持 ws | 握手失败 | Gateway 配置 ws 路由与 Upgrade 头 |
| 消息体过大 | 断开 | 限 `messageSizeLimit` |
| Nginx 未配 Upgrade | 101 失败 | 配 `Upgrade`/`Connection` 头 |

## 8. 面试速答

- **WebSocket 与 HTTP 区别**：HTTP 单向请求-响应；WebSocket 一次握手后全双工长连接，服务端可推送。
- **为什么用 STOMP**：WebSocket 只有传输层语义（帧），STOMP 提供发布/订阅、目的地、鉴权等消息语义，Spring 支持完善。
- **集群下如何做**：内存 broker 不行，用外置 broker（RabbitMQ Relay）或 Redis Pub/Sub 跨节点转发。
- **SSE 与 WebSocket 怎么选**：只需服务端单向推送用 SSE（更简单、自动重连）；需要双向通信用 WebSocket。
