# 22 · 集合框架总览

## 1. 体系结构

```
Iterable
└── Collection
    ├── List        （有序、可重复、有索引）
    │   ├── ArrayList
    │   ├── LinkedList
    │   ├── Vector（遗留）
    │   └── CopyOnWriteArrayList（并发）
    ├── Set         （不重复）
    │   ├── HashSet ── LinkedHashSet
    │   ├── TreeSet（有序）
    │   ├── EnumSet
    │   └── CopyOnWriteArraySet（并发）
    └── Queue       （队列）
        ├── ArrayDeque
        ├── PriorityQueue
        ├── LinkedList（也实现 Deque）
        └── BlockingQueue（并发）
            ├── ArrayBlockingQueue
            ├── LinkedBlockingQueue
            ├── PriorityBlockingQueue
            ├── SynchronousQueue
            ├── DelayQueue
            └── LinkedTransferQueue

Map（独立体系，不继承 Collection）
├── HashMap ── LinkedHashMap
├── TreeMap
├── Hashtable（遗留）
├── EnumMap
├── IdentityHashMap
├── WeakHashMap
└── ConcurrentHashMap / ConcurrentSkipListMap
```

## 2. 核心接口契约

### Collection

```java
int size();
boolean isEmpty();
boolean contains(Object o);
boolean add(E e);
boolean remove(Object o);
void clear();
Iterator<E> iterator();
Object[] toArray();
<T> T[] toArray(T[] a);
boolean addAll(Collection<? extends E> c);
boolean removeIf(Predicate<? super E> filter);   // Java 8
Stream<E> stream();
```

### List 额外

```java
E get(int index);
E set(int index, E element);
void add(int index, E element);
E remove(int index);        // 注意重载：remove(int) vs remove(Object)
int indexOf(Object o);
List<E> subList(int from, int to);   // 视图！
List<E> of(...);            // Java 9+ 不可变
```

### Map

```java
V get(Object key);
V put(K key, V value);
V remove(Object key);
boolean containsKey(Object key);
Set<K> keySet();
Collection<V> values();
Set<Map.Entry<K,V>> entrySet();
V getOrDefault(Object key, V def);
V putIfAbsent(K key, V value);
V computeIfAbsent(K key, Function<K,V> f);
V merge(K key, V value, BiFunction<V,V,V> f);
void forEach(BiConsumer<K,V> action);
```

> `Map` 不是 `Collection`，因为「键值对」不是元素的集合；`entrySet()` 才是 Collection 视图。

## 3. 时间复杂度对照

| 实现 | get | add | remove | contains | 有序 | 线程安全 |
|---|---|---|---|---|---|---|
| ArrayList | O(1) | O(1) 均摊 | O(n) | O(n) | 插入序 | ✘ |
| LinkedList | O(n) | O(1) | O(1)（已知节点） | O(n) | 插入序 | ✘ |
| HashSet | O(1) | O(1) | O(1) | O(1) | 无序 | ✘ |
| LinkedHashSet | O(1) | O(1) | O(1) | O(1) | 插入序 | ✘ |
| TreeSet | O(log n) | O(log n) | O(log n) | O(log n) | 排序 | ✘ |
| HashMap | O(1) | O(1) | O(1) | O(1) | 无序 | ✘ |
| LinkedHashMap | O(1) | O(1) | O(1) | O(1) | 插入/访问序 | ✘ |
| TreeMap | O(log n) | O(log n) | O(log n) | O(log n) | 排序 | ✘ |
| ArrayDeque | O(1)* | O(1) 均摊 | O(1) | O(n) | 插入序 | ✘ |
| PriorityQueue | O(1) peek | O(log n) | O(log n) | O(n) | 堆序 | ✘ |
| ConcurrentHashMap | O(1) | O(1) | O(1) | O(1) | 无序 | ✔ |
| CopyOnWriteArrayList | O(1) | O(n) | O(n) | O(n) | 插入序 | ✔（写时复制） |

## 4. 选型决策树

```
需要键值对？
├─ 是 → 需要排序？ → TreeMap
│        需要插入序？ → LinkedHashMap
│        并发？ → ConcurrentHashMap
│        key 是枚举？ → EnumMap
│        否则 → HashMap
└─ 否 → 需要去重？
     ├─ 是 → 排序？ TreeSet : (插入序？ LinkedHashSet : HashSet)
     └─ 否 → 需要按索引随机访问？
          ├─ 是 → ArrayList
          └─ 否 → 频繁头尾操作？
               ├─ 是 → ArrayDeque
               ├─ 需要优先级 → PriorityQueue
               └─ 线程安全 → CopyOnWriteArrayList / BlockingQueue
```

## 5. 数组 vs 集合

| 维度 | 数组 | 集合 |
|---|---|---|
| 长度 | 固定 | 动态 |
| 元素类型 | 基本类型 + 引用 | 仅引用（泛型） |
| 性能 | 高、内存紧凑 | 有对象头与装箱开销 |
| 功能 | 少（Arrays 工具） | 丰富 |
| 类型安全 | 运行期 ArrayStoreException | 编译期泛型检查 |

