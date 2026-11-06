# 11 · 抽象类与接口

## 1. 抽象类

```java
public abstract class Shape {
    protected final String name;          // 可有字段

    protected Shape(String name) { this.name = name; }

    public abstract double area();        // 抽象方法，无实现

    public String describe() {            // 具体方法，可复用
        return name + " area=" + area();
    }
}

public class Circle extends Shape {
    private final double r;
    public Circle(double r) { super("circle"); this.r = r; }
    @Override public double area() { return Math.PI * r * r; }
}
```

规则：

- 有抽象方法的类**必须**声明为 abstract。
- abstract 类**不能实例化**，但可以有构造器（供子类调用）。
- 可以有 0 个抽象方法（只为禁止实例化）。
- 子类必须实现所有抽象方法，否则子类也得是抽象类。
- `abstract` 不能与 `final`/`static`/`private` 组合修饰方法。

## 2. 接口

```java
public interface PaymentGateway {
    // 常量：隐式 public static final
    String VERSION = "1.0";

    // 抽象方法：隐式 public abstract
    PaymentResult pay(PaymentRequest req);

    // 默认方法（Java 8+）
    default boolean supports(Currency c) { return c == Currency.CNY; }

    // 静态方法（Java 8+）
    static PaymentGateway noop() {
        return req -> PaymentResult.success();
    }

    // 私有方法（Java 9+），供默认方法复用
    private void log(String msg) { System.out.println("[gw] " + msg); }
}
```

成员限定：

| 成员 | 修饰符 |
|---|---|
| 常量 | `public static final`（可省略） |
| 抽象方法 | `public abstract`（可省略） |
| 默认方法 | `public default`（public 可省略） |
| 静态方法 | `public static` |
| 私有方法 | `private`（Java 9+） |

> 接口不能有实例字段、构造器；不能 `new`。

## 3. 多接口与默认方法冲突

```java
interface A { default void hello() { System.out.println("A"); } }
interface B { default void hello() { System.out.println("B"); } }

class C implements A, B {
    @Override
    public void hello() {      // 必须重写，否则编译错误
        A.super.hello();       // 显式选择
    }
}
```

类优先原则：

1. **类中的具体方法** > 接口默认方法（父类方法胜出）。
2. 两个接口默认方法冲突 → 必须重写。
3. 抽象方法声明 > 默认方法（编译器要求实现）。

```java
class Base { public void hello() { System.out.println("Base"); } }
class D extends Base implements A { }   // 调用 Base.hello()，接口默认方法被忽略
```

## 4. 函数式接口

只有一个抽象方法的接口（SAM），可用 Lambda 实现。

```java
@FunctionalInterface
interface Calculator {
    int apply(int a, int b);
    // 可以有 default / static 方法，不影响 SAM
    default Calculator andThen(Calculator next) {
        return (a, b) -> next.apply(this.apply(a, b), b);
    }
}

Calculator add = (a, b) -> a + b;
```

`@FunctionalInterface` 让编译器校验（多于一个抽象方法报错）。

JDK 内置四大接口见 12-55。

## 5. 抽象类 vs 接口

| 维度 | 抽象类 | 接口 |
|---|---|---|
| 继承数量 | 1 | 多 |
| 状态 | 可有实例字段 | 只能常量 |
| 构造器 | 有 | 无 |
| 方法实现 | 任意 | default/static/private |
| 语义 | is-a（是什么） | can-do（能做什么） |
| 演进 | 加抽象方法破坏子类 | 加 default 方法向后兼容 |
| 访问修饰 | 任意 | public（Java 9 私有） |

**选型**：

- 需要**共享状态/代码**、表达**模板骨架** → 抽象类。
- 需要**能力契约**、**多实现**、**跨继承树** → 接口。
- 两者结合：接口定义契约，抽象类提供骨架实现（`AbstractList` + `List`）。

```java
public interface Repository<T, ID> {
    T findById(ID id);
    void save(T entity);
}

public abstract class AbstractJpaRepository<T, ID> implements Repository<T, ID> {
    protected abstract EntityManager em();     // 子类提供
    @Override public T findById(ID id) { return em().find(getType(), id); }
    protected abstract Class<T> getType();
}
```

## 6. 接口的演进历史

| 版本 | 能力 |
|---|---|
| Java 7 及以前 | 常量 + 抽象方法 |
| Java 8 | default 方法、static 方法（支持 Lambda 与集合 Stream） |
| Java 9 | private 方法（默认方法复用） |
| Java 16+ | 可配合 sealed 限制实现者 |

## 7. 标记接口

无任何方法的接口，用于标记语义：

- `Serializable`
- `Cloneable`
- `RandomAccess`（标记 ArrayList 支持快速随机访问）

> 现代替代：注解（`@FunctionalInterface`、`@Deprecated`）。标记接口仍有一个优势：可用于泛型边界 `T extends Serializable`。

## 8. 密封接口（Java 17）

```java
public sealed interface Shape permits Circle, Square, Triangle { }

public record Circle(double r) implements Shape { }
public record Square(double side) implements Shape { }
public final class Triangle implements Shape { }   // 子类必须 final/sealed/non-sealed
```

配合 switch 模式匹配可做穷尽性检查，编译器保证不漏分支。

## 9. 常见设计坑

1. 接口里放常量做配置（应使用配置类/枚举）。
2. 接口方法过多（胖接口），违反 ISP；拆成小接口。
3. 默认方法中调用抽象方法形成隐式契约，文档必须说明。
4. 抽象类构造器中调用抽象方法（子类字段未初始化）。
5. 用接口做「常量池」（`implements Constants`），污染类型层次。
6. 误以为 default 方法能被 Lambda 覆写（Lambda 只实现 SAM）。

## 10. 实战：策略模式

```java
public interface DiscountStrategy {
    BigDecimal apply(BigDecimal price);
    default boolean applicable(Order order) { return true; }
}

@Component
class NoDiscount implements DiscountStrategy {
    public BigDecimal apply(BigDecimal price) { return price; }
}

@Component
class VipDiscount implements DiscountStrategy {
    public BigDecimal apply(BigDecimal price) { return price.multiply(new BigDecimal("0.9")); }
    @Override public boolean applicable(Order order) { return order.isVip(); }
}

@Service
class PriceService {
    private final List<DiscountStrategy> strategies;   // Spring 注入所有实现

    PriceService(List<DiscountStrategy> strategies) { this.strategies = strategies; }

    BigDecimal finalPrice(Order order) {
        return strategies.stream()
            .filter(s -> s.applicable(order))
            .reduce(BigDecimal.ZERO,
                    (p, s) -> s.apply(p),      // 简化示例
                    BigDecimal::add);
    }
}
```

这是接口最典型的价值：**新增策略无需修改调用方**（开闭原则）。
