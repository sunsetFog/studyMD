# 08 · 数组

## 1. 声明与创建

```java
int[] a = new int[5];           // 推荐写法
int b[] = new int[5];           // 合法但不推荐

int[] c = {1, 2, 3};                    // 声明时初始化
int[] d = new int[]{1, 2, 3};           // 匿名数组语法
String[] e = new String[3];             // 元素默认 null

int[] f = new int[0];                   // 空数组，length=0，不是 null
```

数组是**对象**，`length` 是 `final` 字段（不是方法）。

## 2. 内存布局

```
栈: arr ──→ 堆: [对象头][length=3][0][1][2]
```

- 一维基本类型数组：连续存储元素本身。
- 对象数组：连续存储**引用**，对象分散在堆中（缓存不友好）。

```java
int[] a = {1, 2, 3};
int[] b = a;          // 复制引用，a、b 指向同一数组
b[0] = 99;
System.out.println(a[0]);   // 99
```

要真正复制：`Arrays.copyOf`、`System.arraycopy`、`clone()`。

## 3. 默认值

| 元素类型 | 默认值 |
|---|---|
| byte/short/int/long | 0 |
| float/double | 0.0 |
| char | '\u0000' |
| boolean | false |
| 引用类型 | null |

## 4. 二维数组与不规则数组

```java
int[][] m = new int[3][4];        // 3 行 4 列
int[][] n = {
    {1, 2, 3},
    {4, 5},
    {6}
};                                 // 不规则（jagged）

int[][] p = new int[3][];          // 只分配行
p[0] = new int[]{1, 2};
p[1] = new int[5];

// 遍历
for (int i = 0; i < m.length; i++)
    for (int j = 0; j < m[i].length; j++)
        m[i][j] = i * j;

for (int[] row : m)
    for (int v : row)
        System.out.print(v + " ");
```

> Java 没有真正的多维数组，只有「数组的数组」。

## 5. Arrays 工具类

```java
int[] a = {5, 1, 3};

Arrays.sort(a);                          // 双轴快排，基本类型升序
Arrays.sort(objArr, Comparator.reverseOrder());
Arrays.sort(a, 1, 3);                    // 区间 [1,3)
Arrays.binarySearch(a, 3);               // 必须先排序，返回索引或负值
Arrays.fill(a, 7);
Arrays.copyOf(a, 5);                     // 扩容/截断
Arrays.copyOfRange(a, 1, 3);             // [1,3)
Arrays.equals(a, b);                     // 内容比较
Arrays.deepEquals(m1, m2);               // 多维比较
Arrays.hashCode(a);
Arrays.toString(a);                      // "[1, 2, 3]"
Arrays.deepToString(m);                  // 多维
Arrays.stream(a).sum();
Arrays.asList(1, 2, 3);                  // 返回固定大小 List（见下方陷阱）
Arrays.setAll(a, i -> i * 2);
Arrays.parallelSort(bigArray);           // 大数组并行排序
```

### `Arrays.asList` 三大坑

```java
List<String> list = Arrays.asList("a", "b");
list.add("c");            // UnsupportedOperationException
list.set(0, "x");         // OK
// list 是 Arrays 内部类，底层是原数组的视图

List<Integer> nums = Arrays.asList(1, 2, 3);   // 不是 List<int[]>，OK
int[] arr = {1, 2, 3};
List<int[]> wrong = Arrays.asList(arr);        // size=1，元素是数组

// 正确：要可变用 new ArrayList<>(...)
List<String> mutable = new ArrayList<>(Arrays.asList("a", "b"));
// 或 List.of 创建不可变列表
List<String> immutable = List.of("a", "b");
```

## 6. 可变参数（varargs）

```java
public static int sum(int... nums) {     // 编译为 int[]
    int s = 0;
    for (int n : nums) s += n;
    return s;
}
sum();            // 空数组
sum(1, 2, 3);
sum(new int[]{1, 2, 3});
```

规则：

- 必须是**最后一个参数**。
- 方法重载时，`foo(int...)` 与 `foo(int, int)` 同时存在会让编译器优先选精确匹配。
- 每次调用都会创建数组（性能敏感场景注意）。

```java
public void log(String fmt, Object... args) {
    if (args.length == 3 && args[2] instanceof Throwable t) {
        // SLF4J 风格
    }
}
```

## 7. 排序与比较

```java
// 基本类型：只能升序，降序需换包装类
Integer[] arr = {3, 1, 2};
Arrays.sort(arr, Comparator.reverseOrder());

// 对象数组多字段排序
Arrays.sort(users, Comparator
    .comparing(User::getAge)
    .thenComparing(User::getName)
    .reversed());

// 自定义比较器
Arrays.sort(people, (p1, p2) -> p1.getHeight() - p2.getHeight());  // 有溢出风险
Arrays.sort(people, Comparator.comparingInt(Person::getHeight));   // 推荐
```

> 基本类型用双轴快排（不稳定），对象数组用 TimSort（稳定）。需要稳定性时用对象数组。

## 8. 数组 vs 集合

| 维度 | 数组 | 集合 |
|---|---|---|
| 长度 | 固定 | 动态 |
| 类型 | 可基本类型 | 只能引用类型（泛型） |
| 性能 | 高（连续内存、无装箱） | 略低 |
| 功能 | 少 | 丰富 |
| 线程安全 | 无 | 视实现 |

选型：**数值计算/高频访问用数组，业务集合用 List/Map**。

## 9. 数组与 Stream 互转

```java
int[] arr = {1, 2, 3};

// 数组 -> Stream
IntStream s = Arrays.stream(arr);
List<Integer> list = Arrays.stream(arr).boxed().toList();

// List -> 数组
Integer[] boxed = list.toArray(Integer[]::new);
int[] primitive = list.stream().mapToInt(Integer::intValue).toArray();

// String -> char[]
char[] chars = "abc".toCharArray();
String back = new String(chars);
```

## 10. 常见异常

| 异常 | 场景 |
|---|---|
| `ArrayIndexOutOfBoundsException` | 索引 <0 或 ≥length |
| `NegativeArraySizeException` | `new int[-1]` |
| `NullPointerException` | 数组引用为 null 时访问 length 或元素 |
| `ArrayStoreException` | 存入不兼容类型（`Object[] o = new String[1]; o[0]=1;`） |

## 11. 性能与最佳实践

- 预知大小时直接分配准确容量，避免 `Arrays.copyOf` 反复复制。
- 大数组遍历优先普通 for（无迭代器开销）。
- 避免在循环中创建大数组（GC 压力）。
- 缓存 `arr.length` 在极端热点代码中有微小收益，现代 JIT 通常已优化。
- 二维数组按行遍历（缓存友好），不要按列跳跃访问。
- 敏感数据（密码）用 `char[]` 并在使用后 `Arrays.fill(arr, '\0')` 清零。

```java
char[] pwd = readPassword();
try {
    authenticate(pwd);
} finally {
    Arrays.fill(pwd, '\0');
}
```
