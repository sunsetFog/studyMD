# 58 · Netty 基础

## 1. 为什么需要 Netty

原生 NIO 编程复杂（Selector、Buffer、半包/粘包、空轮询 Bug）。Netty 封装了：

- 统一的异步事件驱动模型
- 高性能 Reactor 线程模型
- 丰富的编解码器（HTTP、WebSocket、Protobuf…）
- 内存池（ByteBuf）、零拷贝
- 完善的断线重连、心跳、背压处理

> **WebFlux 默认就是跑在 Netty 上的**（`reactor-netty`）。理解 Netty 有助于排查响应式应用的网络问题。

## 2. 核心组件

| 组件 | 说明 |
|---|---|
| `Bootstrap` / `ServerBootstrap` | 客户端/服务端启动器 |
| `EventLoopGroup` | 线程池（Reactor） |
| `EventLoop` | 单线程事件循环，绑定一个 Selector 与多个 Channel |
| `Channel` | 连接（NioSocketChannel / NioServerSocketChannel） |
| `ChannelPipeline` | 责任链，处理入站/出站事件 |
| `ChannelHandler` | 处理器（Inbound/Outbound） |
| `ChannelHandlerContext` | 上下文 |
| `ByteBuf` | 字节容器（替代 ByteBuffer，支持引用计数与池化） |
| `ChannelFuture` | 异步结果 |
| `Bootstrap.handler / childHandler` | 服务端：boss 处理连接，worker 处理 IO |

## 3. 线程模型（Reactor）

```
BossGroup（1~2 线程）      → accept 连接 → 注册到 WorkerGroup
WorkerGroup（N=核数×2）    → 处理 read/write/编解码/业务
```

- 一个 Channel **绑定一个 EventLoop**，全程同一线程 → **无锁、无并发问题**。
- **不要在 ChannelHandler 里做阻塞操作**（DB、远程调用），否则阻塞整个 EventLoop 上的所有连接。
- 耗时业务丢到独立业务线程池。

## 4. Hello World 服务端

```java
public class NettyServer {
    public static void main(String[] args) throws Exception {
        EventLoopGroup boss = new NioEventLoopGroup(1);
        EventLoopGroup worker = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(boss, worker)
             .channel(NioServerSocketChannel.class)
             .childOption(ChannelOption.SO_KEEPALIVE, true)
             .childOption(ChannelOption.TCP_NODELAY, true)
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override protected void initChannel(SocketChannel ch) {
                     ch.pipeline()
                       .addLast(new IdleStateHandler(60, 30, 0))     // 心跳检测
                       .addLast(new StringDecoder(StandardCharsets.UTF_8))
                       .addLast(new StringEncoder(StandardCharsets.UTF_8))
                       .addLast(new SimpleChannelInboundHandler<String>() {
                           @Override protected void channelRead0(ChannelHandlerContext ctx, String msg) {
                               ctx.writeAndFlush("echo: " + msg);
                           }
                           @Override public void exceptionCaught(ChannelHandlerContext ctx, Throwable c) {
                               log.error("err", c); ctx.close();
                           }
                       });
                 }
             });
            ChannelFuture f = b.bind(8080).sync();
            f.channel().closeFuture().sync();
        } finally { boss.shutdownGracefully(); worker.shutdownGracefully(); }
    }
}
```

## 5. ByteBuf

| 特性 | 说明 |
|---|---|
| 读写双指针 | 无需 `flip()` |
| 动态扩容 | 可指定初始容量 |
| 池化 | `PooledByteBufAllocator`（默认），减少 GC |
| 引用计数 | `retain()` / `release()`，**必须释放**否则内存泄漏 |
| 零拷贝 | `slice()`、`duplicate()`、`CompositeByteBuf` |

```java
ByteBuf buf = Unpooled.buffer(16);
buf.writeInt(1);
int v = buf.readInt();
buf.release();                      // ★ 若非 SimpleChannelInboundHandler 自动释放，需手动
```

> 检测泄漏：`-Dio.netty.leakDetection.level=PARANOID`。

## 6. 粘包/拆包

TCP 是字节流，无消息边界。解决方案：

