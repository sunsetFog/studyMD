# 56 · Stream API

## 1. 核心概念

Stream 是「支持顺序和并行聚合操作的元素序列」，特点：

- **不存储数据**（视图）。
- **惰性求值**（中间操作不执行，终止操作才触发）。
- **不可复用**（一个 Stream 只能消费一次）。
- **函数式风格**（无副作用）。

```java
List<String> result = list.stream()          // 创建
        .filter(s -> !s.isEmpty())           // 中间操作
        .map(String::toUpperCase)            // 中间操作
        .sorted()                            // 中间操作
        .toList();                           // 终止操作
```

## 2. 创建 Stream

```java
// 集合
list.stream();
set.stream();
map.entrySet().stream();

// 数组
Arrays.stream(arr);
Stream.of("a", "b", "c");

// 基本类型流（避免装箱）
IntStream.range(0, 10);          // [0,10)
IntStream.rangeClosed(1, 10);    // [1,10]
IntStream.of(1, 2, 3);
LongStream / DoubleStream

// 无限流
Stream.generate(Math::random).limit(5);
Stream.iterate(0, n -> n + 2).limit(5);                    // Java 8
Stream.iterate(0, n -> n < 100, n -> n + 2);               // Java 9（带谓词）

// 文件
Files.lines(path);
Files.list(dir);
Files.walk(dir);

// 其他
"abc".chars();                   // IntStream
Stream.empty();
Stream.concat(a, b);
Stream.builder().add("a").add("b").build();
```

## 3. 中间操作

### filter（过滤）

```java
list.stream().filter(u -> u.getAge() > 18).toList();
```

### map / mapToXxx（转换）

```java
list.stream().map(User::getName).toList();
list.stream().mapToInt(User::getAge).sum();
list.stream().mapToDouble(Order::getAmount).average();
list.stream().mapToLong(User::getId).max();
```

### flatMap（扁平化）

```java
// List<List<String>> → List<String>
lists.stream().flatMap(List::stream).toList();

// 一对多
orders.stream()
      .flatMap(o -> o.getItems().stream())
      .map(Item::getSku)
      .distinct()
      .toList();

// String → 单词
Stream.of("hello world", "foo bar")
      .flatMap(s -> Arrays.stream(s.split(" ")))
      .toList();
```

### distinct / sorted / peek / limit / skip

```java
list.stream().distinct().toList();
list.stream().sorted().toList();                                    // 自然序
list.stream().sorted(Comparator.comparing(User::getAge)).toList();
list.stream().sorted(Comparator.comparing(User::getAge).reversed()).toList();
list.stream().peek(System.out::println).count();                    // 调试用
list.stream().limit(10).toList();
list.stream().skip(5).toList();
```

> `sorted()` 是**有状态**操作，需要缓冲全部元素。

### takeWhile / dropWhile（Java 9）

```java
Stream.of(1,2,3,4,5,1).takeWhile(n -> n < 4).toList();   // [1,2,3]
Stream.of(1,2,3,4,5,1).dropWhile(n -> n < 4).toList();   // [4,5,1]
```

## 4. 终止操作

### 收集

```java
List<String> list = stream.toList();              // Java 16+，不可变
List<String> list2 = stream.collect(Collectors.toList());   // 可变
Set<String> set = stream.collect(Collectors.toSet());
Set<String> linked = stream.collect(Collectors.toCollection(LinkedHashSet::new));
String s = stream.collect(Collectors.joining(", ", "[", "]"));
```

### 聚合

```java
stream.count();
stream.min(comparator);        // Optional<T>
stream.max(comparator);
stream.findFirst();            // Optional<T>
stream.findAny();              // 并行时更快
stream.anyMatch(p); stream.allMatch(p); stream.noneMatch(p);

IntStream.sum(); average(); summaryStatistics();
```

### 遍历

```java
stream.forEach(System.out::println);
stream.forEachOrdered(System.out::println);   // 并行时保证顺序
```

### 归约

```java
// reduce 三参数
Optional<Integer> sum = Stream.of(1,2,3).reduce(Integer::sum);
Integer sum2 = Stream.of(1,2,3).reduce(0, Integer::sum);         // 带初始值
Integer sum3 = Stream.of(1,2,3).reduce(0, Integer::sum, Integer::sum);  // 并行

// 字符串拼接
String s = Stream.of("a","b").reduce("", String::concat);

// 最大值
Optional<Integer> max = Stream.of(1,5,3).reduce(Integer::max);

// 求积
int product = Stream.of(1,2,3,4).reduce(1, (a,b) -> a * b);
```

