# 58 · Java 8 → 21 版本特性全景

## 1. Java 8（2014，LTS）—— 分水岭

| 特性 | 说明 |
|---|---|
| **Lambda** | 函数式编程基础 |
| **Stream API** | 集合的声明式处理 |
| **Optional** | 显式表达可能为空 |
| **接口默认方法** | `default` / `static` 方法 |
| **方法引用** | `X::method` |
| **新日期时间 API** | `java.time`（JSR-310） |
| **重复注解** | `@Repeatable` |
| **类型注解** | `@Target(TYPE_USE)` |
| **Metaspace** | 替代 PermGen |
| **CompletableFuture** | 异步编排 |
| **HashMap 红黑树** | 链表长度 > 8 转树 |
| **Nashorn** | JS 引擎（JDK 15 移除） |

```java
List<String> names = users.stream()
        .filter(u -> u.getAge() > 18)
        .map(User::getName)
        .sorted()
        .collect(Collectors.toList());

list.forEach(System.out::println);
```

## 2. Java 9（2017）

| 特性 | 说明 |
|---|---|
| **JPMS 模块系统** | `module-info.java` |
| **JShell** | 交互式 REPL |
| **集合工厂方法** | `List.of` / `Set.of` / `Map.of` |
| **私有接口方法** | 复用默认方法逻辑 |
| **Optional 增强** | `ifPresentOrElse` / `or` / `stream` |
| **Stream 增强** | `takeWhile` / `dropWhile` / `iterate` 三参 |
| **try-with-resources 增强** | 可用已有变量 |
| **多版本 JAR** | `META-INF/versions` |
| **G1 成为默认 GC** | — |
| **HTTP/2 客户端（孵化）** | `java.net.http` |
| **String 底层改 byte[]** | 紧凑字符串 |

```java
List.of(1, 2, 3);
Map.of("a", 1, "b", 2);

// try-with-resources 用已有变量
BufferedReader r = ...;
try (r) { ... }
```

## 3. Java 10（2018）

| 特性 | 说明 |
|---|---|
| **var 局部变量类型推断** | `var list = new ArrayList<String>();` |
| **G1 并行 Full GC** | 减少 Full GC 停顿 |
| **容器感知** | `UseContainerSupport` |
| **应用类数据共享（AppCDS）** | 加快启动 |
| **线程局部握手** | 更安全的 `stop` 替代 |

```java
var map = new HashMap<String, List<Integer>>();
for (var entry : map.entrySet()) { }
```

## 4. Java 11（2018，LTS）

| 特性 | 说明 |
|---|---|
| **HttpClient 正式** | 支持 HTTP/2、异步、WebSocket |
| **单文件源码运行** | `java Hello.java` |
| **String 新方法** | `isBlank` / `lines` / `strip` / `repeat` |
| **Files 新方法** | `readString` / `writeString` |
| **Optional.isEmpty** | — |
| **Lambda 参数用 var** | `(var a, var b) -> ...` |
| **移除 Java EE 模块** | 需单独引入 |
| **ZGC（实验）** | 低延迟 GC |
| **Epsilon GC** | 无操作 GC（测试用） |
| **TLS 1.3** | — |

```java
HttpClient client = HttpClient.newHttpClient();
HttpResponse<String> resp = client.send(
    HttpRequest.newBuilder(URI.create("https://example.com")).build(),
    HttpResponse.BodyHandlers.ofString());
```

## 5. Java 12-15（非 LTS）

### Java 12
- `Collectors.teeing`
- `String.indent` / `transform`
- Shenandoah GC（实验）
- switch 表达式（预览）

### Java 13
- **文本块（预览）**
- `String.formatted`
- 动态 CDS 归档
- ZGC 归还未使用内存

### Java 14
- **switch 表达式正式**
- **record（预览）**
- **instanceof 模式匹配（预览）**
- 友好的 NPE 信息（`-XX:+ShowCodeDetailsInExceptionMessages`）
- 移除 CMS GC

### Java 15
- **文本块正式**
- **sealed 类（预览）**
- **隐藏类（Hidden Classes）**
- ZGC 正式（非实验）
- 移除 Nashorn

```java
// 文本块
String json = """
        {"name": "Tom"}
        """;

// switch 表达式
String type = switch (day) {
    case 1, 2, 3, 4, 5 -> "工作日";
    case 6, 7 -> "周末";
    default -> "未知";
};
```

## 6. Java 16（2021）

| 特性 | 说明 |
|---|---|
| **record 正式** | 不可变数据载体 |
| **instanceof 模式匹配正式** | `if (o instanceof String s)` |
| **sealed 类（第二次预览）** | — |
| **Stream.toList()** | 不可变列表 |
| **jpackage 正式** | 打包原生安装包 |
| **Unix-Domain Socket** | — |
| **强封装 JDK 内部 API** | 默认 `--illegal-access=deny` |
| **Vector API（孵化）** | SIMD |

```java
public record Point(int x, int y) { }

if (obj instanceof String s && s.length() > 3) { }
```

## 7. Java 17（2021，LTS）—— 当前主流基线

