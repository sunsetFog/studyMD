# 34 · IO 流体系

## 1. 分类

```
按数据单位：
├── 字节流（8 bit）：InputStream / OutputStream
└── 字符流（16 bit）：Reader / Writer

按方向：
├── 输入（读）
└── 输出（写）

按角色：
├── 节点流（直接连数据源）：FileInputStream、ByteArrayInputStream
└── 处理流（包装其他流）：BufferedInputStream、InputStreamReader
```

**选择规则**：

- 二进制数据（图片、音视频、序列化对象）→ 字节流。
- 文本数据 → 字符流（自动处理编码）。

## 2. 字节流

```java
public abstract class InputStream {
    int read();                       // 读 1 字节，返回 0-255，-1 表示结束
    int read(byte[] b);               // 读入 b，返回字节数
    int read(byte[] b, int off, int len);
    long skip(long n);
    int available();
    void close();
    default byte[] readAllBytes();            // Java 9+
    default long transferTo(OutputStream out); // Java 9+
}

public abstract class OutputStream {
    void write(int b);
    void write(byte[] b);
    void write(byte[] b, int off, int len);
    void flush();
    void close();
}
```

### 文件读写

```java
try (InputStream in = new FileInputStream("a.bin");
     OutputStream out = new FileOutputStream("b.bin")) {
    byte[] buf = new byte[8192];
    int n;
    while ((n = in.read(buf)) != -1) {
        out.write(buf, 0, n);
    }
}
```

> **必须用 `read(buf)` 的返回值作为写入长度**，不能写 `out.write(buf)`（最后一次可能未填满）。

### 三步复制

```java
// Java 9+
try (var in = Files.newInputStream(src); var out = Files.newOutputStream(dst)) {
    in.transferTo(out);            // 高效，内部用缓冲区
}

// 或直接
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);
```

## 3. 字符流

```java
public abstract class Reader {
    int read();
    int read(char[] cbuf);
    int read(CharBuffer target);
    void close();
    default long transferTo(Writer out);
}

public abstract class Writer {
    void write(int c);
    void write(char[] cbuf);
    void write(String str);
    void write(String str, int off, int len);
    void flush();
    void close();
}
```

```java
try (Reader r = new FileReader("a.txt", StandardCharsets.UTF_8);
     Writer w = new FileWriter("b.txt", StandardCharsets.UTF_8)) {
    char[] buf = new char[4096];
    int n;
    while ((n = r.read(buf)) != -1) {
        w.write(buf, 0, n);
    }
}
```

> `FileReader/FileWriter` 老构造器使用平台默认编码，**必须用带 Charset 的构造器**。

## 4. 缓冲流

```java
BufferedInputStream(InputStream)       // 默认 8KB
BufferedOutputStream(OutputStream)
BufferedReader(Reader)                 // 支持 readLine()
BufferedWriter(Writer)                 // 支持 newLine()
```

```java
try (BufferedReader r = new BufferedReader(new FileReader("a.txt", UTF_8))) {
    String line;
    while ((line = r.readLine()) != null) {
        System.out.println(line);
    }
}
```

要点：

- 缓冲流把多次小 IO 合并成大 IO，性能提升数倍到数十倍。
- `BufferedWriter` 必须 `flush()` 或 `close()`，否则数据留在缓冲区。
- `BufferedReader.readLine()` 返回不含换行符，末尾无换行时也能读到。

### 缓冲区大小

| 场景 | 建议 |
|---|---|
| 小文件 | 默认 8KB 足够 |
| 大文件/网络 | 32KB - 64KB |
| 极高吞吐 | 128KB+（需压测） |

## 5. 转换流（字节 ↔ 字符）

```java
// 字节输入 → 字符输入
Reader r = new InputStreamReader(in, StandardCharsets.UTF_8);

// 字符输出 → 字节输出
Writer w = new OutputStreamWriter(out, StandardCharsets.UTF_8);
```

用途：

- 需要按字符处理但数据源是字节（Socket、System.in）。
- 指定编码（`FileReader` 的替代）。

```java
// 标准输入按行读
BufferedReader stdin = new BufferedReader(new InputStreamReader(System.in, UTF_8));
String line = stdin.readLine();
```

## 6. 装饰器体系

```
InputStream
├── FileInputStream           节点
├── ByteArrayInputStream      节点
├── FilterInputStream         装饰器基类
│   ├── BufferedInputStream   缓冲
│   ├── DataInputStream       读基本类型
│   ├── ObjectInputStream     反序列化
│   ├── CheckedInputStream    校验和
│   ├── DigestInputStream     摘要
│   ├── InflaterInputStream   解压
│   └── CipherInputStream     解密
├── SequenceInputStream       串联多个流
└── PipedInputStream          管道
```

**这是装饰器模式的教科书实现**：任意组合、层层包装。

```java
// 读 gzip + 缓冲 + 基本类型
try (DataInputStream in = new DataInputStream(
        new BufferedInputStream(
            new GZIPInputStream(
                new FileInputStream("data.gz"))))) {
    int v = in.readInt();
}
```

## 7. 常用流

### ByteArrayInputStream / OutputStream