### 转数组

```java
Object[] arr = stream.toArray();
String[] arr2 = stream.toArray(String[]::new);
int[] ints = intStream.toArray();
```

## 5. Collectors 详解

### 基础

```java
Collectors.toList()
Collectors.toUnmodifiableList()       // Java 10+
Collectors.toSet()
Collectors.toMap(keyMapper, valueMapper)
Collectors.toMap(k, v, (a,b) -> a)    // 冲突合并
Collectors.toMap(k, v, merge, LinkedHashMap::new)   // 指定 Map 实现
Collectors.joining()
Collectors.joining(delimiter)
Collectors.joining(delimiter, prefix, suffix)
Collectors.counting()
```

### 分组

```java
// 按部门分组
Map<String, List<User>> byDept = users.stream()
        .collect(Collectors.groupingBy(User::getDept));

// 分组后计数
Map<String, Long> count = users.stream()
        .collect(Collectors.groupingBy(User::getDept, Collectors.counting()));

// 分组后取名字列表
Map<String, List<String>> names = users.stream()
        .collect(Collectors.groupingBy(User::getDept,
                 Collectors.mapping(User::getName, Collectors.toList())));

// 分组后求平均
Map<String, Double> avg = users.stream()
        .collect(Collectors.groupingBy(User::getDept,
                 Collectors.averagingInt(User::getAge)));

// 多级分组
Map<String, Map<String, List<User>>> multi = users.stream()
        .collect(Collectors.groupingBy(User::getDept,
                 Collectors.groupingBy(User::getLevel)));

// 指定 Map 类型（保持顺序）
Map<String, List<User>> ordered = users.stream()
        .collect(Collectors.groupingBy(User::getDept, LinkedHashMap::new, Collectors.toList()));
```

### 分区（true/false 两组）

```java
Map<Boolean, List<User>> part = users.stream()
        .collect(Collectors.partitioningBy(u -> u.getAge() >= 18));
```

### 其他

```java
Collectors.summingInt(User::getAge)
Collectors.averagingInt(User::getAge)
Collectors.summarizingInt(User::getAge)      // IntSummaryStatistics
Collectors.minBy(comparator)
Collectors.maxBy(comparator)
Collectors.reducing(...)
Collectors.collectingAndThen(list -> List.copyOf(list), Function.identity())
Collectors.teeing(c1, c2, merge)             // Java 12+，同时两个收集器
```

### teeing 示例

```java
record Stats(long count, double avg) { }

Stats stats = users.stream().collect(Collectors.teeing(
        Collectors.counting(),
        Collectors.averagingInt(User::getAge),
        Stats::new));
```

## 6. toMap 的坑

```java
// 重复 key → IllegalStateException
Map<Long, String> m = users.stream()
        .collect(Collectors.toMap(User::getDept, User::getName));   // 若同部门多人 → 异常

// 解决：提供合并函数
Map<String, String> m2 = users.stream()
        .collect(Collectors.toMap(User::getDept, User::getName, (a, b) -> a));

// value 为 null → NPE（HashMap.merge 不允许 null）
// 解决：过滤或用 forEach 手动 put
```

## 7. 并行流

```java
list.parallelStream()
    .filter(...)
    .map(...)
    .toList();

// 数组
Arrays.stream(arr).parallel();
```

原理：ForkJoinPool.commonPool（默认 CPU-1 线程），拆分-处理-合并。

### 适用条件

| 条件 | 说明 |
|---|---|
| 数据量大 | > 1 万条才有意义 |
| 计算密集 | IO 密集无收益 |
| 无状态 | 操作不依赖顺序/共享状态 |
| 易拆分 | ArrayList/数组好，LinkedList 差 |
| 无副作用 | 不要修改共享变量 |

```java
// 错误：并行流 + 非线程安全集合
List<Integer> result = new ArrayList<>();
IntStream.range(0, 1000).parallel().forEach(result::add);   // 丢数据/异常
// 正确
List<Integer> ok = IntStream.range(0, 1000).parallel().boxed().toList();
```

### 自定义线程池

