# 55 · Lambda 与函数式接口

## 1. Lambda 语法

```java
// (参数) -> 表达式
Runnable r = () -> System.out.println("hi");
Comparator<String> c = (a, b) -> a.length() - b.length();
Function<String, Integer> f = s -> s.length();
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

// 带类型
Comparator<String> c2 = (String a, String b) -> a.compareTo(b);

// 代码块 + return
Function<String, String> trim = s -> {
    String t = s.trim();
    return t.isEmpty() ? null : t;
};
```

## 2. 函数式接口（SAM）

只有一个抽象方法的接口。

```java
@FunctionalInterface
public interface Transformer<T, R> {
    R apply(T t);                       // 唯一抽象方法
    default <V> Transformer<T, V> andThen(Transformer<R, V> next) {
        return t -> next.apply(this.apply(t));
    }
}
```

`@FunctionalInterface` 让编译器校验（0 或 >1 个抽象方法时报错）。

## 3. 四大内置接口

| 接口 | 签名 | 用途 |
|---|---|---|
| `Function<T,R>` | `R apply(T)` | 转换 |
| `Consumer<T>` | `void accept(T)` | 消费 |
| `Supplier<T>` | `T get()` | 生产 |
| `Predicate<T>` | `boolean test(T)` | 判断 |

### 扩展

| 接口 | 签名 |
|---|---|
| `BiFunction<T,U,R>` | `R apply(T,U)` |
| `UnaryOperator<T>` | `T apply(T)`（Function 特化） |
| `BinaryOperator<T>` | `T apply(T,T)` |
| `BiConsumer<T,U>` | `void accept(T,U)` |
| `BiPredicate<T,U>` | `boolean test(T,U)` |
| `IntFunction<R>` / `ToIntFunction<T>` | 基本类型特化 |
| `IntPredicate` / `IntSupplier` / `IntConsumer` | 避免装箱 |
| `IntBinaryOperator` | `int applyAsInt(int,int)` |

### 默认方法

```java
Function<Integer, Integer> f = x -> x + 1;
Function<Integer, Integer> g = x -> x * 2;

f.andThen(g).apply(1);      // g(f(1)) = 4
f.compose(g).apply(1);      // f(g(1)) = 3
Function.identity();         // x -> x

Predicate<String> notEmpty = s -> !s.isEmpty();
Predicate<String> shortEnough = s -> s.length() < 10;
notEmpty.and(shortEnough).test("abc");
notEmpty.or(shortEnough).test("");
notEmpty.negate().test("");

Consumer<String> c1 = System.out::print;
c1.andThen(System.out::println).accept("hi");
```

## 4. 方法引用

| 类型 | 语法 | 等价 Lambda |
|---|---|---|
| 静态方法 | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| 实例方法（特定对象） | `System.out::println` | `x -> System.out.println(x)` |
| 实例方法（任意对象） | `String::length` | `s -> s.length()` |
| 构造器 | `User::new` | `() -> new User()` |
| 数组构造器 | `int[]::new` | `n -> new int[n]` |

```java
list.forEach(System.out::println);
list.stream().map(String::toUpperCase).toList();
list.stream().map(User::getName).toList();
Supplier<List<String>> s = ArrayList::new;
Function<String, User> f = User::new;
```

> 方法引用更简洁，但只在「参数直接传递」时可用；有额外逻辑仍需 Lambda。

## 5. 变量捕获

```java
int base = 10;
Function<Integer, Integer> f = x -> x + base;    // 捕获 base（值拷贝）

// base = 20;    // 编译错误：必须 effectively final
```

规则：

- 只能捕获 **effectively final** 的局部变量。
- 成员变量/静态变量可直接访问（不要求 final）。
- `this` 指向外部类实例（Lambda 不是内部类，无自己的 this）。

```java
class Outer {
    int value = 1;
    Runnable r = () -> System.out.println(this.value);   // Outer.this
}
```

## 6. Lambda vs 匿名内部类

