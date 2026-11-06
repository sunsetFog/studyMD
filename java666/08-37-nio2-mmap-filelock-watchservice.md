# 37 · NIO.2 进阶：内存映射、文件锁与监控

## 1. 内存映射文件（MappedByteBuffer）

把文件的一段直接映射到进程虚拟内存，读写如同操作数组。

```java
try (FileChannel ch = FileChannel.open(path, READ, WRITE)) {
    long size = ch.size();
    MappedByteBuffer buf = ch.map(FileChannel.MapMode.READ_WRITE, 0, size);

    buf.putInt(0, 42);          // 绝对位置写
    int v = buf.getInt(0);      // 绝对位置读

    buf.force();                // 刷盘（重要！）
}
```

### MapMode

| 模式 | 说明 |
|---|---|
| `READ_ONLY` | 只读，修改抛 `ReadOnlyBufferException` |
| `READ_WRITE` | 读写，改动写回文件 |
| `PRIVATE` | 写时复制，改动不落盘 |

### 优势

- **零拷贝**：不需要 read/write 系统调用，无用户态/内核态数据复制。
- 大文件随机访问快（操作系统按页调页）。
- 进程间共享内存（同一文件映射到多个进程）。

### 劣势与限制

1. **映射大小受限于 int**：单次 `map` 最多 2GB，大文件需分段映射。
2. **释放不确定**：`MappedByteBuffer` 无显式 `unmap`，需等 GC + Cleaner，或反射调用 `sun.misc.Unsafe.invokeCleaner`。Windows 上文件被映射时无法删除。
3. **文件被外部修改**：映射内容可能与磁盘不一致（页缓存）。
4. **OOM 风险**：映射计入虚拟内存/直接内存，不受堆限制。
5. **不适合网络 IO**：只适合本地文件。

### 大文件分段映射

```java
public class MappedFileReader implements AutoCloseable {
    private static final long CHUNK = 1L << 30;   // 1GB
    private final FileChannel ch;
    private final List<MappedByteBuffer> chunks = new ArrayList<>();

    public MappedFileReader(Path path) throws IOException {
        this.ch = FileChannel.open(path, StandardOpenOption.READ);
        long size = ch.size();
        for (long pos = 0; pos < size; pos += CHUNK) {
            long len = Math.min(CHUNK, size - pos);
            chunks.add(ch.map(FileChannel.MapMode.READ_ONLY, pos, len));
        }
    }

    public byte get(long index) {
        int ci = (int) (index / CHUNK);
        int off = (int) (index % CHUNK);
        return chunks.get(ci).get(off);
    }

    @Override public void close() throws IOException { ch.close(); }
}
```

### 典型应用

- 高性能日志/队列（Kafka 的 index 文件、RocketMQ 的 CommitLog）。
- 大数据量的只读索引（Lucene）。
- 进程间共享缓存。

### 实战：统计大文件行数

```java
public static long countLines(Path path) throws IOException {
    try (FileChannel ch = FileChannel.open(path, READ)) {
        long size = ch.size();
        long lines = 0;
        long pos = 0;
        int chunk = 8 * 1024 * 1024;
        while (pos < size) {
            int len = (int) Math.min(chunk, size - pos);
            MappedByteBuffer buf = ch.map(READ_ONLY, pos, len);
            for (int i = 0; i < len; i++) {
                if (buf.get(i) == '\n') lines++;
            }
            pos += len;
        }
        return lines;
    }
}
```

## 2. 文件锁

### 排他锁 vs 共享锁

```java
try (FileChannel ch = FileChannel.open(path, READ, WRITE);
     FileLock lock = ch.lock()) {              // 排他锁，阻塞
    // 独占访问
}

try (FileLock lock = ch.lock(0, Long.MAX_VALUE, true)) {   // 共享锁
    // 允许多个读
}

// 非阻塞
FileLock lock = ch.tryLock();
if (lock != null) { try { ... } finally { lock.release(); } }
else { /* 被占用 */ }
```

| 方法 | 阻塞 | 共享 |
|---|---|---|
| `lock()` | ✔ | ✘ |
| `lock(pos, size, shared)` | ✔ | 可指定 |
| `tryLock()` | ✘ | ✘ |
| `tryLock(pos, size, shared)` | ✘ | 可指定 |

