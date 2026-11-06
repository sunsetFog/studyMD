# 16 · String 与字符串处理

## 1. String 为什么不可变

```java
public final class String implements Serializable, Comparable<String>, CharSequence {
    private final byte[] value;    // Java 9+ 用 byte[] + coder 压缩存储
    private int hash;              // 懒计算缓存
}
```

不可变的实现手段：

- `final class` 不可继承。
- `private final byte[] value` 不暴露。
- 所有「修改」方法返回新对象。

收益：

1. **字符串池**可安全共享。
2. **线程安全**（天然不可变）。
3. `hashCode` 可缓存，作为 HashMap key 高效。
4. 安全性（类名、URL、数据库连接串不会被中途篡改）。

Java 9+ 的紧凑字符串（Compact Strings）：Latin-1 用 1 字节/字符，中文用 UTF-16 2 字节，节省内存。

## 2. 字符串池与 intern

```java
String a = "abc";              // 字面量 → 常量池
String b = "abc";              // 复用
String c = new String("abc");  // 堆上新对象
String d = c.intern();         // 返回池中引用

System.out.println(a == b);    // true
System.out.println(a == c);    // false
System.out.println(a == d);    // true
```

编译期常量折叠：

```java
String s1 = "a" + "b";         // 编译期优化为 "ab"，进池
String s2 = "ab";
System.out.println(s1 == s2);  // true

String x = "a";
String s3 = x + "b";           // 运行期 StringBuilder 拼接，堆上新对象
System.out.println(s3 == s2);  // false
```

`final` 变量参与折叠：

```java
final String x = "a";
String s = x + "b";            // 编译期折叠 → "ab"
```

> 不要用 `intern()` 做业务缓存（Java 6 时在永久代，可能 OOM；Java 7+ 移到堆，但仍是全局表，易成瓶颈）。

## 3. 创建方式对比

| 写法 | 对象数 | 位置 |
|---|---|---|
| `"abc"` | 0（复用池）或 1 | 字符串常量池 |
| `new String("abc")` | 1 或 2 | 堆（+ 池中字面量） |
| `new String(char[])` | 1 | 堆 |
| `"a" + "b"` | 编译期 1 | 池 |
| `a + b`（变量） | 多个 | 堆（StringBuilder → toString） |
| `String.valueOf(x)` | 1 | 堆 |

## 4. 常用 API

### 查询

```java
s.length()
s.isEmpty() / s.isBlank()          // isBlank 忽略空白（Java 11+）
s.charAt(i)
s.indexOf("x") / lastIndexOf("x")
s.contains("x")
s.startsWith("x") / endsWith("x")
s.matches(regex)                   // 全匹配，内部编译 Pattern，循环中慎用
s.compareTo(other)                 // 字典序
s.compareToIgnoreCase(other)
s.equalsIgnoreCase(other)
s.contentEquals(sb)                // 与 CharSequence 比较
```

### 截取与替换

```java
s.substring(2)            // 从 2 到末尾
s.substring(2, 5)         // [2,5)
s.replace('a', 'b')       // 字符或 CharSequence，字面量
s.replaceAll("\\d+", "#") // 正则
s.replaceFirst("\\d+", "#")
s.trim()                  // 去首尾 <= ' ' 的字符
s.strip()                 // 去 Unicode 空白（Java 11+，推荐）
s.stripLeading() / stripTrailing()
s.repeat(3)               // Java 11+
s.indent(4)               // Java 12+
```

### 分割与连接

```java
"a,b,c".split(",")              // ["a","b","c"]
"a,b,,c".split(",")             // ["a","b","","c"]
"a,b,,".split(",")              // ["a","b"] —— 尾部空串被丢弃
"a,b,,".split(",", -1)          // ["a","b","",""] —— 保留尾部空串
"a.b".split(".")                // [] —— "." 是正则任意字符！必须 "\\."
String.join("-", "a", "b")      // "a-b"
String.join(",", list)
```

### 转换

```java
s.toCharArray()
s.getBytes(StandardCharsets.UTF_8)
s.toLowerCase() / toUpperCase()
s.toUpperCase(Locale.ROOT)      // 避免土耳其 i 问题
String.valueOf(123)
Integer.parseInt("123")
s.chars()                       // IntStream
s.lines()                       // Stream<String>（Java 11+）
```

## 5. 字符串拼接

```java
// 1. 编译期常量拼接 → 最优
String a = "x" + "y";

// 2. 循环内拼接 → StringBuilder
StringBuilder sb = new StringBuilder(256);
for (String item : items) sb.append(item).append(',');
String result = sb.toString();

// 3. Java 9+ 的 invokedynamic 优化
String s = name + ":" + age;    // 编译为 StringConcatFactory，性能接近 StringBuilder

// 4. String.join
String.join(",", items);

// 5. Collectors.joining
items.stream().collect(Collectors.joining(",", "[", "]"));
```

