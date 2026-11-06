# 27 · Collections 工具类与比较器

## 1. Collections 常用方法

### 只读视图

```java
List<String> ro = Collections.unmodifiableList(list);
Set<String> roSet = Collections.unmodifiableSet(set);
Map<K,V> roMap = Collections.unmodifiableMap(map);
Collection<String> roCol = Collections.unmodifiableCollection(col);
```

视图 ≠ 拷贝：原集合变化会反映到视图；视图上的写操作抛 `UnsupportedOperationException`。

### 空集合

```java
List<String> empty = Collections.emptyList();     // 共享不可变实例
Set<String> emptySet = Collections.emptySet();
Map<K,V> emptyMap = Collections.emptyMap();

// 返回空集合优于返回 null
public List<Order> findOrders(Long userId) {
    List<Order> r = query(userId);
    return r == null ? Collections.emptyList() : r;
}
```

### 单元素集合

```java
Collections.singletonList("a");     // 不可变
Collections.singleton("a");
Collections.singletonMap("k", "v");
```

### 排序与查找

```java
Collections.sort(list);                                  // 要求 Comparable
Collections.sort(list, comparator);
Collections.reverse(list);
Collections.shuffle(list);
Collections.swap(list, i, j);
Collections.rotate(list, 1);                             // 循环右移
Collections.binarySearch(sortedList, key);               // 必须已排序
Collections.max(list); Collections.min(list);
Collections.max(list, comparator);
Collections.frequency(list, "a");                        // 出现次数
Collections.replaceAll(list, "old", "new");
Collections.fill(list, "x");
Collections.nCopies(5, "a");                             // [a,a,a,a,a]
```

### 批量操作

```java
list.addAll(other);
list.removeAll(other);      // 差集
list.retainAll(other);      // 交集
list.containsAll(other);    // 包含
```

> 这些方法在 ArrayList 上是 O(n*m)，大数据量先转 `HashSet`。

### 同步包装（不推荐，了解即可）

```java
List<String> sync = Collections.synchronizedList(new ArrayList<>());
Map<K,V> syncMap = Collections.synchronizedMap(new HashMap<>());
```

遍历仍需手动 `synchronized (sync) { ... }`。生产用 `ConcurrentHashMap`/`CopyOnWriteArrayList`。

## 2. Comparable

```java
public class User implements Comparable<User> {
    private int age;
    private String name;

    @Override
    public int compareTo(User o) {
        return Integer.compare(this.age, o.age);     // 年龄升序
    }
}
```

约定：

- 返回负数：this < o
- 0：相等
- 正数：this > o
- 应与 `equals` 保持一致（否则 `TreeSet`/`TreeMap` 行为反直觉）
- 强烈建议 `compareTo` 与 `equals` 一致，否则 SortedSet/SortedMap 会「吞掉」元素

**不要用减法**：

```java
// 溢出风险
return this.age - o.age;

// 正确
return Integer.compare(this.age, o.age);
```

## 3. Comparator

### 基础

```java
Comparator<User> byAge = (a, b) -> Integer.compare(a.getAge(), b.getAge());
Comparator<User> byAge2 = Comparator.comparingInt(User::getAge);
Comparator<User> byName = Comparator.comparing(User::getName);
```

### 链式组合

```java
Comparator<User> cmp = Comparator
        .comparing(User::getDept)
        .thenComparingInt(User::getAge)
        .thenComparing(User::getName, Comparator.nullsLast(String::compareTo));

users.sort(cmp);
```

### 常用工厂

```java
Comparator.naturalOrder();
Comparator.reverseOrder();
Comparator.comparing(Function);
Comparator.comparingInt/Long/Double(ToIntFunction);
Comparator.nullsFirst(cmp);
Comparator.nullsLast(cmp);

// 逆序（注意：不要用 -compare 结果，会溢出）
cmp.reversed();

// 处理 null 字段
Comparator.comparing(User::getName, Comparator.nullsLast(Comparator.naturalOrder()));
```

### 多字段 + null 安全完整示例

```java
Comparator<User> cmp = Comparator
        .comparing(User::getDept, Comparator.nullsLast(Comparator.naturalOrder()))
        .thenComparing(User::getAge, Comparator.nullsLast(Comparator.naturalOrder()))
        .thenComparing(User::getId);
```

### 中文排序

```java
Collator collator = Collator.getInstance(Locale.CHINA);
list.sort(Comparator.comparing(User::getName, collator));
```

## 4. reversed 的坑