**结论**：数值密集计算用数组；业务逻辑用集合；极端性能场景考虑 `int[]` + 自定义容器（或 Eclipse Collections / fastutil）。

## 6. 不可变集合（Java 9+）

```java
List<String> l = List.of("a", "b");
Set<Integer> s = Set.of(1, 2, 3);
Map<String, Integer> m = Map.of("a", 1, "b", 2);
Map<String, Integer> m2 = Map.ofEntries(Map.entry("a", 1), Map.entry("b", 2));

List<String> copy = List.copyOf(existingList);   // 防御性拷贝
```

特性：

- 不可增删改（`add` 抛 `UnsupportedOperationException`）。
- **不允许 null**（`List.of(null)` 抛 NPE）。
- `Set.of` 重复元素抛 `IllegalArgumentException`。
- 序列化安全（不会产生重复实例）。
- 实现紧凑（内存占用小）。

对比 `Collections.unmodifiableList(list)`：后者是**视图**，原 list 改变仍会反映；`List.copyOf` 是**真拷贝**。

```java
List<String> src = new ArrayList<>(List.of("a"));
List<String> view = Collections.unmodifiableList(src);
src.add("b");
System.out.println(view);        // [a, b] —— 视图变了
```

## 7. 集合的线程安全

| 方式 | 说明 | 推荐度 |
|---|---|---|
| `Collections.synchronizedList` | 全表锁，性能差 | 低 |
| `CopyOnWriteArrayList` | 读多写极少 | 中 |
| `ConcurrentHashMap` | 高并发读写 | 高 |
| `BlockingQueue` | 生产者消费者 | 高 |
| 不可变集合 | 只读共享 | 高 |
| 线程私有（方法内局部变量） | 无需同步 | 最高 |

```java
// 同步包装的遍历仍需手动加锁
List<String> sync = Collections.synchronizedList(new ArrayList<>());
synchronized (sync) {
    for (String s : sync) { }
}
```

## 8. 遍历方式

```java
// 1. 增强 for（最常用）
for (String s : list) { }

// 2. 迭代器（需要删除时）
Iterator<String> it = list.iterator();
while (it.hasNext()) { if (cond(it.next())) it.remove(); }

// 3. forEach（Java 8）
list.forEach(System.out::println);

// 4. 索引 for（需要索引时）
for (int i = 0; i < list.size(); i++) { list.get(i); }

// 5. Stream
list.stream().filter(...).forEach(...);

// 6. Map 遍历
map.forEach((k, v) -> { });
for (Map.Entry<K, V> e : map.entrySet()) { }
```

## 9. 集合与 Stream

```java
List<String> names = users.stream()
        .filter(User::isActive)
        .map(User::getName)
        .sorted()
        .toList();              // Java 16+，不可变

Map<Long, User> byId = users.stream()
        .collect(Collectors.toMap(User::getId, u -> u));

Map<String, List<User>> grouped = users.stream()
        .collect(Collectors.groupingBy(User::getDept));
```

详见 12-56 Stream 篇。

## 10. 常见坑

1. `Arrays.asList` / `List.of` 返回固定大小或不可变列表，`add` 抛异常。
2. `subList` 是视图，原列表结构性修改后使用会抛 `ConcurrentModificationException`。
3. 增强 for 中删除元素。
4. `List.remove(int)` vs `remove(Object)`：`list.remove(1)` 删索引，`list.remove(Integer.valueOf(1))` 删元素。
5. `HashMap` 的 key 用可变对象。
6. `HashSet` 存自定义对象未重写 `equals/hashCode`。
7. 初始容量设置不当导致频繁扩容。
8. 用 `==` 比较集合内容。
9. `Collection.toArray()` 返回 `Object[]`，强转会 `ClassCastException`，用 `toArray(new T[0])`。
10. `Collections.emptyList()` 是共享不可变实例，别试图修改。

## 11. 初始化容量建议

| 集合 | 默认容量 | 负载因子 | 建议 |
|---|---|---|---|
| ArrayList | 10（首次 add 时） | — | 预估 size 直接指定 |
| HashMap | 16 | 0.75 | `(int)(n / 0.75) + 1` |
| StringBuilder | 16 | — | 预估长度 |
| ConcurrentHashMap | 16 | 0.75 | 同上 |

```java
// 已知要放 1000 个元素
Map<String, User> map = new HashMap<>(1024);   // 不必精确，取 2 的幂
List<User> list = new ArrayList<>(1000);
```

## 12. 遗留类（不要在新代码中使用）

| 类 | 问题 | 替代 |
|---|---|---|
| `Vector` | 全方法 synchronized，性能差 | ArrayList / CopyOnWriteArrayList |
| `Stack` | 继承 Vector，API 混乱 | ArrayDeque |
| `Hashtable` | 全表锁，不允许 null | ConcurrentHashMap |
| `Enumeration` | 老迭代器 | Iterator |
| `Collections.synchronizedXxx` | 全表锁 | 并发容器 |

```java
// 栈：用 Deque 而不是 Stack
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.pop(); stack.peek();
```
