# 07 · 封装、继承、多态

## 一、封装

**封装 = 隐藏内部状态 + 暴露受控接口**。不只是 private + getter：

```java
public class Account {
    private BigDecimal balance = BigDecimal.ZERO;   // 不直接暴露字段

    public void withdraw(BigDecimal amount) {
        if (amount.compareTo(balance) > 0) {
            throw new IllegalStateException("余额不足");
        }
        balance = balance.subtract(amount);    // 业务规则内聚在类里
    }
}
```

封装的意义：调用方不能绕过规则；修改内部实现（如加日志/锁）不影响外部。

## 二、继承（extends）

```java
public class Animal {
    protected String name;
    public Animal(String name) { this.name = name; }

    public void eat() { System.out.println(name + " eating"); }
}

public class Dog extends Animal {

    public Dog(String name) {
        super(name);                    // 调用父类构造器，必须第一行
    }

    @Override                           // 必须加！编译期检查
    public void eat() {
        super.eat();                    // 复用父类逻辑
        System.out.println("dog style");
    }

    public void bark() { }              // 子类扩展
}
```

**Java 单继承**（一个类只能 extends 一个父类），多能力靠接口（08 篇）。

继承要点：
- 子类拥有父类非 private 成员
- 构造器不继承，子类构造必须链到父类构造
- `final` 类不可继承（String、Integer）
- private 成员对子类不可见

**is-a 判断**：只有真正的「是一种」关系才继承；「有一个」用组合（12 篇）。

## 三、多态（核心）

**多态 = 父类引用指向子类对象，调用方法时执行子类的实现**。

```java
Animal a = new Dog("tom");   // 向上转型，天然安全
a.eat();                     // 执行 Dog.eat() —— 运行时动态绑定
// a.bark();                 // 编译错误：编译期看引用类型 Animal

// 向下转型需要显式 + 检查
if (a instanceof Dog d) {    // instanceof 模式匹配（Java 16+）
    d.bark();                // 自动转成 Dog d
}
```

**动态绑定的机制**：方法调用在运行时按**实际对象类型**查虚方法表（vtable）分派——这是 Java 多态的实现基础。

### 多态的实战价值：面向接口编程

```java
// 处理器框架：新增形状无需改这个方法
public double totalArea(List<Shape> shapes) {
    double sum = 0;
    for (Shape s : shapes) sum += s.area();   // 各自实现
    return sum;
}

// 消息渠道多态
public interface Notifier { void send(String msg); }
// EmailNotifier / SmsNotifier / WebhookNotifier 各自实现
// 调用侧按配置注入具体实现，新增渠道零改动（开闭原则）
```

### 方法调用三步判断（考试级）

1. 编译期：按**引用类型**确定能调哪些方法（方法表范围）
2. 运行期：按**实际对象类型**确定执行哪个版本（动态绑定）
3. static/final/private 方法例外：静态绑定，无多态

## 四、方法重写（Override）规则

```java
父类方法：public Object process(Number n) throws Exception

子类重写要求：
- 方法名、参数列表完全相同
- 访问权限 ≥ 父类（public → public，protected → protected/public）
- 返回值类型相同或为其子类（协变返回）
- 抛出的受检异常 ≤ 父类
- static 方法是「隐藏」非重写；private/final 不可重写
```

**Override vs Overload**：
- Override（重写）：父子类间，签名相同，运行期决定
- Overload（重载）：同类内，签名不同，编译期决定

## 五、super 与父子初始化顺序

```java
// 完整顺序：父静态块 → 子静态块 → 父实例块 → 父构造器 → 子实例块 → 子构造器
```

## 六、多态陷阱：字段没有多态

```java
class Parent { int x = 1; }
class Child extends Parent { int x = 2; }

Parent p = new Child();
p.x;        // 1 ！字段按引用类型静态解析，没有动态绑定
((Child) p).x;  // 2
// 结论：字段一律 private + 方法访问，别玩字段遮蔽
```

## 常见坑

- **坑 1**：重写忘加 @Override → 签名打错变成重载，悄无声息的 bug
- **坑 2**：构造器里调用可重写方法 → 子类版本在子类字段初始化前执行
- **坑 3**：instanceof 后忘强转（Java 16 前写法） → 直接用模式匹配
- **坑 4**：误以为字段有动态绑定
- **坑 5**：滥用继承做代码复用（不是 is-a 关系）→ 12 篇组合优于继承

## Checklist

- [ ] 重写四规则（签名/权限/返回值/异常）+ 必加 @Override
- [ ] 熟悉 instanceof 模式匹配写法
- [ ] 能完整讲出动态绑定三步判断
- [ ] 面向接口编程的代码风格
