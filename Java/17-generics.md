# 17 · 泛型

## 一、泛型类与接口

```java
public class Box<T> {
    private T value;
    public Box(T value) { this.value = value; }
    public T get() { return value; }
    public void set(T value) { this.value = value; }
}

Box<String> box = new Box<>("hello");    // 编译期类型安全，无需强转
String s = box.get();

// 多类型参数
public class Pair<K, V> {
    private final K key;
    private final V value;
    // 构造器...
}
Pair<String, Integer> p = new Pair<>("age", 18);

// 类型参数命名约定：T 类型、E 元素、K/V 键值、R 返回、N 数字
```

## 二、泛型方法

```java
// 类型参数在方法级声明（返回类型前）
public static <T> T firstOf(List<T> list) {
    return list.isEmpty() ? null : list.get(0);
}

// <T extends 上界>：约束能力
public static <T extends Comparable<T>> T max(List<T> list) {
    T max = list.get(0);
    for (T item : list) {
        if (item.compareTo(max) > 0) max = item;
    }
    return max;
}

// 多上界：& 连接
public static <T extends Number & Comparable<T>> void process(T t) { }

firstOf(List.of("a", "b"));      // 类型推断，无需显式
this.<String>firstOf(list);      // 显式指定（歧义时）
```

## 三、通配符（?）—— 生成与消费

```java
// 无界：?
static int size(List<?> list) { return list.size(); }    // 只读长度

// 上界通配符：? extends —— 产者（Producer），只能读出
static double sum(List<? extends Number> nums) {
    double t = 0;
    for (Number n : nums) t += n.doubleValue();   // 读 OK
    // nums.add(1);   编译错误！不能写入（不知道确切类型）
    return t;
}
sum(List.of(1, 2, 3));       // List<Integer> ✓
sum(List.of(1.5, 2.5));      // List<Double> ✓

// 下界通配符：? super —— 消费者（Consumer），只能写入
static void fill(List<? super Integer> sink) {
    sink.add(1);                  // 写 OK（Integer 及其子类都能装）
    // Object o = sink.get(0);    读出只能当 Object
}
fill(new ArrayList<Number>());
fill(new ArrayList<Object>());
```

### PECS 原则（面试高频）

```
Producer Extends, Consumer Super
- 该结构给你"产出"数据（你读它）    → ? extends T
- 该结构为你"消费"数据（你写它）    → ? super T
- 既读又写                          → 精确类型 T，不用通配符
```

JDK 示例：`Collections.copy(List<? super T> dst, List<? extends T> src)` —— src 产出、dst 消费。

## 四、类型擦除（理解泛型的钥匙）

```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
a.getClass() == b.getClass();        // true！运行时都是 ArrayList

// 编译期检查 + 强转插入，运行时擦除为 Object（或上界）
// class Box<T> → class Box { Object value; }
```

**擦除带来的限制**：

```java
// 1. 不能 new T() / new T[] —— 类型参数编译后不存在
class Box<T> {
    // T t = new T();        ✗
    // T[] arr = new T[10];  ✗ → 用 ArrayList<T> 或 (T[]) new Object[10]
}

// 2. 不能对类型参数用 instanceof
// if (obj instanceof T)   ✗ → Java 21 之前不可；反射/传 Class<T> 绕过

// 3. 静态成员不能使用类的类型参数
class Box<T> {
    // static T shared;      ✗
}

// 4. 泛型不能是基本类型
// List<int> ✗ → List<Integer>（装箱）

// 5. 泛型数组创建受限
// new ArrayList<String>[10]  ✗
```

## 五、实战模式

```java
// 1. 泛型 DAO 基类
public abstract class BaseRepository<T, ID> {
    public abstract T findById(ID id);
    public abstract List<T> findAll();
}
public class UserRepository extends BaseRepository<User, Long> { }

// 2. 泛型 Builder
public class Builder<T> {
    private final Function<Map<String, Object>, T> factory;
    private final Map<String, Object> props = new HashMap<>();
    public Builder(Function<Map<String, Object>, T> f) { this.factory = f; }
    public <V> Builder<T> with(String key, V value) {
        props.put(key, value);
        return this;
    }
    public T build() { return factory.apply(props); }
}

// 3. JSON 反序列化带泛型（Jackson TypeReference）
Map<String, List<User>> data = mapper.readValue(json,
        new TypeReference<Map<String, List<User>>>() { });

// 4. 递归类型边界（自比较）
static <T extends Comparable<? super T>> T max(List<? extends T> list) { }
```

## 六、通配符冲突警告

```java
// 通配符类型的容器不能写入
List<? extends Number> nums = List.of(1, 2);
// nums.add(3.14);   ✗ 编译错误

// var 声明 + 通配符的可读性问题：返回类型避免通配符（API 设计）
// 坏：public List<? extends User> getUsers()    → 调用方无法往回放
// 好：public List<User> getUsers()
```

## 常见坑

- **坑 1**：`List<Object>` ≠ `List<String>` 的父类 → 需要通配符，直接赋值编译错
- **坑 2**：PECS 用反 → extends 里写入编译报错
- **坑 3**：泛型类的 Class<T> 想直接 new T() → 擦除不可行；构造器传 Class<T> 反射创建
- **坑 4**：`(T[]) new Object[n]` 泛型数组强转 → 调用方 ClassCastException（堆污染），集合替代
- **坑 5**：Map<String, List<Integer>> JSON 反序列化成 Map<String, List> → 用 TypeReference
- **坑 6**：API 返回类型带 `? extends` → 调用方被锁死，返回精确类型

## Checklist

- [ ] PECS 记牢并能举例
- [ ] 理解擦除及其五条限制
- [ ] 泛型 DAO/Builder 模式会用
- [ ] 对外 API 返回精确类型，通配符只在方法参数
