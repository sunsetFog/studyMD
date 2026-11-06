# 10 · 内部类、枚举与 Record

## 一、内部类四种形态

```java
public class Outer {
    private int x = 10;

    // 1. 成员内部类：依赖外部实例
    class Inner {
        void show() { System.out.println(x); }          // 可访问外部私有成员
        Outer getOuter() { return Outer.this; }         // 外部引用
    }
    // 创建：Outer.Inner in = outer.new Inner();

    // 2. 静态嵌套类：不依赖外部实例（推荐首选）
    static class Nested {
        static int y;
        void show() { /* 不能访问 Outer 的实例成员 x */ }
    }
    // 创建：new Outer.Nested()

    // 3. 局部内部类：方法内定义（少用）
    void method() {
        class Local { void run() { } }
        new Local().run();
    }

    // 4. 匿名内部类：一次性实现（Java 8 后多被 Lambda 替代）
    Runnable r = new Runnable() {
        @Override
        public void run() { System.out.println(x); }
    };
}
```

**静态嵌套类优先**：无外部实例依赖就加 static——防止隐式持有外部引用导致泄漏（非静态内部类隐式持有 Outer.this，若被静态集合持有则外部类无法回收）。

## 二、枚举（enum）：类型安全的常量集

```java
public enum OrderStatus {
    CREATED("已创建"),
    PAID("已支付"),
    SHIPPED("已发货"),
    COMPLETED("已完成");

    private final String desc;              // 枚举可以有字段

    OrderStatus(String desc) { this.desc = desc; }   // 构造器私有
    public String getDesc() { return desc; }

    // 枚举可以有方法
    public boolean isTerminal() {
        return this == COMPLETED;
    }

    // 也可以重写 toString
    @Override
    public String toString() { return name() + "(" + desc + ")"; }
}
```

内置方法：

```java
OrderStatus.PAID.name();               // "PAID"
OrderStatus.valueOf("PAID");           // 字符串→枚举（不存在抛 IllegalArgumentException）
OrderStatus.values();                  // 全部枚举值数组
OrderStatus.PAID.ordinal();            // 序号（0 起，慎用于持久化！）
```

### 枚举实现接口 / 抽象方法

```java
public enum Operation {
    ADD { double apply(double a, double b) { return a + b; } },   // 每个常量各自实现
    SUB { double apply(double a, double b) { return a - b; } };

    abstract double apply(double a, double b);     // 枚举的抽象方法
}
```

### 枚举天然特性

- 天生单例、线程安全、有序（values 顺序）、可 switch
- **valueOf/ordinal 不稳定风险**：新增枚举值改变 ordinal；DB 存 name 不存 ordinal
- 枚举单例是《Effective Java》推荐写法

## 三、Record（Java 16+）：不可变数据载体

```java
// 一行替代：字段 + 构造器 + getter + equals + hashCode + toString
public record Point(int x, int y) { }

Point p = new Point(1, 2);
p.x();               // 访问器叫字段名，不是 getX()
p.toString();        // "Point[x=1, y=2]"
p.equals(new Point(1, 2));   // true —— 值语义

// 构造器内校验（紧凑构造器）
public record User(String username, int age) {
    public User {
        if (age < 0) throw new IllegalArgumentException("age");
        username = username.trim();    // 可规范化参数
    }
}

// 可加静态工厂与方法
public record Money(BigDecimal amount, String currency) {
    public static Money ofCNY(String amount) {
        return new Money(new BigDecimal(amount), "CNY");
    }
    public Money add(Money other) { ... }
}

// Record 模式匹配（Java 21）——解构
if (obj instanceof Point(int x, int y)) {
    System.out.println(x + y);
}
```

**Record 适用**：DTO、坐标、键值对、事件对象、方法多返回值。**不适用**：需要可变状态或 JPA 实体（Hibernate 需要无参构造器）。

## 四、选择决策

```
固定常量集/状态机          → enum
纯数据传输（不可变）        → record
有行为的复杂对象           → class + Lombok
匿名回调                   → lambda（替代匿名内部类）
嵌套辅助类                 → static 嵌套类
```

## 常见坑

- **坑 1**：switch 枚举漏 case → 无编译警告（加 default 或用表达式 switch）
- **坑 2**：持久化存 ordinal → 插入新枚举值后全乱
- **坑 3**：非静态内部类实例被长生命周期对象持有 → 外部类泄漏
- **坑 4**：Record 访问器习惯性写 getX() → 是 x()
- **坑 5**：Record 想要 setter → 违背设计初衷，用普通类
- **坑 6**：valueOf 大小写敏感 → 严谨处理用 `Arrays.stream(values()).filter(...)`

## Checklist

- [ ] 嵌套类默认 static
- [ ] 枚举持久化存 name()，自校验 valueOf
- [ ] 纯数据用 record + 紧凑构造器校验
- [ ] 枚举做状态机 + isTerminal 类语义方法
