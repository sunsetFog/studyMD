# 36 · Java 8 → 21 新特性演进

## 一、版本时间线与选型

```
8(2014) → 9 → 10 → 11 LTS(2018) → 12-16 → 17 LTS(2021) → 18-20 → 21 LTS(2023) → 22-25
主线：8 = 函数式+Stream；17 = 语言现代化成熟；21 = 虚拟线程时代
```

## 二、Java 8：函数式基石（全部企业代码的地基）

- **Lambda 与 Stream**（11/18 篇）
- **Optional**：null 安全容器
- **新日期时间 java.time**（14 篇）
- **接口 default/static 方法**（08 篇）
- CompletableFuture（22 篇）

## 三、Java 9-11：模块化与便利 API

```java
// 9：模块系统 JPMS（31 篇）、集合工厂方法
List.of("a", "b"); Map.of("k", 1); Set.of(1, 2);

// 9：Stream 增强
stream.takeWhile(x -> x < 5); dropWhile(...); iterate(0, n -> n < 10, n -> n + 1);

// 10：var 局部变量类型推断（02 篇）
var list = new ArrayList<String>();

// 11：HTTP Client（27 篇）、String 增强
str.isBlank(); str.strip(); str.lines(); str.repeat(2);
"1,2,3".split(",") → List.of → .toArray
Files.readString/writeString

// 11：单文件直接运行
java Hello.java
```

## 四、Java 12-17：语言现代化高潮

```java
// 14：switch 表达式（03 篇）
String type = switch (day) { case 1, 7 -> "周末"; default -> "工作日"; };

// 14：友好 NPE 提示
// "Cannot invoke String.length() because 's' is null" —— 直接指出空的是谁

// 15：Text Blocks（文本块）
String json = """
        {
          "name": "tom",
          "age": 18
        }
        """;

// 16：instanceof 模式匹配（07 篇）
if (obj instanceof String s) { s.length(); }

// 16：Record（10 篇）—— 不可变数据载体
record Point(int x, int y) { }

// 17：sealed 密封类 —— 受控继承
public sealed class Shape permits Circle, Square, Triangle { }
// 配合 switch 模式匹配 → 编译器穷举检查：
double area = switch (shape) {          // 无 default 也编译通过（穷举了所有子类）
    case Circle c -> Math.PI * c.r() * c.r();
    case Square s -> s.side() * s.side();
    case Triangle t -> 0.5 * t.base() * t.height();
};

// 17：垃圾收集器默认 G1，CMS 移除（32 篇）
```

**sealed + record + switch 模式匹配 = 现代代数数据类型（ADT）风格**，领域建模利器。

## 五、Java 18-21：并发与模式匹配时代

```java
// 21：虚拟线程（25 篇）—— 并发范式转变
Thread.startVirtualThread(() -> handle());

// 21：Record Pattern（解构）
if (obj instanceof Point(int x, int y)) { x + y; }

// 21：switch 模式匹配正式化
Object obj = ...;
String desc = switch (obj) {
    case Integer i when i > 0 -> "正整数";
    case String s -> "字符串";
    case null -> "空";                  // 21 允许匹配 null
    default -> "其他";
};

// 21：分代 ZGC（32 篇）、结构化并发 preview（25 篇）

// 19/20：ScopedValue（preview）—— ThreadLocal 的现代替代
static final ScopedValue<String> USER = new ScopedValue<>();
ScopedValue.where(USER, "tom").run(() -> { USER.get(); });
```

## 六、弃用与移除清单（升级注意）

```
- Java 9：弃用 Applet；13：移除 Nashorn JS 引擎
- Java 11：移除 JavaFX 独立发布、JEE 模块（javax.annotation 等）
- Java 15：偏向锁移除（32 篇锁升级变化）
- Java 17：强封装 JDK 内部 API（--add-opens 过渡）
- finalize 全程废弃（15 篇 Cleaner 替代）
- SecurityManager 弃用
```

**升级清单动作**：javax.* 依赖排查、内部 API 反射检查、`--add-opens` 处理、Lombok/框架版本跟上。

## 七、按特性分类速查表

| 领域 | 特性 | 版本 |
|---|---|---|
| 语言 | Lambda/Stream/Optional | 8 |
| 语言 | var | 10 |
| 语言 | switch 表达式 / Text Block | 14/15 |
| 语言 | Record / instanceof 模式 | 16 |
| 语言 | sealed / switch 模式 / Record Pattern | 17/21 |
| 并发 | CompletableFuture | 8 |
| 并发 | 虚拟线程 / 结构化并发 | 21 |
| API | java.time | 8 |
| API | List.of / HTTP Client | 9/11 |
| JVM | G1 默认 / ZGC / 分代 ZGC | 9/15/21 |
| 平台 | JPMS | 9 |

## 常见坑

- **坑 1**：老项目直接升 17 → `javax.*`/反射/依赖兼容炸；先 dependency:tree + add-opens 过渡
- **坑 2**：Record 用于 JPA/MyBatis 实体 → 框架需要无参构造/可变；实体用类，DTO 用 record
- **坑 3**：sealed 子类忘 extends 继承关系 → 编译错（子类必须继承密封类本身）
- **坑 4**：switch 模式匹配漏 case → sealed 穷举才免 default
- **坑 5**：Text Block 缩进理解（以最后行缩进为基准剥离）→ 行尾位置决定前导空格

## Checklist

- [ ] 新项目基线 17/21，能列出各版本标志特性
- [ ] record + sealed + switch 模式组合建模会用
- [ ] 升级清单（javax/反射/依赖）有预案
- [ ] 虚拟线程按 25 篇评估启用