### 特性

- **进程级锁**，同一 JVM 内多个线程不受限制（需自行同步）。
- **建议锁（advisory）**：不阻止其他进程直接读写文件，只阻止其他 `FileLock` 请求。
- 锁在 Channel 关闭或 JVM 退出时自动释放。
- 跨平台行为差异较大（Linux 用 `fcntl`，Windows 强制锁）。

### 单实例保护

```java
public class SingleInstanceGuard {
    private final FileChannel channel;
    private final FileLock lock;

    public SingleInstanceGuard(Path lockFile) throws IOException {
        this.channel = FileChannel.open(lockFile,
                StandardOpenOption.CREATE, StandardOpenOption.WRITE);
        this.lock = channel.tryLock();
        if (lock == null) throw new IllegalStateException("已有实例在运行");
    }

    public void close() throws IOException {
        lock.release();
        channel.close();
    }
}
```

> 分布式场景用 Redis 分布式锁或 ZooKeeper，文件锁只在单机有效。

### 常见坑

1. 同一 JVM 内对同一文件重复加锁 → `OverlappingFileLockException`。
2. 锁的区间重叠 → 异常。
3. 忘记 `release`（虽然 Channel 关闭会释放，但显式更好）。
4. 认为文件锁能防止其他程序读写（实际只是建议锁）。

## 3. WatchService 文件监控

```java
public class DirWatcher {
    public static void main(String[] args) throws Exception {
        Path dir = Path.of("/tmp/watch");
        Files.createDirectories(dir);

        try (WatchService ws = FileSystems.getDefault().newWatchService()) {
            dir.register(ws,
                StandardWatchEventKinds.ENTRY_CREATE,
                StandardWatchEventKinds.ENTRY_MODIFY,
                StandardWatchEventKinds.ENTRY_DELETE);

            while (true) {
                WatchKey key = ws.take();              // 阻塞
                // WatchKey key = ws.poll(1, TimeUnit.SECONDS);   // 超时
                for (WatchEvent<?> event : key.pollEvents()) {
                    WatchEvent.Kind<?> kind = event.kind();
                    if (kind == StandardWatchEventKinds.OVERFLOW) continue;
                    Path name = (Path) event.context();
                    System.out.println(kind.name() + ": " + name);
                }
                boolean valid = key.reset();          // 必须 reset
                if (!valid) break;                    // 目录不可访问
            }
        }
    }
}
```

### 特性与限制

| 项 | 说明 |
|---|---|
| 监控范围 | **只监控当前目录，不递归** |
| 事件 | 创建、修改、删除（无「重命名」，表现为删+建） |
| 可靠性 | 可能丢失事件（高频率写入） |
| OVERFLOW | 事件队列溢出，需全量重扫 |
| 平台 | Linux inotify、macOS FSEvents、Windows ReadDirectoryChangesW |

### 递归监控实现

```java
public static void watchRecursively(Path root, WatchService ws) throws IOException {
    try (Stream<Path> s = Files.walk(root)) {
        s.filter(Files::isDirectory).forEach(d -> {
            try {
                d.register(ws, ENTRY_CREATE, ENTRY_MODIFY, ENTRY_DELETE);
            } catch (IOException e) { throw new UncheckedIOException(e); }
        });
    }
}
// 新目录创建时需要动态注册
```

> 生产环境用 Apache Commons IO 的 `FileAlterationMonitor` 或 `WatchService` 封装库（如 directory-watcher）。

## 4. 异步文件通道

```java
try (AsynchronousFileChannel ch = AsynchronousFileChannel.open(
        path, StandardOpenOption.READ)) {

    ByteBuffer buf = ByteBuffer.allocate(1024);

    Future<Integer> future = ch.read(buf, 0);
    int n = future.get();                 // 阻塞等待

    // 或回调
    ch.read(buf, 0, buf, new CompletionHandler<Integer, ByteBuffer>() {
        @Override public void completed(Integer result, ByteBuffer attachment) {
            attachment.flip();
            // 处理
        }
        @Override public void failed(Throwable exc, ByteBuffer attachment) {
            exc.printStackTrace();
        }
    });
}
```