| 特性 | 说明 |
|---|---|
| **sealed 类正式** | 受控继承 |
| **switch 模式匹配（预览）** | — |
| **文本块正式**（15 已正式，17 起广泛使用） | — |
| **强封装 JDK 内部** | 需 `--add-opens` |
| **移除 SecurityManager（废弃）** | — |
| ** macOS AArch64 支持** | Apple Silicon |
| **Context-Specific Deserialization Filters** | 反序列化安全 |
| **Foreign Function & Memory API（孵化）** | 替代 JNI |
| **Pseudo-Random Number Generators** | 新随机数接口 |

```java
public sealed interface Shape permits Circle, Square { }
public record Circle(double r) implements Shape { }
public record Square(double s) implements Shape { }
```

## 8. Java 18（2022）

- 默认字符集 **UTF-8**（JEP 400）
- 简单 Web 服务器（`jwebserver`）
- `@snippet` Javadoc 标签
- 代码片段（Snippets in Java API Doc）
- 矢量 API（第三次孵化）
- 弃用 Finalization

## 9. Java 19（2022）

- **虚拟线程（预览）** JEP 425
- **结构化并发（孵化）** JEP 428
- **record 模式匹配（预览）**
- 外部函数与内存 API（预览）
- `Thread.ofVirtual()` / `Thread.ofPlatform()`

```java
Thread.startVirtualThread(() -> System.out.println("virtual"));
```

## 10. Java 20（2023）

- 虚拟线程（第二次预览）
- 结构化并发（第二次孵化）
- record 模式匹配（第二次预览）
- switch 模式匹配（第四次预览）
- 作用域值（Scoped Values，孵化）

## 11. Java 21（2023，LTS）—— 现代化起点

| 特性 | 状态 | 说明 |
|---|---|---|
| **虚拟线程** | 正式 | JEP 444，高并发 IO 场景革命 |
| **switch 模式匹配** | 正式 | 类型模式 + 守卫 + null |
| **record 模式匹配** | 正式 | 嵌套解构 |
| **sealed 类** | 正式（17） | 配合穷尽性检查 |
| **有序集合** | 新增接口 | `SequencedCollection` / `SequencedSet` / `SequencedMap` |
| **分代 ZGC** | 正式 | 更低开销 |
| **结构化并发** | 预览 | `StructuredTaskScope` |
| **作用域值** | 预览 | 替代 ThreadLocal |
| **字符串模板** | 预览 | `STR."Hello \{name}"` |
| **未命名模式** | 预览 | `_` |
| **未命名类与实例 main** | 预览 | 简化入门 |

### 有序集合（SequencedCollection）

```java
List<String> list = new ArrayList<>(List.of("a","b","c"));
list.getFirst();          // a
list.getLast();           // c
list.reversed();          // 反转视图
list.addFirst("z");
list.addLast("y");
```

### switch 模式匹配

```java
String describe(Object o) {
    return switch (o) {
        case null -> "null";
        case Integer i when i > 0 -> "正整数 " + i;
        case Integer i -> "整数 " + i;
        case String s -> "字符串长度 " + s.length();
        default -> "其他";
    };
}
```

### record 模式

```java
record Point(int x, int y) { }

if (o instanceof Point(int x, int y)) {
    System.out.println(x + "," + y);
}
```

## 12. Java 22-25（后续演进）

| 版本 | 关键点 |
|---|---|
| **22** | 未命名变量正式、外部函数 API 正式、字符串模板第二次预览 |
| **23** | 原语模式（预览）、ZGC 默认分代、Markdown 文档注释 |
| **24** | **synchronized 不再 pin 虚拟线程**（JEP 491）、Class-File API 正式、移除 SecurityManager |
| **25（LTS）** | 作用域值正式、紧凑源文件、实例 main、模式匹配增强 |

> 具体以官方 JEP 列表为准，非 LTS 版本特性可能调整。

## 13. 版本选型建议

| 场景 | 推荐 |
|---|---|
| 新项目 | **Java 21 LTS**（虚拟线程 + 模式匹配） |
| 存量升级 | Java 17 → 21（兼容性好） |
| 极致低延迟 | Java 21 + ZGC |
| 遗留系统 | 至少 Java 11（8 已停止免费商业支持） |
| 尝鲜 | 最新非 LTS（每 6 个月） |

### 升级检查清单

1. 依赖库兼容性（Spring Boot 3 要求 17+）。
2. 反射/内部 API 使用（模块强封装）。
3. 移除的 GC（CMS）与工具（Nashorn）。
4. 第三方 agent（Arthas、APM）版本。
5. 序列化兼容（record 需要 Jackson 2.12+）。
6. JVM 参数变化（`-XX:+UseG1GC` 默认）。
7. 容器镜像基础镜像更新。

## 14. 特性对照速查

| 特性 | 引入版本 | 正式版本 |
|---|---|---|
| Lambda / Stream | 8 | 8 |
| 模块系统 | 9 | 9 |
| var | 10 | 10 |
| HttpClient | 9（孵化） | 11 |
| 文本块 | 13 | 15 |
| switch 表达式 | 12 | 14 |
| record | 14 | 16 |
| instanceof 模式 | 14 | 16 |
| sealed | 15 | 17 |
| 虚拟线程 | 19 | 21 |
| switch 模式匹配 | 17 | 21 |
| record 模式 | 19 | 21 |
| 分代 ZGC | 17（实验） | 21 |
| 字符串模板 | 21 | 未定 |
| 结构化并发 | 19 | 未定 |
