# 30 · 通配符与类型擦除

## 1. 三类通配符

| 写法 | 名称 | 可读 | 可写 | 场景 |
|---|---|---|---|---|
| `List<?>` | 无界 | ✔（Object） | ✘（只能 null） | 任意类型只读 |
| `List<? extends Number>` | 上界 | ✔（Number） | ✘ | 生产者（读取数据） |
| `List<? super Integer>` | 下界 | ✔（Object） | ✔（Integer） | 消费者（写入数据） |

```java
void read(List<? extends Number> list) {
    Number n = list.get(0);        // 可读，类型至少是 Number
    // list.add(1);                // 不能写（不知道具体是 List<Integer> 还是 List<Double>）
}

void write(List<? super Integer> list) {
    list.add(1);                   // 可写 Integer 及其子类
    Object o = list.get(0);        // 读只能当 Object
}
```

## 2. PECS 原则

> **P**roducer **E**xtends, **C**onsumer **S**uper

- 数据**从集合流出**（你读它）→ `extends`。
- 数据**流入集合**（你写它）→ `super`。
- 既读又写 → 不用通配符（用精确类型或泛型方法）。

```java
// JDK 实例：Collections.copy
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (int i = 0; i < src.size(); i++) {
        dest.set(i, src.get(i));
    }
}

// JDK 实例：Collection.addAll
boolean addAll(Collection<? extends E> c);   // 只读源，所以 extends
```

### 为什么上界不能写

```java
List<? extends Number> list = new ArrayList<Integer>();
list.add(1.5);      // 若允许，Integer 列表里就混入了 Double
```

编译器无法确定具体类型，为安全起见一律禁止（null 除外）。

### 为什么下界可以写

```java
List<? super Integer> list = new ArrayList<Number>();   // 或 ArrayList<Object>
list.add(1);        // Integer 一定是 Number/Object 的子类型，安全
```

## 3. 通配符捕获

```java
void swap(List<?> list, int i, int j) {
    // list.set(i, list.get(j));      // 编译错误：无法写入
    swapHelper(list, i, j);
}

private <T> void swapHelper(List<T> list, int i, int j) {
    T tmp = list.get(i);
    list.set(i, list.get(j));
    list.set(j, tmp);
}
```

编译器把 `?` 捕获成一个未知类型 `T`，通过辅助泛型方法恢复可操作性。`Collections.swap` 就是这么实现的。

## 4. 通配符 vs 泛型方法

```java
// 通配符版：只能读
public static double sum(List<? extends Number> list) { ... }

// 泛型方法版：类型可控
public static <T extends Number> double sum(List<T> list) { ... }
```

选择：

- 类型参数只在**一个位置**出现 → 通配符（更简洁）。
- 类型参数在**多个位置**出现（参数间有关联）→ 泛型方法。

```java
// 需要关联：T 同时出现在参数与返回值
public static <T> T[] toArray(List<T> list, T[] arr) { ... }

// 无需关联
public static void print(List<?> list) { ... }
```

## 5. 类型擦除

### 什么是擦除

泛型只存在于**编译期**，编译后类型参数被擦除：

```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
System.out.println(a.getClass() == b.getClass());   // true，都是 ArrayList
```

擦除规则：

- 无界 `T` → `Object`
- `<T extends Number>` → `Number`
- `List<T>` → `List`
- 必要时插入**强制转换**

```java
// 源码
public <T> T get(List<T> list) { return list.get(0); }

// 擦除后（伪代码）
public Object get(List list) { return list.get(0); }
// 调用处插入 (String) 强转
```

### 查看擦除结果

```bash
javap -c -s MyClass.class      # 看方法签名
javap -v MyClass.class         # 看 Signature 属性（保留了泛型信息）
```

## 6. 桥方法（Bridge Method）

```java
public class Node<T> {
    private T data;
    public void setData(T data) { this.data = data; }
}

public class StringNode extends Node<String> {
    @Override public void setData(String data) { super.setData(data); }
}
```

擦除后父类方法是 `setData(Object)`，子类是 `setData(String)`——**没有真正重写**，多态会失效。

编译器生成**桥方法**：

```java
// StringNode 编译产物（伪代码）
public void setData(String data) { super.setData(data); }

// 编译器自动生成
public void setData(Object data) {          // 桥方法
    setData((String) data);                 // 转发 + 强转
}
```

桥方法带 `ACC_BRIDGE | ACC_SYNTHETIC` 标志。

```java
// 反射会看到两个 setData
for (Method m : StringNode.class.getDeclaredMethods()) {
    System.out.println(m.getName() + " bridge=" + m.isBridge());
}
// setData bridge=false
// setData bridge=true
```

## 7. 擦除带来的限制与应对

### 限制 1：不能创建泛型数组

