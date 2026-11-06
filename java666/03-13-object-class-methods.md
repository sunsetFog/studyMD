# 13 · Object 类方法详解

## 1. Object 的 11 个方法

| 方法 | 说明 | 是否推荐重写 |
|---|---|---|
| `equals(Object)` | 逻辑相等 | ✔ 必须 |
| `hashCode()` | 哈希值 | ✔ 与 equals 一起 |
| `toString()` | 字符串表示 | ✔ 建议 |
| `clone()` | 浅拷贝 | △ 谨慎，优先拷贝构造器 |
| `finalize()` | 回收前回调 | ✘ 已废弃（Java 9 deprecated, 18 移除相关） |
| `getClass()` | 运行时类 | ✘ final |
| `notify/notifyAll/wait` | 线程通信 | ✘ final，改用 JUC |
| `registerNatives` | 私有 native | ✘ |

## 2. equals 契约

重写必须满足：

1. **自反性**：`x.equals(x) == true`
2. **对称性**：`x.equals(y) == y.equals(x)`
3. **传递性**：`x=y, y=z → x=z`
4. **一致性**：多次调用结果不变（不依赖可变状态）
5. **非空性**：`x.equals(null) == false`

### 标准实现

```java
public final class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                    // 1. 同一引用，快速路径
        if (!(o instanceof Point p)) return false;     // 2. 类型检查 + 模式匹配（Java 16+）
        return x == p.x && y == p.y;                   // 3. 逐字段比较
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);                     // 或 31 * x + y
    }
}
```

为什么用 `instanceof` 而不是 `getClass()`：

- `instanceof` 允许子类相等（需保证对称性，通常配合 final 类）。
- `getClass()` 严格同类才相等，适合有继承且语义要求严格的场景。

### 对称性破坏案例

```java
class CaseInsensitiveString {
    private final String s;
    @Override public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString c) return s.equalsIgnoreCase(c.s);
        if (o instanceof String str) return s.equalsIgnoreCase(str);   // 单方面相等
        return false;
    }
}
// "abc".equals(ciStr) 为 false，但 ciStr.equals("abc") 为 true → 违反对称性
```

## 3. hashCode 契约

1. 同一对象多次调用返回相同值（前提：equals 用到的字段未变）。
2. `equals` 相等 → `hashCode` 必须相等。
3. `equals` 不等 → `hashCode` **尽量**不同（影响哈希表性能，不要求）。

**只重写 equals 不重写 hashCode 的后果**：两个相等对象哈希不同，HashMap 中存入两个「重复」键，`get` 取不到。

```java
Map<Point, String> map = new HashMap<>();
map.put(new Point(1, 2), "a");
map.get(new Point(1, 2));     // 若没重写 hashCode → null
```

### 高质量 hashCode

```java
// 方式 1：Objects.hash（简洁，有数组创建开销）
@Override public int hashCode() { return Objects.hash(x, y); }

// 方式 2：手写（性能好，IntelliJ 生成风格）
@Override public int hashCode() {
    int result = Integer.hashCode(x);
    result = 31 * result + Integer.hashCode(y);
    return result;
}

// 方式 3：record 自动生成
public record Point(int x, int y) { }
```

为什么用 31：奇素数，`31 * i == (i << 5) - i` 可被 JIT 优化，且分布较好。

### 可变字段做 key 的灾难

```java
Map<List<String>, String> map = new HashMap<>();
List<String> key = new ArrayList<>(List.of("a"));
map.put(key, "v");
key.add("b");                 // 修改了参与 hashCode 的字段
map.get(key);                 // null —— 哈希变了，落在不同桶
```

结论：**Map 的 key 必须不可变**（String、Integer、record、不可变值对象）。

## 4. toString

```java
@Override
public String toString() {
    return "Point{x=%d, y=%d}".formatted(x, y);
}
```

要求：

- 包含类名与所有关键字段，便于日志排查。
- 不要输出敏感信息（密码、token、身份证）。
- 不要在 toString 中做重计算或远程调用（日志量大时性能灾难）。
- 双向引用要防止无限递归（A 引用 B，B 引用 A）。

