# 20 · 正则表达式

## 1. 基本用法

```java
// 一次性匹配（每次编译 Pattern，循环中慎用）
boolean ok = "13800138000".matches("1[3-9]\\d{9}");

// 推荐：预编译
private static final Pattern PHONE = Pattern.compile("^1[3-9]\\d{9}$");

Matcher m = PHONE.matcher(input);
if (m.matches()) { }        // 全串匹配
if (m.find()) { }           // 查找子串
```

`matches()` 要求**整个字符串**匹配；等价于正则首尾加 `^...$`。

## 2. 字符类

| 语法 | 含义 |
|---|---|
| `.` | 任意字符（默认不含换行） |
| `\d` | 数字 `[0-9]` |
| `\D` | 非数字 |
| `\w` | 单词字符 `[a-zA-Z0-9_]` |
| `\W` | 非单词字符 |
| `\s` | 空白 `[ \t\n\x0B\f\r]` |
| `\S` | 非空白 |
| `[abc]` | a、b 或 c |
| `[^abc]` | 非 a/b/c |
| `[a-z]` | 范围 |
| `[a-zA-Z0-9_]` | 组合 |

Java 字符串中 `\` 需转义：`"\\d"`。

## 3. 量词

| 量词 | 含义 |
|---|---|
| `X?` | 0 或 1 次 |
| `X*` | 0 或多次 |
| `X+` | 1 或多次 |
| `X{n}` | 恰好 n 次 |
| `X{n,}` | 至少 n 次 |
| `X{n,m}` | n 到 m 次 |
| `X*?` | 懒惰（最短匹配） |
| `X*+` | 占有（不回溯） |

```java
"a+".matches("aaa");        // true
"a+?".matcher("aaa").find(); // 匹配 "a"（最短）
```

## 4. 位置与边界

| 语法 | 含义 |
|---|---|
| `^` | 行/串开头（MULTILINE 下为行首） |
| `$` | 行/串结尾 |
| `\b` | 单词边界 |
| `\B` | 非单词边界 |
| `\A` | 输入开头 |
| `\z` | 输入结尾 |
| `\Z` | 输入结尾（允许末尾换行符） |

```java
"cat".matches("\\bcat\\b");   // true
"concatenate".matches(".*\\bcat\\b.*");  // false
```

## 5. 分组

```java
// 捕获组
Pattern p = Pattern.compile("(\\d{4})-(\\d{2})-(\\d{2})");
Matcher m = p.matcher("2026-01-05");
if (m.matches()) {
    m.group(1);   // "2026"
    m.group(2);   // "01"
    m.group(0);   // 整个匹配
    m.groupCount();// 3
}

// 命名组（Java 7+）
Pattern named = Pattern.compile("(?<year>\\d{4})-(?<month>\\d{2})");
Matcher nm = named.matcher("2026-01");
nm.matches();
nm.group("year");    // "2026"

// 非捕获组（性能更好）
Pattern.compile("(?:abc)+");

// 反向引用
Pattern.compile("(\\w)\\1");        // 重复字符，如 "aa"
"abab".matches("(ab)\\1");          // true

// 命名反向引用
Pattern.compile("(?<c>\\w)\\k<c>");
```

## 6. 前后查找（零宽断言）

| 语法 | 含义 |
|---|---|
| `(?=X)` | 正向肯定：后面是 X |
| `(?!X)` | 正向否定：后面不是 X |
| `(?<=X)` | 反向肯定：前面是 X |
| `(?<!X)` | 反向否定：前面不是 X |

```java
// 提取金额数字（前面是 ¥）
Pattern.compile("(?<=¥)\\d+").matcher("价格¥199 折扣¥50");
// 匹配 199、50

// 密码必须含数字且长度 8-20（不消耗字符）
"^(?=.*\\d)(?=.*[a-zA-Z]).{8,20}$"

// 不以某前缀开头
"^(?!admin).*$"
```

Java 的反向查找**要求长度固定**（`(?<=a{1,3})` 会报错），这是与 PCRE 的差异。

## 7. 模式标志

```java
Pattern.compile("abc", Pattern.CASE_INSENSITIVE);
Pattern.compile("^line", Pattern.MULTILINE);      // ^$ 匹配每行
Pattern.compile("a.b", Pattern.DOTALL);           // . 匹配换行
Pattern.compile("x y", Pattern.COMMENTS);         // 忽略空白，可加注释
Pattern.compile("a", Pattern.UNICODE_CASE | Pattern.CASE_INSENSITIVE);

// 内联写法
"(?i)abc";        // 忽略大小写
"(?m)^abc";       // 多行
"(?s).";          // DOTALL
"(?i:abc)def";    // 仅组内生效
```

## 8. Matcher 常用方法

```java
Matcher m = p.matcher(text);

