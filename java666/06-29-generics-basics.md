# 29 · 泛型基础

## 1. 为什么需要泛型

```java
// 没有泛型的时代（Java 5 之前）
List list = new ArrayList();
list.add("a");
list.add(1);
String s = (String) list.get(1);    // ClassCastException，运行时才暴露
```

泛型带来：

1. **编译期类型检查**（错误提前）。
2. **消除强制转换**。
3. **代码复用**（一套逻辑支持多种类型）。
4. **更好的可读性**（API 自文档化）。

## 2. 泛型类

```java
public class Box<T> {
    private T value;

    public void set(T value) { this.value = value; }
    public T get() { return value; }

    public <U> Box<U> map(Function<T, U> f) {      // 泛型方法
        Box<U> box = new Box<>();
        box.set(f.apply(value));
        return box;
    }
}

Box<String> b = new Box<>();
b.set("hello");
String s = b.get();
```

多类型参数：

```java
public class Pair<K, V> {
    private final K key;
    private final V value;
    public Pair(K key, V value) { this.key = key; this.value = value; }
    public K key() { return key; }
    public V value() { return value; }
}
```

## 3. 泛型接口

```java
public interface Repository<T, ID> {
    T findById(ID id);
    void save(T entity);
    List<T> findAll();
}

public class UserRepository implements Repository<User, Long> {
    @Override public User findById(Long id) { ... }
    @Override public void save(User u) { ... }
    @Override public List<User> findAll() { ... }
}
```

实现时也可以保留泛型：

```java
public class InMemoryRepository<T, ID> implements Repository<T, ID> {
    private final Map<ID, T> store = new HashMap<>();
    @Override public T findById(ID id) { return store.get(id); }
    @Override public void save(T entity) { }
    @Override public List<T> findAll() { return new ArrayList<>(store.values()); }
}
```

## 4. 泛型方法

```java
public static <T> T firstOrNull(List<T> list) {
    return list == null || list.isEmpty() ? null : list.get(0);
}

public static <T extends Comparable<T>> T max(List<T> list) {
    T max = list.get(0);
    for (T t : list) if (t.compareTo(max) > 0) max = t;
    return max;
}
```

要点：

- 类型参数声明在**返回类型之前**：`<T> T method(...)`。
- 泛型方法可以独立于类的泛型参数。
- 静态方法不能使用类的类型参数（类参数属于实例）：`static <T> void f(T t)` 是合法的泛型方法。

## 5. 类型推断

```java
Map<String, List<User>> map = new HashMap<>();     // 菱形运算符（Java 7+）
List<String> list = Collections.emptyList();       // 目标类型推断

// 方法推断
T first = firstOrNull(list);

// 显式指定（推断失败时）
Collections.<String>emptyList();
this.<String>method();

// 匿名类不能用菱形（Java 9+ 可以）
Comparator<String> cmp = new Comparator<>() { ... };   // Java 9+
```

### 推断失败场景

```java
// 链式泛型调用，编译器无法推断
List<String> l = Collections.emptyList().stream().toList();   // 可能需要显式类型

// var + 泛型（Java 10+）
var list = new ArrayList<String>();     // 推断为 ArrayList<String>
var map = Map.of("a", 1);               // Map<String, Integer>
```

## 6. 泛型边界（Bounded Type）

```java
// 上界：T 必须是 Number 或其子类
public <T extends Number> double sum(List<T> list) {
    double s = 0;
    for (T t : list) s += t.doubleValue();   // 可调用 Number 的方法
    return s;
}

// 多边界：类必须在前，接口可多个
public <T extends Comparable<T> & Serializable> T pick(List<T> list) { ... }

// 无界等价于 extends Object
public <T> void f(T t) { }
```

边界的作用：

- 限定可接受的类型。
- 允许调用边界类型的方法。
- 影响类型擦除后的结果（擦除到第一个边界）。

## 7. 泛型与继承

```java
List<String> 不是 List<Object> 的子类型！

List<String>  ls = new ArrayList<>();
List<Object> lo = ls;           // 编译错误
```