| 维度 | Lambda | 匿名内部类 |
|---|---|---|
| 适用 | 函数式接口 | 任意接口/抽象类 |
| 字节码 | `invokedynamic` | 生成 `Outer$1.class` |
| this | 外部类实例 | 匿名类实例 |
| 变量捕获 | effectively final | 同 |
| 性能 | 略优（可内联） | 有对象创建 |
| 可读性 | 简洁 | 冗长 |

```java
// Lambda
Runnable r1 = () -> System.out.println("hi");

// 匿名类
Runnable r2 = new Runnable() {
    @Override public void run() { System.out.println("hi"); }
};
```

## 7. 自定义函数式接口

```java
@FunctionalInterface
public interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;

    // 桥接为普通 Function，把受检异常转为非受检
    static <T, R> Function<T, R> unchecked(ThrowingFunction<T, R> f) {
        return t -> {
            try { return f.apply(t); }
            catch (Exception e) { throw new RuntimeException(e); }
        };
    }
}

// 使用
list.stream().map(ThrowingFunction.unchecked(this::parse)).toList();
```

## 8. 闭包与状态

```java
// 用数组/AtomicInteger 绕过 effectively final
AtomicInteger count = new AtomicInteger();
list.forEach(x -> count.incrementAndGet());

// 或改用 reduce/count
long c = list.stream().count();
```

> 不要为了「修改外部变量」而滥用可变容器，优先用 Stream 的聚合操作。

## 9. 高阶函数

```java
// 返回函数
public static <T> Predicate<T> not(Predicate<T> p) {
    return p.negate();
}

// 接受函数
public static <T, R> List<R> map(List<T> list, Function<T, R> f) {
    return list.stream().map(f).toList();
}

// 柯里化
public static <A, B, C> Function<A, Function<B, C>> curry(BiFunction<A, B, C> f) {
    return a -> b -> f.apply(a, b);
}

var add = curry((Integer a, Integer b) -> a + b);
add.apply(1).apply(2);      // 3
```

## 10. 组合与管道

```java
Function<String, String> trim = String::trim;
Function<String, String> lower = String::toLowerCase;
Function<String, String> normalize = trim.andThen(lower);

Predicate<String> valid = ((Predicate<String>) String::isEmpty).negate()
        .and(s -> s.length() <= 20);

List<String> result = inputs.stream()
        .map(normalize)
        .filter(valid)
        .toList();
```

## 11. 性能注意

- Lambda 首次调用有 `invokedynamic` 链接开销（一次性）。
- 非捕获 Lambda 通常被缓存为单例（无对象分配）。
- 捕获 Lambda 每次执行可能创建对象（JIT 逃逸分析可优化）。
- 基本类型流（`IntStream`）避免装箱。
- 方法引用与 Lambda 性能基本相同。

## 12. 常见坑

1. 试图修改捕获的局部变量（编译错误）。
2. 在 Lambda 中 `this` 指向外部类（以为是内部类）。
3. Lambda 中抛受检异常（函数式接口未声明 throws）。
4. 在循环中创建捕获循环变量的 Lambda：

```java
List<Runnable> tasks = new ArrayList<>();
for (int i = 0; i < 3; i++) {
    int finalI = i;                       // 必须
    tasks.add(() -> System.out.println(finalI));
}
// 若直接捕获 i → 编译错误（i 不是 effectively final）
```

5. 过度使用 Lambda 导致可读性下降（复杂逻辑应用方法）。
6. 在 Lambda 中做阻塞操作（Stream 并行时更危险）。
7. 误以为 Lambda 是「语法糖的匿名类」——字节码实现不同（`invokedynamic`）。
8. 在 `@Transactional` 方法内部用 Lambda 调用（仍受自调用限制）。

## 13. 速查

| 需求 | 接口 |
|---|---|
| 转换 | `Function<T,R>` |
| 消费 | `Consumer<T>` |
| 生产 | `Supplier<T>` |
| 判断 | `Predicate<T>` |
| 二元运算 | `BinaryOperator<T>` |
| 基本类型 | `IntFunction` / `ToIntFunction` / `IntPredicate` |
| 无参无返回 | `Runnable` |
| 有返回无参 | `Callable<V>` |
| 自定义 | `@FunctionalInterface` |
| 组合 | `andThen` / `compose` / `and` / `or` / `negate` |