**结论**：单次表达式拼接直接用 `+`（JIT 已优化）；**循环内必须 StringBuilder**。

## 6. StringBuilder vs StringBuffer

| 维度 | StringBuilder | StringBuffer |
|---|---|---|
| 线程安全 | ✘ | ✔（synchronized） |
| 性能 | 高 | 低约 10-20% |
| 使用场景 | 方法内局部变量 | 极少（跨线程共享） |

```java
StringBuilder sb = new StringBuilder(64);   // 预估容量，减少扩容
sb.append("a").append(1).append(true);
sb.insert(0, ">");
sb.reverse();
sb.deleteCharAt(0);
sb.setLength(0);          // 清空复用（比 new 高效）
sb.toString();
```

扩容规则：默认容量 16，不足时 `newCapacity = old * 2 + 2`，然后 `Arrays.copyOf`。

## 7. 文本块（Java 15+）

```java
String json = """
        {
          "name": "Tom",
          "age": 18
        }
        """;
```

规则：

- 以 `"""` 开始，换行后内容开始。
- 结束符位置决定**最小缩进**，会被剥离。
- 自动把 `\n` 统一为 LF。
- 需要保留行尾空格时用 `\s`。
- 支持 `\` 续行（不换行）。

```java
String sql = """
        SELECT *
        FROM user
        WHERE name = ?
        """;
```

## 8. 格式化

```java
String.format("name=%s, age=%d, pi=%.2f", name, age, Math.PI);
"id=%d".formatted(1);              // Java 15+
"%05d".formatted(42);              // 00042
"%,d".formatted(1234567);          // 1,234,567
"%-10s|".formatted("a");           // 左对齐
"%s".formatted(null);              // "null"（不抛异常）
```

常用占位符：

| 占位符 | 说明 |
|---|---|
| `%s` / `%S` | 字符串 / 大写 |
| `%d` | 整数 |
| `%f` | 浮点（默认 6 位小数） |
| `%.2f` | 保留 2 位（四舍五入） |
| `%e` | 科学计数 |
| `%x` / `%o` | 十六进制 / 八进制 |
| `%b` | 布尔 |
| `%c` | 字符 |
| `%n` | 换行（平台相关） |
| `%%` | 字面量 % |

## 9. 编码与乱码

```java
byte[] bytes = "中文".getBytes(StandardCharsets.UTF_8);
String s = new String(bytes, StandardCharsets.UTF_8);
```

乱码三大原因：

1. 编码与解码字符集不一致。
2. 用平台默认编码（`getBytes()` 无参）跨环境传输。
3. 字节被截断（按字节切分 UTF-8 多字节字符）。

规范：

- **永远显式指定 `StandardCharsets.UTF_8`**。
- 文件读写用 `Files.newBufferedReader(path, UTF_8)`。
- HTTP 请求/响应设置 charset。
- 数据库连接串指定 `characterEncoding=utf8mb4`。

`Charset.defaultCharset()` 在 Java 18 起默认 UTF-8（JEP 400）。

## 10. 性能与陷阱

```java
// 1. 循环内 + 拼接：O(n²)
String s = "";
for (int i = 0; i < 10000; i++) s += i;   // 生成 10000 个临时对象

// 2. 循环内 matches：每次编译正则
for (String x : list) if (x.matches("\\d+")) { }   // 差
Pattern P = Pattern.compile("\\d+");               // 好
for (String x : list) if (P.matcher(x).matches()) { }

// 3. split 用正则，"." 需转义

// 4. 大字符串 substring：Java 7+ 会复制，不会有内存泄漏，但仍有拷贝成本

// 5. equals 常量在前防 NPE
if ("Y".equals(flag)) { }

// 6. 比较忽略大小写用 equalsIgnoreCase，不要 toLowerCase 后再比（Locale 问题）
```

## 11. 常用工具片段

```java
// 判空
public static boolean isBlank(String s) { return s == null || s.isBlank(); }

// 空值转默认
String v = Objects.requireNonNullElse(input, "");

// 脱敏
String mask(String phone) {
    return phone.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2");
}

// 驼峰下划线互转
String camelToSnake(String s) { return s.replaceAll("([a-z])([A-Z])", "$1_$2").toLowerCase(); }

// 字符串反转（正确处理代理对）
String reverse(String s) {
    return new StringBuilder(s).reverse().toString();   // 对 emoji 仍可能出错
}
```

## 12. 速记

- `String` 不可变，拼接产生新对象。
- 字面量进池，`new` 进堆。
- 循环拼接用 `StringBuilder`。
- 比较用 `equals`，常量在左。
- 编码永远显式 UTF-8。
- `split` 参数是正则。
- `trim` vs `strip`：后者支持 Unicode 空白。
