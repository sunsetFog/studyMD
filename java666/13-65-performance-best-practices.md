# 65 · 性能优化与编码规范

## 1. 优化原则

1. **先测量再优化**（不要凭直觉）。
2. **优化瓶颈**（Amdahl 定律：优化非瓶颈无意义）。
3. **优先架构与算法**，其次代码细节，最后 JVM 参数。
4. **可读性优先**，除非有数据证明此处是热点。
5. **避免过早优化**（Knuth： premature optimization is the root of all evil）。

## 2. 常见性能陷阱

### 字符串

```java
// ✘ 循环内拼接
String s = "";
for (int i = 0; i < 10000; i++) s += i;    // O(n²)

// ✔
StringBuilder sb = new StringBuilder(10000 * 4);
for (int i = 0; i < 10000; i++) sb.append(i);
```

```java
// ✘ 循环内编译正则
for (String x : list) if (x.matches("\\d+")) { }

// ✔
private static final Pattern NUM = Pattern.compile("\\d+");
for (String x : list) if (NUM.matcher(x).matches()) { }
```

### 集合

```java
// ✘ List.contains 在循环内 → O(n²)
for (Long id : ids) if (list.contains(id)) { }

// ✔ Set
Set<Long> set = new HashSet<>(list);
for (Long id : ids) if (set.contains(id)) { }

// ✘ LinkedList 随机访问
for (int i = 0; i < list.size(); i++) list.get(i);    // O(n²)

// ✔ 迭代器
for (T t : list) { }
```

```java
// ✘ 未预估容量
List<User> list = new ArrayList<>();     // 多次扩容
Map<K,V> map = new HashMap<>();

// ✔
List<User> list = new ArrayList<>(1000);
Map<K,V> map = new HashMap<>(2048);
```

### 装箱

```java
// ✘ 大量装箱
Long sum = 0L;
for (long i = 0; i < 1_000_000; i++) sum += i;

// ✔ 基本类型
long sum = 0L;

// ✔ Stream 用 mapToInt/mapToLong
list.stream().mapToLong(User::getScore).sum();
```

### 循环内重计算

```java
// ✘
for (int i = 0; i < list.size(); i++) { }        // ArrayList 无所谓，LinkedList 是 O(n)
for (int i = 0; i < expensive(); i++) { }

// ✔
int size = list.size();
for (int i = 0; i < size; i++) { }
```

### 循环内远程调用（N+1）

```java
// ✘ 100 次查询
for (Long id : ids) userMapper.selectById(id);

// ✔ 一次批量
userMapper.selectByIds(ids);
```

### 循环内建对象

```java
// ✘ 每次 new SimpleDateFormat
for (String s : dates) new SimpleDateFormat("yyyy-MM-dd").parse(s);

// ✔ 复用或 DateTimeFormatter（线程安全）
private static final DateTimeFormatter F = DateTimeFormatter.ofPattern("yyyy-MM-dd");
for (String s : dates) LocalDate.parse(s, F);
```

## 3. 并发性能

```java
// ✘ 高并发计数用 AtomicLong
private final AtomicLong count = new AtomicLong();
count.incrementAndGet();      // 高竞争 CAS 失败重试

// ✔ LongAdder
private final LongAdder count = new LongAdder();
count.increment();

// ✘ 大锁
synchronized (this) {
    parseConfig();
    queryDb();          // 慢操作在锁内
}

// ✔ 缩小临界区
Config cfg = parseConfig();
synchronized (this) { this.cfg = cfg; }
queryDb();
```

## 4. IO 性能

```java
// ✘ 单字节读
int b;
while ((b = in.read()) != -1) { }

// ✔ 缓冲 + 批量
byte[] buf = new byte[64 * 1024];
int n;
while ((n = in.read(buf)) != -1) out.write(buf, 0, n);

// ✔ 零拷贝（大文件）
in.transferTo(0, in.size(), out);

// ✔ 大文件行处理（不 OOM）
Files.lines(path).forEach(this::handle);
```

## 5. 数据库性能

| 问题 | 优化 |
|---|---|
| 全表扫描 | 加索引（最左前缀、覆盖索引） |
| N+1 查询 | 批量查询 / JOIN |
| 大事务 | 拆分、缩短事务 |
| 深分页 `LIMIT 100000, 10` | 游标/延迟关联 |
| 循环单条 insert | 批量插入 |
| `SELECT *` | 只查需要的列 |
| 无索引的 `LIKE '%x%'` | 全文索引 / ES |
| 频繁 count | 缓存/近似值 |
| 连接池过大 | 按公式配置 |

```java
// 深分页优化
// ✘
select * from t order by id limit 100000, 10;
// ✔
select * from t where id > 100000 order by id limit 10;
```

## 6. 缓存

```java
// 本地缓存（Caffeine）
Cache<Long, User> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(10))
        .recordStats()
        .build();

User u = cache.get(id, k -> repo.findById(k));

// 分布式缓存（Redis）
// - 缓存穿透：空值缓存 / 布隆过滤器
// - 缓存击穿：互斥锁 / 逻辑过期
// - 缓存雪崩：过期时间加随机
```

