# 36 · NIO：Buffer、Channel 与 Selector

## 1. NIO 与传统 IO

| 维度 | BIO（java.io） | NIO（java.nio） |
|---|---|---|
| 模型 | 流（Stream），单向 | 通道（Channel），双向 |
| 阻塞 | 阻塞 | 支持非阻塞 |
| 数据单位 | 字节/字符 | 缓冲区（Buffer） |
| 复用 | 无 | Selector 多路复用 |
| 适用 | 连接数少、大文件 | 高并发、大量连接 |

Java 7 又引入 NIO.2（`java.nio.file`、异步通道），见 08-37。

## 2. Buffer 三要素

```
 capacity = 10
 ┌──────────────────────┐
 │ 0 1 2 3 4 5 6 7 8 9  │
 └──────────────────────┘
       ↑       ↑        ↑
    position limit   capacity
```

| 属性 | 含义 |
|---|---|
| `capacity` | 总容量，创建后不变 |
| `position` | 下一个读/写位置 |
| `limit` | 可读/可写的边界 |

### 核心方法

```java
ByteBuffer buf = ByteBuffer.allocate(1024);      // 堆内
ByteBuffer direct = ByteBuffer.allocateDirect(1024);  // 堆外（零拷贝）
ByteBuffer wrapped = ByteBuffer.wrap(bytes);     // 包装数组

buf.put((byte) 1);          // 写模式：position++
buf.flip();                 // 切换读模式：limit = position, position = 0
buf.get();                  // 读：position++
buf.clear();                // 重置：position=0, limit=capacity（数据未清）
buf.compact();              // 压缩：保留未读数据到开头，切换写模式
buf.rewind();               // position=0，limit 不变（重读）
buf.mark(); buf.reset();    // 标记/回到标记
buf.remaining();            // limit - position
buf.hasRemaining();
```

### 状态转换

```
      allocate
         ↓
   [写模式] position=0, limit=capacity
         ↓ put()
         ↓ flip()
   [读模式] position=0, limit=原position
         ↓ get()
         ↓ clear()  → 回到写模式（丢弃数据）
         ↓ compact() → 回到写模式（保留未读）
```

**忘记 `flip()` 是最常见 bug**：写完直接读会读到 position 之后的位置（无数据）。

### 示例：读写循环

```java
ByteBuffer buf = ByteBuffer.allocate(8);
buf.put("hello".getBytes(UTF_8));
buf.flip();
while (buf.hasRemaining()) {
    System.out.print((char) buf.get());
}
buf.clear();
```

## 3. Buffer 类型

| 类型 | 元素 |
|---|---|
| `ByteBuffer` | byte（最常用） |
| `CharBuffer` | char |
| `ShortBuffer` / `IntBuffer` / `LongBuffer` | 对应基本类型 |
| `FloatBuffer` / `DoubleBuffer` | 浮点 |
| `MappedByteBuffer` | 内存映射文件 |

### 视图与转换

```java
ByteBuffer b = ByteBuffer.allocate(16);
IntBuffer ib = b.asIntBuffer();        // 视图，共享底层数据
CharBuffer cb = b.asCharBuffer();

ByteBuffer slice = b.slice();          // 从 position 到 limit 的切片
ByteBuffer dup = b.duplicate();        // 复制（独立 position）
ByteBuffer ro = b.asReadOnlyBuffer();
```

### 字节序

```java
buf.order(ByteOrder.LITTLE_ENDIAN);     // 默认 BIG_ENDIAN
```

网络协议通常大端，x86 文件格式可能小端。

## 4. Channel

```java
public interface Channel extends Closeable {
    boolean isOpen();
    void close();
}
```

常用实现：

| Channel | 说明 |
|---|---|
| `FileChannel` | 文件读写、内存映射、文件锁 |
| `SocketChannel` | TCP 客户端 |
| `ServerSocketChannel` | TCP 服务端 |
| `DatagramChannel` | UDP |
| `AsynchronousFileChannel` | 异步文件 IO |

### FileChannel 读写

