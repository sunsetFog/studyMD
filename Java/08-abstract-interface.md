# 08 · 抽象类与接口

## 一、抽象类（abstract class）

```java
public abstract class Shape {           // 不能被实例化

    protected String name;

    public Shape(String name) { this.name = name; }   // 可以有构造器/字段/具体方法

    public abstract double area();        // 抽象方法：无实现，子类必须重写

    public void describe() {              // 具体方法：模板代码
        System.out.println(name + " area=" + area());   // 调用抽象方法 → 多态
    }
}

public class Circle extends Shape {
    private final double radius;
    public Circle(double r) { super("circle"); this.radius = r; }

    @Override
    public double area() { return Math.PI * radius * radius; }
}
```

**模板方法模式**就在此实现：父类定流程（具体方法），子类填步骤（抽象方法）。

## 二、接口（interface）

```java
public interface Comparable2<T> {
    int compareTo(T other);               // 默认 public abstract
}

public interface Runnable2 {
    void run();
}

// Java 8+ 接口可以有默认方法与静态方法
public interface Vehicle {

    // 默认方法：实现类可直接用或选择性重写
    default void start() {
        System.out.println("starting");
    }

    // 静态方法：工具性质，不参与继承
    static Vehicle of(String type) { return null; }

    // 常量（public static final 隐式）
    int MAX_SPEED = 300;
}

// 多实现
public class Car implements Vehicle, Comparable2<Car> {
    @Override public void start() { Vehicle.super.start(); }  // 冲突时显式指定
    @Override public int compareTo(Car o) { return 0; }
}
```

## 三、抽象类 vs 接口（选型核心）

| 维度 | 抽象类 | 接口 |
|---|---|---|
| 关系语义 | is-a（是什么） | can-do（能做什么） |
| 构造器/状态字段 | 有（protected 状态） | 无实例状态（仅常量） |
| 继承数量 | 单继承 | 多实现 |
| 具体方法 | 任意 | default/static |
| 典型用途 | 模板方法、公共骨架 | 能力契约、回调、解耦 |

**决策口诀**：
```
要共享状态和部分实现 → 抽象类
要定义能力契约、可多份实现并存 → 接口
核心库回调/扩展点 → 一律接口（如 Comparator、Comparable）
```

现代设计主流：**接口 + 组合**，抽象类退居模板方法场景。

## 四、函数式接口（衔接 11/18 篇）

```java
@FunctionalInterface        // 编译期校验：只能有一个抽象方法
public interface Calculator {
    int calc(int a, int b);
    default String desc() { return "calculator"; }   // default 不算
}

Calculator add = (a, b) -> a + b;       // Lambda 实现
```

JDK 内置四大函数式接口（18 篇 Stream 全用到）：

| 接口 | 抽象方法 | 语义 |
|---|---|---|
| Supplier<T> | T get() | 生产者 |
| Consumer<T> | void accept(T) | 消费者 |
| Function<T,R> | R apply(T) | 转换 |
| Predicate<T> | boolean test(T) | 判断 |

## 五、接口的设计价值

```java
// 1. 解耦：调用方依赖接口而非实现
public class OrderService {
    private final PayChannel payChannel;      // 接口注入
    public OrderService(PayChannel pc) { this.payChannel = pc; }
}

// 2. 多实现并存与运行时选择（策略模式）
Map<String, PayChannel> channels = Map.of(
        "alipay", new AlipayChannel(),
        "wechat", new WechatChannel());

// 3. 能力标记接口（如 Serializable、Cloneable——无方法纯标记）

// 4. 接口隔离：大接口拆小（12 篇 ISP）
```

## 六、final 三兄弟

```java
final class Utility { }              // 类：不可继承
final void keyStep() { }             // 方法：不可重写
final int MAX = 100;                 // 变量：不可重新赋值

// final 引用 vs 不可变对象
final List<String> list = new ArrayList<>();
list.add("x");        // OK！引用不变，对象内容可变
list = new ArrayList<>();   // 编译错误
// 真不可变：List.of(...)（16 篇）
```

final 字段 + 构造器初始化 = 不可变对象基础 → 天然线程安全（21 篇）。

## 常见坑

- **坑 1**：抽象类有构造器却不能 new → 构造器给子类 super() 调用用
- **坑 2**：实现类没实现全部抽象方法 → 自己也要声明 abstract
- **坑 3**：两个接口 default 方法签名冲突 → 必须 Override 并用 `InterfaceA.super.method()` 指定
- **坑 4**：接口里定义可变状态 → 接口字段全是 public static final 常量
- **坑 5**：接口仅为了「代码复用」而设计 → 组合更合适

## Checklist

- [ ] 抽象类/接口按 is-a / can-do 选型
- [ ] 回调与扩展点一律接口
- [ ] default 方法的冲突解决方案
- [ ] final 引用 ≠ 不可变对象
