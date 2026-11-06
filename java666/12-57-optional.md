# 57 · Optional

## 1. 解决什么问题

Optional 是一个容器，明确表达「可能没有值」，减少 NPE 与 null 判断散落。

```java
// 传统
User u = findUser(id);
if (u != null) {
    String name = u.getName();
    if (name != null) { ... }
}

// Optional
findUser(id)
    .map(User::getName)
    .ifPresent(name -> ...);
```

> **设计初衷**：主要用作**返回值**，表示「可能无结果」。

## 2. 创建

```java
Optional<String> empty = Optional.empty();
Optional<String> ofValue = Optional.of("a");            // value 不能为 null（抛 NPE）
Optional<String> nullable = Optional.ofNullable(maybeNull);
```

## 3. 常用方法

### 判断与获取

```java
opt.isPresent();
opt.isEmpty();                     // Java 11+
opt.get();                         // 无值时抛 NoSuchElementException（尽量别用）
opt.orElse("default");             // 无值返回默认值（总会构造默认值）
opt.orElseGet(() -> compute());    // 无值时才计算（推荐）
opt.orElseThrow();                 // 无值抛 NoSuchElementException（Java 10+）
opt.orElseThrow(() -> new BizException("not found"));
```

### 转换与过滤

```java
opt.map(String::trim);                     // 有值才转换，返回 Optional
opt.flatMap(s -> findByName(s));           // 转换结果已是 Optional
opt.filter(s -> s.length() > 3);
opt.stream();                              // 0 或 1 个元素的 Stream（Java 9+）
```

### 消费

```java
opt.ifPresent(System.out::println);
opt.ifPresentOrElse(
    v -> System.out.println("有值: " + v),
    () -> System.out.println("无值")
);                                          // Java 9+
```

## 4. 正确用法

### 返回值

```java
public Optional<User> findByEmail(String email) {
    return Optional.ofNullable(repo.selectByEmail(email));
}
```

### 链式处理

```java
String city = findUser(id)
        .map(User::getAddress)
        .map(Address::getCity)
        .orElse("未知");
```

### 替代 null 检查

```java
// 旧
if (user != null && user.getAddress() != null) {
    return user.getAddress().getCity();
}
return "unknown";

// 新
return Optional.ofNullable(user)
        .map(User::getAddress)
        .map(Address::getCity)
        .orElse("unknown");
```

## 5. 反模式（禁止）

### 1. 用作字段/参数

```java
// ✘
private Optional<String> name;
public void set(Optional<String> name) { }

// ✔
private String name;                    // 字段用 null，配合 @Nullable
public void set(String name) { }
```

原因：Optional 不可序列化、增加内存开销、调用方必须再包一层。

### 2. 集合里放 Optional

```java
// ✘
List<Optional<User>> list;

// ✔
List<User> list;      // 用 null 或过滤掉
```

### 3. isPresent + get

```java
// ✘ 与 null 判断无异，没有价值
if (opt.isPresent()) {
    return opt.get().getName();
}

// ✔
return opt.map(User::getName).orElse(null);
```

### 4. 用 get() 而不检查

```java
// ✘ 可能抛 NoSuchElementException
opt.get();

// ✔
opt.orElseThrow(() -> new BizException("not found"));
```

### 5. orElse 中做重计算

```java
// ✘ 无论有没有值都会执行 createDefault()
opt.orElse(createDefault());

// ✔
opt.orElseGet(this::createDefault);
```

### 6. 嵌套 Optional

```java
// ✘
Optional<Optional<User>>;

// ✔ 用 flatMap
Optional<User> u = opt.flatMap(this::findUser);
```

### 7. 用于集合返回值

```java
// ✘ 空集合已经能表达「没有」
Optional<List<User>> findUsers();

// ✔
List<User> findUsers();      // 返回空列表
```

## 6. 与 Stream 结合

```java
// Optional 转 Stream（0 或 1 个元素）
opt.stream().forEach(System.out::println);

// 过滤 Stream 中的空 Optional
List<User> users = ids.stream()
        .map(this::findById)          // Stream<Optional<User>>
        .flatMap(Optional::stream)    // Stream<User>
        .toList();

// 找第一个非空
Optional<User> first = ids.stream()
        .map(this::findById)
        .flatMap(Optional::stream)
        .findFirst();
```

## 7. 实现原理

```java
public final class Optional<T> {
    private static final Optional<?> EMPTY = new Optional<>();
    private final T value;              // null 表示 empty

    private Optional() { this.value = null; }
    public static <T> Optional<T> empty() { return (Optional<T>) EMPTY; }
    public static <T> Optional<T> of(T value) {
        return new Optional<>(Objects.requireNonNull(value));
    }
}
```

- `Optional.empty()` 是单例。
- 每个 `of/ofNullable` 都创建新对象（轻微开销）。

## 8. 与 null 的对比

| 维度 | null | Optional |
|---|---|---|
| 表达「无值」 | 隐式（易忘检查） | 显式（类型强制） |
| 链式调用 | 需要层层判空 | `map` 链式 |
| 性能 | 零开销 | 对象开销 |
| 序列化 | 支持 | 不支持 |
| 适用位置 | 字段/参数/局部变量 | 返回值 |

## 9. 实战：多层嵌套取值

```java
// 取用户所在公司的部门经理的邮箱
String email = Optional.ofNullable(order)
        .map(Order::getUser)
        .map(User::getCompany)
        .map(Company::getDepartment)
        .map(Department::getManager)
        .map(Manager::getEmail)
        .orElse("n/a");
```

对比传统写法要 5 层 if。

## 10. 配合校验

```java
public User getUser(Long id) {
    return Optional.ofNullable(id)
            .flatMap(repo::findById)
            .orElseThrow(() -> new BizException("USER_NOT_FOUND", "用户不存在: " + id));
}
```

## 11. 性能说明

- Optional 是普通对象（16 字节左右），高频路径有开销。
- 热路径（每秒百万次）可用 null + 注解（`@Nullable`/`@NonNull`）。
- `OptionalInt/Long/Double` 避免装箱：

```java
OptionalInt oi = IntStream.of(1,2,3).max();
oi.orElse(0);
```

## 12. API 速查

| 需求 | 方法 |
|---|---|
| 创建 | `of` / `ofNullable` / `empty` |
| 判断 | `isPresent` / `isEmpty` |
| 取值 | `get`（慎用）/ `orElse` / `orElseGet` / `orElseThrow` |
| 转换 | `map` / `flatMap` / `filter` |
| 消费 | `ifPresent` / `ifPresentOrElse` |
| 转 Stream | `stream` |
| 基本类型 | `OptionalInt` / `OptionalLong` / `OptionalDouble` |
| 组合 | `or`（Java 9+，返回另一个 Optional） |

```java
// Java 9+ 的 or
Optional<User> u = findCached(id).or(() -> findDb(id));
```

## 13. 团队规范建议

1. Optional 只用于**返回值**。
2. 禁止 `Optional.get()`（除已 `isPresent` 检查）。
3. 优先 `orElseGet` 而非 `orElse`（有计算时）。
4. 不要用 Optional 替代所有 null（字段、参数保持 null + 注解）。
5. 集合返回值用空集合，不用 Optional。
6. 提供 `Optional<T>` 的方法要在 Javadoc 说明何时为空。
7. 不要为了「炫技」把简单逻辑写成 5 层 Optional 链（可读性优先）。
