# 05 · 方法

## 一、方法定义

```java
修饰符 返回类型 方法名(参数列表) [throws 异常] {
    // 方法体
    return 返回值;   // void 可省略或 return;
}

public static int max(int a, int b) {
    return a >= b ? a : b;
}

public void process() throws IOException { ... }   // 声明受检异常
```

## 二、传值语义（Java 只有值传递 —— 高频面试题）

```java
void change(int x) { x = 100; }          // 基本类型：拷贝值，外部不变

void changeArray(int[] arr) { arr[0] = 100; }   // 数组/对象：拷贝的是引用（地址）
                                                 // 通过地址改对象内容 → 外部可见！
void reassign(int[] arr) { arr = new int[]{9,9}; }  // 重新赋引用 → 只改拷贝，外部不变
```

**结论**：Java 传的是「值的拷贝」。基本类型拷贝值，引用类型拷贝引用——所以**方法内不能让外部引用指向新对象，但能修改对象内部状态**。

## 三、重载（Overload）

```java
public class Printer {
    void print(String s) { }
    void print(int i) { }
    void print(String s, int times) { }
    // 规则：方法名相同，参数列表不同（个数/类型/顺序）
    // 与返回值无关！仅返回值不同不能重载
}
```

**重载解析顺序**：精确匹配 → 更宽的自动类型转换（int→long）→ 装箱（Integer）→ 可变参数。

```java
print(1);          // 匹配 print(int)
print(1, 2);       // 若只有 print(long, long) → 自动加宽匹配
```

## 四、可变参数

```java
public static int sum(int... nums) {      // 本质是 int[] 参数
    int total = 0;
    for (int n : nums) total += n;
    return total;
}

sum(); sum(1); sum(1, 2, 3);
sum(new int[]{1, 2});

// 重载冲突注意：sum(int... ) 与 sum(int, int...) 会让调用歧义
// 可变参数必须是最后一个参数，且最多一个
```

## 五、递归

```java
// 阶乘
long fact(int n) {
    if (n <= 1) return 1;           // 1. 递归出口（必须有！）
    return n * fact(n - 1);         // 2. 问题规模缩小
}

// 斐波那契：朴素递归 O(2^n) → 备忘录/迭代 O(n)
long fib(int n, Map<Integer, Long> memo) {
    if (n <= 2) return 1;
    return memo.computeIfAbsent(n,
            k -> fib(k - 1, memo) + fib(k - 2, memo));
}
```

**栈溢出**：递归深度过大 → StackOverflowError。深度可预估时改迭代；真实分层数据（如树）递归没问题。

## 六、方法设计规范

```java
// 1. 单一职责 + 动词命名
calculateTax() / isValid() / buildUser()

// 2. 参数不宜过多（>4 个改用参数对象）
// 坏
void createUser(String name, String phone, String email, int age, String city, ...)
// 好
void createUser(UserCreateCommand cmd)

// 3. 布尔参数拆分方法（可读性）
// 坏：render(true)   好：renderAsHtml() / renderAsPlainText()

// 4. 卫语句提前返回，避免深层嵌套（03 篇）

// 5. 返回空集合而不是 null
List<User> findUsers() { return Collections.emptyList(); }
```

## 七、构造器与方法重载的组合

```java
public class Rectangle {
    private final int width, height;

    public Rectangle(int side) {          // 重载构造器 + this 链式调用
        this(side, side);
    }
    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }
}
```

## 常见坑

- **坑 1**：以为 Java 有引用传递 → 只有值传递；方法内 reassign 外部不可见
- **坑 2**：重载时 Integer 与 int 参数歧义 → null 调用编译错误
- **坑 3**：可变参数方法传 null → nums 本身为 null 而非空数组，遍历 NPE
- **坑 4**：递归没出口或出口不可达 → StackOverflowError
- **坑 5**：方法返回 null 表示「没找到」→ 调用方 NPE；返回 Optional 或空集合

## Checklist

- [ ] 能准确解释「Java 只有值传递」及两种拷贝差异
- [ ] 重载只看参数列表，不看返回值
- [ ] 递归有出口，深度大改迭代
- [ ] 方法参数 ≤ 4，返回空集合/Optional 而非 null
