# 21 · 格式化与国际化

## 1. printf 风格格式化

```java
System.out.printf("name=%s age=%d score=%.2f%n", name, age, score);
String s = String.format("%s-%d", "a", 1);
String t = "id=%d".formatted(1);        // Java 15+
```

### 完整格式说明符

```
%[argument_index$][flags][width][.precision]conversion
```

| 部分 | 说明 | 示例 |
|---|---|---|
| `argument_index$` | 参数位置（1 起） | `%1$s` |
| `flags` | `-` 左对齐、`+` 强制符号、`0` 补零、`,` 千分位、`(` 负数括号、空格 | `%-10s` |
| `width` | 最小宽度 | `%10d` |
| `.precision` | 小数位/最大字符数 | `%.3f` |
| `conversion` | 类型 | `s d f x b c e %` |

```java
String.format("%1$-10s|%2$10s|", "left", "right");   // left      |     right|
String.format("%08.3f", 3.14159);                    // 0003.142
String.format("%,d", 1234567);                       // 1,234,567
String.format("%+d", 5);                             // +5
String.format("%(d", -5);                            // (5)
String.format("%.3s", "abcdef");                     // abc
String.format("%,.2f", 1234567.891);                 // 1,234,567.89
```

## 2. 数值格式化

### DecimalFormat

```java
DecimalFormat df = new DecimalFormat("#,##0.00");
df.format(1234.5);            // "1,234.50"

DecimalFormat pct = new DecimalFormat("0.0%");
pct.format(0.856);            // "85.6%"

DecimalFormat sci = new DecimalFormat("0.###E0");
sci.format(12345);            // "1.234E4"
```

模式字符：

| 字符 | 含义 |
|---|---|
| `0` | 必填数字（不足补 0） |
| `#` | 可选数字（不足不显示） |
| `.` | 小数点 |
| `,` | 分组分隔符 |
| `%` | 百分比（乘以 100） |
| `‰` | 千分比 |
| `¤` | 货币符号 |
| `;` | 正负模式分隔（`"0.00;(0.00)"`） |
| `E` | 科学计数 |

**DecimalFormat 非线程安全**！不要作为静态字段共享：

```java
// 错误
private static final DecimalFormat DF = new DecimalFormat("#,##0.00");

// 正确 1：每次新建
String s = new DecimalFormat("#,##0.00").format(v);

// 正确 2：ThreadLocal
private static final ThreadLocal<DecimalFormat> DF =
    ThreadLocal.withInitial(() -> new DecimalFormat("#,##0.00"));

// 正确 3：用 BigDecimal + setScale（金额推荐）
```

### NumberFormat

```java
NumberFormat nf = NumberFormat.getInstance(Locale.CHINA);
nf.setMaximumFractionDigits(2);
nf.setMinimumFractionDigits(2);
nf.format(1234.5);            // "1,234.50"

NumberFormat currency = NumberFormat.getCurrencyInstance(Locale.CHINA);
currency.format(1234.5);      // "¥1,234.50"

NumberFormat percent = NumberFormat.getPercentInstance();
percent.format(0.856);        // 86%（默认 0 位小数，四舍五入）
```

## 3. 日期时间格式化

见 04-17，这里补充要点：

```java
DateTimeFormatter f = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String s = LocalDateTime.now().format(f);
LocalDateTime dt = LocalDateTime.parse("2026-01-01 12:00:00", f);

// 本地化（自动选格式）
DateTimeFormatter localized = DateTimeFormatter
        .ofLocalizedDate(FormatStyle.FULL)
        .withLocale(Locale.CHINA);
LocalDate.now().format(localized);    // 2026年1月1日星期四
```

`DateTimeFormatter` **线程安全**，可作为 static final 常量复用。

## 4. Locale

```java
Locale cn = Locale.CHINA;                       // zh_CN
Locale us = Locale.US;                          // en_US
Locale jp = Locale.JAPAN;                       // ja_JP
Locale root = Locale.ROOT;                      // 语言中立
Locale def = Locale.getDefault();
Locale.setDefault(Locale.CHINA);                // 全局修改，慎用

Locale.forLanguageTag("zh-CN");
Locale.getAvailableLocales();
```

解析字符串：

```java
Locale.forLanguageTag("en-US");
```

### 土耳其 i 问题（经典坑）

```java
"title".toUpperCase();                    // 在 tr_TR 下变成 "TİTLE"（带点 I）
"title".toUpperCase(Locale.ROOT);         // "TITLE"（正确）

// 协议/代码中的大小写转换永远指定 Locale.ROOT 或 Locale.ENGLISH
```

## 5. 国际化（i18n）

### ResourceBundle

```
src/main/resources/
├── messages.properties           # 默认
├── messages_zh_CN.properties
├── messages_en_US.properties
└── messages_ja_JP.properties
```

