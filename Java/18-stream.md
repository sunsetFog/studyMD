# 18 · Stream 流与 Optional

## 一、Stream 心智模型

**Stream = 数据源的声明式处理流水线**：中间操作（惰性）→ 终端操作（触发执行）。

```
源 → filter(过滤) → map(变换) → sorted(排序) → collect/forEach(终端)
```

- **不存储数据**：加工数据源的视图
- **惰性执行**：终端操作才真正遍历（短路操作可提前结束）
- **一次性**：流被终端操作消费后关闭，再用抛 IllegalStateException
- **不修改源**：函数式、无副作用为设计前提

## 二、创建流

```java
Stream<String> s1 = list.stream();                  // 集合
Stream<String> s2 = Arrays.stream(array);           // 数组
Stream<String> s3 = Stream.of("a", "b");
Stream<String> s4 = Stream.iterate(1, n -> n * 2).limit(10);   // 无限流+截断
Stream<Double> s5 = Stream.generate(Math::random).limit(5);
IntStream.range(0, 10);                             // 数字范围
IntStream.rangeClosed(1, 10);
"abc".chars();                                      // 字符流
Files.lines(Path.of("f.txt"));                      // 文件行流
```

## 三、中间操作

```java
list.stream()
    .filter(u -> u.getAge() > 18)          // 过滤
    .map(User::getName)                    // 一对一变换
    .mapToInt(Integer::parseInt)           // 特化流（防装箱）
    .flatMap(u -> u.getTags().stream())    // 一对多展平
    .distinct()                            // 去重（依赖 equals）
    .sorted(Comparator.comparing(User::getAge))    // 排序
    .peek(System.out::println)             // 调试窥视（勿用于业务副作用）
    .limit(10)                             // 截取前 N
    .skip(5)                               // 跳过前 N
    .takeWhile(x -> x < 100)               // 条件截断（Java 9+）
    .dropWhile(x -> x < 10);
```

**map vs flatMap**：map 是「变换」，flatMap 是「变换+展平一层」——多级嵌套集合（List<List<T>>）拉平、或每个元素产生流再合并。

## 四、终端操作

```java
// 收集
List<String> names = stream.collect(Collectors.toList());
List<String> names2 = stream.toList();        // Java 16+，不可变，首选
Set<String> uniq = stream.collect(Collectors.toSet());

// 查找判断
Optional<User> first = stream.filter(...).findFirst();
Optional<User> any  = stream.filter(...).findAny();     // 并行下任意
boolean hasAdult = stream.anyMatch(u -> u.getAge() >= 18);
boolean allAdult = stream.allMatch(...);
boolean noneBanned = stream.noneMatch(...);
long count = stream.count();

// 归约
int total = IntStream.of(1,2,3).sum();
OptionalInt max = IntStream.of(1,2,3).max();
Integer sum = stream.reduce(0, Integer::sum);
String joined = stream.reduce("", String::concat);

// 遍历（终端副作用，仅打印/写库这类末端）
stream.forEach(System.out::println);
stream.forEachOrdered(...);      // 并行保序
```

## 五、Collectors 收集器（最常用板块）

```java
// 分组：Map<K, List<V>>
Map<String, List<User>> byCity =
        users.stream().collect(Collectors.groupingBy(User::getCity));

// 分组计数
Map<String, Long> counts =
        users.stream().collect(Collectors.groupingBy(User::getCity, Collectors.counting()));

// 分组统计
Map<String, IntSummaryStatistics> stats =
        users.stream().collect(Collectors.groupingBy(User::getCity,
                Collectors.summarizingInt(User::getAge)));
stats.get("BJ").getMax(); getAverage(); getSum();

// 分区：true/false 两桶
Map<Boolean, List<User>> adultSplit =
        users.stream().collect(Collectors.partitioningBy(u -> u.getAge() >= 18));

// 拼接
String names = users.stream()
        .map(User::getName)
        .collect(Collectors.joining(", ", "[", "]"));    // [tom,jerry]

// 转换后收集
Map<Long, String> idName =
        users.stream().collect(Collectors.toMap(User::getId, User::getName));

// toMap 重复键必须给 merge 函数（否则 IllegalStateException！）
Map<String, Integer> scores = list.stream()
        .collect(Collectors.toMap(Student::getName, Student::getScore, Integer::sum));

// toMap 值为 null 会 NPE → 需要 null 时用 collect(HashMap::new, ...)

// 下游收集器嵌套：分组内再取最大/排序
Map<String, Optional<User>> topByCity = users.stream()
        .collect(Collectors.groupingBy(User::getCity,
                Collectors.maxBy(Comparator.comparing(User::getAge))));
```