| 方案 | 解码器 |
|---|---|
| 固定长度 | `FixedLengthFrameDecoder` |
| 分隔符 | `DelimiterBasedFrameDecoder`（如 `\n`） |
| 长度字段（最常用） | `LengthFieldBasedFrameDecoder` |
| 行分隔 | `LineBasedFrameDecoder` |

```java
// 协议：| 4字节长度 | 数据 |
pipeline.addLast(new LengthFieldBasedFrameDecoder(1024 * 1024, 0, 4, 0, 4));
pipeline.addLast(new LengthFieldPrepender(4));
```

## 7. 编解码器

| 类型 | 类 |
|---|---|
| 字节↔消息 | `ByteToMessageDecoder` / `MessageToByteEncoder` |
| 消息↔消息 | `MessageToMessageDecoder` / `MessageToMessageEncoder` |
| 通用 | `StringDecoder/Encoder`、`ObjectDecoder/Encoder`（不推荐）、`ProtobufDecoder`、`HttpServerCodec`、`WebSocketServerProtocolHandler` |

自定义协议示例：魔数 + 版本 + 序列化方式 + 指令 + 长度 + 数据。

## 8. 心跳与空闲检测

```java
pipeline.addLast(new IdleStateHandler(60, 30, 0, TimeUnit.SECONDS));   // 读空闲/写空闲/全部
// 在 handler 的 userEventTriggered 中处理：
@Override public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
    if (evt instanceof IdleStateEvent e && e.state() == IdleState.READER_IDLE) {
        ctx.close();               // 超时断开
    }
}
// 客户端定时发心跳包
```

## 9. 高性能要点

| 点 | 说明 |
|---|---|
| `TCP_NODELAY` | 关闭 Nagle，降低延迟 |
| `SO_BACKLOG` | 连接队列 |
| `SO_RCVBUF/SNDBUF` | 缓冲区 |
| 池化 ByteBuf | 默认开启 |
| 零拷贝 | `FileRegion` 传文件（`sendfile`） |
| `Epoll`（Linux） | `EpollEventLoopGroup` 性能更好 |
| 业务线程池 | 耗时任务剥离 |
| 对象复用 | 减少 GC |
| 避免阻塞 | EventLoop 内禁止阻塞 |

## 10. 与 Spring Boot 集成

```java
@Component
public class NettyServerRunner implements ApplicationRunner {

    @Value("${netty.port}") private int port;
    @Autowired private ServerHandler serverHandler;

    @Override public void run(ApplicationArguments args) throws Exception {
        new NettyServer(port, serverHandler).start();
    }

    @PreDestroy public void stop() { boss.shutdownGracefully(); worker.shutdownGracefully(); }
}
```

> 生产 IM/物联网常用：Netty 做长连接接入层，业务用 Spring 处理，通过 MQ/事件桥接。

## 11. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| EventLoop 阻塞 | 所有连接卡死 | 业务丢线程池 |
| ByteBuf 未释放 | 内存泄漏 | 用 `SimpleChannelInboundHandler`；`ReferenceCountUtil.release` |
| 粘包拆包 | 数据错乱 | 加帧解码器 |
| 大包未限 | OOM | 限制帧长度 |
| 忘记 `shutdownGracefully` | 资源不释放 | `@PreDestroy` |
| 线程数设置不当 | 性能差 | boss=1，worker=核数×2 |
| `sync()` 在主线程 | 阻塞启动 | 异步启动或专用线程 |
| 无心跳 | 假死连接 | `IdleStateHandler` |

## 12. 面试速答

- **Netty 是什么**：基于 NIO 的异步事件驱动网络框架，高性能、高可靠。
- **Reactor 线程模型**：Boss 接收连接，Worker 处理 IO；一个 Channel 绑定一个 EventLoop（单线程无锁）。
- **ByteBuf 与 ByteBuffer**：ByteBuf 读写双指针无需 flip、支持池化与引用计数、可动态扩容。
- **粘包拆包怎么解决**：固定长度、分隔符、**长度字段**（最常用）。
- **零拷贝**：`CompositeByteBuf`、`FileRegion`（sendfile）、`slice`，减少内存拷贝次数。
- **为什么不能在 Handler 里做阻塞**：EventLoop 线程被占满，其上所有 Channel 都无法处理 IO。
