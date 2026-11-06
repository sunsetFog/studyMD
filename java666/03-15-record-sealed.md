# 15 · record、sealed 与模式匹配（Java 16/17+）

## 1. record：不可变数据载体

```java
public record Point(int x, int y) { }
```

一行等价于：

```java
public final class Point extends Record {
    private final int x;
    private final int y;

    public Point(int x, int y) { this.x = x; this.y = y; }

    public int x() { return x; }        // 访问器，不是 getX()
    public int y() { return y; }

    @Override public boolean equals(Object o) { ... }   // 逐字段
    @Override public int hashCode() { ... }             // 逐字段
    @Override public String toString() { return "Point[x=" + x + ", y=" + y + "]"; }
}
```

特性：

- 隐式 `final`，不可继承。
- 字段全部 `private final`。
- 自动生成**规范构造器**、访问器、`equals/hashCode/toString`。
- 可实现接口，可定义额外方法与静态字段（不能有实例字段）。

### 紧凑构造器（校验）

```java
public record Range(int lo, int hi) {
    public Range {                      // 无参数列表
        if (lo > hi) throw new IllegalArgumentException("lo > hi");
        // 可对参数重新赋值，自动赋给字段
    }
}
```

### 自定义方法

```java
public record Money(BigDecimal amount, String currency) {
    public Money {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
    }

    public static Money cny(String v) { return new Money(new BigDecimal(v), "CNY"); }

    public Money plus(Money other) {
        if (!currency.equals(other.currency)) throw new IllegalArgumentException("币种不同");
        return new Money(amount.add(other.amount), currency);
    }

    // 可覆盖自动生成的方法
    @Override public String toString() {
        return amount + " " + currency;
    }
}
```

### 使用场景

| 适合 | 不适合 |
|---|---|
| DTO / 值对象 | 需要可变状态（JPA 实体） |
| 方法多返回值 | 需要无参构造 + setter 的框架 |
| 模式匹配的载体 | 字段很多且需增量构建（用 Builder） |
| Map 的复合 key | 需要继承层次 |

### 与 Lombok 对比

```java
@Value                      // Lombok
public class Point {
    int x;
    int y;
}
```

| 维度 | record | Lombok @Value |
|---|---|---|
| 依赖 | 无（JDK 内置） | 需注解处理器 |
| 继承 | 不能继承类 | 同 |
| 与框架兼容 | Jackson 需 2.12+，JPA 不支持 | 更灵活 |
| IDE 支持 | 原生 | 需插件 |

> 纯数据载体优先 record；需要兼容老框架时用 Lombok 或手写。

### record 的限制

1. 不能继承其他类（已继承 `Record`）。
2. 不能声明实例字段。
3. 不能是 abstract。
4. 访问器不能叫 `getXxx()`（除非自己加）。
5. 反射创建时需用规范构造器。

```java
// Jackson 反序列化 record 需要 jackson-databind 2.12+ 且开启 ParameterNamesModule
ObjectMapper mapper = new ObjectMapper().registerModule(new ParameterNamesModule());
```

## 2. sealed：受控继承

```java
public sealed interface Shape permits Circle, Square, Triangle { }

public record Circle(double radius) implements Shape { }
public record Square(double side) implements Shape { }
public final class Triangle implements Shape {
    private final double a, b, c;
    // ...
}
```

规则：

- `permits` 列出直接子类型，子类型必须在同一模块（同一包 for 未命名模块）。
- 子类型必须是 `final`、`sealed` 或 `non-sealed` 之一。
- 省略 `permits` 时，编译器从同一文件中的子类型推断。

```java
// 同一文件可省略 permits
public sealed interface Expr {
    record Num(int v) implements Expr { }
    record Add(Expr l, Expr r) implements Expr { }
}
```

### non-sealed

```java
public sealed interface Animal permits Dog, Cat { }
public non-sealed class Dog implements Animal { }   // 允许任意子类继续扩展
public final class Cat implements Animal { }
```

### 用途

- 明确封闭的领域模型（支付方式、表达式、AST）。
- 配合 switch 实现**穷尽性检查**，新增子类型时编译报错。
- 替代「枚举 + 附加数据」的笨重写法。