## 7. 对象创建与 GC

- 减少短命大对象（如大数组、大字符串）。
- 复用不可变对象（常量、枚举）。
- 避免在热点路径创建 Lambda（捕获型）与 Stream。
- 大集合及时清空引用（`list.clear()`、置 null）。
- 关注晋升速率（过早晋升导致 Full GC）。

```java
// ✘ 热点路径创建 Stream
for (Order o : orders) {
    list.add(o.getItems().stream().map(Item::getSku).toList());
}

// ✔ 简单循环（减少管道开销）
for (Order o : orders) {
    for (Item i : o.getItems()) list.add(i.getSku());
}
```

## 8. 编码规范（阿里/Google 精简版）

### 命名

- 类：大驼峰；方法/变量：小驼峰；常量：全大写下划线。
- 禁止拼音、单字母（循环除外）、`$`。
- 布尔字段不加 `is` 前缀（`active` 而非 `isActive`）。
- 接口实现类：`XxxImpl`；抽象类：`AbstractXxx`；异常：`XxxException`。

### 常量与魔法值

```java
// ✘
if (status == 3) { }
Thread.sleep(86400000);

// ✔
private static final int STATUS_PAID = 3;
private static final Duration ONE_DAY = Duration.ofDays(1);
```

### 方法

- 单个方法不超过 80 行。
- 参数不超过 4 个（多了用对象封装）。
- 避免过深嵌套（> 3 层用卫语句）。
- 一个方法只做一件事。

### 判空与 Optional

```java
// 字符串判空
if (s == null || s.isBlank()) { }
// 或 StringUtils.isBlank(s)（Spring/Commons）

// 集合判空
if (list == null || list.isEmpty()) { }
// 或 CollectionUtils.isEmpty(list)

// 对象比较
Objects.equals(a, b);
```

### 异常

- 不要吞异常。
- 不要用异常控制流程。
- 捕获具体异常。
- 日志带上下文与异常对象。

### 日志

- 用 SLF4J + 占位符。
- 禁止 `System.out`。
- 敏感信息脱敏。

### 集合

```java
// 返回空集合而非 null
return Collections.emptyList();

// 防御性拷贝
return List.copyOf(list);

// 不可变常量
private static final List<String> TYPES = List.of("a", "b");
```

## 9. 代码坏味道

| 坏味道 | 重构 |
|---|---|
| 超长方法 | 提取方法 |
| 超长类 | 拆分职责 |
| 重复代码 | 提取公共方法/父类/工具 |
| 过多参数 | 参数对象 |
| 魔法数字 | 常量/枚举 |
| 深层嵌套 | 卫语句、提前 return |
| 注释掉的代码 | 删除（有 Git） |
| 布尔参数 | 拆成两个方法 |
| 可变全局状态 | 依赖注入 |
| 过度继承 | 组合 |
| `catch (Exception e) {}` | 具体处理 |
| 一个类 import 超过 30 个包 | 职责过多 |

## 10. 性能测试

```java
// JMH（微基准，避免 JIT 预热/死码消除问题）
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(1)
@State(Scope.Thread)
public class MyBenchmark {
    @Benchmark
    public int testMethod() { return compute(); }
}
```

压测工具：JMeter、wrk、Gatling、k6。

关注指标：QPS、P50/P95/P99 RT、错误率、CPU/内存/GC。

## 11. 上线前检查清单

- [ ] 无 `System.out.println`
- [ ] 无 `e.printStackTrace()`
- [ ] 无硬编码的密钥/密码
- [ ] 所有外部调用有超时
- [ ] 线程池参数已评估
- [ ] 连接池大小合理
- [ ] 慢 SQL 已优化（explain）
- [ ] 缓存有淘汰策略
- [ ] 大文件/大数据分页处理
- [ ] 日志有 traceId
- [ ] 异常有全局处理
- [ ] 配置外置（不硬编码）
- [ ] 压测通过
- [ ] 监控告警已配置

## 12. 性能优化速查

| 症状 | 可能原因 | 优化 |
|---|---|---|
| CPU 高 | 死循环、正则回溯、频繁 GC | 线程栈 + JFR |
| 内存高 | 泄漏、大对象、缓存无界 | heap dump + MAT |
| RT 高 | 慢 SQL、锁竞争、N+1 | 链路追踪 |
| QPS 上不去 | 线程池小、连接池小、串行调用 | 压测调参、并行化 |
| GC 频繁 | 分配率高、新生代小 | 减少对象、调堆 |
| 启动慢 | 类太多、初始化重 | AppCDS、延迟初始化 |

## 13. 一句话总结

> 性能优化的顺序：**架构 > 算法 > 数据结构 > IO/DB > 并发 > JVM 参数 > 代码微优化**。
> 没有测量数据的优化都是猜测。
