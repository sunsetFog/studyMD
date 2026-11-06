# 60 · 网络编程

## 1. TCP 基础

```java
// 服务端
try (ServerSocket server = new ServerSocket(8080)) {
    while (true) {
        Socket client = server.accept();          // 阻塞等待连接
        new Thread(() -> handle(client)).start(); // 每连接一线程（简单但有上限）
    }
}

// 客户端
try (Socket socket = new Socket("localhost", 8080)) {
    OutputStream out = socket.getOutputStream();
    out.write("hello".getBytes(UTF_8));
    out.flush();

    InputStream in = socket.getInputStream();
    byte[] buf = new byte[1024];
    int n = in.read(buf);
}
```

### Socket 常用方法

```java
socket.getInputStream();
socket.getOutputStream();
socket.getRemoteSocketAddress();
socket.getLocalPort();
socket.setSoTimeout(3000);           // 读超时（毫秒），0 = 无限
socket.setTcpNoDelay(true);          // 禁用 Nagle（低延迟）
socket.setKeepAlive(true);
socket.shutdownOutput();             // 半关闭
socket.close();
```

### 服务端优化

```java
ServerSocket server = new ServerSocket();
server.setReuseAddress(true);         // TIME_WAIT 状态下可重用端口
server.bind(new InetSocketAddress(8080), 100);   // backlog = 100
```

## 2. UDP

```java
// 接收
try (DatagramSocket socket = new DatagramSocket(9999)) {
    byte[] buf = new byte[1024];
    DatagramPacket packet = new DatagramPacket(buf, buf.length);
    socket.receive(packet);
    String msg = new String(packet.getData(), 0, packet.getLength(), UTF_8);
}

// 发送
try (DatagramSocket socket = new DatagramSocket()) {
    byte[] data = "ping".getBytes(UTF_8);
    DatagramPacket packet = new DatagramPacket(
            data, data.length, InetAddress.getByName("localhost"), 9999);
    socket.send(packet);
}
```

特点：无连接、不保证可靠/有序、开销小。适合实时音视频、DNS、游戏。

## 3. HTTP Client（Java 11+）

```java
HttpClient client = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_2)
        .connectTimeout(Duration.ofSeconds(5))
        .followRedirects(HttpClient.Redirect.NORMAL)
        .build();

// GET
HttpRequest req = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users"))
        .header("Accept", "application/json")
        .timeout(Duration.ofSeconds(10))
        .GET()
        .build();

HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
System.out.println(resp.statusCode());
System.out.println(resp.body());

// POST JSON
HttpRequest post = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users"))
        .header("Content-Type", "application/json")
        .POST(HttpRequest.BodyPublishers.ofString("{\"name\":\"Tom\"}"))
        .build();

// 异步
client.sendAsync(req, HttpResponse.BodyHandlers.ofString())
      .thenApply(HttpResponse::body)
      .thenAccept(System.out::println);

// 表单
BodyPublishers.ofString("a=1&b=2");
// 文件
BodyPublishers.ofFile(Path.of("f.txt"));
// 流式（大响应）
client.send(req, HttpResponse.BodyHandlers.ofInputStream());
```

### 与 HttpURLConnection 对比

| 维度 | HttpURLConnection | HttpClient |
|---|---|---|
| HTTP/2 | ✘ | ✔ |
| 异步 | ✘ | ✔（CompletableFuture） |
| WebSocket | ✘ | ✔ |
| API | 繁琐 | 现代流式 |
| 推荐 | 遗留 | ✔ |

## 4. 编解码

```java
// URL 编码
URLEncoder.encode("张三 & 李四", StandardCharsets.UTF_8);   // %E5%BC%A0%E4%B8%89+%26+...
URLDecoder.decode(encoded, StandardCharsets.UTF_8);

// Base64
Base64.getEncoder().encodeToString(bytes);
Base64.getDecoder().decode(str);
Base64.getUrlEncoder().withoutPadding();      // URL 安全
Base64.getMimeEncoder();                       // 换行

// Hex（Java 17+）
HexFormat.of().formatHex(bytes);
HexFormat.of().parseHex("0a1b");

// URI 构建
URI uri = new URIBuilder()...   // 需第三方库
URI.create("https://x.com/path?q=1");

// URL vs URI
URL url = URI.create("https://x.com/a").toURL();
```

## 5. 粘包与半包

TCP 是**字节流**，没有消息边界。一次 `read` 可能：
- 读到半个消息（半包）。
- 读到多个消息（粘包）。

解决方案：

| 方案 | 说明 |
|---|---|
| **固定长度** | 每条消息定长，简单但浪费 |
| **分隔符** | 用 `\n`、`\0` 分隔（如 Redis 协议） |
| **长度字段** | 先发 4 字节长度，再发内容（最常用） |
| **自描述协议** | Protobuf 自带长度 |

