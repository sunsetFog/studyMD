# 01 · Java 是什么

## 1. 一句话定义

Java 是一门**静态类型、面向对象、编译 + 解释混合执行、带自动内存管理**的通用编程语言，核心卖点是「一次编写，到处运行」（Write Once, Run Anywhere）。

## 2. 技术体系三代名词

| 名词 | 含义 | 说明 |
|---|---|---|
| **JDK** | Java Development Kit | 开发工具包：javac、jar、javadoc、jdb + JRE |
| **JRE** | Java Runtime Environment | 运行环境：JVM + 核心类库 |
| **JVM** | Java Virtual Machine | 执行字节码的虚拟机，屏蔽操作系统差异 |

关系：`JDK ⊃ JRE ⊃ JVM`。

> Java 11 之后官方不再单独发布 JRE 安装包，但 `jlink` 可以自定义裁剪运行时。

## 3. 执行流程

```
Hello.java  --javac-->  Hello.class(字节码)  --java-->  JVM 解释/JIT 编译 --> 机器码执行
```

关键点：

1. **javac 只做前端**：词法/语法分析 → 语义分析 → 生成字节码，不做全局优化。
2. **JVM 负责后端**：解释执行 + 热点代码 JIT 编译（C1/C2）+ 逃逸分析 + 内联。
3. 字节码是**平台中立**的，但 JVM 是平台相关的。

查看字节码：

```bash
javac Hello.java
javap -c -p Hello.class     # 反汇编
javap -v Hello.class        # 常量池、方法表等完整信息
```

## 4. 语言特性清单

- **面向对象**：封装、继承、多态；但 `int` 等 8 种基本类型不是对象（性能取舍）。
- **静态类型 + 强类型**：编译期检查，隐式转换受限。
- **自动内存管理**：GC 回收，无手动 free；仍有 OOM/内存泄漏风险（引用未释放）。
- **单继承多实现**：类单继承，接口多实现，避免 C++ 菱形继承问题。
- **泛型是编译期糖**：类型擦除，运行时无泛型信息。
- **值传递**：Java **只有值传递**，对象传递的是引用的副本。
- **无指针运算**：但有引用，安全性更高。
- **丰富的生态**：Spring、MyBatis、Netty、Kafka、Elasticsearch……

## 5. 平台版本

| 平台 | 用途 |
|---|---|
| Java SE | 标准版，桌面/命令行/基础库 |
| Java EE → Jakarta EE | 企业版，Servlet、JPA、JMS（现由 Eclipse 基金会维护） |
| Java ME | 嵌入式（已被 Android/物联网方案取代） |
| Java Card | 智能卡 |

## 6. 版本与 LTS

- **LTS（长期支持）**：8、11、17、21、25。生产选 LTS。
- **非 LTS**：每 6 个月一个特性版本（9、10、12…），适合尝鲜。
- **发布节奏**：每年 3 月、9 月各一版。

```java
// 查看当前版本
public class Version {
    public static void main(String[] args) {
        System.out.println(System.getProperty("java.version"));
        System.out.println(System.getProperty("java.vendor"));
        System.out.println(Runtime.version());
    }
}
```

## 7. Java 的对手与定位

| 语言 | 场景 | 对比 |
|---|---|---|
| Go | 云原生、中间件 | 启动快、内存小；生态与 JVM 调优工具不如 Java |
| Kotlin | Android、后端 | 与 Java 100% 互操作，语法更简洁 |
| C# | Windows/.NET | 语言特性接近，生态不同 |
| Rust | 系统级 | 无 GC、零成本抽象，学习曲线陡 |

Java 的护城河：**JVM 生态 + 成熟中间件 + 海量人才与轮子**。

## 8. 第一个可运行程序（不建工程）

```java
// Hello.java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, Java " + Runtime.version().feature());
    }
}
```

```bash
javac Hello.java && java Hello
# Java 11+ 可直接单文件运行（源码启动模式）
java Hello.java
```

## 9. 常见误区

1. ❌「Java 是解释型语言」——是**编译成字节码 + 运行时解释/JIT**。
2. ❌「JVM 只有 HotSpot」——还有 OpenJ9、GraalVM（可 AOT 编译成原生镜像）。
3. ❌「JDK 版本 = 语言版本」——语法由 `--release`/`source` 决定，运行在更高 JVM 上也可行（向下兼容）。
4. ❌「Java 慢」——现代 JIT 后接近 C++，慢的是启动和内存占用（GraalVM Native Image 可缓解）。