```properties
# messages_zh_CN.properties（需 UTF-8，或 JDK 9+ 直接读 UTF-8）
greeting=你好，{0}
error.notfound=未找到资源：{0}
```

```java
ResourceBundle bundle = ResourceBundle.getBundle("messages", Locale.CHINA);
String msg = bundle.getString("greeting");
String formatted = MessageFormat.format(msg, "张三");
```

### MessageFormat

```java
MessageFormat.format("你好，{0}！你有 {1,number,integer} 条消息。", "张三", 5);

// 类型化占位符
"{0,number,#,##0.00}"
"{0,date,yyyy-MM-dd}"
"{0,time,HH:mm}"
"{0,choice,0#无|1#一条|1<多条}"

// 注意单引号转义
MessageFormat.format("It''s {0}", "ok");   // It's ok
```

**坑**：MessageFormat 中单引号是转义符，`"don't {0}"` 会吞掉占位符，需写 `"don''t {0}"`。

## 6. Spring 中的国际化

```java
@Configuration
public class I18nConfig {
    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource ms = new ResourceBundleMessageSource();
        ms.setBasename("i18n/messages");
        ms.setDefaultEncoding("UTF-8");
        ms.setUseCodeAsDefaultMessage(true);
        return ms;
    }
}
```

```java
@Service
public class GreetingService {
    private final MessageSource messageSource;

    public String greet(String name, Locale locale) {
        return messageSource.getMessage("greeting", new Object[]{name}, locale);
    }
}
```

```yaml
spring:
  messages:
    basename: i18n/messages
    encoding: UTF-8
```

获取 Locale：

```java
// 从请求头 Accept-Language
Locale locale = LocaleContextHolder.getLocale();
```

## 7. 字符编码与排序

```java
// 排序：Collator 处理语言相关顺序
Collator collator = Collator.getInstance(Locale.CHINA);
List<String> names = new ArrayList<>(List.of("张三", "李四", "王五"));
names.sort(collator);            // 按拼音序

// 中文拼音排序可用 Collator 或 pinyin4j
Collator pinyin = Collator.getInstance(Locale.CHINA);
pinyin.setStrength(Collator.PRIMARY);
```

## 8. 常用格式化片段

```java
// 文件大小
public static String humanSize(long bytes) {
    if (bytes < 1024) return bytes + " B";
    String[] units = {"KB", "MB", "GB", "TB", "PB"};
    double v = bytes;
    int i = -1;
    while (v >= 1024 && i < units.length - 1) { v /= 1024; i++; }
    return "%.1f %s".formatted(v, units[i]);
}

// 耗时
public static String humanDuration(long millis) {
    long s = millis / 1000;
    return "%d:%02d:%02d".formatted(s / 3600, s % 3600 / 60, s % 60);
}

// 左对齐表格
System.out.printf("%-10s %-8s %s%n", "ID", "NAME", "SCORE");
System.out.printf("%-10d %-8s %.1f%n", 1, "Tom", 95.5);
```

## 9. 常见坑

1. `SimpleDateFormat`/`DecimalFormat` 作为静态共享字段（线程不安全）。
2. `toUpperCase()` 未指定 Locale（土耳其语环境出错）。
3. Properties 文件中文乱码（IDE 存成 ISO-8859-1）→ 统一 UTF-8，JDK 9+ 默认支持。
4. `MessageFormat` 单引号吞占位符。
5. `String.format` 用 `%d` 传 `Long` 之外的类型（如 `Integer` 传 `%f` 抛异常）。
6. `%n` 与 `\n`：`%n` 是平台换行符，跨平台文本用 `%n`，网络协议用 `\n`。
7. `%s` 传 null 输出 "null"，需要自定义显示用 `Objects.toString(x, "-")`。
8. 金额格式化用 `double`（精度问题），应用 `BigDecimal`。
9. Locale 全局修改影响所有线程。
10. 时区未指定导致日期相差 8 小时（见 04-17）。

## 10. 速查表

| 需求 | 用法 |
|---|---|
| 补零 | `%05d` |
| 千分位 | `%,d` 或 `DecimalFormat("#,##0")` |
| 保留 2 位 | `%.2f`（double）/ `setScale(2, HALF_UP)`（BigDecimal） |
| 百分比 | `NumberFormat.getPercentInstance()` |
| 货币 | `NumberFormat.getCurrencyInstance(locale)` |
| 左对齐 | `%-10s` |
| 日期格式化 | `DateTimeFormatter.ofPattern(...)` |
| 多语言 | `ResourceBundle` / `MessageSource` |
| 排序 | `Collator.getInstance(locale)` |
| 文件大小 | 自定义工具方法 |
