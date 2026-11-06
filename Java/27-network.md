# 27 · 网络编程

## 一、TCP/UDP 与 Socket 心智

```
TCP：可靠、有序、面向连接（三次握手/四次挥手）  → HTTP/DB/RPC 全走它
UDP：不可靠、无连接、低延迟                     → DNS、音视频、游戏
```

Java 抽象：
- 客户端：`Socket`（TCP）/ `DatagramSocket`（UDP）
- 服务端：`ServerSocket`（TCP）/ `DatagramSocket`（UDP）

## 二、TCP 客户端/服务端（BIO 模型）

### 服务端

```java
try (ServerSocket server = new ServerSocket(8080)) {
    while (true) {
        Socket socket = server.accept();          // 阻塞等待连接
        // 每连接一线程（BIO 模型；高并发换 NIO/Netty/虚拟线程）
        Thread.startVirtualThread(() -> handle(socket));
    }
}

void handle(Socket socket) {
    try (socket;
         var in = new BufferedReader(
                 new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8));
         var out = new PrintWriter(
                 new OutputStreamWriter(socket.getOutputStream(), StandardCharsets.UTF_8), true)) {
        String line;
        while ((line = in.readLine()) != null) {   // readLine 阻塞直到换行
            out.println("echo: " + line);          // println 自动 flush（true 参数）
            if ("bye".equals(line)) break;
        }
    } catch (IOException e) {
        log.error("连接异常", e);
    }
}
```

### 客户端

```java
try (Socket socket = new Socket("localhost", 8080);
     var in = new BufferedReader(new InputStreamReader(socket.getInputStream(), UTF_8));
     var out = new PrintWriter(new OutputStreamWriter(socket.getOutputStream(), UTF_8), true)) {

    socket.setSoTimeout(5000);          // 读超时：防永久阻塞！
    out.println("hello");
    String resp = in.readLine();
}
```

**TCP 粘包/拆包**：TCP 是字节流无消息边界 → 应用层定界（换行符/长度前缀/固定长度/分隔符）。readLine 就是「换行定界」；生产协议多用「长度前缀」（Netty LengthFieldBasedFrameDecoder）。

## 三、UDP

```java
// 发送
try (DatagramSocket socket = new DatagramSocket()) {
    byte[] data = "ping".getBytes(UTF_8);
    socket.send(new DatagramPacket(data, data.length,
            InetAddress.getByName("localhost"), 9999));
}

// 接收
try (DatagramSocket socket = new DatagramSocket(9999)) {
    byte[] buf = new byte[1024];
    DatagramPacket packet = new DatagramPacket(buf, buf.length);
    socket.receive(packet);                       // 阻塞
    String msg = new String(packet.getData(), 0, packet.getLength(), UTF_8);
}
```

## 四、HTTP 客户端（Java 11+ java.net.http）

```java
HttpClient client = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_2)
        .connectTimeout(Duration.ofSeconds(5))
        .build();

// 同步 GET
HttpRequest req = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users/1"))
        .header("Authorization", "Bearer token")
        .timeout(Duration.ofSeconds(10))
        .GET()
        .build();

HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
resp.statusCode();
resp.body();

// 异步
client.sendAsync(req, BodyHandlers.ofString())
      .thenApply(HttpResponse::body)
      .thenAccept(System.out::println);

// POST JSON
HttpRequest post = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/users"))
        .header("Content-Type", "application/json")
        .POST(HttpRequest.BodyPublishers.ofString("{\"name\":\"tom\"}"))
        .build();
```

**实战注意**：
- HttpClient 实例**全局复用**（内置连接池）
- 超时三件套齐全：connectTimeout（客户端级）+ request timeout（请求级）+ 对方响应慢靠超时兜底
- 生产复杂场景用 OkHttp/Apache HttpClient（重试/拦截器生态）

## 五、URL 处理

```java
URI uri = URI.create("https://api.example.com:443/users?page=2");
uri.getScheme(); uri.getHost(); uri.getPort(); uri.getPath(); uri.getQuery();

// 编码：参数值必须编码
String q = URLEncoder.encode("java 并发", StandardCharsets.UTF_8);   // java+%E5%B9%B6...
String decoded = URLDecoder.decode(q, StandardCharsets.UTF_8);
// 注意：encode 会把空格转 +，适用于 query 参数；路径段用路径安全编码
```

## 六、网络排查工具箱

```bash
curl -v http://localhost:8080/api     # 请求细节
telnet host port / nc -zv host port   # 连通性
netstat -an | grep 8080               # 连接状态（TIME_WAIT 多=短连接泛滥）
tcpdump -i any port 8080              # 抓包
```

**TIME_WAIT 洪峰**：高频短连接 → HttpClient 复用 keep-alive 长连接。

## 七、超时与重试（网络编程的生命线）

```java
// 三类超时缺一不可
connectTimeout      // 建连超时（网络不通快速失败）
readTimeout         // 响应等待超时
overallTimeout      // 整体请求超时

// 重试原则
1. 只重试幂等请求（GET/PUT/DELETE；POST 需幂等键）
2. 指数退避 + 抖动：delay = base * 2^n + random
3. 重试次数上限 + 熔断（连续失败快速失败）
```

## 常见坑

- **坑 1**：Socket 不设 SoTimeout → 对端挂起则永久阻塞
- **坑 2**：TCP 当消息协议不处理粘包 → 半包/粘连
- **坑 3**：每次请求 new Socket/new HttpClient → 连接风暴 + TIME_WAIT
- **坑 4**：URLEncoder 用于路径 → + 号混入路径；路径用 URI 直接构建
- **坑 5**：POST 重试 → 重复下单；幂等键兜底
- **坑 6**：忽略 statusCode 判断 → 200 之外直接 body 解析炸

## Checklist

- [ ] Socket 必设超时
- [ ] HttpClient 全局复用 + 三类超时
- [ ] TCP 消息定界方案明确
- [ ] 重试只对幂等 + 指数退避
