# 16 · 集合框架

## 一、全景图

```
Collection
├── List（有序可重复）：ArrayList / LinkedList / Vector(过时)
├── Set（不重复）：HashSet / LinkedHashSet / TreeSet
└── Queue（队列）：ArrayDeque / LinkedList / PriorityQueue

Map（键值对，独立体系）
├── HashMap / LinkedHashMap / TreeMap / Hashtable(过时)
└── Concurrent 系列（24 篇）
```

## 二、List

### ArrayList（默认选择）

```java
List<String> list = new ArrayList<>();
list.add("a"); list.add(0, "head"); list.set(0, "x");
list.get(0); list.remove(0); list.remove("a");
list.size(); list.contains("a"); list.indexOf("a");
list.isEmpty(); list.clear();

// 构建
new ArrayList<>(List.of("a", "b"));   // List.of 不可变
new ArrayList<>(100);                  // 预估容量，减少扩容拷贝
```

**底层**：Object[] 数组。get/set O(1)；**头/中间插入删除 O(n)（整体搬移）**；扩容 1.5 倍拷贝。
**选择**：随机访问多、尾部增删 → ArrayList（90% 场景）。

### LinkedList

**底层**：双向链表。头尾增删 O(1)，**随机访问 O(n)**。实际业务极少优于 ArrayList（缓存局部性差）；当 Deque/Queue 用。

### List 工具

```java
List.of("a", "b");                    // 不可变（Java 9+）
Collections.unmodifiableList(list);   // 只读视图
Collections.emptyList();              // 空集合单例（返回值规范）
Collections.reverse(list);
Collections.shuffle(list);
Collections.sort(list, cmp);          // 或 list.sort(cmp)
Collections.max/min(list);
List.subList(1, 3);                   // 视图！改子列表影响原列表
```

## 三、Set

```java
Set<String> hash = new HashSet<>();       // 最快，无序（哈希桶）
Set<String> linked = new LinkedHashSet<>();  // 保持插入序
Set<String> tree = new TreeSet<>();       // 排序（红黑树），自然序或 Comparator

hash.add("a");          // 返回 false 表示已存在（去重判断）
hash.contains("a");     // O(1)
tree.first(); tree.last(); tree.headSet("m");

Set.of("a", "b");       // 不可变
// 数组去重惯用法
Set<Integer> uniq = new HashSet<>(List.of(1, 2, 2, 3));
```

**选择**：默认 HashSet；需保序 LinkedHashSet；需排序/范围查询 TreeSet。

## 四、Map

### HashMap（默认选择）

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.get("a");                      // 无键返回 null（不是异常）
map.getOrDefault("x", 0);
map.containsKey("a");
map.remove("a");

// 四大遍历方式
for (var e : map.entrySet()) e.getKey() + e.getValue();   // 推荐
map.forEach((k, v) -> { });
for (String k : map.keySet()) { }
map.values();

// Java 8 计算增强（避免 get-check-put 竞态与样板）
map.computeIfAbsent("key", k -> new ArrayList<>()).add(item);   // 分组必用
map.computeIfPresent("k", (k, v) -> v + 1);
map.merge("k", 1, Integer::sum);        // 计数必用
map.putIfAbsent("k", v);
```

**底层（高频面试）**：数组 + 链表 + 红黑树。
- put：key.hashCode() → 扰动函数 → (n-1) & hash 定桶 → 冲突挂链 → 链长 ≥ 8 且容量 ≥ 64 转红黑树（查询 O(log n)）
- 默认容量 16，负载因子 0.75，超阈值扩容 2 倍（rehash 全量搬移）
- **指定初始容量**：`new HashMap<>(expected / 0.75 + 1)` 避免扩容

### TreeMap / LinkedHashMap

```java
TreeMap<Integer, String> tm = new TreeMap<>();   // 按 key 排序
tm.firstKey(); tm.floorKey(5); tm.tailMap(10);   // 范围导航

LinkedHashMap<String, Integer> lhm = new LinkedHashMap<>(16, 0.75f, true); // true=访问序
// accessOrder=true → LRU 缓存基础：重写 removeEldestEntry
Map<String, Integer> lru = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Integer> e) {
        return size() > 100;
    }
};
```

### Map 创建工具

```java
Map.of("a", 1, "b", 2);              // 不可变（≤10 对）
Map.entry("k", 1);                   // 单条目
Collections.emptyMap();
```

## 五、Queue / Deque

```java
// 队列：offer/poll/peek（失败返回 null/false，优于 add/remove 的抛异常版）
Queue<Integer> q = new ArrayDeque<>();
q.offer(1); q.poll(); q.peek();

// 双端队列（栈的替代——禁用 legacy Stack）
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);       // addFirst
stack.pop();         // removeFirst
stack.peek();

// 优先队列（小顶堆）
PriorityQueue<Integer> pq = new PriorityQueue<>();
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
pq.offer(3); pq.poll();     // 每次出最小/最大 O(log n)
```

## 六、遍历修改的坑

```java
// 反例：遍历中 remove → ConcurrentModificationException（fail-fast）
for (String s : list) { if (cond) list.remove(s); }

// 正确姿势
list.removeIf(s -> cond);                        // 首选
Iterator<String> it = list.iterator();           // 复杂逻辑
while (it.hasNext()) { if (cond(it.next())) it.remove(); }

// Map 遍历删除
map.entrySet().removeIf(e -> e.getValue() < 0);
```

## 七、选择决策表

| 需求 | 选择 |
|---|---|
| 列表（增删查混合） | ArrayList |
| 头尾频繁操作/队列 | ArrayDeque |
| 去重 | HashSet |
| 去重+保序 | LinkedHashSet |
| 排序/范围 | TreeSet/TreeMap |
| 键值映射 | HashMap |
| 计数/分组 | merge / computeIfAbsent |
| LRU | LinkedHashMap(accessOrder) 或 Caffeine |
| 只读集合 | List.of / Map.of |
| 多线程 | 24 篇并发容器 |

## 常见坑

- **坑 1**：集合元素没重写 equals/hashCode → contains/去重失效（09 篇）
- **坑 2**：遍历中结构性修改 → CME；用 removeIf/Iterator
- **坑 3**：Arrays.asList / Map.of / List.of 不可变却当普通 List 用 → add 抛异常
- **坑 4**：subList 是视图 → 对原列表结构修改后使用抛异常
- **坑 5**：HashMap 无序遍历却依赖顺序 → 需要 Seq 用 LinkedHashMap
- **坑 6**：new ArrayList<>() 不预估容量大批量 add → 多次扩容拷贝
- **坑 7**：LinkedList 当 List 用以为更快 → 实际更慢（缓存不友好）

## Checklist

- [ ] 默认 ArrayList/HashMap/HashSet，换型有理由
- [ ] merge/computeIfAbsent 消灭 get-check-put 样板
- [ ] 元素类 equals/hashCode 已正确重写
- [ ] 删除用 removeIf，遍历不结构性修改
- [ ] 不可变集合用于常量/返回防御副本
