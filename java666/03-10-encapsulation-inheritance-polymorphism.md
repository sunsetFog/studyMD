# 10 · 封装、继承与多态

## 1. 封装（Encapsulation）

**目标**：隐藏实现细节，只暴露稳定契约，保护不变量。

```java
public class Account {
    private BigDecimal balance = BigDecimal.ZERO;   // 私有状态

    public BigDecimal getBalance() { return balance; }   // 只读

    public void deposit(BigDecimal amount) {
        if (amount == null || amount.signum() <= 0) {
            throw new IllegalArgumentException("金额必须为正");
        }
        balance = balance.add(amount);                   // 唯一修改入口
    }

    public void withdraw(BigDecimal amount) {
        if (amount.compareTo(balance) > 0) {
            throw new IllegalStateException("余额不足");
        }
        balance = balance.subtract(amount);
    }
}
```

收益：

- 可在 setter 中做校验，保证对象始终合法。
- 内部结构可替换（如 balance 改为分存储），不影响调用方。
- 便于加日志、审计、懒加载。

### 防御性拷贝

```java
// 错误：暴露内部可变集合
public List<Item> getItems() { return items; }

// 正确
public List<Item> getItems() { return List.copyOf(items); }   // 不可变副本
// 或
public List<Item> getItems() { return new ArrayList<>(items); }
```

## 2. 继承（Inheritance）

```java
public class Animal {
    protected String name;

    public Animal(String name) { this.name = name; }

    public void speak() { System.out.println(name + " makes a sound"); }
}

public class Dog extends Animal {
    private String breed;

    public Dog(String name, String breed) {
        super(name);              // 必须第一行
        this.breed = breed;
    }

    @Override
    public void speak() {
        super.speak();            // 可选：复用父类逻辑
        System.out.println(name + " barks");
    }
}
```

规则：

- Java **单继承**，`extends` 一个类。
- 子类构造器必须调用父类构造器（显式 `super(...)` 或隐式无参）；父类无无参构造器时必须显式调用。
- `protected` 成员对子类可见（即使不同包）。
- 子类不能降低方法可见性。
- `final` 类不可继承，`final` 方法不可重写。

### 继承的代价

- **强耦合**：父类改动波及所有子类（脆弱基类问题）。
- 破坏封装：子类依赖父类实现细节。
- 方法覆盖可能违反父类契约（里氏替换）。

> 设计原则：**组合优于继承**（Effective Java 第 18 条）。

```java
// 差：为了复用代码而继承
class MyList extends ArrayList<String> { }

// 好：包装（装饰器）
class MyList {
    private final List<String> delegate;
    MyList(List<String> delegate) { this.delegate = delegate; }
}
```

## 3. 多态（Polymorphism）

**同一接口，不同实现**；调用时由**运行期类型**决定执行哪个方法。

```java
Animal a = new Dog("旺财", "柴犬");
a.speak();          // 调用 Dog.speak()

Animal[] animals = { new Dog("d", "x"), new Cat("c") };
for (Animal an : animals) an.speak();     // 多态分发
```

### 编译期 vs 运行期

| 项 | 决定时机 | 依据 |
|---|---|---|
| 方法重载（overload） | 编译期 | 引用类型 + 参数静态类型 |
| 方法重写（override） | 运行期 | 实际对象类型（虚方法分派） |
| 字段访问 | 编译期 | 引用类型（字段不参与多态） |
| 静态方法 | 编译期 | 引用类型（隐藏，非重写） |

```java
class A { String name = "A"; static void s() { print("A.s"); } void m() { print("A.m"); } }
class B extends A { String name = "B"; static void s() { print("B.s"); } void m() { print("B.m"); } }

A a = new B();
System.out.println(a.name);   // A（字段看引用类型）
a.s();                        // A.s（静态方法看引用类型）
a.m();                        // B.m（实例方法看实际类型）
```

## 4. 重载 vs 重写

| 维度 | 重载 Overload | 重写 Override |
|---|---|---|
| 签名 | 方法名同，参数不同 | 签名完全相同（协变返回除外） |
| 位置 | 同类（或父子类） | 子类 |
| 绑定 | 编译期 | 运行期 |
| 返回类型 | 任意 | 相同或子类型（协变） |
| 异常 | 任意 | 不能抛出更宽的受检异常 |
| 可见性 | 任意 | 不能更严格 |
| 注解 | 无 | `@Override`（强烈建议） |

重载陷阱：

```java
void f(int x) { }
void f(long x) { }
f(1);          // 选 f(int)
f(1L);         // 选 f(long)

void g(Object o) { }
void g(String s) { }
g(null);       // 编译错误：ambiguous（两个都匹配且无更具体？实际选 String，因更具体）
```

> 更精确规则：选**最具体**的适用方法；若存在两个互不包含的类型则编译错误。

```java
void h(Integer i) {}
void h(Long l) {}
h(null);   // 编译错误：Integer 与 Long 无继承关系，ambiguous
```

## 5. super 关键字

| 用法 | 说明 |
|---|---|
| `super(...)` | 调用父类构造器，必须第一行 |
| `super.method()` | 调用父类被重写的方法 |
| `super.field` | 访问父类字段（被子类隐藏时） |

## 6. final 关键字

| 位置 | 含义 |
|---|---|
| `final class` | 不可继承（String、Integer、record） |
| `final method` | 不可重写（模板方法模式常用） |
| `final field` | 只能赋值一次（基本类型值不变，引用不可换） |
| `final 局部变量` | 只能赋值一次，Lambda 捕获要求 effectively final |
| `final 参数` | 方法内不可重新赋值 |

```java
final List<String> list = new ArrayList<>();
list.add("a");          // 合法：引用不变，内容可变
// list = new ArrayList<>();  // 编译错误
```

`final` 的**内存语义**：正确构造的 final 字段在构造器完成后对其他线程可见（无需同步），这是安全发布不可变对象的基础。

## 7. 里氏替换原则（LSP）

子类必须能替换父类而不破坏程序正确性：

- 前置条件不能加强（不能要求更多）。
- 后置条件不能减弱（不能返回更少）。
- 不抛出父类未声明的受检异常。
- 不修改父类的语义约定。

经典反例：`Square extends Rectangle`，设置宽高会互相影响，破坏面积计算预期。

## 8. 模板方法模式（继承的正当用法）

```java
public abstract class AbstractImporter {
    public final void importData(Path path) {    // final 固定流程
        validate(path);
        List<String> lines = read(path);
        List<Record> records = parse(lines);
        save(records);
        afterImport();
    }
    protected abstract List<String> read(Path path);
    protected abstract List<Record> parse(List<String> lines);
    protected void validate(Path path) { /* 默认空实现 */ }
    protected void afterImport() { }
}
```

## 9. 常见坑

1. 在构造器中调用可重写方法。
2. 用继承复用代码而非表达「is-a」。
3. 重写 `equals` 却不重写 `hashCode`。
4. 子类字段与父类字段同名（隐藏，易混淆）。
5. 静态方法「重写」其实只是隐藏。
6. 重写时缩小可见性 → 编译错误。
7. 重写抛出更宽的受检异常 → 编译错误。
8. `@Override` 不加，参数写错变成重载，运行时行为不符预期。

## 10. 设计检查清单

- 字段是否全部 private？集合返回是否防御性拷贝？
- 是否存在 3 层以上继承？考虑改为组合。
- 父类方法是否 all `final` 或 `abstract`（避免中间态）？
- 多态分支是否用 `instanceof` 硬编码？考虑访问者模式或 sealed + switch。
- 子类是否真正满足 is-a 语义？
