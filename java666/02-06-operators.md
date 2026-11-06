# 06 · 运算符

## 1. 分类总览

| 类别 | 运算符 |
|---|---|
| 算术 | `+ - * / % ++ --` |
| 关系 | `== != > < >= <= instanceof` |
| 逻辑 | `&& \|\| ! & \| ^` |
| 位 | `& \| ^ ~ << >> >>>` |
| 赋值 | `= += -= *= /= %= &= \|= ^= <<= >>= >>>=` |
| 三元 | `? :` |
| 其他 | `. () [] new :: ->` |

## 2. 算术运算符

```java
int a = 7, b = 2;
a / b      // 3    整数除法，向零取整
a % b      // 1    取余，符号随被除数
a / 2.0    // 3.5  提升为 double

-7 / 2     // -3
-7 % 2     // -1
7 % -2     // 1
```

### 自增自减

```java
int i = 1;
int x = i++;   // x=1, i=2（先取值后自增）
int y = ++i;   // i=3, y=3（先自增后取值）
```

**未定义行为警告**：

```java
int i = 0;
i = i++;        // i 仍为 0（先保存旧值，自增后又赋值回去）
int[] arr = {0};
arr[i] = i++;   // 结果依赖求值顺序，Java 规定先求左侧索引
```

Java 规范保证了求值顺序（从左到右），但这类代码可读性极差，禁止在生产代码中出现。

### 溢出

```java
int max = Integer.MAX_VALUE;
max + 1                       // -2147483648，静默溢出
Math.addExact(max, 1)         // 抛 ArithmeticException
Math.multiplyExact(a, b)
```

`int` 除以 0 抛 `ArithmeticException`；`double` 除以 0 得到 `Infinity`/`NaN`：

```java
System.out.println(1.0 / 0);      // Infinity
System.out.println(0.0 / 0);      // NaN
System.out.println(1 % 0.0);      // NaN
```

## 3. 关系与相等

```java
5 == 5.0        // true，数值提升后比较
'中' == 20013   // true
"a" == "a"      // 可能 true（字符串池），不可依赖
"a".equals("a") // true，正确写法
```

`instanceof`（Java 16+ 模式匹配）：

```java
if (obj instanceof String s && s.length() > 3) {
    System.out.println(s.toUpperCase());   // s 已强转，作用域在条件为真分支
}
```

## 4. 逻辑运算符与短路

| 运算符 | 说明 |
|---|---|
| `&&` | 短路与：左侧 false 则不计算右侧 |
| `\|\|` | 短路或：左侧 true 则不计算右侧 |
| `&` / `\|` | 非短路，两侧都计算（也可用于位运算） |
| `^` | 异或，布尔上表示「不同为 true」 |

```java
if (obj != null && obj.size() > 0) { }   // 依赖短路避免 NPE

boolean r = check() | sideEffect();      // sideEffect 一定会执行
```

**陷阱**：把 `&&` 写成 `&` 会在对象为 null 时继续调用方法导致 NPE。

## 5. 位运算

```java
int a = 0b1100;   // 12
int b = 0b1010;   // 10
a & b     // 1000 = 8
a | b     // 1110 = 14
a ^ b     // 0110 = 6
~a        // ...11110011 = -13
a << 2    // 48   左移，低位补 0
a >> 2    // 3    算术右移，高位补符号位
a >>> 2   // 3    逻辑右移，高位补 0
```

### 移位细节

- 移位位数会先对 32（long 为 64）取模：`a << 33` 等价 `a << 1`。
- `byte/short/char` 移位前会提升为 int。
- 负数右移：

```java
System.out.println(-8 >> 1);    // -4
System.out.println(-8 >>> 1);   // 2147483644
```

### 位运算实战

```java
// 1. 判断奇偶（比 % 快，但可读性差，现代 JIT 无差别）
boolean odd = (n & 1) == 1;

// 2. 交换两数（无需临时变量，但有溢出风险，不推荐实际使用）
a ^= b; b ^= a; a ^= b;

// 3. 权限位标志
static final int READ = 1 << 0;    // 1
static final int WRITE = 1 << 1;   // 2
static final int EXEC = 1 << 2;    // 4
int perm = READ | WRITE;
boolean canRead = (perm & READ) != 0;
perm &= ~WRITE;                    // 清除权限

// 4. HashMap 扰动函数思想：让高位参与索引计算
static int spread(int h) { return h ^ (h >>> 16); }

// 5. 取绝对值（不推荐手写，仅演示）
int abs = (n ^ (n >> 31)) - (n >> 31);
```

## 6. 三元运算符

```java
int max = a > b ? a : b;
String s = obj == null ? "unknown" : obj.toString();
```

类型规则：两分支类型会统一提升。

```java
Object o = true ? 1 : "a";      // 编译错误：int 与 String 无公共数值类型
Number n = true ? 1 : 2.0;      // 1.0（double）
```

嵌套三元可读性差，超过一层建议改用 `if` 或 `switch`。

## 7. 复合赋值

```java
int i = 10;
i += 5;      // 15
i *= 2;      // 30

byte b = 10;
b += 5;      // 等价 b = (byte)(b + 5)，编译器插入隐式强转
// b = b + 5;  // 编译错误
```

副作用只求值一次：

```java
arr[i++] += 5;   // i 只自增一次
```

## 8. 运算符优先级（从高到低）

| 优先级 | 运算符 |
|---|---|
| 1 | `()` `[]` `.` |
| 2 | `++ -- ~ !`（一元） |
| 3 | `new` `(type)` 强转 |
| 4 | `* / %` |
| 5 | `+ -` |
| 6 | `<< >> >>>` |
| 7 | `< > <= >= instanceof` |
| 8 | `== !=` |
| 9 | `&` |
| 10 | `^` |
| 11 | `\|` |
| 12 | `&&` |
| 13 | `\|\|` |
| 14 | `? :` |
| 15 | `= += -= ...`（右结合） |

**结论：永远不要依赖记忆优先级，复杂表达式加括号。**

```java
// 错误预期：以为先算 1 + 1
boolean r = 1 + 1 == 2;     // 实际 (1+1) == 2 → true（+ 优先于 ==）
int x = 1 << 2 + 3;         // 1 << 5 = 32，不是 (1<<2)+3
```

## 9. 常见陷阱

1. `==` 用于对象比较（String、Integer、BigDecimal）。
2. `BigDecimal.equals` 会比较 scale：`1.0` ≠ `1.00`，比较用 `compareTo`。
3. 浮点直接用 `==`。
4. `&&` 与 `&` 混淆。
5. 溢出后未检查（尤其金额、ID 计算）。
6. `i = i++`。
7. 三元表达式两分支类型不一致导致意外提升。
8. 位运算优先级低于 `==`：`(a & 1) == 1` 必须加括号。

## 10. 编码建议

- `if (flag == true)` → `if (flag)`；`if (flag == false)` → `if (!flag)`。
- 条件取反用 `!` 而不是 `== false`。
- 多重条件封装成有名字的布尔方法：`if (isEligible(user))`。
- 用 `Objects.equals(a, b)` 避免 null 判断。
- 用 `Math.addExact` 等显式检查溢出（金融/计费场景必须）。