## 六、Optional：消灭 null 判断

```java
Optional<User> opt = repository.findById(id);

// 创建
Optional.of(value);          // 值不能 null（否则 NPE）
Optional.ofNullable(value);  // 可 null
Optional.empty();

// 使用（链式，别 get 裸取）
opt.map(User::getName).orElse("unknown");
opt.map(User::getEmail)
   .filter(e -> e.endsWith("@x.com"))
   .ifPresent(e -> sendMail(e));        // 存在才执行
opt.orElseGet(() -> loadDefault());     // orElse 总求值 / orElseGet 惰性
opt.orElseThrow(() -> new BizException("用户不存在"));
opt.isPresent(); opt.isEmpty();         // 判断（需要时才用，能用链式就链式）

// Java 9+
Optional<User> merged = opt.or(() -> fallbackOpt);
opt.ifPresentOrElse(u -> pay(u), () -> notifyEmpty());
opt.stream();                           // 转流接回 Stream 管道
```

**Optional 使用规范**：
- 只作**返回类型**（表示「可能没有」）；不作字段/参数（设计噪音）
- 永远不 get() 裸取 → orElse/ifPresent/orElseThrow

## 七、并行流

```java
long count = hugeList.parallelStream()
        .filter(...).count();

// 适用：数据量大（万级+）+ 无状态操作 + 非共享可变集合
// 线程池 = ForkJoinPool.commonPool()，全局共享 —— IO 阻塞任务会拖垮全局！

// 自定义线程池（不要默认 commonPool 做阻塞任务）
ForkJoinPool pool = new ForkJoinPool(8);
pool.submit(() -> list.parallelStream().forEach(this::ioTask)).get();

// 保持有序：parallel().forEachOrdered(...)
```

**默认不并行**：多数业务场景普通 stream 更快（数据小、装箱、merge 开销）。

## 八、Stream 陷阱清单

| 陷阱 | 说明 |
|---|---|
| 流复用 | 终端后关闭；每次从源新建 |
| 副作用 | filter 内改外部状态 → 并行下灾难；用 collect |
| toMap 重复键 | 不给 merge 函数 → IllegalStateException |
| toMap 值 null | NPE → 收集到 HashMap 手工填 |
| groupingBy null 键 | NPE → key 先规范化 |
| 装箱开销 | 大数据用 mapToInt/Long/Double 特化流 |
| 无限流不截断 | generate/iterate 忘 limit → 挂死 |
| sorted 后 limit | 无状态组合下可短路，但 sorted 全量排序 O(n log n) 意识 |
| 文件流不关 | Files.lines 必须 try-with-resources |

## 常见坑

- **坑 1**：collect 后再 .stream() 同一流 → IllegalStateException
- **坑 2**：toMap 键冲突 → 补 merge 函数
- **坑 3**：Optional 字段/参数滥用 → 只做返回值
- **坑 4**：parallelStream 里的外部共享集合 add → 数据错乱
- **坑 5**：flatMap 概念混淆 → 记「展平一层」

## Checklist

- [ ] 声明式替代命令式循环（过滤/变换/聚合）
- [ ] groupingBy/toMap 组合器熟练
- [ ] Optional 只做返回类型，链式消灭 null 检查
- [ ] 并行流谨慎，阻塞任务自定义池
