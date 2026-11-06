# 06 · 类与对象

## 一、类的完整结构

```java
public class User {

    // ── 字段（成员变量）──
    private String username;
    private int age;

    // ── 静态常量 ──
    public static final int MAX_AGE = 150;

    // ── 静态变量（类级，所有实例共享）──
    private static int userCount = 0;

    // ── 静态代码块：类加载时执行一次 ──
    static {
        System.out.println("类初始化");
    }

    // ── 实例代码块：每次构造前执行 ──
    {
        System.out.println("实例初始化");
    }

    // ── 构造器 ──
    public User(String username, int age) {
        this.username = username;
        this.age = age;
        userCount++;
    }

    public User(String username) {
        this(username, 0);            // this(...) 调用其他构造器，必须第一行
    }

    // ── 方法 ──
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
}
```

**执行顺序**：静态代码块（类加载，一次）→ 实例代码块 → 构造器。

## 二、对象创建与内存

```java
User u = new User("tom", 18);
```

```
栈：u 存对象的引用（地址）
堆：User 对象 {username→"tom", age:18}
方法区/元空间：类信息（Class 对象、方法元数据）
```

`null` 调用成员 → NullPointerException。

## 三、this 关键字

```java
this.username = username;    // 区分成员变量与参数
this(username, 0);           // 构造器内调用重载构造器
return this;                 // 返回自身（Builder 链式）
obj.compareTo(this);         // 把当前对象作为参数传递
```

## 四、static 详解

```java
static int count = 0;              // 类变量：属于类，不属于实例
static void staticMethod() { }     // 静态方法：无 this，不能访问实例成员

User.count;        // 推荐通过类名访问
u.count;           // 合法但误导（实际仍访问类变量），禁止写法

// static 方法不能重写（静态绑定）——是「隐藏」不是多态
// static 在工具类/常量/工厂方法中合理使用；泛滥则说明设计有问题
```

**static 典型场景**：常量（`Math.PI`）、工具方法（`Math.max`）、工厂方法（`Integer.valueOf`）、单例（36 篇）、实例计数。

## 五、JavaBean 规范（getter/setter）

```java
public class UserDTO {
    private String username;
    private boolean active;

    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }

    public boolean isActive() { return active; }      // boolean 用 is 前缀
    public void setActive(boolean active) { this.active = active; }
}
```

 getter/setter 价值：控制可变性、未来加校验/通知不破坏调用方。**Lombok 自动生成**：`@Data`、`@Getter/@Setter`、`@Builder`（38 篇）。

## 六、包与导入

```java
package com.example.demo.model;      // 首行，倒置域名命名

import java.util.List;               // 单个导入
import java.util.*;                  // 通配（不递归子包）
import static java.lang.Math.max;    // 静态导入：直接写 max(1,2)
```

包 = 命名空间 + 访问控制单元。

## 七、访问修饰符

| 修饰符 | 同类 | 同包 | 子类 | 任意 |
|---|---|---|---|---|
| private | ✓ | ✗ | ✗ | ✗ |
| （默认/包私有） | ✓ | ✓ | ✗ | ✗ |
| protected | ✓ | ✓ | ✓ | ✗ |
| public | ✓ | ✓ | ✓ | ✓ |

原则：**字段全部 private，方法按需放宽**。最小可见性。

## 八、对象销毁与 finalize

- Java 无手动析构；GC 自动回收不可达对象
- `finalize()` 已废弃（Java 9+），不可依赖
- 需要确定性释放的资源（文件/连接）→ try-with-resources + AutoCloseable（26 篇）

## 常见坑

- **坑 1**：static 方法里用 this / 访问实例成员 → 编译错误
- **坑 2**：构造器里调用可被重写的方法 → 子类重写方法在父类构造时执行，子类字段未初始化
- **坑 3**：boolean 的 isXxx() 配合 JSON 序列化 → 部分框架去掉 is 前缀导致字段名错位
- **坑 4**：静态变量被多线程修改 → 并发问题；计数用 AtomicInteger（21 篇）
- **坑 5**：构造器参数与字段同名忘记 this → 参数赋给自身

## Checklist

- [ ] 记住初始化顺序：静态块 → 实例块 → 构造器
- [ ] this(...) 必须是构造器第一行
- [ ] 字段 private + 最小可见性
- [ ] static 慎用，优先实例方法