```java
// 防递归
@Override public String toString() {
    return "Dept{id=" + id + ", name='" + name + "'}";   // 不打印 children
}
```

## 5. clone 与拷贝

### 浅拷贝 vs 深拷贝

```java
public class Order implements Cloneable {
    private List<Item> items;

    @Override
    public Order clone() {
        try {
            Order copy = (Order) super.clone();      // 浅拷贝：items 共享
            copy.items = new ArrayList<>(this.items); // 深一层
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);
        }
    }
}
```

`clone` 的问题：

- `Cloneable` 是标记接口，不实现就抛 `CloneNotSupportedException`。
- 绕过构造器，破坏不变量。
- 浅拷贝容易漏，深拷贝手写繁琐。
- 对 final 字段不友好。

**推荐替代**：

```java
// 1. 拷贝构造器
public Order(Order other) {
    this.id = other.id;
    this.items = new ArrayList<>(other.items);
}

// 2. 静态工厂
public static Order copyOf(Order o) { return new Order(o); }

// 3. 序列化/JSON 深拷贝（简单但慢）
// 4. record 天然适合复制：new Point(p.x(), p.y())
```

## 6. getClass

```java
obj.getClass()               // 运行时类
obj.getClass().getName()     // 全限定名
obj.getClass().getSimpleName()
```

`getClass()` 是 `final native`，不能重写，常用于：

- 日志中打印真实类型。
- 反射入口。
- 严格 equals 实现。

注意：代理对象（Spring AOP）的 `getClass()` 返回代理类，不是原始类；需要 `AopUtils.getTargetClass()`。

## 7. finalize（已废弃）

```java
// 不要这样写
@Override
protected void finalize() throws Throwable {
    resource.close();
}
```

问题：

- 执行时机不确定，可能永不执行。
- 拖慢 GC（需要额外队列与二次标记）。
- 异常被吞掉。
- 可被「复活」。

替代：`try-with-resources`、`AutoCloseable`、`Cleaner`（Java 9+）。

```java
public class Resource implements AutoCloseable {
    private static final Cleaner CLEANER = Cleaner.create();
    private final Cleaner.Cleanable cleanable;

    public Resource() {
        this.cleanable = CLEANER.register(this, () -> System.out.println("cleanup"));
    }

    @Override public void close() { cleanable.clean(); }
}
```

## 8. wait / notify

```java
synchronized (lock) {
    while (!condition) lock.wait();   // 必须 while，防虚假唤醒
    // 处理
}

synchronized (lock) {
    condition = true;
    lock.notifyAll();
}
```

现代替代：`Lock + Condition`、`BlockingQueue`、`CompletableFuture`。但面试常考，需掌握原理（见并发篇）。

## 9. 完整值对象模板

```java
public final class UserId implements Comparable<UserId> {
    private final long value;

    private UserId(long value) {
        if (value <= 0) throw new IllegalArgumentException("id must be positive");
        this.value = value;
    }

    public static UserId of(long value) { return new UserId(value); }

    public long value() { return value; }

    @Override public boolean equals(Object o) {
        return o instanceof UserId other && value == other.value;
    }

    @Override public int hashCode() { return Long.hashCode(value); }

    @Override public String toString() { return "UserId(" + value + ")"; }

    @Override public int compareTo(UserId o) { return Long.compare(value, o.value); }
}
```

好处：类型安全（不会把 orderId 传给 userId）、不可变、可作 Map key。

## 10. 速查表

| 场景 | 做法 |
|---|---|
| 判断相等 | `Objects.equals(a, b)` |
| 生成 hash | `Objects.hash(...)` |
| 打印 | `Objects.toString(obj, "default")` |
| 非空校验 | `Objects.requireNonNull(x, "x")` |
| 比较大小 | `Objects.compare(a, b, cmp)` 或 `Comparator` |
| 数组内容比较 | `Arrays.equals` / `Arrays.deepEquals` |
| BigDecimal 比较 | `compareTo == 0`（不是 equals） |
