# 04 · 数组与字符串

## 一、数组

### 声明与初始化

```java
int[] a = new int[5];                  // 动态：默认值 0
int[] b = {1, 2, 3};                   // 静态
int[] c = new int[]{1, 2, 3};          // 完整写法
String[] names = new String[]{"tom", "jerry"};   // 引用类型默认 null

int[][] matrix = new int[3][4];        // 二维：3 行 4 列
int[][] jagged = {{1, 2}, {3, 4, 5}};  // 不规则数组
```

**数组特性**：
- 长度固定：`arr.length`
- 连续内存、按下标 O(1) 随机访问
- 下标越界 → ArrayIndexOutOfBoundsException（运行时）
- 引用类型数组存的是引用

### 常用操作

```java
import java.util.Arrays;

Arrays.toString(arr);            // [1, 2, 3]（直接打印是地址！）
Arrays.deepToString(matrix);     // 二维
Arrays.sort(arr);                // 排序（原地）
Arrays.copyOf(arr, 5);           // 扩容复制
Arrays.copyOfRange(arr, 1, 3);   // 范围复制 [1,3)
Arrays.fill(arr, 0);             // 填充
Arrays.equals(a, b);             // 比较（== 比的是地址！）
Arrays.binarySearch(sorted, 3);  // 二分查找（必须已排序）
System.arraycopy(src, 0, dst, 0, len);  // 高效复制

// 集合与数组互转
String[] array = list.toArray(new String[0]);
List<String> list = Arrays.asList("a", "b");       // 定长视图，不能 add！
List<String> list2 = List.of("a", "b");            // 不可变（Java 9+）
```

## 二、String 深度解析

### 不可变性（核心心智）

```java
String s = "hello";
s.toUpperCase();            // 返回新对象，s 还是 "hello"！
s = s.toUpperCase();        // 才是重新赋值
```

- String 内部 `final byte[] value`（Java 9+ 紧凑存储），无任何修改方法
- 不可变的好处：字符串池安全共享、hashCode 缓存、线程安全、可作 Map key
- 每次「修改」都产生新对象 → 循环拼接性能灾难

### 字符串池（String Pool）

```java
String a = "hello";           // 指向池中对象
String b = "hello";           // 同一个池对象
a == b                        // true（字面量入池）

String c = new String("hello");  // 堆中新对象
c == a                           // false
c.equals(a)                      // true
c.intern() == a                  // true（入池）
```

### 常用 API

```java
str.length();
str.charAt(0);
str.indexOf("ll");              // 子串位置，-1 不存在
str.substring(1, 3);            // [1,3)
str.toUpperCase() / toLowerCase();
str.trim() / strip();           // strip 处理全角空格（Java 11+）
str.replace("a", "b");          // 全部替换
str.split(",");                 // 分割（正则！特殊字符要转义）
str.contains("x") / startsWith / endsWith;
str.matches("\\d+");            // 正则全匹配
str.isBlank();                  // 空/纯空白（Java 11+）
str.repeat(3);                  // "aaa"
str.isEmpty()                   // length==0
str.compareTo(other);           // 字典序
String.join(",", "a", "b");     // a,b
String.format("%s=%d", "n", 1); // 格式化
```

### 比较铁律

```java
// 永远用 equals 比内容
if ("admin".equals(input)) { }   // 常量放前面防 NPE

// null 安全
Objects.equals(a, b);            // 双方 null 都 OK
```

## 三、StringBuilder / StringBuffer

```java
StringBuilder sb = new StringBuilder();   // 非线程安全（单线程首选）
// StringBuffer：加 synchronized 的老版本（线程安全但慢，新代码几乎不用）

sb.append("a").append(1).append(true);    // 链式
sb.insert(0, "head");
sb.deleteCharAt(0);
sb.replace(0, 2, "xx");
sb.reverse();
String result = sb.toString();

// 预估容量避免扩容拷贝
StringBuilder sb2 = new StringBuilder(1024);
```

**拼接性能结论**：
```
单次拼接         → str1 + str2（编译器优化，无所谓）
循环内拼接       → StringBuilder.append
少量已知项       → String.format / + （可读性优先）
```

## 四、正则表达式

```java
String regex = "^1[3-9]\\d{9}$";           // 手机号
boolean ok = phone.matches(regex);

Pattern p = Pattern.compile("(\\d+)-(\\d+)");
Matcher m = p.matcher("10-20");
if (m.find()) {
    m.group(1);  // "10"
    m.group(2);  // "20"
}

// 常用元字符
//  \d 数字  \w 单词字符  \s 空白   . 任意字符
//  * ≥0 次  + ≥1 次   ? 0或1次  {n,m} n到m次
//  ^ 开头  $ 结尾  [...] 字符集  | 或  () 分组
```

## 常见坑

- **坑 1**：`==` 比较字符串内容 → 引用比较；new String 的对象永远 != 字面量
- **坑 2**：`Arrays.asList(...)` 返回定长视图 → add/remove 抛 UnsupportedOperationException
- **坑 3**：`split(".")` 返回空数组 → `.` 是正则元字符，要 `split("\\.")`
- **坑 4**：直接打印数组得到地址 → 用 Arrays.toString
- **坑 5**：循环内 `+=` 拼接 → 每次创建新 StringBuilder，O(n²)
- **坑 6**：`substring` 在超大字符串截取子串长期持有 → Java 7u 前 subtitle 共享原数组内存泄漏（新版本已修复，了解即可）

## Checklist

- [ ] 理解 String 不可变与字符串池
- [ ] equals 比较铁律，常量在前防 NPE
- [ ] 循环拼接用 StringBuilder 并预估容量
- [ ] 正则元字符转义意识（split("\\.")）