底层用线程池（Linux 上是 `io_uring` 或线程模拟），真正的异步取决于操作系统。

## 5. 异步 Socket 通道

```java
AsynchronousSocketChannel client = AsynchronousSocketChannel.open();
client.connect(new InetSocketAddress("example.com", 80)).get();

ByteBuffer req = ByteBuffer.wrap("GET / HTTP/1.0\r\n\r\n".getBytes(UTF_8));
client.write(req).get();

ByteBuffer resp = ByteBuffer.allocate(8192);
client.read(resp).get();
```

> 实践中高并发网络用 Netty（基于 NIO + epoll），异步 NIO 的 API 使用繁琐。

## 6. 文件属性与 POSIX

```java
Path p = Path.of("/tmp/a.txt");

// 基础属性
BasicFileAttributes a = Files.readAttributes(p, BasicFileAttributes.class);
a.size(); a.isDirectory(); a.creationTime(); a.lastModifiedTime();

// POSIX 属性
PosixFileAttributes pa = Files.readAttributes(p, PosixFileAttributes.class);
pa.owner(); pa.group();
pa.permissions();     // [OWNER_READ, OWNER_WRITE, ...]

// 修改权限
Files.setPosixFilePermissions(p, PosixFilePermissions.fromString("rw-r--r--"));
Files.setOwner(p, userPrincipal);
```

Windows 上 POSIX 属性不可用，抛 `UnsupportedOperationException`。

### 扩展属性（xattr）

```java
UserDefinedFileAttributeView view =
    Files.getFileAttributeView(p, UserDefinedFileAttributeView.class);
view.write("author", StandardCharsets.UTF_8.encode("ivory"));
```

## 7. FileSystem 与 Zip 文件系统

```java
// 把 zip 当成文件系统读取（无需解压）
try (FileSystem zfs = FileSystems.newFileSystem(Path.of("a.zip"))) {
    Path root = zfs.getPath("/");
    try (Stream<Path> s = Files.walk(root)) {
        s.filter(Files::isRegularFile).forEach(System.out::println);
    }
}

// 自定义协议（如内存文件系统 jimfs）
```

## 8. 性能对比：读文件

| 方式 | 1GB 文件耗时（参考） | 说明 |
|---|---|---|
| `FileInputStream` 单字节 | 极慢（秒级到分钟） | 禁止 |
| `BufferedInputStream` 8KB | 秒级 | 通用 |
| 大缓冲区 64KB | 更快 | 推荐 |
| `FileChannel.transferTo` | 最快 | 零拷贝 |
| `MappedByteBuffer` | 很快（随机访问尤佳） | 大文件随机读 |
| `Files.readAllBytes` | 占内存 | 仅小文件 |

## 9. 常见坑

1. `MappedByteBuffer` 修改后忘记 `force()`（进程崩溃丢数据）。
2. 映射超过 2GB 的文件（`map` 参数是 long 但实现限制 2GB/次）。
3. 映射文件未释放就删除（Windows 报错）。
4. 认为文件锁能阻止一切访问。
5. 同一 JVM 重复 `lock` 抛 `OverlappingFileLockException`。
6. `WatchService` 期望递归监控（实际不递归）。
7. `WatchKey` 未 `reset()`（后续事件收不到）。
8. 高频率写文件导致 `OVERFLOW`。
9. `AsynchronousFileChannel` 的 `Future.get()` 阻塞了事件线程。
10. 堆外/映射内存不计入堆，遇 `OutOfMemoryError: Map failed` 需查虚拟内存限制。

## 10. 速查

| 需求 | API |
|---|---|
| 大文件随机读 | `FileChannel.map(...)` |
| 大文件顺序复制 | `transferTo` |
| 单机防重复启动 | `FileLock.tryLock()` |
| 监控目录变化 | `WatchService` |
| 异步读写文件 | `AsynchronousFileChannel` |
| 不解压读 zip | `FileSystems.newFileSystem` |
| 修改权限 | `Files.setPosixFilePermissions` |
| 内存文件系统 | `jimfs`（Google） |
| 强制刷盘 | `MappedByteBuffer.force()` / `channel.force(true)` |
