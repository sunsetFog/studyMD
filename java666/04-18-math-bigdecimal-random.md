# 18 · Math、BigDecimal、BigInteger 与随机数

## 1. Math

```java
Math.abs(-5);          // 5
Math.max(1, 2); Math.min(1, 2);
Math.pow(2, 10);       // 1024.0
Math.sqrt(16);         // 4.0
Math.cbrt(27);         // 3.0
Math.round(3.5);       // 4（long，四舍五入，负数向 0 舍入：Math.round(-3.5) = -3）
Math.ceil(3.1);        // 4.0
Math.floor(3.9);       // 3.0
Math.signum(-2);       // -1.0
Math.hypot(3, 4);      // 5.0（避免溢出）
Math.toRadians(180);   // π
Math.log(Math.E);      // 1.0
Math.exp(1);           // e
```

精确溢出检查（Java 8+）：

```java
Math.addExact(a, b);        // 溢出抛 ArithmeticException
Math.subtractExact(a, b);
Math.multiplyExact(a, b);
Math.incrementExact(a);
Math.negateExact(a);
Math.toIntExact(longVal);   // 超范围抛异常
```

`Math.abs(Integer.MIN_VALUE)` 仍返回负数（无正数对应），必须用 `Math.absExact`（Java 15+）。

其他：

```java
Math.random();             // [0.0, 1.0)
Math.copySign(3, -1);      // -3.0
Math.fma(a, b, c);         // a*b+c，单次舍入，精度更高
StrictMath                   // 跨平台结果完全一致（Math 允许平台优化）
```

## 2. BigDecimal

### 为什么需要

```java
System.out.println(0.1 + 0.2);   // 0.30000000000000004
```

`double` 是二进制浮点，无法精确表示十进制小数。**金额、利率、税率必须用 BigDecimal**。

### 创建

```java
new BigDecimal("0.1");        // ✔ 推荐（字符串）
BigDecimal.valueOf(0.1);      // ✔ 内部走 Double.toString
new BigDecimal(0.1);          // ✘ 0.1000000000000000055511151231257827...
BigDecimal.ZERO / ONE / TEN;
```

### 运算（不可变，返回新对象）

```java
BigDecimal a = new BigDecimal("10.00");
BigDecimal b = new BigDecimal("3");

a.add(b);          // 13.00
a.subtract(b);     // 7.00
a.multiply(b);     // 30.00
a.divide(b);       // ArithmeticException: Non-terminating decimal expansion
a.divide(b, 4, RoundingMode.HALF_UP);   // 3.3333
a.divide(b, MathContext.DECIMAL128);

a.remainder(b);
a.pow(2);
a.negate();
a.abs();
a.max(b); a.min(b);
```

**除法必须指定精度和舍入模式**（除非能整除），否则可能抛异常。

### 比较

```java
new BigDecimal("1.0").equals(new BigDecimal("1.00"));      // false（scale 不同）
new BigDecimal("1.0").compareTo(new BigDecimal("1.00"));   // 0（数值相等）

// 正确比较
if (a.compareTo(b) == 0) { }
if (a.compareTo(b) > 0) { }    // a > b
```

### 舍入模式

| 模式 | 说明 | 1.5 → 0 位 | -1.5 → 0 位 |
|---|---|---|---|
| `UP` | 远离零 | 2 | -2 |
| `DOWN` | 向零 | 1 | -1 |
| `CEILING` | 向正无穷 | 2 | -1 |
| `FLOOR` | 向负无穷 | 1 | -2 |
| `HALF_UP` | 四舍五入 | 2 | -2 |
| `HALF_DOWN` | 五舍六入 | 1 | -1 |
| `HALF_EVEN` | 银行家舍入 | 2 | -2 |

`HALF_EVEN`（银行家舍入）：向最近的偶数舍入，1.5→2，2.5→2，3.5→4。金融统计推荐，误差更小。

```java
BigDecimal v = new BigDecimal("2.345");
v.setScale(2, RoundingMode.HALF_UP);    // 2.35
v.setScale(2, RoundingMode.HALF_EVEN);  // 2.34（4 是偶数）
```

### scale 与精度

```java
BigDecimal x = new BigDecimal("123.4500");
x.scale();       // 4（小数位数）
x.precision();   // 7（有效数字位数）
x.stripTrailingZeros();   // 123.45，scale=2
x.toPlainString();        // "123.4500"（不用科学计数法）
x.toString();             // 可能输出 1.2345E+2
```

> 对外输出统一用 `toPlainString()`，避免科学计数法。

### 工具封装

```java
public final class Money {
    public static final int SCALE = 2;
    public static final RoundingMode RM = RoundingMode.HALF_UP;

    public static BigDecimal of(String v) { return new BigDecimal(v).setScale(SCALE, RM); }
    public static BigDecimal add(BigDecimal a, BigDecimal b) { return a.add(b).setScale(SCALE, RM); }
    public static BigDecimal mul(BigDecimal a, BigDecimal b) { return a.multiply(b).setScale(SCALE, RM); }
    public static BigDecimal div(BigDecimal a, BigDecimal b) { return a.divide(b, SCALE, RM); }
    public static String format(BigDecimal v) { return v.setScale(SCALE, RM).toPlainString(); }
}
```

### 常见坑

