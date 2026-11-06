# 11 · Lambda 与函数式接口

## 一、Lambda 语法

```java
// 无参
Runnable r = () -> System.out.println("run");

// 单参（括号可省）
Consumer<String> c = s -> System.out.println(s);

// 多参
Comparator<Integer> cmp = (a, b) -> a - b;      // 等价 (Integer a, Integer b)

// 带代码块与 return
Function<Integer, Integer> square = x -> {
    int result = x * x;
    return result;
};

// 方法引用：已有方法直接当 Lambda 用（更简洁）
Function<String, Integer> len = String::length;
Supplier<List<String>> maker = ArrayList::new;
BiFunction<String, String, Boolean> eq = String::equals;
list.forEach(System.out::println);
```

**方法引用四种**：`类::静态方法`、`对象::实例方法`、`类::实例方法`（首参为接收者）、`类::new` 构造引用。

## 二、函数式接口 = Lambda 的目标类型

Lambda 的类型由**目标接口**决定：

```java
Object o = (Runnable) () -> { };       // 必须显式转，Lambda 不能直接赋 Object
Runnable r = () -> { };                // 赋给函数式接口 → 有类型了
```

### JDK 内置函数式接口（java.util.function）

| 接口 | 签名 | 用途示例 |
|---|---|---|
| Supplier\<T\> | T get() | 惰性求值、工厂 |
| Consumer\<T\> | void accept(T) | forEach、副作用 |
| BiConsumer\<T,U\> | accept(T,U) | Map.forEach |
| Function\<T,R\> | R apply(T) | map 转换 |
| BiFunction\<T,U,R\> | apply(T,U) | 双参转换 |
| Predicate\<T\> | boolean test(T) | filter 条件 |
| UnaryOperator\<T\> | T apply(T) | 同类型变换 |

带变体的记忆法：`Bi`（两参）、`Int/Long/Double` 前缀（避免装箱）、`ToXxx`（指定返回）。

### Comparator 组合器

```java
Comparator<User> cmp = Comparator
        .comparing(User::getAge)
        .thenComparing(User::getName)
        .reversed();

list.sort(cmp);
```

### Predicate / Function 组合

```java
Predicate<String> isLong = s -> s.length() > 5;
Predicate<String> hasDigit = s -> s.matches(".*\\d.*");
isLong.and(hasDigit);        // 与
isLong.or(hasDigit);         // 或
isLong.negate();             // 非

Function<Integer, Integer> f = x -> x + 1;
f.andThen(x -> x * 2);       // 先 f 后 g：(x+1)*2
f.compose(x -> x * 2);       // 先 g 后 f：x*2+1
Function.identity();         // 恒等
```

## 三、自定义函数式接口

```java
@FunctionalInterface           // 建议加：编译期保证单一抽象方法
public interface RetryAction<T> {
    T execute() throws Exception;
}
```

能用 JDK 内置就不要自造——**类型可读性与生态兼容**。

## 四、实战模式

```java
// 1. 策略注入
list.sort((a, b) -> a.length() - b.length());
list.sort(Comparator.comparingInt(String::length));   // 防 int 溢出写法更佳

// 2. 惰性求值（避免不必要开销）
logger.debug("cost {}", expensive());     // 坏：总是执行
logger.debug("cost {}", () -> expensive());  // 好重载版：级别不满足才执行

// 3. 回调
void processAsync(Runnable onSuccess, Consumer<Exception> onError) { }

// 4. 工厂
Map<String, Supplier<Shape>> factories = Map.of(
        "circle", Circle::new, "square", Square::new);
Shape s = factories.get(type).get();
```

## 五、作用域与限制

```java
int base = 10;
Function<Integer, Integer> f = x -> x + base;   // 捕获 effectively final 局部变量
// base = 20;   → 编译错误：Lambda 捕获的变量必须事实最终（不重新赋值）

// this：Lambda 里 this 指外部类实例（匿名内部类里 this 指匿名类本身）
// 局部变量必须 effectively final 的原因：Lambda 可能延迟执行/跨线程，副本快照
```

## 六、Lambda vs 匿名内部类

| 维度 | Lambda | 匿名内部类 |
|---|---|---|
| 只能实现函数式接口 | 是 | 可实现多方法接口/继承类 |
| this | 外部实例 | 匿名类自身 |
| 性能 | invokedynamic 惰性生成 | 编译即生成 class |
| 可读性（多方法接口） | — | 唯一选择 |

## 常见坑

- **坑 1**：Lambda 捕获可变计数器做并行处理 → 线程不安全；用 Stream reduce / Atomic
- **坑 2**：`(a, b) -> a - b` 减法比较溢出 → `Comparator.comparingInt` / `Integer.compare`
- **坑 3**：方法引用到重载方法产生歧义 → 显式类型标注参数
- **坑 4**：Lambda 中修改外部集合同时遍历 → 并发修改；预期副作用逻辑用普通循环更清晰
- **坑 5**：把所有循环无脑改 Stream → 简单遍历 for-each 更高效可读

## Checklist

- [ ] 四大函数式接口 + 组合器方法熟练
- [ ] 方法引用替代简单 Lambda
- [ ] effectively final 理解并遵守
- [ ] 排序用 Comparator 组合器不用手写减法