m.matches();          // 全串匹配
m.find();             // 查找下一个
m.find(int start);    // 从指定位置开始
m.group();            // 当前匹配
m.start(); m.end();   // 位置
m.reset();            // 重置
m.replaceAll("x");    // 替换所有
m.replaceFirst("x");
m.appendReplacement(sb, "$1");  // 高级替换
m.appendTail(sb);
m.results();          // Java 9+ Stream<MatchResult>
```

### 替换中的 `$` 与 `\`

```java
"2026-01-05".replaceAll("(\\d+)-(\\d+)-(\\d+)", "$3/$2/$1");  // 05/01/2026
"a1b2".replaceAll("\\d", "\\$");          // 替换为字面 $ 需转义
Matcher.quoteReplacement("$100");         // 安全替换字符串
```

### 遍历所有匹配

```java
Matcher m = Pattern.compile("\\d+").matcher("a1 b22 c333");
while (m.find()) {
    System.out.println(m.group() + " @" + m.start());
}
// 1 @1, 22 @4, 333 @8

// Stream 风格
Pattern.compile("\\d+").matcher(text).results()
       .map(MatchResult::group)
       .forEach(System.out::println);
```

## 9. 常用正则库

```java
// 手机号（中国大陆）
"^1[3-9]\\d{9}$"

// 邮箱（实用版，不追求 RFC 完整性）
"^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$"

// 身份证（18 位）
"^[1-9]\\d{5}(18|19|20)\\d{2}(0[1-9]|1[0-2])(0[1-9]|[12]\\d|3[01])\\d{3}[\\dXx]$"

// 强密码（8-20 位，含大小写数字，可含特殊字符）
"^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)[A-Za-z\\d@$!%*?&]{8,20}$"

// URL
"^https?://[\\w.-]+(:\\d+)?(/[\\w./?%&=-]*)?$"

// IPv4
"^((25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]?\\d)\\.){3}(25[0-5]|2[0-4]\\d|1\\d{2}|[1-9]?\\d)$"

// 中文
"^[\\u4e00-\\u9fa5]+$"

// 日期 yyyy-MM-dd（含简单合法性）
"^\\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\\d|3[01])$"

// 金额（最多两位小数）
"^\\d+(\\.\\d{1,2})?$"

// 车牌号
"^[京津沪渝冀豫云辽黑湘皖鲁新苏浙赣鄂桂甘晋蒙陕吉闽贵粤青藏川宁琼使领][A-HJ-NP-Z][A-HJ-NP-Z0-9]{4,5}[A-HJ-NP-Z0-9挂学警港澳]$"
```

> 身份证/手机号等校验建议最后一两位再用算法校验（校验位、号段库），正则只做格式初筛。

## 10. 性能与安全

### ReDoS（正则拒绝服务）

```java
// 危险：嵌套量词 + 回溯
Pattern.compile("^(a+)+$").matcher("aaaaaaaaaaaaaaaaaaaaaaaaX").matches();
// 指数级回溯，CPU 打满
```

防护：

1. 避免嵌套量词 `(a+)+`、`(a|a)*`。
2. 用占有量词 `a++` 或原子组 `(?>...)` 消除回溯。
3. 限制输入长度（如 ≤ 200 字符）。
4. 设置匹配超时（Java 无原生支持，可用独立线程 + 中断）。
5. 优先用简单字符串 API（`contains`、`startsWith`）。

### 其他性能点

- **预编译 Pattern** 为 static final 常量。
- 简单场景别用正则：`"abc".contains("b")` 比 `matches(".*b.*")` 快。
- `split` 内部每次编译，热点代码用 `Pattern.compile(...).split(...)`。
- 大文本用 `Matcher.find()` 流式，不要 `replaceAll` 生成巨大字符串。

```java
private static final Pattern SPLIT = Pattern.compile("\\s*,\\s*");
String[] parts = SPLIT.split(input);
```

## 11. 常见坑

1. `"a.b".split(".")` → 空数组（`.` 是任意字符，应用 `"\\."`）。
2. `matches` 是**全匹配**，不是包含。
3. Java 字符串里 `\\` 才是正则的 `\`。
4. `$` 在替换串中是组引用，字面量需 `Matcher.quoteReplacement`。
5. `\b` 在 Java 字符串中是退格符，正则单词边界要写 `"\\b"`。
6. 贪婪量词导致匹配过多，用 `?` 变懒惰。
7. `Pattern` 线程安全，`Matcher` 不是（每次 `matcher()` 新建）。
8. `^$` 默认只匹配整串首尾，多行需 `MULTILINE`。
9. 中文 `\w` 默认不匹配（需 `UNICODE_CHARACTER_CLASS`）。
10. `group(n)` 前必须先 `find()`/`matches()`，否则抛 `IllegalStateException`。

## 12. 调试建议

- 用 regex101.com / regexr.com 可视化，注意选择 **Java** 方言。
- 先写小测试用例覆盖：空串、边界、超长、特殊字符、Unicode。
- 复杂正则加 `COMMENTS` 模式分行写注释。

```java
private static final Pattern EMAIL = Pattern.compile("""
        ^[A-Za-z0-9._%+-]+      # 本地部分
        @[A-Za-z0-9.-]+         # 域名
        \\.[A-Za-z]{2,}$        # 顶级域
        """, Pattern.COMMENTS);
```