```java
Comparator<User> cmp = Comparator.comparingInt(User::getAge).thenComparing(User::getName);

cmp.reversed();          // 整体反转：年龄降序 + 姓名降序
// 若只想年龄降序、姓名升序：
Comparator<User> mixed = Comparator
        .comparingInt(User::getAge).reversed()
        .thenComparing(User::getName);
```

`reversed()` 作用于**调用前的整个链**，位置很关键。

## 5. 排序稳定性

| 算法 | 稳定性 | 适用 |
|---|---|---|
| TimSort（对象数组、List） | ✔ 稳定 | `Collections.sort`、`List.sort` |
| 双轴快排（基本类型数组） | ✘ 不稳定 | `Arrays.sort(int[])` |

需要稳定排序但元素是基本类型 → 用包装类数组或索引排序。

## 6. 不可变集合对比

| 方式 | 是否拷贝 | 允许 null | 修改 |
|---|---|---|---|
| `List.of(...)` | 新建 | ✘ | 抛异常 |
| `List.copyOf(coll)` | ✔ 拷贝 | ✘ | 抛异常 |
| `Collections.unmodifiableList(list)` | ✘ 视图 | ✔（随原集合） | 抛异常 |
| `Collections.emptyList()` | 共享 | — | 抛异常 |
| Guava `ImmutableList.copyOf` | ✔ | ✘ | 抛异常 |

```java
// 防御性返回
public List<Item> getItems() {
    return List.copyOf(items);       // 每次都拷贝，安全但有成本
}

// 内部已是不可变时可零拷贝
private final List<Item> items = List.of(...);
```

## 7. 实用工具片段

```java
// 分组后排序
Map<String, List<User>> byDept = users.stream()
    .sorted(Comparator.comparing(User::getAge))
    .collect(Collectors.groupingBy(User::getDept, LinkedHashMap::new, Collectors.toList()));

// 取前 N
List<User> top10 = users.stream()
    .sorted(Comparator.comparingInt(User::getScore).reversed())
    .limit(10)
    .toList();

// 找最大
Optional<User> oldest = users.stream().max(Comparator.comparingInt(User::getAge));

// 按多个 key 去重（保留第一个）
Map<String, User> unique = users.stream()
    .collect(Collectors.toMap(
        u -> u.getDept() + ":" + u.getName(),
        u -> u,
        (a, b) -> a,
        LinkedHashMap::new));

// 集合转 Map 注意重复 key
Map<Long, User> byId = users.stream()
    .collect(Collectors.toMap(User::getId, u -> u, (a, b) -> a));
```

## 8. 比较器实现的常见错误

```java
// 1. 减法溢出
(a, b) -> a.getAge() - b.getAge();          // 极端值出错
Integer.compare(a.getAge(), b.getAge());    // 正确

// 2. 返回 boolean 转 int
(a, b) -> a.getAge() > b.getAge() ? 1 : -1; // 相等时返回 -1，违反契约！

// 3. 不一致的比较导致 TimSort 抛
// "Comparison method violates its general contract!"
// 原因：比较器不满足传递性（常见于有状态比较器）

// 4. 用 == 比较字符串
(a, b) -> a.getName() == b.getName() ? 0 : 1;   // 错误
```

### 契约违例异常

```
java.lang.IllegalArgumentException: Comparison method violates its general contract!
```

触发条件：比较器不满足自反/反对称/传递性。

排查：

- 检查是否有随机值、时间戳、外部可变状态参与比较。
- 检查 null 处理是否一致。
- 用 `-Djava.util.Arrays.useLegacyMergeSort=true` 临时规避（不推荐，掩盖问题）。

## 9. 排序性能

```java
// ArrayList 排序：内部转数组 → TimSort → 写回
list.sort(cmp);

// 避免在比较器中做重计算
// 差
Comparator.comparing(u -> expensive(u));
// 好：预先计算
Map<User, Integer> cache = ...;
```

- 排序 100 万元素约几十毫秒。
- 若只需 Top K，用 `PriorityQueue`（O(n log k)）而非全排序。
- 大数据集可用并行排序 `Arrays.parallelSort`。

## 10. 速查表

| 需求 | 代码 |
|---|---|
| 升序 | `Comparator.comparing(X::getField)` |
| 降序 | `Comparator.comparing(X::getField).reversed()` |
| 多字段 | `.thenComparing(...)` |
| null 安全 | `Comparator.nullsLast(...)` |
| 基本类型 | `comparingInt/Long/Double` |
| 自定义规则 | `(a, b) -> ...` 返回 int |
| 空集合 | `Collections.emptyList()` |
| 只读 | `List.copyOf(...)` |
| 频次 | `Collections.frequency(list, x)` |
| 二分 | `Collections.binarySearch(sorted, key)` |
