# 33 · File、Path 与 Files

## 1. File（老 API）

```java
File f = new File("/tmp/a.txt");
File g = new File("/tmp", "a.txt");

f.exists(); f.isFile(); f.isDirectory();
f.length();                       // 字节数
f.getName(); f.getParent(); f.getPath(); f.getAbsolutePath();
f.canRead(); f.canWrite(); f.canExecute();
f.lastModified();                 // 毫秒时间戳
f.mkdir();                        // 单层目录，父不存在则失败
f.mkdirs();                       // 多层
f.createNewFile();
f.delete();                       // 目录必须为空
f.list();                         // String[]，失败返回 null
f.listFiles();                    // File[]，失败返回 null
f.renameTo(target);               // 跨文件系统可能失败
```

缺点：错误只返回 boolean（无原因）、方法命名混乱（`delete` vs `deleteOnExit`）、不支持符号链接、性能一般。

## 2. Path（NIO.2，推荐）

```java
Path p = Path.of("/tmp/a.txt");            // Java 11+
Path p2 = Paths.get("/tmp", "a.txt");      // Java 7+
Path p3 = Path.of(System.getProperty("user.home"), "logs", "app.log");

p.getFileName();          // a.txt
p.getParent();            // /tmp
p.getRoot();              // /
p.getNameCount();         // 3
p.getName(0);             // tmp
p.toAbsolutePath();
p.normalize();            // 去掉 . 与 ..
p.toRealPath();           // 解析符号链接（文件必须存在）
p.subpath(0, 2);          // tmp/a.txt
p.resolve("b.txt");       // /tmp/a.txt/b.txt（绝对路径参数会直接返回）
p.resolveSibling("b");    // /tmp/b
p.relativize(other);

p.isAbsolute();
Files.exists(p);
```

### Path 与 File 互转

```java
File file = p.toFile();
Path path = file.toPath();
```

## 3. Files 常用方法

### 读写小文件

```java
// 读
String content = Files.readString(path);                       // Java 11+
List<String> lines = Files.readAllLines(path);                 // UTF-8 默认
byte[] bytes = Files.readAllBytes(path);

// 写
Files.writeString(path, "hello");                              // Java 11+
Files.write(path, bytes);
Files.write(path, lines);                                      // 每行加分隔符

// 追加
Files.writeString(path, "more\n", StandardOpenOption.APPEND);
Files.write(path, bytes, StandardOpenOption.CREATE, StandardOpenOption.APPEND);
```

> 小文件（< 几十 MB）用上面这些；大文件用流式处理。

### 流式读写

```java
try (BufferedReader r = Files.newBufferedReader(path, StandardCharsets.UTF_8)) {
    r.lines().filter(l -> !l.isBlank()).forEach(System.out::println);
}

try (BufferedWriter w = Files.newBufferedWriter(path, StandardCharsets.UTF_8)) {
    for (String line : lines) {
        w.write(line);
        w.newLine();
    }
}

try (Stream<String> stream = Files.lines(path)) {
    long count = stream.count();
}
```

### 复制、移动、删除

```java
Files.copy(src, dest);                                        // 目标存在则抛异常
Files.copy(src, dest, StandardCopyOption.REPLACE_EXISTING);
Files.copy(in, path, StandardCopyOption.REPLACE_EXISTING);    // 从 InputStream

Files.move(src, dest, StandardCopyOption.REPLACE_EXISTING);
Files.move(src, dest, StandardCopyOption.ATOMIC_MOVE);        // 原子（同分区）

Files.delete(path);              // 不存在抛 NoSuchFileException
Files.deleteIfExists(path);

// 递归删除
Files.walk(path)
     .sorted(Comparator.reverseOrder())
     .forEach(p -> { try { Files.delete(p); } catch (IOException e) { throw new UncheckedIOException(e); } });
```

### 目录操作

```java
Files.createDirectory(path);                       // 父必须存在
Files.createDirectories(path);                     // 递归创建
Files.createTempFile("prefix", ".tmp");            // 系统临时目录
Files.createTempDirectory("dir");
Files.createSymbolicLink(link, target);
Files.createLink(hardLink, existing);
Files.isSymbolicLink(p);
```

### 属性

```java
Files.size(path);
Files.getLastModifiedTime(path);
Files.setLastModifiedTime(path, FileTime.fromMillis(System.currentTimeMillis()));
Files.isDirectory(path);
Files.isRegularFile(path);
Files.isReadable(path);
Files.isHidden(path);
Files.getOwner(path);
Files.setPosixFilePermissions(path, PosixFilePermissions.fromString("rw-r--r--"));
Files.getPosixFilePermissions(path);

// 批量读取属性（避免多次系统调用）
BasicFileAttributes attrs = Files.readAttributes(path, BasicFileAttributes.class);
attrs.size(); attrs.creationTime(); attrs.lastAccessTime();
```

## 4. 目录遍历

```java
// 1. 单层
try (Stream<Path> s = Files.list(dir)) {
    s.filter(Files::isRegularFile).forEach(System.out::println);
}

// 2. 递归（深度优先）
try (Stream<Path> s = Files.walk(dir)) {
    s.filter(p -> p.toString().endsWith(".java")).forEach(System.out::println);
}

// 3. 限定深度
try (Stream<Path> s = Files.walk(dir, 2)) { ... }

// 4. 按扩展名查找
try (Stream<Path> s = Files.find(dir, Integer.MAX_VALUE,
        (p, attrs) -> attrs.isRegularFile() && p.toString().endsWith(".log"))) {
    s.forEach(System.out::println);
}

// 5. 文件树遍历器（大目录、可控中断）
Files.walkFileTree(dir, new SimpleFileVisitor<>() {
    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
        System.out.println(file);
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult visitFileFailed(Path file, IOException exc) {
        return FileVisitResult.CONTINUE;      // 跳过无权限文件
    }

    @Override
    public FileVisitResult postVisitDirectory(Path d, IOException exc) {
        return FileVisitResult.CONTINUE;
    }
});
```

