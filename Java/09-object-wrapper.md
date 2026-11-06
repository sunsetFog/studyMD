# 09 · Object 类、equals/hashCode 与包装类

## 一、Object：万物之母

```java
public class Object {
    public boolean equals(Object obj)   // 默认 ==（引用比较）
    public int hashCode()               // 默认地址派生
    public String toString()            // 默认 "类名@十六进制地址"
    protected Object clone()            // 浅拷贝（慎用）
    public final Class<?> getClass()
    public final void wait()/notify()/notifyAll()   // 线程协作（20 篇）
    protected void finalize()           // 已废弃
}
```

所有类直接/间接继承 Object，可重写 equals/hashCode/toString。

## 二、equals 与 hashCode 契约（重灾区）

### 正确重写

```java
public class Point {
    private final int x, y;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                    // 1. 自反捷径
        if (!(o instanceof Point p)) return false;     // 2. 类型检查（模式匹配）
        return x == p.x && y == p.y;                   // 3. 逐字段比较
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);                     // 等价于 31*x + 31*y + y
    }

    @Override
    public String toString() {
        return "Point{x=%d, y=%d}".formatted(x, y);
    }
}
```

### 契约规则

```
1. equals 相等  →  hashCode 必须相等
2. hashCode 相等  →  equals 不必相等（哈希碰撞）
3. equals 不相等  →  hashCode 尽量不同（分布均匀）
4. 重写 equals 必须重写 hashCode（IDE 可生成 / Lombok @EqualsAndHashCode）
```

**违反的后果**：不重写 hashCode → 两个「相等」对象在 HashMap 中落到不同桶 → contains/get 找不到。

```java
Set<Point> set = new HashSet<>();
set.add(new Point(1, 2));
set.contains(new Point(1, 2));   // 没重写 hashCode → false！
```

### == vs equals

| | == | equals |
|---|---|---|
| 基本类型 | 值比较 | — |
| 引用类型 | 地址比较 | 内容比较（重写后） |
| null 安全 | 可以 | 左操作数不能 null |

## 三、包装类（Wrapper）

| 基本类型 | 包装类 | 基本类型 | 包装类 |
|---|---|---|---|
| byte | Byte | float | Float |
| short | Short | double | Double |
| int | **Integer** | char | **Character** |
| long | Long | boolean | Boolean |

泛型/集合只能用包装类；包装类是对象，可为 null。

### 自动装箱与拆箱

```java
Integer a = 100;        // 自动装箱：Integer.valueOf(100)
int b = a;              // 自动拆箱：a.intValue()

// 性能：循环内大量装箱 → 生成大量对象
// 拆箱 NPE：Integer x = null; int y = x;  → NullPointerException！

// Integer 缓存陷阱（-128~127 缓存）
Integer a1 = 127, b1 = 127;
a1 == b1                // true —— 命中缓存，同一对象
Integer a2 = 128, b2 = 128;
a2 == b2                // false ！超缓存，不同对象
// 铁律：包装类比较永远用 equals 或先拆箱
```

### 常用转换

```java
Integer.parseInt("123");       // String → int
String.valueOf(123);           // int → String
Integer.toString(123, 2);      // 二进制字符串
Integer.toBinaryString(10);
Integer.compare(a, b);         // 安全比较（避免减法溢出）
Integer.max(a, b);
```

## 四、经典陷阱：减法比较溢出

```java
// 坏：大数值相减可能溢出
if (a - b < 0) { }                 // a=Integer.MIN_VALUE 时错乱

// 好
if (Integer.compare(a, b) < 0) { }
```

## 五、可空包装类的处理

```java
// 从 DB/JSON 反序列化来的 Integer 可能为 null
int count = Objects.requireNonNullElse(dto.getCount(), 0);   // 默认值
if (dto.getAge() != null && dto.getAge() > 18) { }           // 拆箱前判空
```

## 常见坑

- **坑 1**：重写 equals 不重写 hashCode → HashSet/HashMap 行为异常
- **坑 2**：equals 参数写成具体类型（`equals(Point p)`）→ 变成重载不是重写；参数必须 Object
- **坑 3**：包装类 `==` 比较 → 127 边界行为分裂；一律 equals
- **坑 4**：拆箱 NPE → null 包装类参与算术/赋基本类型
- **坑 5**：hashCode 只用一个字段而 equals 比较多字段 → 违反契约

## Checklist

- [ ] equals/hashCode 成对重写，用 Objects.hash
- [ ] 包装类比较一律 equals
- [ ] 拆箱前判 null
- [ ] 排序比较用 compare 不用减法