原因：如果允许，就能往 `List<String>` 里塞 Integer，破坏类型安全。

但：

```java
Object[] arr = new String[3];    // 数组是协变的（运行期检查）
arr[0] = 1;                      // ArrayStoreException 运行期才报
```

> **数组协变，泛型不变**（invariant）。这是泛型更安全的原因，也是通配符存在的原因。

## 8. 泛型方法 vs 通配符

```java
// 通配符：适合「只读消费」的参数
public void printAll(List<?> list) {
    for (Object o : list) System.out.println(o);
}

// 泛型方法：需要类型关联时
public <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (T t : src) dest.add(t);
}
```

规则：

- 参数类型之间**有关系** → 泛型方法。
- 只需要「任意类型」→ 通配符。
- 返回值依赖类型参数 → 泛型方法。

## 9. 泛型的限制

```java
// 1. 不能 new 泛型数组
// T[] arr = new T[10];            // 编译错误
T[] arr = (T[]) new Object[10];    // 需要强转 + @SuppressWarnings

// 2. 不能对泛型类型使用 instanceof（除非无界通配）
// if (o instanceof List<String>)   // 编译错误
if (o instanceof List<?>) { }       // 合法

// 3. 不能 new 泛型实例
// T t = new T();                   // 只能通过 Supplier<T>

// 4. 不能有静态泛型字段
// static T value;                  // 编译错误

// 5. 不能 catch 泛型异常
// catch (T e) { }

// 6. 基本类型不能作类型参数
// List<int>  ✘     List<Integer> ✔

// 7. 不能重载仅泛型参数不同的方法
// void f(List<String> l);
// void f(List<Integer> l);         // 擦除后签名相同 → 编译错误
```

类型参数创建实例的正确方式：

```java
public <T> T create(Supplier<T> factory) {
    return factory.get();
}
User u = create(User::new);
```

## 10. 泛型与反射

```java
// 运行时拿不到 List<String> 的 String（擦除）
// 但可以拿到字段/方法签名中的泛型信息（存在 Signature 属性里）

Field f = MyClass.class.getDeclaredField("users");
ParameterizedType t = (ParameterizedType) f.getGenericType();
Type arg = t.getActualTypeArguments()[0];   // class java.lang.String
```

Jackson 的 `TypeReference`、Spring 的 `ResolvableType` 都依赖这个能力：

```java
new TypeReference<List<User>>() { };   // 匿名子类保留泛型信息
```

## 11. 实战：泛型结果封装

```java
public record Result<T>(int code, String message, T data) {
    public static <T> Result<T> ok(T data) { return new Result<>(0, "ok", data); }
    public static <T> Result<T> fail(int code, String msg) { return new Result<>(code, msg, null); }

    public boolean isSuccess() { return code == 0; }

    public <R> Result<R> map(Function<T, R> f) {
        return isSuccess() ? Result.ok(f.apply(data)) : Result.fail(code, message);
    }
}

Result<User> r = Result.ok(new User(...));
Result<String> name = r.map(User::getName);
```

## 12. 常见坑

1. `List<Object>` 接收 `List<String>`（不变性）。
2. 泛型数组创建（`(T[]) new Object[n]` 有隐患，见 30 篇）。
3. 静态方法使用类类型参数。
4. 原始类型（raw type）`List list` 混用（丢失类型检查，只应在遗留代码）。
5. `Class<T>` 与 `T` 混淆（前者是类型令牌，用于反射/反序列化）。
6. 通配符用于返回值（`List<?>` 返回值调用方难用）。
7. 认为泛型有运行时开销（实际擦除后与普通代码一致）。

## 13. 速查

| 写法 | 含义 |
|---|---|
| `<T>` | 无界类型参数 |
| `<T extends X>` | 上界 |
| `<?>` | 无界通配符 |
| `<? extends T>` | 上界通配（只读，生产者） |
| `<? super T>` | 下界通配（只写，消费者） |
| `<>` | 菱形推断 |
| `Class<T>` | 类型令牌 |
| `T...` | 泛型可变参数（有警告） |