```java
ForkJoinPool pool = new ForkJoinPool(8);
List<R> result = pool.submit(() -> list.parallelStream().map(...).toList()).get();
```

> 并行流在容器/共享环境下不一定更快，务必压测。

## 8. 惰性求值与短路

```java
Stream.of(1,2,3,4,5)
      .peek(System.out::println)      // 不打印
      .filter(n -> n > 10)
      .findFirst();                   // 触发执行，打印 1..5

// 短路：找到即停
Stream.iterate(1, n -> n + 1)
      .filter(n -> n % 7 == 0)
      .findFirst();                   // 只计算到 7
```

## 9. 实战模式

### 分组 Top N

```java
Map<String, List<User>> top3 = users.stream()
        .collect(Collectors.groupingBy(User::getDept,
                Collectors.collectingAndThen(Collectors.toList(),
                        l -> l.stream()
                              .sorted(Comparator.comparingInt(User::getScore).reversed())
                              .limit(3)
                              .toList())));
```

### 扁平化 + 去重

```java
List<String> tags = articles.stream()
        .flatMap(a -> a.getTags().stream())
        .distinct()
        .sorted()
        .toList();
```

### 转 Map（id → 对象）

```java
Map<Long, User> byId = users.stream()
        .collect(Collectors.toMap(User::getId, Function.identity()));
```

### 求和 / 平均值

```java
BigDecimal total = orders.stream()
        .map(Order::getAmount)
        .reduce(BigDecimal.ZERO, BigDecimal::add);

double avg = orders.stream().mapToDouble(o -> o.getAmount().doubleValue()).average().orElse(0);
```

### 分批处理

```java
int batchSize = 100;
List<List<Order>> batches = IntStream.range(0, (orders.size() + batchSize - 1) / batchSize)
        .mapToObj(i -> orders.subList(i * batchSize, Math.min((i + 1) * batchSize, orders.size())))
        .toList();
```

### 索引遍历

```java
List<String> indexed = IntStream.range(0, list.size())
        .mapToObj(i -> i + ":" + list.get(i))
        .toList();
```

## 10. 性能与陷阱

1. **Stream 不是永远更快**：小集合用 for 循环更快（无管道开销）。
2. **避免装箱**：用 `mapToInt` / `IntStream`。
3. **避免在 Stream 中做重操作**（每条记录查库）。
4. **`sorted` + `limit`** 可用 `PriorityQueue` 优化 Top K。
5. **`distinct`** 依赖 hashCode，自定义对象需正确实现。
6. **无限流必须 limit**，否则 OOM/死循环。
7. **不要复用 Stream**（`IllegalStateException: stream has already been operated upon or closed`）。
8. **`peek` 只用于调试**（可能被优化掉，不要放业务逻辑）。
9. **`findFirst` vs `findAny`**：并行时 `findAny` 更快但不保证第一个。
10. **`forEach` 不保证顺序**（并行时用 `forEachOrdered`）。
11. **在 Stream 中修改外部集合**（副作用，并行时线程不安全）。
12. **`Collectors.toMap` 重复 key / null value**（见上）。

## 11. Stream vs 循环

| 场景 | 推荐 |
|---|---|
| 简单遍历/累加 | for 循环 |
| 复杂转换/过滤/分组 | Stream |
| 需要 break/continue | for 循环 |
| 需要索引 | for 循环或 `IntStream.range` |
| 并行处理大数据 | parallelStream（谨慎） |
| 链式可读性 | Stream |

```java
// 循环：提前退出
for (User u : users) {
    if (u.isTarget()) { found = u; break; }
}

// Stream：短路
User found = users.stream().filter(User::isTarget).findFirst().orElse(null);
```

## 12. 速查

| 需求 | API |
|---|---|
| 过滤 | `filter` |
| 转换 | `map` / `mapToInt` |
| 扁平化 | `flatMap` |
| 去重 | `distinct` |
| 排序 | `sorted` |
| 截取 | `limit` / `skip` |
| 判断 | `anyMatch` / `allMatch` / `noneMatch` |
| 查找 | `findFirst` / `findAny` |
| 聚合 | `count` / `sum` / `average` / `min` / `max` |
| 归约 | `reduce` |
| 收集 | `collect` / `toList` |
| 分组 | `groupingBy` |
| 分区 | `partitioningBy` |
| 拼接 | `joining` |
| 转 Map | `toMap` |
