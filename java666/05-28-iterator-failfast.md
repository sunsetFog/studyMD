# 28 · 迭代器与 fail-fast 机制

## 1. Iterator 接口

```java
public interface Iterator<E> {
    boolean hasNext();
    E next();
    default void remove() { throw new UnsupportedOperationException(); }
    default void forEachRemaining(Consumer<? super E> action) { }
}
```

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.isEmpty()) it.remove();     // 唯一安全的删除方式
}
```

规则：

- `next()` 前必须 `hasNext()`，否则抛 `NoSuchElementException`。
- `remove()` 必须在 `next()` 之后调用，且每次 `next()` 只能 `remove()` 一次。
- `forEachRemaining` 性能更好（批量处理）。

## 2. Iterable 与增强 for

```java
public interface Iterable<T> {
    Iterator<T> iterator();
    default void forEach(Consumer<? super T> action) { ... }
    default Spliterator<T> spliterator() { ... }
}
```

增强 for 编译后：

```java
for (String s : list) { ... }

// 等价于
for (Iterator<String> it = list.iterator(); it.hasNext(); ) {
    String s = it.next();
    ...
}
```

所以：**任何实现 Iterable 的对象都能用增强 for**（数组是编译器特例）。

自定义可迭代类：

```java
public class Range implements Iterable<Integer> {
    private final int start, end;

    public Range(int start, int end) { this.start = start; this.end = end; }

    @Override
    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            private int cur = start;
            @Override public boolean hasNext() { return cur < end; }
            @Override public Integer next() { return cur++; }
        };
    }
}

for (int i : new Range(0, 5)) { }    // 0 1 2 3 4
```

## 3. fail-fast 机制

### 原理

集合内部维护 `modCount`（结构性修改次数）：

```java
// ArrayList.Itr
int expectedModCount = modCount;

public E next() {
    checkForComodification();     // 每次 next 检查
    ...
}

final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

**结构性修改**：add、remove、clear、扩容等改变 size 的操作。`set`（替换元素）不算。

### 触发场景

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

for (String s : list) {
    if (s.equals("b")) list.remove(s);     // ConcurrentModificationException
}
```

注意：**不保证一定抛出**。若删除后恰好遍历结束（如删除倒数第二个元素），可能不报错——这是最危险的地方，测试环境可能不复现。

```java
List<String> l = new ArrayList<>(List.of("a", "b"));
for (String s : l) l.remove(s);   // 可能不抛异常但结果错误
```

### 正确做法

```java
// 1. Iterator.remove（唯一官方安全方式）
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (cond(it.next())) it.remove();
}

// 2. removeIf（内部用位图批量删除，性能更好）
list.removeIf(this::cond);

// 3. 倒序 for（只适用于 List）
for (int i = list.size() - 1; i >= 0; i--) {
    if (cond(list.get(i))) list.remove(i);
}

// 4. 收集后统一删除
List<String> toRemove = list.stream().filter(this::cond).toList();
list.removeAll(toRemove);

// 5. 新建集合
List<String> filtered = list.stream().filter(s -> !cond(s)).toList();
```

### Map 遍历删除

```java
// 正确：entrySet 的 Iterator
Iterator<Map.Entry<K,V>> it = map.entrySet().iterator();
while (it.hasNext()) {
    if (cond(it.next().getValue())) it.remove();
}

// 正确：removeIf（Java 8+）
map.entrySet().removeIf(e -> cond(e.getValue()));

// 正确：Collection.removeIf
map.values().removeIf(this::cond);

// 错误
map.forEach((k, v) -> { if (cond(v)) map.remove(k); });   // CME
```

## 4. fail-safe 机制

并发容器使用**弱一致性**迭代器，遍历时不检查 modCount，也不保证看到最新数据。

| 集合 | 迭代器类型 | 遍历时修改 | 一致性 |
|---|---|---|---|
| ArrayList/HashMap | fail-fast | 抛 CME | 强（快照语义无） |
| ConcurrentHashMap | fail-safe | 不抛异常 | 弱一致 |
| CopyOnWriteArrayList | fail-safe | 不抛异常 | 快照（看不到新改动） |
| ConcurrentLinkedQueue | fail-safe | 不抛异常 | 弱一致 |
| ConcurrentSkipListMap | fail-safe | 不抛异常 | 弱一致 |

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
for (String k : map.keySet()) {
    map.put("b", 2);        // 不抛异常，但本次遍历可能看不到 b
}
```