1. `new BigDecimal(double)` 精度丢失。
2. `equals` 比 scale，必须用 `compareTo`。
3. `divide` 未指定舍入 → 异常。
4. BigDecimal 不可变，`a.add(b)` 不改变 a，必须接收返回值。
5. 金额用 `double` 存储/传输。
6. 数据库字段用 `double`（应用 `DECIMAL(19,4)`）。
7. `setScale` 不指定 RoundingMode 在需要舍入时抛异常。

## 3. BigInteger

任意精度整数，用于超大数（加密、组合数、阶乘）。

```java
BigInteger a = new BigInteger("123456789012345678901234567890");
BigInteger b = BigInteger.valueOf(1000);

a.add(b); a.subtract(b); a.multiply(b);
a.divide(b);                          // 整除
a.mod(b);                             // 取模（结果非负）
a.pow(10);
a.gcd(b);
a.isProbablePrime(100);               // Miller-Rabin，误判率 2^-100
a.nextProbablePrime();
a.modPow(exp, mod);                   // 模幂，RSA 核心
a.shiftLeft(10); a.shiftRight(10);
a.bitLength();
a.toByteArray();
new BigInteger(1, bytes);             // 字节转正整数
```

## 4. 随机数

### Math.random()

```java
double d = Math.random();                       // [0,1)
int n = (int) (Math.random() * 100);            // [0,100)
```

内部用共享的 `Random` 实例，**多线程下有竞争**，不适合高并发。

### Random

```java
Random r = new Random();          // 以当前时间播种
Random r2 = new Random(42);       // 固定种子，可复现
r.nextInt();                      // 全 int 范围
r.nextInt(100);                   // [0,100)
r.nextLong();
r.nextDouble();
r.nextBoolean();
r.nextBytes(byte[]);
r.ints(10, 1, 100).toArray();     // Java 8+ 流式
```

**线性同余算法，可预测，绝不可用于安全场景**。

### ThreadLocalRandom

```java
int n = ThreadLocalRandom.current().nextInt(1, 100);
long l = ThreadLocalRandom.current().nextLong();
double d = ThreadLocalRandom.current().nextDouble(0, 1);
```

高并发首选，无竞争，性能远高于共享 `Random`。

### SecureRandom

```java
SecureRandom sr = new SecureRandom();          // 自动选最强算法
byte[] token = new byte[32];
sr.nextBytes(token);
String hex = HexFormat.of().formatHex(token);

sr.nextInt(100);                               // 安全随机整数
sr.generateSeed(16);
```

用途：密码盐、会话 token、验证码、密钥、抽奖（防预测）。

性能比 Random 慢一个数量级，不要用于普通场景。

### 加权随机

```java
static <T> T weightedPick(List<T> items, List<Integer> weights) {
    int total = weights.stream().mapToInt(Integer::intValue).sum();
    int r = ThreadLocalRandom.current().nextInt(total);
    for (int i = 0; i < items.size(); i++) {
        r -= weights.get(i);
        if (r < 0) return items.get(i);
    }
    throw new IllegalStateException();
}
```

### 洗牌

```java
List<Integer> list = new ArrayList<>(List.of(1,2,3,4,5));
Collections.shuffle(list);
Collections.shuffle(list, new Random(42));       // 可复现
```

### 抽奖去重

```java
// 从 n 中抽 k 个不重复
Set<Integer> picked = new LinkedHashSet<>();
while (picked.size() < k) picked.add(ThreadLocalRandom.current().nextInt(n));
```

## 5. 数值类型选型

| 场景 | 类型 |
|---|---|
| 计数、索引 | `int`（< 21 亿） |
| 时间戳、ID | `long` |
| 金额 | `BigDecimal`（数据库 `DECIMAL`） |
| 科学计算 | `double` |
| 图形/游戏坐标 | `float`/`int` |
| 超大整数（加密） | `BigInteger` |
| 状态码 | `enum` |

## 6. 精度与舍入速查

```java
// 保留 2 位小数
double d = 3.14159;
BigDecimal bd = BigDecimal.valueOf(d).setScale(2, RoundingMode.HALF_UP);

// 千分位
DecimalFormat df = new DecimalFormat("#,##0.00");
df.format(new BigDecimal("1234567.891"));   // 1,234,567.89

// 百分比
NumberFormat pf = NumberFormat.getPercentInstance();
pf.setMaximumFractionDigits(1);
pf.format(0.856);                            // 85.6%

// 向上取整到分（避免少收钱）
BigDecimal ceil = amount.setScale(2, RoundingMode.CEILING);
```

## 7. 常见坑汇总

1. 用 `==` 比较 `Integer`（缓存陷阱），用 `equals`。
2. `Math.round(-1.5)` 返回 -1，不是 -2。
3. `Math.abs(Integer.MIN_VALUE)` 溢出为负。
4. `(int) 3.99` 是 3，不是 4。
5. BigDecimal `equals` 与 `compareTo` 语义不同。
6. BigDecimal 运算忘了接收返回值。
7. `Random` 用于生成 token（可预测）。
8. `Math.random()` 在高并发下成为瓶颈。
9. 浮点累加误差：`0.1` 累加 10 次 ≠ 1.0。
10. `Integer.parseInt` 对超范围字符串抛 `NumberFormatException`。