```java
ByteArrayOutputStream bos = new ByteArrayOutputStream();
bos.write("hello".getBytes(UTF_8));
byte[] data = bos.toByteArray();       // 内存中的字节数组

ByteArrayInputStream bis = new ByteArrayInputStream(data);
```

用途：单元测试、内存转换、图片处理（配合 ImageIO）。

### DataInputStream / DataOutputStream

```java
try (DataOutputStream out = new DataOutputStream(new FileOutputStream("d.bin"))) {
    out.writeInt(42);
    out.writeUTF("hello");
    out.writeDouble(3.14);
}

try (DataInputStream in = new DataInputStream(new FileInputStream("d.bin"))) {
    int i = in.readInt();          // 必须与写入顺序一致
    String s = in.readUTF();
    double d = in.readDouble();
}
```

大端序（Big-Endian），与平台无关。

### ObjectInputStream / ObjectOutputStream

见 08-35 序列化。

### PrintStream / PrintWriter

```java
PrintWriter pw = new PrintWriter(new FileWriter("a.txt", UTF_8));
pw.println("line");
pw.printf("%d%n", 1);
pw.flush();
```

`PrintStream`（`System.out`）吞掉 IOException，用 `checkError()` 查询。

### 压缩流

```java
// GZIP
try (var out = new GZIPOutputStream(Files.newOutputStream(path))) {
    out.write(data);
}

// ZIP（多文件）
try (ZipOutputStream zos = new ZipOutputStream(Files.newOutputStream(path))) {
    ZipEntry e = new ZipEntry("a.txt");
    zos.putNextEntry(e);
    zos.write(content.getBytes(UTF_8));
    zos.closeEntry();
}
```

### 管道流（线程间通信）

```java
PipedInputStream pin = new PipedInputStream();
PipedOutputStream pout = new PipedOutputStream(pin);

new Thread(() -> { try { pout.write("data".getBytes()); pout.close(); } catch (IOException e) { } }).start();
// 主线程从 pin 读
```

> 实际更推荐 `BlockingQueue` 或 `Exchanger`。

## 8. 序列化流（简述）

```java
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("u.bin"))) {
    oos.writeObject(user);
}

try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("u.bin"))) {
    User u = (User) ois.readObject();
}
```

详见 08-35。

## 9. 高效读写模板

```java
// ✅ 大文件复制（缓冲区 + try-with-resources）
public static void copy(Path src, Path dst) throws IOException {
    try (InputStream in = new BufferedInputStream(Files.newInputStream(src), 64 * 1024);
         OutputStream out = new BufferedOutputStream(Files.newOutputStream(dst), 64 * 1024)) {
        byte[] buf = new byte[64 * 1024];
        int n;
        while ((n = in.read(buf)) != -1) {
            out.write(buf, 0, n);
        }
    }
}

// ✅ 逐行处理大文本（不 OOM）
public static void process(Path path, Consumer<String> handler) throws IOException {
    try (BufferedReader r = Files.newBufferedReader(path, UTF_8)) {
        r.lines().forEach(handler);
    }
}

// ✅ 分块读取（限流/进度）
public static void chunked(Path path, int chunkSize, Consumer<byte[]> handler) throws IOException {
    try (InputStream in = Files.newInputStream(path)) {
        byte[] buf = new byte[chunkSize];
        int n;
        while ((n = in.read(buf)) != -1) {
            handler.accept(Arrays.copyOf(buf, n));
        }
    }
}
```

## 10. 常见坑

1. 用 `read()` 单字节读大文件（每个字节一次系统调用，极慢）。
2. `out.write(buf)` 而不是 `out.write(buf, 0, n)`（写入垃圾数据）。
3. 忘记 `flush()`（`BufferedWriter`、`PrintWriter`）。
4. 未关闭流（句柄泄漏，`lsof` 查看）。
5. `FileReader` 无编码参数（平台默认，跨环境乱码）。
6. 在 finally 中 close 又抛异常，覆盖原异常（用 try-with-resources 解决）。
7. `available()` 当文件总长度（对网络流不可靠）。
8. 用字符流处理二进制（图片、压缩包损坏）。
9. `readLine()` 丢失换行符（需要原样保留时用 `read()`）。
10. 大量小文件读写没有缓冲。

## 11. 字节流 vs 字符流选择

| 数据类型 | 选择 |
|---|---|
| 文本（UTF-8 等） | Reader/Writer |
| 图片、音频、视频 | InputStream/OutputStream |
| 序列化对象 | ObjectOutputStream |
| 网络协议（二进制） | InputStream |
| HTTP 响应文本 | InputStreamReader + 缓冲 |
| CSV/JSON 文件 | BufferedReader |

## 12. 速查

| 需求 | 类 |
|---|---|
| 文件字节读写 | `FileInputStream` / `FileOutputStream` |
| 文件字符读写 | `Files.newBufferedReader/Writer` |
| 缓冲 | `Buffered*` |
| 行读取 | `BufferedReader.readLine` |
| 基本类型 | `DataInputStream/OutputStream` |
| 对象 | `ObjectInputStream/OutputStream` |
| 字节↔字符 | `InputStreamReader/OutputStreamWriter` |
| 内存 | `ByteArrayInputStream/OutputStream` |
| 压缩 | `GZIPInputStream` / `ZipInputStream` |
| 格式化输出 | `PrintWriter` |