## 5. ListIterator

`List` 专属，支持双向遍历与修改。

```java
ListIterator<String> it = list.listIterator();
while (it.hasNext()) {
    int idx = it.nextIndex();
    String s = it.next();
    it.set(s.toUpperCase());       // 替换当前元素
    it.add("new");                  // 在当前元素后插入
}

// 反向遍历（必须先到末尾）
ListIterator<String> rev = list.listIterator(list.size());
while (rev.hasPrevious()) {
    System.out.println(rev.previous());
}
```

## 6. Spliterator（了解）

并行流的基础，把数据源切成多段。

```java
Spliterator<String> sp = list.spliterator();
sp.tryAdvance(System.out::println);
sp.forEachRemaining(System.out::println);
sp.estimateSize();
sp.characteristics();     // ORDERED / SIZED / DISTINCT ...
```

`ArrayList` 的 Spliterator 支持 O(1) 二分切分（基于索引），所以并行流高效；`LinkedList` 切分差，并行流反而不如串行。

## 7. 迭代器 vs 索引遍历

```java
// ArrayList：索引遍历最快
for (int i = 0; i < list.size(); i++) list.get(i);

// LinkedList：只能用迭代器
for (String s : list) { }        // 内部迭代器

// 通用建议：优先增强 for，除非需要索引
```

| 集合 | 索引 for | 增强 for |
|---|---|---|
| ArrayList | 快（O(1) get） | 略慢 |
| LinkedList | O(n²) 灾难 | O(n) |
| HashSet | 不支持 | O(n) |
| CopyOnWriteArrayList | 快（快照数组） | 快（快照数组） |

## 8. 自定义集合的 fail-fast

```java
public class SimpleList<E> extends AbstractList<E> {
    private Object[] data = new Object[10];
    private int size;

    @Override
    public E get(int index) {
        Objects.checkIndex(index, size);
        return (E) data[index];
    }

    @Override
    public int size() { return size; }

    @Override
    public boolean add(E e) {
        if (size == data.length) data = Arrays.copyOf(data, size * 2);
        data[size++] = e;
        modCount++;                 // 必须维护 modCount
        return true;
    }
}
```

继承 `AbstractList` 自动获得 fail-fast 迭代器（依赖 `modCount`）。

## 9. 场景化最佳实践

```java
// 1. 遍历中删除 → removeIf
list.removeIf(Objects::isNull);

// 2. 遍历中新增 → 收集到临时列表后 addAll
List<String> newItems = new ArrayList<>();
for (String s : list) if (cond(s)) newItems.add(s + "_x");
list.addAll(newItems);

// 3. 遍历中修改其他元素 → ListIterator.set
ListIterator<Integer> it = list.listIterator();
while (it.hasNext()) {
    int v = it.next();
    if (v < 0) it.set(0);
}

// 4. 嵌套遍历同一集合 → 用索引或复制
for (String a : List.copyOf(list)) {
    for (String b : list) { }
}

// 5. 并发遍历 → 并发容器或加锁
```

## 10. 常见坑

1. 增强 for 中 `list.remove` / `map.put`（CME）。
2. 用 `for (String s : list)` 同时修改 `list` 的元素字段（合法，但可能引发业务 bug）。
3. `Iterator.remove` 前未调用 `next`（`IllegalStateException`）。
4. 连续两次 `remove`（第二次抛异常）。
5. 认为 fail-fast 一定抛异常（实际不保证，取决于 modCount 时机）。
6. 在 `forEach` 的 Lambda 中修改集合。
7. `subList` 视图 + 原列表修改 → CME。
8. 并发场景用 ArrayList 期望 fail-fast 保护（它只是尽力检测，不是线程安全机制）。

```java
// 典型错误
map.forEach((k, v) -> {
    if (v == null) map.remove(k);      // CME
});
// 正确
map.entrySet().removeIf(e -> e.getValue() == null);
```

## 11. 速查

| 需求 | 写法 |
|---|---|
| 删除满足条件的元素 | `coll.removeIf(pred)` |
| 遍历时删除 | `Iterator.remove()` |
| Map 遍历删除 | `map.entrySet().removeIf(...)` |
| 遍历时替换 | `ListIterator.set()` |
| 遍历时插入 | 收集后 `addAll` |
| 并发遍历 | `ConcurrentHashMap` / `CopyOnWriteArrayList` |
| 并行遍历 | `stream().parallel()` |
