# 09 · 类与对象

## 1. 类的定义

```java
public class User {
    // 字段（状态）
    private Long id;
    private String name;
    private int age;

    // 构造器
    public User() { }

    public User(Long id, String name) {
        this.id = id;
        this.name = name;
    }

    // 方法（行为）
    public String greeting() {
        return "Hi, " + name;
    }
}
```

对象创建：

```java
User u = new User(1L, "Tom");
```

`new` 做的事：

1. 在堆上分配内存（类加载完成的前提下）。
2. 字段赋默认值（0/null/false）。
3. 执行实例初始化块与字段初始化器。
4. 调用构造器。

## 2. 构造器

```java
public class Order {
    private final Long id;
    private String remark;

    public Order(Long id) {
        this(id, null);        // 必须第一行
    }

    public Order(Long id, String remark) {
        this.id = id;          // final 字段必须赋值
        this.remark = remark;
    }
}
```

规则：

- 名字与类名相同，无返回类型。
- 不写任何构造器时，编译器生成**无参默认构造器**；一旦写了有参构造器，默认构造器消失。
- `this(...)` 调用同类其他构造器，必须第一行；`super(...)` 同理。
- 构造器中不要调用可被重写的方法（子类字段尚未初始化，经典 bug）。

```java
class Parent {
    Parent() { init(); }                 // 危险
    void init() { }
}
class Child extends Parent {
    private int value = 10;
    @Override void init() { System.out.println(value); }  // 输出 0
}
```

## 3. this 关键字

| 用法 | 说明 |
|---|---|
| `this.field` | 区分同名字段与参数 |
| `this.method()` | 调用本类方法 |
| `this(...)` | 调用本类其他构造器 |
| `return this` | 链式调用（Builder） |

```java
public User name(String name) { this.name = name; return this; }
public User age(int age) { this.age = age; return this; }

User u = new User().name("Tom").age(18);
```

## 4. 静态成员

```java
public class Counter {
    private static int total;              // 类变量，所有实例共享
    public static final int MAX = 100;     // 常量

    static {                               // 静态初始化块
        total = 0;
    }

    public static int getTotal() { return total; }  // 静态方法
}
```

- 静态成员属于类，可通过 `类名.成员` 访问（推荐），也可通过实例访问（不推荐）。
- 静态方法中**不能使用 `this`/`super`**，不能直接访问实例成员。
- 静态方法不能被重写（只能被隐藏）。
- 静态变量在类初始化时赋值，生命周期与类相同。

## 5. 初始化顺序（高频面试）

```java
public class Init {
    static int s = print("1 静态字段");
    static { print("2 静态块"); }
    int i = print("3 实例字段");
    { print("4 实例块"); }
    Init() { print("5 构造器"); }

    static int print(String msg) {
        System.out.println(msg);
        return 0;
    }

    public static void main(String[] args) {
        new Init();
        System.out.println("--- 第二次 ---");
        new Init();
    }
}
```

输出：

```
1 静态字段
2 静态块
3 实例字段
4 实例块
5 构造器
--- 第二次 ---
3 实例字段
4 实例块
5 构造器
```

完整规则（含继承）：

1. 父类静态字段/静态块（按源码顺序）
2. 子类静态字段/静态块
3. 父类实例字段/实例块
4. 父类构造器
5. 子类实例字段/实例块
6. 子类构造器

> 「静态只一次，实例每次 new 都来一遍」。

## 6. 对象与引用

```java
User a = new User("Tom");
User b = a;              // 同一对象
b.setName("Jerry");
System.out.println(a.getName());   // Jerry

a = null;                // a 不再引用，对象仍被 b 引用，不回收
```

对象生命周期：创建 → 使用 → 不可达 → GC 回收。

## 7. toString / equals / hashCode

IDE 可自动生成，也可用 record 或 Lombok。详见 03-13。

```java
@Override
public String toString() {
    return "User{id=" + id + ", name='" + name + "'}";
}
```

## 8. 值传递（Java 只有值传递）

```java
void change(int x) { x = 100; }                 // 不影响调用方

void change(User u) { u.setName("new"); }       // 影响，修改对象内容

void reassign(User u) { u = new User("other"); }// 不影响，只改了副本引用
```

## 9. 类的组织原则

- **单一职责**：一个类只做一件事，字段与方法围绕同一概念。
- **高内聚低耦合**：字段私有，通过方法暴露必要行为。
- **不可变优先**：字段 `final`，无 setter，返回防御性拷贝。
- **避免上帝类**：超过 500 行 / 20 个方法就该拆分。
- **组合优于继承**：优先把能力作为字段注入。

```java
// 不可变类模板
public final class Money {
    private final BigDecimal amount;
    private final String currency;

    public Money(BigDecimal amount, String currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public Money plus(Money other) {
        requireSameCurrency(other);
        return new Money(amount.add(other.amount), currency);  // 返回新对象
    }
    // 只有 getter，没有 setter
}
```

## 10. 常见错误

| 错误 | 说明 |
|---|---|
| 构造器中调用可重写方法 | 子类字段未初始化 |
| 用 `==` 比较对象 | 比较地址 |
| 忘记 `this` 导致自赋值 | `name = name;` |
| 静态方法里用实例变量 | 编译错误 |
| 公有可变字段 | 破坏封装 |
| 构造器抛异常后仍使用对象引用 | 对象可能半初始化 |
| `finalize` 中复活对象 | 已废弃，禁止使用 |

## 11. 对象头简述

HotSpot 对象内存布局（64 位）：

```
| Mark Word (8B) | Klass Pointer (4B/8B) | 实例数据 | 对齐填充 |
```

- Mark Word 存 hash、GC 年龄、锁标志。
- 开启压缩指针（`-XX:+UseCompressedOops`，默认）引用为 4 字节。
- 对象大小按 8 字节对齐。

`new Object()` 通常占 16 字节。字段重排（long/double 优先，引用与 int 分组）可减少 padding。
