# 28 · WebSocket 实时通信

## 一、依赖与配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

## 二、原生 WebSocket（简单场景）

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new ChatHandler(), "/ws/chat")
                .addInterceptors(new AuthHandshakeInterceptor())   // 握手时鉴权
                .setAllowedOriginPatterns("*");
    }
}

public class ChatHandler extends TextWebSocketHandler {

    // 会话管理：用户 → 会话（多实例部署需外部化，见第六节）
    private static final Map<Long, WebSocketSession> SESSIONS = new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        Long userId = (Long) session.getAttributes().get("userId");
        SESSIONS.put(userId, session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        // 业务处理……
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        Long userId = (Long) session.getAttributes().get("userId");
        SESSIONS.remove(userId);
    }
}
```

握手鉴权（token 通常无法放在 WebSocket header，放 query 参数）：

```java
public class AuthHandshakeInterceptor implements HandshakeInterceptor {
    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
                                   WebSocketHandler handler, Map<String, Object> attrs) {
        String token = UriComponentsBuilder.fromUri(request.getURI())
                .build().getQueryParams().getFirst("token");
        Claims claims = jwtUtil.parse(token);   // 失败抛异常 → 拒绝握手
        attrs.put("userId", Long.valueOf(claims.getSubject()));
        return true;
    }

    @Override
    public void afterHandshake(ServerHttpRequest r, ServerHttpResponse resp,
                               WebSocketHandler h, Exception e) { }
}
```

## 三、STOMP（推荐：消息路由/订阅模型）

```java
@Configuration
@EnableWebSocketMessageBroker
public class StompConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();    // 降级兼容
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/app");   // 客户端发消息前缀
        registry.enableSimpleBroker("/topic", "/queue");      // 服务端推送前缀
    }
}
```

```java
@Controller
public class ChatController {

    @MessageMapping("/chat.send")          // 客户端发到 /app/chat.send
    @SendTo("/topic/chat/{roomId}")        // 广播给订阅者
    public ChatMessage send(@DestinationVariable String roomId, ChatMessage msg) {
        return msg;
    }

    // 点对点：发给指定用户
    @MessageMapping("/private")
    public void sendPrivate(PrivateMessage msg, Principal principal) {
        messagingTemplate.convertAndSendToUser(
                msg.getToUser(), "/queue/notify", msg);
    }
}
```

前端（JS）：

```js
const client = new StompJs.Client({ brokerURL: 'ws://localhost:8080/ws' });
client.onConnect = () => {
  client.subscribe('/topic/chat/room1', m => console.log(JSON.parse(m.body)));
  client.subscribe('/user/queue/notify', m => console.log(JSON.parse(m.body)));
};
client.publish({ destination: '/app/chat.send', body: JSON.stringify(msg) });
```

## 四、心跳与保活

```java
registry.enableSimpleBroker("/topic")
        .setHeartbeatValue(new long[]{10000, 10000});   // 10s 心跳
```

Nginx 反向代理需调超时：

```nginx
location /ws {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 300s;
}
```

## 五、业务场景：服务端主动推送

```java
@RequiredArgsConstructor
@Service
public class NotifyPushService {

    private final SimpMessagingTemplate messagingTemplate;

    // 订单状态变更推送给用户
    public void pushOrderStatus(Long userId, OrderStatusMessage msg) {
        messagingTemplate.convertAndSendToUser(String.valueOf(userId), "/queue/order", msg);
    }
}
```

## 六、多实例部署（关键难点）

单机 SESSIONS Map 只在本机内存 → 用户连在 A 机，消息在 B 机产生 → 推送丢失：

```
方案 1：外部 broker（RabbitMQ STOMP relay / Redis pub-sub）
  registry.enableStompBrokerRelay("/topic", "/queue")   // rabbitmq-stomp
方案 2：消息统一进 MQ，每个实例消费并推给自己持有的会话
方案 3：网关层按 userId 会话路由（复杂）
```

## 常见坑

- **坑 1**：token 校验放在 STOMP 消息层而非握手层 → 任意连接可连上
- **坑 2**：Nginx 未配 Upgrade 头 → 握手失败 400/502
- **坑 3**：单机 Session Map，多实例推送丢失 → 外部 broker / MQ 广播
- **坑 4**：推送大消息/高频推送 → 客户端卡顿；分页/节流/合并
- **坑 5**：SimpleBroker 不支持集群与持久化 → 规模化用 StompBrokerRelay
- **坑 6**：连接泄漏（客户端异常断开未触发 close）→ 心跳检测 + 空闲清理

## Checklist

- [ ] 握手层完成 token 鉴权
- [ ] 心跳与 Nginx 超时配置匹配
- [ ] 多实例推送方案明确（broker relay / MQ）
- [ ] 高频推送节流合并