```java
T[] arr = new T[10];                       // 编译错误

// 方式 A：强转（有 unchecked 警告）
@SuppressWarnings("unchecked")
T[] arr = (T[]) new Object[10];

// 方式 B：Array.newInstance（运行时需要 Class<T>）
@SuppressWarnings("unchecked")
T[] arr = (T[]) Array.newInstance(clazz, 10);

// 方式 C：用集合代替数组
List<T> list = new ArrayList<>();
```

**为什么不能**：数组在运行时需要知道元素类型（用于 ArrayStoreException 检查），而擦除后 T 的类型未知。`new Object[10]` 转成 `T[]` 后，向其中放入错误类型不会报错，直到取出时才 ClassCastException。

```java
Object[] objs = new String[1];   // 合法（数组协变）
List<String>[] lists = new List[1];  // 合法但有 unchecked 警告
```

### 限制 2：不能用 instanceof 判断泛型

```java
// if (obj instanceof List<String>)   // 编译错误
if (obj instanceof List<?>) { }       // 合法（无界）
```

### 限制 3：不能重载擦除后相同的方法

```java
void f(List<String> l) { }
void f(List<Integer> l) { }    // 编译错误：both have same erasure
```

### 限制 4：静态成员不能用类类型参数

```java
class Box<T> {
    // static T value;            // 编译错误
    static <U> U create() { }     // 合法：自己的类型参数
}
```

### 限制 5：不能 catch/throw 泛型异常

```java
// catch (T e) { }                 // 编译错误
// throw new T();                 // 编译错误
```

### 限制 6：不能实例化 T

```java
// T t = new T();                  // 编译错误
T t = clazz.getDeclaredConstructor().newInstance();   // 传 Class<T>
T t = supplier.get();                                  // 传 Supplier<T>
```

## 8. 类型令牌（Type Token）

```java
public class TypeRef<T> {
    private final Type type;

    protected TypeRef() {
        Type superclass = getClass().getGenericSuperclass();
        this.type = ((ParameterizedType) superclass).getActualTypeArguments()[0];
    }

    public Type getType() { return type; }
}

// 用法：匿名子类保留泛型信息
TypeRef<List<User>> ref = new TypeRef<>() { };
System.out.println(ref.getType());   // java.util.List<com.acme.User>
```

这正是 Jackson 的 `TypeReference`、Gson 的 `TypeToken` 的实现原理。

```java
// Jackson
List<User> users = mapper.readValue(json, new TypeReference<List<User>>() { });

// Gson
Type type = new TypeToken<List<User>>(){}.getType();
List<User> users = gson.fromJson(json, type);
```

## 9. 擦除与重载冲突实战

```java
// 想提供两个方法，实际不可行
public void process(List<String> list) { }
public void process(List<Integer> list) { }     // 编译错误

// 方案 1：不同方法名
public void processStrings(List<String> list) { }
public void processIntegers(List<Integer> list) { }

// 方案 2：包装类型
public void process(StringList list) { }
public void process(IntList list) { }

// 方案 3：用 Class 参数区分
public <T> void process(List<T> list, Class<T> type) { }
```

## 10. 泛型的运行时开销

- **零开销**：擦除后与手写 Object 代码一致，无额外对象、无运行时类型检查。
- 代价在编译期（类型检查）与代码体积（桥方法）。
- 唯一的运行时成本：装箱（`List<Integer>`）+ 必要的强转（可忽略）。

## 11. 常见坑

1. `${}` 认为 `List<String>` 能在运行时判断元素类型。
2. 泛型数组强转后放入错误类型，取出时才炸。
3. 重载泛型方法擦除冲突。
4. 用 `getClass()` 比较泛型集合类型（永远相等）。
5. 反射获取泛型参数时未处理 `TypeVariable`/`WildcardType`。
6. 序列化框架未保留泛型信息（JSON 反序列化成 `LinkedHashMap`）。
7. `Class<T>` 与 `Type` 混用。
8. 泛型可变参数 `T...` 产生 unchecked 警告（`@SafeVarargs` 可抑制，仅限 final/static/private 方法）。

```java
@SafeVarargs
public static <T> List<T> listOf(T... items) {
    return new ArrayList<>(Arrays.asList(items));
}
```

## 12. 速查

| 问题 | 答案 |
|---|---|
| 泛型何时擦除 | 编译期存在，class 文件中仅保留 Signature 属性 |
| 擦除到哪 | 无界 → Object；有界 → 第一个边界 |
| 桥方法作用 | 保证擦除后多态正确 |
| PECS | 生产者 extends，消费者 super |
| 为什么不能 new T[] | 运行时无法确定元素类型 |
| 如何获取泛型信息 | 字段/方法签名 + `ParameterizedType` |
| 类型令牌 | 匿名子类 + `getGenericSuperclass()` |