```java
sealed interface Payment permits Alipay, Wechat, Card { }
record Alipay(String account) implements Payment { }
record Wechat(String openId) implements Payment { }
record Card(String no, String cvv) implements Payment { }

String pay(Payment p) {
    return switch (p) {
        case Alipay a -> "支付宝 " + a.account();
        case Wechat w -> "微信 " + w.openId();
        case Card c -> "银行卡 " + c.no().substring(c.no().length() - 4);
    };   // 无 default，编译器保证穷尽
}
```

## 3. 模式匹配

### 3.1 instanceof 模式（Java 16 正式）

```java
if (obj instanceof String s && !s.isBlank()) {
    System.out.println(s.trim());
}
```

作用域：模式变量在**条件为真**的分支内可用，且遵循短路逻辑：

```java
if (!(obj instanceof String s)) return;
System.out.println(s.length());     // 此处 s 可用
```

### 3.2 switch 类型模式（Java 21 正式）

```java
static String format(Object o) {
    return switch (o) {
        case null -> "null";
        case Integer i when i < 0 -> "负数 " + i;
        case Integer i -> "整数 " + i;
        case String s -> "字符串(" + s.length() + ")";
        case int[] a -> "数组(" + a.length + ")";
        case Point(int x, int y) -> "点(" + x + "," + y + ")";   // 记录模式（21 预览/后续正式）
        default -> "其他";
    };
}
```

### 3.3 记录模式（Record Patterns）

```java
record Point(int x, int y) { }
record Line(Point start, Point end) { }

void print(Object o) {
    if (o instanceof Line(Point(var x1, var y1), Point(var x2, var y2))) {
        System.out.printf("(%d,%d)-(%d,%d)%n", x1, y1, x2, y2);
    }
}
```

嵌套解构，省去大量 getter 调用。

### 3.4 守卫（when）

```java
switch (obj) {
    case String s when s.length() > 10 -> "长字符串";
    case String s -> "短字符串";
    default -> "其他";
}
```

### 3.5 空值处理

传统 switch 遇 null 抛 NPE。Java 21 支持 `case null`：

```java
switch (s) {
    case null -> System.out.println("null");
    case "a" -> System.out.println("a");
    default -> System.out.println("other");
}
```

## 4. 组合实战：表达式求值器

```java
public sealed interface Expr permits Num, Add, Mul, Neg { }

public record Num(double value) implements Expr { }
public record Add(Expr left, Expr right) implements Expr { }
public record Mul(Expr left, Expr right) implements Expr { }
public record Neg(Expr expr) implements Expr { }

public final class Evaluator {
    public static double eval(Expr e) {
        return switch (e) {
            case Num(double v) -> v;
            case Add(var l, var r) -> eval(l) + eval(r);
            case Mul(var l, var r) -> eval(l) * eval(r);
            case Neg(var x) -> -eval(x);
        };
    }

    public static void main(String[] args) {
        Expr e = new Add(new Num(1), new Mul(new Num(2), new Num(3)));
        System.out.println(eval(e));   // 7.0
    }
}
```

新增 `Div` 时，switch 立即编译报错，不会漏处理——这是 sealed + 模式匹配最大的价值。

## 5. 版本对应表

| 特性 | 预览版本 | 正式版本 |
|---|---|---|
| instanceof 模式 | 14、15 | **16** |
| record | 14、15 | **16** |
| sealed 类 | 15、16 | **17** |
| switch 模式匹配 | 17–20 | **21** |
| 记录模式 | 20、21 | **21** |
| 未命名变量 `_` | 21 | 22 |

## 6. 常见坑

1. record 的 `equals` 用字段类型各自的 equals，`double` 用 `Double.compare` 语义（NaN 相等、+0.0 ≠ -0.0）。
2. record 数组字段的 equals 是引用比较（数组不重写 equals），需要 `Arrays.equals` 自定义。
3. sealed 子类必须在同一模块；跨模块需在同一 module 中声明。
4. switch 模式匹配的顺序敏感：更具体的类型放前面，否则被更宽的模式拦截。
5. `case null` 必须显式写，`default` 不覆盖 null。

```java
// 顺序错误示例
switch (o) {
    case Object x -> "总是命中";     // 后面的 case 永远不可达 → 编译错误
    case String s -> "字符串";
}
```

6. record 不能用作 JPA 实体（Hibernate 6 支持 record 作为 embeddable/投影，但不支持作为 entity）。