> `Files.walk` 返回的 Stream 必须关闭（try-with-resources），否则文件句柄泄漏。

## 5. 临时文件与清理

```java
Path tmp = Files.createTempFile("app-", ".log");
tmp.toFile().deleteOnExit();      // JVM 退出时删除（文件多时内存占用）

// 自建临时目录管理
Path dir = Files.createTempDirectory("batch-");
try {
    // 使用
} finally {
    deleteRecursively(dir);
}
```

## 6. 文件监控（WatchService 简述）

```java
try (WatchService ws = FileSystems.getDefault().newWatchService()) {
    Path dir = Path.of("/tmp/watch");
    dir.register(ws, ENTRY_CREATE, ENTRY_MODIFY, ENTRY_DELETE);

    while (true) {
        WatchKey key = ws.take();
        for (WatchEvent<?> e : key.pollEvents()) {
            System.out.println(e.kind() + " " + e.context());
        }
        if (!key.reset()) break;
    }
}
```

详见 08-37。

## 7. 编码

```java
// 始终显式指定
Files.readString(path, StandardCharsets.UTF_8);
Files.writeString(path, content, StandardCharsets.UTF_8);
Files.newBufferedReader(path, StandardCharsets.UTF_8);

// 检测 BOM（UTF-8 with BOM 会产生 \uFEFF）
String s = Files.readString(path);
if (s.startsWith("\uFEFF")) s = s.substring(1);
```

Java 18 起默认 charset 为 UTF-8（JEP 400），但显式指定仍是好习惯（避免运行在旧版本或特殊配置下出错）。

## 8. 路径安全（重要）

```java
// 危险：用户输入直接拼路径 → 路径穿越
Path target = baseDir.resolve(userInput);        // userInput = "../../etc/passwd"

// 正确：规范化后校验前缀
Path base = Path.of("/data/uploads").toAbsolutePath().normalize();
Path target = base.resolve(userInput).normalize();
if (!target.startsWith(base)) {
    throw new SecurityException("非法路径");
}
```

规则：

- 文件名只允许白名单字符（`[a-zA-Z0-9._-]`）。
- 不使用用户输入作为完整路径。
- 上传文件重命名为 UUID，不保留原始文件名。

## 9. 常见工具方法

```java
// 递归删除
public static void deleteRecursively(Path path) throws IOException {
    if (!Files.exists(path)) return;
    try (Stream<Path> s = Files.walk(path)) {
        s.sorted(Comparator.reverseOrder()).forEach(p -> {
            try { Files.delete(p); }
            catch (IOException e) { throw new UncheckedIOException(e); }
        });
    }
}

// 读取 classpath 资源
public static String readResource(String name) throws IOException {
    try (InputStream in = MyClass.class.getResourceAsStream("/" + name)) {
        if (in == null) throw new FileNotFoundException(name);
        return new String(in.readAllBytes(), StandardCharsets.UTF_8);
    }
}

// 按行处理大文件（不 OOM）
public static void processLargeFile(Path path, Consumer<String> consumer) throws IOException {
    try (BufferedReader r = Files.newBufferedReader(path, StandardCharsets.UTF_8)) {
        String line;
        while ((line = r.readLine()) != null) consumer.accept(line);
    }
}

// 获取扩展名
public static String ext(String name) {
    int i = name.lastIndexOf('.');
    return i > 0 ? name.substring(i + 1) : "";
}

// 确保目录存在
Files.createDirectories(path.getParent());
```

## 10. 常见坑

1. Windows 路径分隔符硬编码 `\`（用 `Path.of` 或 `File.separator`）。
2. 忘记关闭 Stream（文件句柄泄漏，Linux 上 `Too many open files`）。
3. `Files.delete` 删除非空目录（抛 `DirectoryNotEmptyException`）。
4. `Files.copy` 目标存在抛异常（需 `REPLACE_EXISTING`）。
5. `Files.walk` 遇到无权限目录抛异常（用 `walkFileTree` 的 `visitFileFailed`）。
6. `File.list()` 返回 null 未判空。
7. 用 `File.length()` 判断是否为空（目录返回 0 或平台相关）。
8. 路径拼接用字符串 `+`（应用 `resolve`）。
9. 上传文件用原始文件名（路径穿越 + 覆盖）。
10. 读取大文件用 `readAllBytes`（OOM）。

## 11. File vs Path/Files

| 维度 | File | Path + Files |
|---|---|---|
| 引入版本 | 1.0 | 7 |
| 错误处理 | boolean | 异常（有原因） |
| 符号链接 | 支持差 | 完整支持 |
| 性能 | 一般 | 好 |
| 原子操作 | 无 | `ATOMIC_MOVE` |
| 推荐 | 仅兼容老 API | ✔ 新代码 |

## 12. 速查

| 需求 | 代码 |
|---|---|
| 读文本 | `Files.readString(p)` |
| 写文本 | `Files.writeString(p, s)` |
| 按行读 | `Files.lines(p)` / `newBufferedReader` |
| 复制 | `Files.copy(src, dst, REPLACE_EXISTING)` |
| 递归删 | `Files.walk(p).sorted(reverseOrder())` |
| 遍历 | `Files.walk(dir)` / `Files.find(...)` |
| 建目录 | `Files.createDirectories(p)` |
| 临时文件 | `Files.createTempFile(prefix, suffix)` |
| 路径安全 | `normalize()` + `startsWith(base)` |
