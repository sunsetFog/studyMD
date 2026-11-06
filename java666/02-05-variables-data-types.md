# 05 · 变量、数据类型与类型转换

## 1. 8 种基本类型（primitive）

| 类型 | 字节 | 位 | 范围 | 默认值 | 包装类 |
|---|---|---|---|---|---|
| `byte` | 1 | 8 | -128 ~ 127 | 0 | Byte |
| `short` | 2 | 16 | -32768 ~ 32767 | 0 | Short |
| `int` | 4 | 32 | -2³¹ ~ 2³¹-1 | 0 | Integer |
| `long` | 8 | 64 | -2⁶³ ~ 2⁶³-1 | 0L | Long |
| `float` | 4 | 32 | ±3.4e-38 ~ ±3.4e38 | 0.0f | Float |
| `double` | 8 | 64 | ±4.9e-324 ~ ±1.8e308 | 0.0d | Double |
| `char` | 2 | 16 | 0 ~ 65535（无符号 UTF-16 单元） | '\u0000' | Character |
| `boolean` | 1(虚拟机相关) | — | true/false | false | Boolean |

> 基本类型**不是对象**，存放在栈/对象字段中，无对象头开销——这是 Java 的性能取舍。

## 2. 字面量与后缀

```java
int dec = 100;
int bin = 0b1010;        // 10
int oct = 0777;          // 511
int hex = 0xFF;          // 255
int underscore = 1_000_000;   // 下划线仅可读性

long big = 10_000_000_000L;   // 超出 int 范围必须加 L
float f = 3.14f;              // 必须加 f
double d = 3.14;              // 默认 double
char c = 'A';
char unicode = '\u4e2d';      // 中
String s = """
    文本块（Java 15+），
    保留换行与缩进处理
    """;
```

## 3. 引用类型

- 类、接口、枚举、记录、数组、注解。
- 默认值 `null`；变量存的是**地址引用**。
- 三大引用层级：强引用（默认）、软引用、弱引用、虚引用（见 JVM 篇）。

```java
Object o = "abc";      // 编译期类型 Object，运行期类型 String
int[] arr = new int[3];// 数组是对象，arr.length 是字段
```

## 4. 变量分类与作用域

| 类别 | 位置 | 默认值 | 生命周期 |
|---|---|---|---|
| 局部变量 | 方法/代码块内 | **无默认值，必须初始化** | 方法调用期间 |
| 实例变量 | 类中、非 static | 有 | 随对象 |
| 静态变量 | 类中、static | 有 | 随类（类卸载时销毁） |
| 参数 | 方法签名 | 无 | 方法调用期间 |

```java
public class Scope {
    int instance;           // 0
    static int stat;        // 0

    void m(int param) {
        int local;          // 未初始化
        // System.out.println(local);  // 编译错误
        for (int i = 0; i < 3; i++) { }
        // i 在此不可见
    }
}
```

### 变量遮蔽（shadowing）

```java
int x = 1;
{ int x = 2; }   // 局部块内不能重复声明同名变量（Java 不允许内层遮蔽局部变量）
class A { int x; void m(int x) { this.x = x; } }  // 参数遮蔽字段，用 this 区分
```

## 5. 常量

```java
static final int MAX = 100;        // 编译期常量（编译时内联到调用方！）
static final List<String> LIST = List.of("a");  // 引用不可变，内容仍可能可变
```

**重要陷阱**：`static final` 基本类型/String 字面量会被**内联**到使用处，修改常量后依赖方不重新编译则读到旧值（分布式/多模块常见事故）。解决：全量重编译，或改用非编译期常量（如 `getMax()`）。

## 6. 类型转换

### 6.1 自动类型提升（widening）

```
byte → short → int → long → float → double
                char ↗
```

```java
int i = 'A';          // 65
long l = i;           // 隐式
double d = 1;         // 1.0
```

注意 `long → float` 会**丢失精度**（float 只有 24 位有效尾数）。

### 6.2 强制转换（narrowing）

