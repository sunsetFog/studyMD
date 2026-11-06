# 03 · 程序结构、包与命名规范

## 1. 一个 Java 源文件的组成

```java
// 1) 包声明（0 或 1 条，必须第一行有效代码）
package com.example.demo;

// 2) 导入（可多条，支持静态导入）
import java.util.List;
import java.util.Map;
import static java.util.Objects.requireNonNull;

// 3) 类型声明（类/接口/枚举/记录/注解，可有多个，public 最多一个）
public class Demo {
    // 成员：字段 → 构造器 → 方法 → 内部类
}
```

合法但少见的写法：

```java
class A { }        // 包级私有
class B { }        // 同一文件多个非 public 类
// public class C {}  // 只有与文件名同名的类才能 public
```

## 2. 包（package）

作用：命名空间隔离、访问控制、便于模块化与按包发布。

命名规范：**反向域名 + 小写**，如 `com.alibaba.fastjson2`、`org.springframework.core`。

```java
package com.acme.order.service;   // 目录必须对应 com/acme/order/service
```

> 目录结构与包名不一致时，`javac` 仍能编译（找得到文件即可），但**运行期按包名查目录**，会造成混乱，务必保持一致。

### 包的可见性

包级私有（无修饰符）成员只能被**同一个包**中的类访问——这是 Java 里最容易被忽略的封装层级。

## 3. import 规则

| 写法 | 说明 |
|---|---|
| `import java.util.List;` | 单类型导入 |
| `import java.util.*;` | 按需导入（不影响性能，只影响可读性） |
| `import static java.lang.Math.max;` | 静态导入（常量/静态方法） |
| `java.lang.*` | 自动导入 |
| 同包类 | 自动可见，无需导入 |

注意：

- `*` **不会**递归导入子包。
- 同名类冲突时必须用全限定名，如 `java.util.Date` vs `java.sql.Date`。
- 静态导入滥用会降低可读性，建议仅用于常量与断言（`assertEquals`）。

## 4. 类体成员顺序（约定）

```java
public class Order {
    // 1. 静态常量
    public static final int MAX_SIZE = 100;
    // 2. 静态变量
    private static int counter;
    // 3. 实例变量
    private Long id;
    private List<Item> items = new ArrayList<>();
    // 4. 静态初始化块
    static { counter = 0; }
    // 5. 实例初始化块
    { id = 0L; }
    // 6. 构造器（由少到多，互相调用）
    public Order() { this(0L); }
    public Order(Long id) { this.id = id; }
    // 7. 公共方法 → 私有方法
    public Long getId() { return id; }
    // 8. 内部类
    static class Item { }
}
```

## 5. main 方法

```java
public static void main(String[] args) { }
```

- `public`：JVM 需要能从外部调用。
- `static`：无需实例化。
- `void`：无返回值。
- 参数：`String[]` 或 `String...`，命令行参数以空格分隔，含空格的参数需引号。
- Java 21 预览支持**实例 main 方法**（JEP 445，后续版本演进为未命名类），了解即可。

```bash
java Demo a "b c" d
# args = ["a", "b c", "d"]
```

## 6. 命名规范（Oracle 官方约定）

| 元素 | 规则 | 示例 |
|---|---|---|
| 包 | 全小写，反向域名 | `com.acme.user` |
| 类/接口/枚举/记录 | 大驼峰 | `OrderService`、`Runnable` |
| 方法/变量 | 小驼峰 | `calculateTotal`、`itemCount` |
| 常量 | 全大写 + 下划线 | `MAX_RETRY_COUNT` |
| 泛型参数 | 单大写字母 | `T`、`E`、`K`、`V` |
| 布尔字段 | 不加 `is` 前缀（避免序列化问题） | `active` 而非 `isActive` |
| 实现类 | 接口名 + Impl | `UserServiceImpl` |
| 异常 | 以 Exception 结尾 | `BizException` |
| 测试 | 被测类 + Test | `OrderServiceTest` |

避免：拼音命名、单字母变量（循环 i/j 除外）、`$`、无意义缩写（`tmp1`、`data2`）。

## 7. 注释与 Javadoc

```java
/**
 * 订单服务：负责下单、取消与查询。
 *
 * @author ivory
 * @since 1.0
 */
public class OrderService {

    /**
     * 创建订单。
     *
     * @param userId 用户 ID，不能为 null
     * @param items  商品列表，不能为空
     * @return 新建订单，永不为 null
     * @throws IllegalArgumentException 参数非法时抛出
     */
    public Order create(Long userId, List<Item> items) { ... }
}
```

常用标签：`@param`、`@return`、`@throws`、`@see`、`@deprecated`、`@since`、`{@code xxx}`、`{@link Foo}`。

原则：

- 注释解释**为什么**，代码表达**做什么**。
- 不要写 `// i++ 自增` 这类废话。
- 公开 API 必须有 Javadoc；实现细节用行内注释。
- 修改代码时同步修改注释，过期注释比没有更危险。

## 8. 编码与文件规范

- 源文件统一 **UTF-8**，编译加 `-encoding UTF-8`。
- 缩进 4 空格（Google 风格 2 空格），不用 Tab。
- 单行不超过 120 字符。
- 行尾不留空格；文件末尾留一个空行。
- 换行符统一 LF（`.gitattributes` 配置）。

## 9. 标识符规则（语法层面）

- 首字符：字母、`_`、`$`（后两者不建议）。
- 后续：字母、数字、`_`、`$`、Unicode 字母（可用中文但不建议）。
- 不能是关键字/保留字：`class`、`interface`、`enum`、`record`、`sealed`(上下文)、`var`(上下文)、`true`、`false`、`null` 等。
- 大小写敏感：`User` ≠ `user`。

## 10. 一个规范的小程序

```java
package com.acme.demo;

import java.util.Arrays;

/**
 * 命令行计算器：支持 sum / max。
 */
public final class Calculator {

    private static final String USAGE = "usage: java Calculator <sum|max> 1 2 3";

    private Calculator() { }   // 工具类禁止实例化

    public static void main(String[] args) {
        if (args.length < 2) {
            System.err.println(USAGE);
            System.exit(1);
        }
        int[] nums = Arrays.stream(args, 1, args.length)
                           .mapToInt(Integer::parseInt)
                           .toArray();
        int result = switch (args[0]) {
            case "sum" -> Arrays.stream(nums).sum();
            case "max" -> Arrays.stream(nums).max().orElseThrow();
            default -> throw new IllegalArgumentException("unknown op: " + args[0]);
        };
        System.out.println(result);
    }
}
```