```java
// 长度字段实现
DataOutputStream out = new DataOutputStream(socket.getOutputStream());
byte[] body = "hello".getBytes(UTF_8);
out.writeInt(body.length);
out.write(body);
out.flush();

// 读取
DataInputStream in = new DataInputStream(socket.getInputStream());
int len = in.readInt();
byte[] data = new byte[len];
in.readFully(data);
```

## 6. NIO 高并发（简述）

```java
// 见 08-36 的 Selector 示例
Selector selector = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.bind(new InetSocketAddress(8080));
server.configureBlocking(false);
server.register(selector, SelectionKey.OP_ACCEPT);
```

单线程管理数万连接。生产用 Netty。

## 7. RPC 基础

一次 RPC 调用的组成：

```
客户端代理 → 序列化 → 网络传输 → 反序列化 → 服务端处理 → 序列化响应 → 返回
```

关键设计：

| 环节 | 方案 |
|---|---|
| 传输协议 | TCP（长连接）/ HTTP2 |
| 序列化 | Protobuf / Hessian / JSON |
| 服务发现 | 注册中心（Nacos、ZK） |
| 负载均衡 | 轮询、加权、一致性哈希 |
| 超时重试 | 幂等 + 退避 |
| 熔断限流 | Sentinel、Resilience4j |

```java
// 简易 RPC 客户端代理（JDK 动态代理）
public class RpcProxy implements InvocationHandler {
    private final String host;
    private final int port;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try (Socket socket = new Socket(host, port)) {
            ObjectOutputStream out = new ObjectOutputStream(socket.getOutputStream());
            out.writeUTF(method.getName());
            out.writeObject(args);
            out.flush();

            ObjectInputStream in = new ObjectInputStream(socket.getInputStream());
            return in.readObject();
        }
    }

    @SuppressWarnings("unchecked")
    public static <T> T create(Class<T> iface, String host, int port) {
        return (T) Proxy.newProxyInstance(
                iface.getClassLoader(),
                new Class[]{iface},
                new RpcProxy(host, port));
    }
}
```

## 8. 网络排查工具

| 命令 | 用途 |
|---|---|
| `ping` | 连通性 |
| `telnet host port` / `nc -vz host port` | 端口可达性 |
| `curl -v` | HTTP 细节 |
| `netstat -anp \| grep 8080` | 端口占用 |
| `lsof -i :8080` | 进程占用端口 |
| `tcpdump -i any port 8080 -A` | 抓包 |
| `ss -s` | 连接统计 |
| `mtr host` | 路由追踪 |

### Java 层排查

```java
// 连接超时 vs 读超时
socket.connect(addr, 3000);       // 连接超时
socket.setSoTimeout(5000);        // 读超时

// 常见异常
// java.net.ConnectException: Connection refused        → 服务未启动/端口错
// java.net.SocketTimeoutException: Read timed out      → 服务端处理慢
// java.net.UnknownHostException                        → DNS 解析失败
// java.net.NoRouteToHostException                      → 网络不通
// java.io.IOException: Broken pipe                     → 对端已关闭，仍写入
// java.net.SocketException: Connection reset           → 对端 RST（进程崩溃/超时关闭）
```

## 9. 连接池与长连接

HTTP 短连接的问题：每次 TCP 三次握手 + 慢启动。

方案：

- **HTTP Keep-Alive**（`Connection: keep-alive`）。
- **连接池**（Apache HttpClient `PoolingHttpClientConnectionManager`、OkHttp）。
- **长连接 RPC**（Dubbo、gRPC）。

```java
// OkHttp 连接池
new OkHttpClient.Builder()
    .connectionPool(new ConnectionPool(50, 5, TimeUnit.MINUTES))
    .connectTimeout(3, TimeUnit.SECONDS)
    .readTimeout(5, TimeUnit.SECONDS)
    .build();
```

## 10. 常见坑

1. 未设置超时（连接/读），线程永久阻塞。
2. 每请求新建 `HttpClient`（应复用，内部有连接池）。
3. 未关闭 `InputStream`/`Socket`（连接泄漏）。
4. 忘记处理 TCP 粘包。
5. 在 `OutputStream` 写后未 `flush`（缓冲未发送）。
6. 用 `URL` 加载大文件到内存（应用流式）。
7. 服务端每连接一线程（连接数上万时 OOM）。
8. 未处理 `InterruptedException` 与中断关闭。
9. DNS 缓存问题（`InetAddress` 有缓存，`networkaddress.cache.ttl`）。
10. 忽略 TCP_NODELAY（小包延迟高）。

## 11. 速查

| 需求 | API |
|---|---|
| TCP 服务端 | `ServerSocket` |
| TCP 客户端 | `Socket` |
| UDP | `DatagramSocket` |
| HTTP 客户端 | `java.net.http.HttpClient` |
| NIO 多路复用 | `Selector` + `SocketChannel` |
| URL 编码 | `URLEncoder` / `URLDecoder` |
| Base64 | `java.util.Base64` |
| 十六进制 | `java.util.HexFormat`（17+） |
| 高性能网络框架 | Netty |
| 抓包 | `tcpdump` / Wireshark |