```java
try (FileChannel ch = FileChannel.open(path, READ, WRITE)) {
    ByteBuffer buf = ByteBuffer.allocate(1024);
    int n;
    while ((n = ch.read(buf)) != -1) {
        buf.flip();
        // 处理 buf
        buf.clear();
    }

    ByteBuffer out = ByteBuffer.wrap("data".getBytes(UTF_8));
    while (out.hasRemaining()) {
        ch.write(out);              // 可能部分写入，必须循环
    }
}
```

### 高效复制

```java
try (FileChannel in = FileChannel.open(src, READ);
     FileChannel out = FileChannel.open(dst, CREATE, WRITE)) {
    in.transferTo(0, in.size(), out);
    // 或 out.transferFrom(in, 0, in.size());
}
```

`transferTo/transferFrom` 底层用 **sendfile**，避免用户态拷贝（零拷贝）。

### 定位读写

```java
ch.position(100);
ch.read(buf, 1024);            // 从指定位置读，不改变 position
ch.size();
ch.truncate(500);              // 截断
ch.force(true);                // 强制刷盘（相当于 fsync）
```

## 5. 零拷贝

传统读取文件并发送到网络：

```
磁盘 → 内核缓冲区 → 用户缓冲区 → Socket 缓冲区 → 网卡
        4 次拷贝，4 次上下文切换
```

`FileChannel.transferTo`：

```
磁盘 → 内核缓冲区 → 网卡
        2 次拷贝（DMA），少量上下文切换
```

应用：Kafka、Netty、Nginx 静态文件服务。

Java 中的零拷贝 API：

```java
FileChannel.transferTo(position, count, targetChannel);
FileChannel.transferFrom(sourceChannel, position, count);
```

注意：`transferTo` 单次最多传 2GB（int 参数），大文件需循环：

```java
long pos = 0, size = in.size();
while (pos < size) {
    pos += in.transferTo(pos, size - pos, out);
}
```

## 6. Selector 多路复用

### 原理

一个线程管理多个 Channel，通过操作系统 epoll/kqueue 通知就绪事件。

```
Selector
├── Channel A (OP_READ)
├── Channel B (OP_WRITE)
└── Channel C (OP_ACCEPT)
```

### 事件类型

| 事件 | 值 | 触发条件 |
|---|---|---|
| `OP_READ` | 1 | 可读 |
| `OP_WRITE` | 4 | 可写 |
| `OP_CONNECT` | 8 | 连接完成（客户端） |
| `OP_ACCEPT` | 16 | 有新连接（服务端） |

### 服务端模板

```java
public class NioServer {
    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();
        ServerSocketChannel server = ServerSocketChannel.open();
        server.bind(new InetSocketAddress(8080));
        server.configureBlocking(false);
        server.register(selector, SelectionKey.OP_ACCEPT);

        ByteBuffer buf = ByteBuffer.allocate(1024);

        while (true) {
            selector.select();                     // 阻塞直到有事件
            Iterator<SelectionKey> it = selector.selectedKeys().iterator();

            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();                       // 必须移除，否则重复处理

                if (!key.isValid()) continue;

                if (key.isAcceptable()) {
                    ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                    SocketChannel client = ssc.accept();
                    client.configureBlocking(false);
                    client.register(selector, SelectionKey.OP_READ);
                    System.out.println("accept " + client.getRemoteAddress());
                } else if (key.isReadable()) {
                    SocketChannel client = (SocketChannel) key.channel();
                    buf.clear();
                    int n = client.read(buf);
                    if (n == -1) {                 // 对端关闭
                        key.cancel();
                        client.close();
                        continue;
                    }
                    buf.flip();
                    byte[] data = new byte[buf.remaining()];
                    buf.get(data);
                    System.out.println("recv: " + new String(data, UTF_8));
                }
            }
        }
    }
}
```

### 关键细节

1. `selectedKeys()` 必须 `iterator.remove()`，否则下次仍处理。
2. `read` 返回 -1 表示对端关闭，必须 `cancel + close`。
3. 写操作也可能返回 0，需要注册 `OP_WRITE` 并维护待写队列。
4. `selector.select()` 可能虚假唤醒，需 `while` 循环检查。
5. `select(timeout)` 可设置超时，用于定时任务。
6. `wakeup()` 可唤醒阻塞的 select（跨线程）。

