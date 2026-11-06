# 02 · 语法基础

## 一、变量与标识符

```java
int age = 18;               // 局部变量：方法内，无默认值必须初始化
static int counter;         // 成员变量：类级，有默认值（0/null/false）
final double PI = 3.14159;  // 常量：final + 大写蛇形命名
```

命名规范：
- 类：大驼峰 `UserService`
- 方法/变量：小驼峰 `userName`
- 常量：`MAX_RETRY_COUNT`
- 标识符：字母/数字/_/$ 组成，不能数字开头，不能是关键字

## 二、基本数据类型（8 种）

| 类型 | 字节 | 范围 | 默认值 |
|---|---|---|---|
| byte | 1 | -128 ~ 127 | 0 |
| short | 2 | ±3.2万 | 0 |
| **int** | 4 | ±21亿 | 0 |
| **long** | 8 | ±9.2×10^18 | 0L |
| float | 4 | ~±3.4×10^38 | 0.0f |
| **double** | 8 | ~±1.8×10^308 | 0.0d |
| char | 2 | 单个 Unicode 字符 | '\u0000' |
| **boolean** | — | true/false（无 0/1 概念） | false |

加粗为最常用。整数默认 int，小数默认 double：

```java
long big = 10_000_000_000L;   // long 字面量必须加 L
float f = 3.14f;              // float 必须加 f
char c = 'A';                 // 单引号，一个字符
char cn = '中';               // Unicode 没问题
```

## 三、引用类型

```java
String name = "Java";             // 字符串（特殊：有字面量池）
int[] arr = {1, 2, 3};            // 数组
List<String> list = new ArrayList<>();   // 对象
User user = null;                 // 引用可为 null（空指针之源）
```

**基本类型 vs 引用类型的本质**：
- 基本类型：变量直接存值（栈上）
- 引用类型：变量存对象地址，对象本体在堆

## 四、类型转换

### 自动（隐式）：小 → 大

```java
int i = 100;
long l = i;        // 自动：byte→short→int→long→float→double
double d = i + 1;  // int 参与运算自动提升
```

### 强制（显式）：大 → 小（可能丢失精度）

```java
double d = 3.99;
int i = (int) d;        // 3 —— 截断不是四舍五入！
long l = 10_000_000_000L;
int overflow = (int) l; // 溢出：截断低位，值错乱
```

### 经典陷阱：byte/short 运算

```java
byte a = 10, b = 20;
byte c = (byte) (a + b);   // a+b 自动提升为 int，必须强转
```

### 字符串与数值互转

```java
int i = Integer.parseInt("123");
double d = Double.parseDouble("3.14");
String s = String.valueOf(123);        // 或 123 + ""
```

## 五、运算符

```java
// 算术
int q = 10 / 3;        // 3 —— 整数除法截断！
int r = 10 % 3;        // 1 取模
double dq = 10.0 / 3;  // 3.33... 至少一方是小数

// 自增前置/后置
int a = 5;
int b = a++;   // b=5, a=6（先取值后自增）
int c = ++a;   // c=7, a=7（先自增后取值）

// 关系/逻辑
&&  与（短路）  ||  或（短路）  !  非
&   与（非短路）|   或（非短路）^  异或

// 三元
int max = a > b ? a : b;

// 位运算
5 & 3   // 1   按位与
5 | 3   // 7   按位或
5 ^ 3   // 6   按位异或（可用于交换/加密）
5 << 1  // 10  左移=乘2
5 >> 1  // 2   右移=除2
-5 >>> 1 // 无符号右移，高位补0

// 经典位运算用法
n & 1 == 1      // 判断奇偶
n & (n - 1) == 0 // 判断 2 的幂
```

## 六、Scanner 输入与运算优先级

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();
String line = sc.nextLine();   // 注意与 nextInt 混用时的换行符残留问题
sc.close();
```

优先级口诀：`算术 > 关系 > 逻辑 > 赋值`；拿不准就加括号——**可读性优先于省括号**。

## 七、var 与类型推断（Java 10+）

```java
var list = new ArrayList<String>();   // 局部变量类型推断
var entry = Map.entry("key", 1);

// 只能用于局部变量（有明确初始化值）
// 不能用于：成员变量、方法参数、返回值、null
```

## 常见坑

- **坑 1**：整数除法 `1/2 == 0` → 需要小数时一方转 double
- **坑 2**：浮点比较 `0.1+0.2 == 0.3` 为 false → 货币用 BigDecimal（13 篇），比较用 `Math.abs(a-b) < eps`
- **坑 3**：`(int) 3.99` 截断为 3 → 四舍五入用 `Math.round`
- **坑 4**：char 参与 `+` 变成 ASCII 加法：`'a' + 1 == 98`，拼接需 `"" + 'a' + 1`
- **坑 5**：long 变量不加 L 编译报错或 int 溢出后才报错
- **坑 6**：字符串拼接循环内用 `+` → 性能差，用 StringBuilder（04 篇）

## Checklist

- [ ] 默写 8 种基本类型及默认值
- [ ] 理解整数提升：byte/short/char 运算提升为 int
- [ ] 熟练位运算奇偶/2 的幂判断
- [ ] 字面量后缀 L/f 不遗漏
