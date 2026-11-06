# 26 · IO 与 NIO

## 一、IO 体系总览

```
字节流                        字符流（含编码转换）
InputStream / OutputStream    Reader / Writer
├── FileInputStream           ├── FileReader
├── BufferedInputStream       ├── BufferedReader
├── ObjectInputStream         ├── InputStreamReader   ← 字节→字符的桥
└── DataInputStream           └── PrintWriter
```

**装饰器模式**贯穿整个 IO：基础流 + 缓冲/转换装饰。

## 二、字节流

```java
// 读取
try (InputStream in = new FileInputStream("in.bin")) {
    int b;
    while ((b = in.read()) != -1) { }        // 逐字节（慢，教学用）
}

// 缓冲（推荐）
try (var in = new BufferedInputStream(new FileInputStream("in.bin"), 8192);
     var out = new BufferedOutputStream(new FileOutputStream("out.bin"))) {
    byte[] buf = new byte[8192];
    int n;
    while ((n = in.read(buf)) != -1) {
        out.write(buf, 0, n);                // 注意写 n 不是 buf.length！
    }
}

// 文件复制一行（Java 9+）
Files.copy(Path.of("a.bin"), Path.of("b.bin"), StandardCopyOption.REPLACE_EXISTING);

// 全量读取
byte[] all = Files.readAllBytes(Path.of("small.bin"));   // 仅小文件！大文件 OOM
```

## 三、字符流与编码

```java
// 显式编码（永远指定！默认编码随平台变化）
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(new FileInputStream("in.txt"), StandardCharsets.UTF_8))) {
    String line;
    while ((line = reader.readLine()) != null) process(line);
}

try (PrintWriter writer = new PrintWriter(
        new OutputStreamWriter(new FileOutputStream("out.txt"), StandardCharsets.UTF_8))) {
    writer.println("hello");
}

// Java 11+ 便捷
String content = Files.readString(Path.of("a.txt"), StandardCharsets.UTF_8);
Files.writeString(Path.of("b.txt"), "hello", StandardCharsets.UTF_8);
```

**编码常识**：UTF-8 变长（中文 3 字节）、GBK 中文 2 字节；乱码 = 编解码不一致。

## 四、对象流（序列化，详见 28 篇）

```java
try (var out = new ObjectOutputStream(new FileOutputStream("obj.bin"))) {
    out.writeObject(user);        // 类需实现 Serializable
}
```

## 五、NIO：Channel + Buffer + Selector

```java
// 核心：非阻塞 + 零拷贝导向。业务代码常用 FileChannel/MappedByteBuffer
FileChannel ch = FileChannel.open(Path.of("big.bin"), StandardOpenOption.READ);

// Buffer 读写循环
ByteBuffer buf = ByteBuffer.allocate(1024);
while (ch.read(buf) != -1) {
    buf.flip();                    // 切读模式（limit=position, position=0）
    while (buf.hasRemaining()) {
        byte b = buf.get();
    }
    buf.clear();                   // 切写模式
}

// 内存映射文件（大文件随机访问利器）
MappedByteBuffer mapped = ch.map(FileChannel.MapMode.READ_ONLY, 0, ch.size());

// transferTo：内核态零拷贝复制
try (FileChannel dst = FileChannel.open(outPath, WRITE, CREATE)) {
    ch.transferTo(0, ch.size(), dst);
}
```

**Buffer 四属性**：capacity（容量）≥ limit（界限）≥ position（位置）≥ mark。flip/clear/rewind 是模式切换核心。

## 六、NIO 的三种 IO 模型（理解 Netty 的基础）

```
BIO（阻塞）：一线程一连接，阻塞 read        → 连接少时简单可靠
NIO（同步非阻塞）：Selector 轮询多 Channel，事件驱动 → 单线程管千连接
AIO（异步）：完成回调                       → Linux 实现不理想，生态少
```

Selector 模型 = Netty/Redis/NGINX 的事件循环基石。现代框架（Netty/虚拟线程）已封装 NIO 细节——**应用层直接写 Selector 的场景极少**，理解模型即可。

## 七、Files / Path 工具（现代首选）

```java
Path p = Path.of("dir", "file.txt");
Files.exists(p); Files.size(p);
Files.isDirectory(p); Files.isReadable(p);

Files.createDirectories(Path.of("a/b/c"));
Files.createTempFile("app", ".tmp");
Files.list(dir);                        // 流式列出（记得 close）
Files.walk(Path.of("."), 3);            // 递归遍历流
Files.delete(p);
Files.move(src, dst, REPLACE_EXISTING);
Files.readAllLines(p);
Files.lines(p);                         // 大文件流式逐行（惰性）
```

## 八、大文件/高性能要点

- 大文件：Buffer 流分块 + 复用 buffer，不 readAllBytes
- 复制：transferTo（零拷贝）> buffer 循环 > 逐字节
- 网络发送文件：sendfile（OS 级零拷贝，框架层透明）
- 随机访问大文件：MappedByteBuffer / RandomAccessFile

## 常见坑

- **坑 1**：不指定字符编码 → 跨平台乱码
- **坑 2**：`read(buf)` 返回 0/-1 判断错 → 循环条件 `!= -1`，写长度用返回值 n
- **坑 3**：Buffer 忘 flip → 读不到数据
- **坑 4**：Files.lines 流没 close → 文件句柄泄漏
- **坑 5**：readAllBytes/readAllLines 处理大文件 → OOM
- **坑 6**：Windows/Linux 路径分隔符 → 统一 Path.of，不手拼 "\"

## Checklist

- [ ] 所有 IO 显式 UTF-8
- [ ] 资源 try-with-resources
- [ ] 大文件分块/transferTo，拒绝全量读
- [ ] 文件操作用 Files/Path API
- [ ] 能讲清 BIO/NIO 模型与适用场景