### 水平触发 vs 边缘触发

Java NIO 是**水平触发**（LT）：只要缓冲区有数据，每次都通知。Netty 通过 `epoll` 原生传输支持边缘触发（ET），减少通知次数。

## 7. 为什么需要 Netty

手写 NIO 的问题：

- 半包/粘包处理（TCP 是字节流）。
- 缓冲区管理复杂。
- 空闲检测、心跳、重连。
- 编解码、序列化。
- 连接数万时的性能优化。

Netty 提供：ByteBuf（引用计数、池化）、Pipeline、编解码器、心跳、优雅关闭、零拷贝封装。

```java
// Netty 服务端骨架
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class)
 .childHandler(new ChannelInitializer<SocketChannel>() {
     @Override protected void initChannel(SocketChannel ch) {
         ch.pipeline()
           .addLast(new LengthFieldBasedFrameDecoder(1024 * 1024, 0, 4, 0, 4))
           .addLast(new StringDecoder(UTF_8))
           .addLast(new SimpleChannelInboundHandler<String>() {
               @Override protected void channelRead0(ChannelHandlerContext ctx, String msg) {
                   ctx.writeAndFlush("echo: " + msg);
               }
           });
     }
 });
b.bind(8080).sync();
```

## 8. 堆内 vs 堆外缓冲

| 维度 | `allocate`（堆内） | `allocateDirect`（堆外） |
|---|---|---|
| 分配速度 | 快 | 慢 |
| 读写速度 | 需拷贝到内核 | 直接 IO，快 |
| GC 影响 | 受 GC 管理 | 不受 GC（需手动释放/等待 Cleaner） |
| 内存监控 | 堆监控可见 | 需 `-XX:MaxDirectMemorySize` |
| 适用 | 短期/小量 | 长期/高吞吐 IO |

```java
// 堆外内存泄漏会 OOM: Direct buffer memory
-XX:MaxDirectMemorySize=512m
```

Netty 用池化 ByteBuf 管理堆外内存（`PooledByteBufAllocator`）。

## 9. 常见坑

1. 忘记 `flip()`（读到空数据）。
2. 忘记 `clear()`/`compact()`（缓冲区满后无法继续写）。
3. `channel.write` 未循环（部分写入）。
4. `selectedKeys` 未 remove（重复处理、CPU 空转）。
5. `read` 返回 -1 未关闭连接（连接泄漏）。
6. 未处理半包（一次 read 可能只有半个消息）。
7. 堆外内存未释放（`Direct buffer memory` OOM）。
8. 在 Selector 线程做耗时业务（阻塞所有连接）。
9. `selector.select()` 空转（未处理事件却一直循环）。
10. 关闭 Channel 后未 cancel key（`CancelledKeyException`）。

## 10. 完整示例：文件复制（NIO）

```java
public static void copyNio(Path src, Path dst) throws IOException {
    try (FileChannel in = FileChannel.open(src, StandardOpenOption.READ);
         FileChannel out = FileChannel.open(dst,
                StandardOpenOption.CREATE,
                StandardOpenOption.WRITE,
                StandardOpenOption.TRUNCATE_EXISTING)) {

        long size = in.size();
        longpos = 0;
        while (pos < size) {
            pos += in.transferTo(pos, size - pos, out);
        }
    }
}
```

## 11. 速查

| 操作 | API |
|---|---|
| 创建缓冲 | `ByteBuffer.allocate(n)` |
| 写转读 | `flip()` |
| 读转写（清空） | `clear()` |
| 读转写（保留） | `compact()` |
| 通道读写 | `channel.read(buf)` / `write(buf)` |
| 零拷贝 | `transferTo/transferFrom` |
| 内存映射 | `map(READ_WRITE, 0, size)` |
| 多路复用 | `Selector.open()` + `register` |
| 文件锁 | `channel.lock()` |
| 刷盘 | `channel.force(true)` |
