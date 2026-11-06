# 12 · 修饰符、内部类与匿名类

## 1. 访问修饰符

| 修饰符 | 同类 | 同包 | 子类（不同包） | 任意位置 |
|---|---|---|---|---|
| `private` | ✔ | ✘ | ✘ | ✘ |
| 默认（包私有） | ✔ | ✔ | ✘ | ✘ |
| `protected` | ✔ | ✔ | ✔ | ✘ |
| `public` | ✔ | ✔ | ✔ | ✔ |

> `protected` 的「子类可见」仅限**通过子类引用访问自身继承的成员**，不能访问父类实例的 protected 成员（同包除外）。

```java
package p1;
public class Base { protected int x; }

package p2;
public class Sub extends Base {
    void f(Base b) {
        // System.out.println(b.x);  // 编译错误：不同包的父类实例
        System.out.println(this.x);   // OK
    }
}
```

### 可修饰位置

| 修饰符 | 类 | 方法 | 字段 | 构造器 | 局部变量 |
|---|---|---|---|---|---|
| public / protected / private | ✔（顶级类只能 public/默认） | ✔ | ✔ | ✔ | ✘ |
| static | ✔（嵌套类） | ✔ | ✔ | ✘ | ✘ |
| final | ✔ | ✔ | ✔ | ✘ | ✔ |
| abstract | ✔ | ✔ | ✘ | ✘ | ✘ |
| native | ✘ | ✔ | ✘ | ✘ | ✘ |
| synchronized | ✘ | ✔ | ✘ | ✘ | ✘ |
| transient / volatile | ✘ | ✘ | ✔ | ✘ | ✘ |
| sealed / non-sealed | ✔ | ✘ | ✘ | ✘ | ✘ |

## 2. 四种内部类

### 2.1 成员内部类（非静态）

```java
public class Outer {
    private int x = 10;

    class Inner {                     // 持有外部类引用 Outer.this
        void print() {
            System.out.println(x);            // 直接访问外部字段
            System.out.println(Outer.this.x); // 显式
        }
    }

    Inner create() { return new Inner(); }
}

Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();     // 必须先有外部实例
```

问题：**隐式持有外部类引用**，可能导致内存泄漏（内部类被长生命周期对象持有 → 外部 Activity/大对象无法回收）。

### 2.2 静态嵌套类

```java
public class Outer {
    private static int s = 1;

    static class Nested {              // 不持有外部实例
        void print() { System.out.println(s); }   // 只能访问静态成员
    }
}

Outer.Nested n = new Outer.Nested();   // 不需要外部实例
```

**优先使用静态嵌套类**（Effective Java 第 24 条）。Builder、Map.Entry 都是静态嵌套类。

### 2.3 局部内部类

```java
void method() {
    int local = 10;                    // effectively final
    class Local {
        void print() { System.out.println(local); }
    }
    new Local().print();
}
```

只能访问 effectively final 的局部变量（因为内部类对象可能比方法活得久）。

### 2.4 匿名内部类

```java
Runnable r = new Runnable() {
    @Override
    public void run() { System.out.println("run"); }
};

Comparator<String> cmp = new Comparator<>() {
    @Override
    public int compare(String a, String b) { return a.length() - b.length(); }
};

// Java 9+ 菱形语法可用于匿名类
```

特点：一次性使用、不能有构造器（可用实例初始化块）、编译后生成 `Outer$1.class`。

Lambda 替代（仅函数式接口）：

```java
Runnable r = () -> System.out.println("run");
```

## 3. 双大括号初始化（反模式）

```java
// 看起来简洁，实际上创建了匿名子类 + 实例初始化块
List<String> list = new ArrayList<>() {{
    add("a");
    add("b");
}};
```

缺点：

- 生成额外类文件，增加类加载开销。
- 匿名类持有外部引用 → 潜在内存泄漏。
- 不能序列化。
- 部分静态分析工具报警。

替代：

```java
List<String> list = new ArrayList<>(List.of("a", "b"));
List<String> immutable = List.of("a", "b");
Map<String, Integer> map = Map.of("a", 1, "b", 2);
```

## 4. 内部类编译产物

| 源码 | 生成的 class |
|---|---|
| `Outer` | `Outer.class` |
| `Outer.Inner`（成员） | `Outer$Inner.class` |
| `Outer.Nested`（静态） | `Outer$Nested.class` |
| 局部类 | `Outer$1Local.class` |
| 匿名类 | `Outer$1.class`、`Outer$2.class` |

内部类访问外部私有字段时，编译器生成 `access$000` 之类的合成方法（Java 11+ 用 nestmates 机制优化，不再需要）。

## 5. 内存泄漏案例

```java
public class Leaky {
    private final List<Runnable> tasks = new ArrayList<>();

    public void register() {
        tasks.add(new Runnable() {          // 匿名类持有 Leaky.this
            @Override public void run() { System.out.println("tick"); }
        });
    }
}

// tasks 若被静态集合/线程池长期持有，Leaky 实例永远无法回收
```

修复：改为 `static` 嵌套类，或 Lambda 不捕获外部实例，或显式清理。

Android 经典场景：非静态 Handler/内部类持有 Activity。

## 6. 闭包与 effectively final

```java
int count = 0;
Runnable r = () -> count++;    // 编译错误：count 不是 effectively final
```

原因：Lambda 捕获的是**值的副本**，若允许修改会造成两个「count」语义混乱。需要可变计数用：

```java
AtomicInteger count = new AtomicInteger();
Runnable r = count::incrementAndGet;

// 或数组/单元素容器（不推荐，可读性差）
int[] box = {0};
Runnable r2 = () -> box[0]++;
```

## 7. 静态导入与嵌套可见性

```java
import static com.acme.Constants.MAX_SIZE;

public class Demo {
    int a = MAX_SIZE;
}
```

内部类可以访问外部类的所有成员（包括 private），这是编译器通过合成访问器（或 nestmates）实现的，不是真正的「同一作用域」。

## 8. 最佳实践清单

1. 内部类能静态就静态。
2. 匿名类超过 10 行改用命名类或 Lambda。
3. 不要在内部类里做耗时任务并长期持有（线程池任务尤其注意）。
4. Builder 用静态嵌套类。
5. 避免双大括号初始化。
6. 内部类不要暴露在 public API 中（外部使用 `Outer.Inner` 很别扭）。
7. 序列化场景避免非静态内部类（会连带序列化外部实例）。

## 9. 修饰符组合速查

```java
public static final class Constants { }        // 常量类
private static final Logger log = ...;         // 日志
public synchronized void m() { }               // 同步方法（等价 synchronized(this)）
private transient String cache;                // 不参与序列化
private volatile boolean running = true;       // 可见性
public native void syscall();                  // JNI
public strictfp double calc() { }              // 严格浮点（Java 17 起默认严格）
```

## 10. 常见编译错误对照

| 报错 | 原因 |
|---|---|
| `modifier static not allowed here` | 静态成员不能修饰局部类/局部变量 |
| `cannot assign a value to final variable` | 重复赋值 final |
| `non-static variable x cannot be referenced from a static context` | 静态方法访问实例成员 |
| `local variables referenced from a lambda expression must be final or effectively final` | 捕获可变局部变量 |
| `class X is public, should be declared in a file named X.java` | 文件名不匹配 |
| `duplicate class` | 同一包下重复类（含依赖冲突） |