```java
int i = 300;
byte b = (byte) i;    // 44（截断低 8 位）
double d = 3.99;
int n = (int) d;      // 3（截断，非四舍五入）
```

### 6.3 表达式提升规则

- `byte/short/char` 参与运算自动提升为 `int`。
- 有 `long` 则结果 `long`；有 `float` 则 `float`；有 `double` 则 `double`。

```java
byte a = 1, b = 2;
byte c = (byte) (a + b);   // 必须强转，因为结果是 int
short s = 1;
s += 1;   // 复合赋值隐含强转，等价 s = (short)(s + 1)
```

## 7. 包装类与装箱

```java
Integer a = 100;      // 自动装箱 Integer.valueOf(100)
int b = a;            // 自动拆箱 a.intValue()
```

### 缓存陷阱

```java
Integer x = 127, y = 127;
System.out.println(x == y);   // true —— IntegerCache 缓存 -128~127

Integer m = 128, n = 128;
System.out.println(m == n);   // false —— 超出缓存范围，新建对象

System.out.println(m.equals(n)); // true
```

结论：**包装类比较必须用 `equals`**，或用 `Objects.equals`。

### null 拆箱 NPE

```java
Integer v = null;
int i = v;      // NullPointerException
```

建议：方法返回值尽量用基本类型；集合/DTO 用包装类（表达「无值」）。

### 各包装类常用方法

```java
Integer.parseInt("123");
Integer.valueOf("123");
Integer.toBinaryString(10);       // "1010"
Integer.MAX_VALUE; Integer.MIN_VALUE;
Double.isNaN(0.0/0.0);
Character.isDigit('5'); Character.isLetter('中');
Boolean.parseBoolean("TRUE");     // true（忽略大小写）
```

## 8. 浮点数精度问题

```java
System.out.println(0.1 + 0.2);          // 0.30000000000000004
System.out.println(0.1f + 0.2f == 0.3f);// false
```

原因：IEEE 754 二进制无法精确表示 0.1。

解决：

- 金额用 `BigDecimal`（**必须用 String 构造器**）。
- 比较浮点用误差范围：`Math.abs(a - b) < 1e-9`。

```java
new BigDecimal("0.1").add(new BigDecimal("0.2"));  // 0.3
new BigDecimal(0.1);   // 0.1000000000000000055511151231257827...  错误用法
```

## 9. char 与 String 的关系

```java
char c = '中';                 // 一个 UTF-16 单元
String emoji = "😀";           // 代理对，占 2 个 char
System.out.println(emoji.length());          // 2
System.out.println(emoji.codePointCount(0, emoji.length())); // 1
```

处理 Unicode 用 `codePointAt` / `offsetByCodePoints`，不要按 `char` 遍历。

## 10. var（Java 10+）

```java
var list = new ArrayList<String>();   // 推断为 ArrayList<String>
var map = new HashMap<String, List<Integer>>();
for (var entry : map.entrySet()) { }
```

限制：

- 只能用于**局部变量**（含 for、try-with-resources），不能用于字段/方法参数/返回值。
- 必须**声明时初始化**，不能为 null（无法推断）。
- 不是动态类型，仍是编译期强类型。

## 11. 常见坑清单

1. `int` 溢出静默发生：`Integer.MAX_VALUE + 1 == Integer.MIN_VALUE`；用 `Math.addExact` 抛异常。
2. `long` 乘法溢出：`int * int` 先溢出再赋给 long，需 `1L * a * b`。
3. `char + char` 得到 int，不是 String。
4. `==` 比较引用地址，对象比较用 `equals`。
5. 三元运算符类型提升：`true ? 1 : 1.0` 结果是 `double`。
6. `switch` 不支持 `long`/`double`（支持 int、String、enum、char）。
7. 位运算 `>>` 是算术右移（保留符号），`>>>` 是逻辑右移。

```java
System.out.println(-8 >> 1);   // -4
System.out.println(-8 >>> 1);  // 2147483644
```
