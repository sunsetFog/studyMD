# 59 · 模块系统（JPMS）

## 1. 为什么需要模块

Java 8 及之前的问题：

1. **classpath 地狱**：无法表达依赖，缺 jar 运行时才报错。
2. **包冲突**：同名类在不同 jar 中，先找到的生效。
3. **无强封装**：反射可访问任何 private 成员。
4. **JDK 过大**：无法裁剪，最小运行时也有 200MB+。
5. **启动慢**：需扫描大量 jar。

## 2. 模块定义

```java
// src/main/java/module-info.java
module com.acme.order {
    requires java.base;                 // 隐式，可省略
    requires java.sql;
    requires transitive com.acme.common;   // 传递依赖（使用方也可见）
    requires static lombok;                // 编译期必需，运行期可选

    exports com.acme.order.api;            // 导出包
    exports com.acme.order.spi to com.acme.plugin;   // 限定导出

    opens com.acme.order.entity to com.fasterxml.jackson.databind;  // 允许反射
    opens com.acme.order.internal;         // 对所有人开放反射

    uses com.acme.order.spi.PaymentProvider;        // 声明使用 SPI
    provides com.acme.order.spi.PaymentProvider     // 提供服务实现
        with com.acme.order.alipay.AlipayProvider;

    // 传递打开（Java 9+）
}
```

## 3. 指令详解

| 指令 | 含义 |
|---|---|
| `requires X` | 依赖模块 X |
| `requires transitive X` | 依赖并传递给使用本模块的模块 |
| `requires static X` | 编译期依赖（运行期可选） |
| `exports pkg` | 导出包（其他模块可访问 public 成员） |
| `exports pkg to M` | 仅对模块 M 导出 |
| `opens pkg` | 允许运行期反射访问所有成员 |
| `opens pkg to M` | 仅对 M 开放反射 |
| `uses S` | 声明使用服务接口 S（配合 ServiceLoader） |
| `provides S with Impl` | 提供 S 的实现 |

### 关键区分：exports vs opens

| | exports | opens |
|---|---|---|
| 编译期访问 | ✔ | ✘ |
| 运行期反射（public） | ✔ | ✔ |
| 运行期反射（private） | ✘ | ✔ |
| 典型用途 | API 包 | 实体类（给 Jackson/Hibernate） |

## 4. 模块路径 vs 类路径

```bash
# 模块路径（有 module-info 的 jar）
java --module-path mods --module com.acme.app/com.acme.app.Main

# 类路径（传统）
java -cp "libs/*" com.acme.app.Main

# 自动模块（无 module-info 的 jar 放在 module-path 上）
# 模块名由 jar 文件名推导（a-b-1.0.jar → a.b）
```

> 自动模块 `requires` 所有模块，且 `exports` 所有包——兼容旧库的过渡机制。

## 5. 服务加载（ServiceLoader）

```java
// 服务接口模块
module com.acme.spi {
    exports com.acme.spi;
}

// 提供者模块
module com.acme.alipay {
    requires com.acme.spi;
    provides com.acme.spi.PaymentProvider with com.acme.alipay.AlipayProvider;
}

// 使用方
module com.acme.app {
    requires com.acme.spi;
    uses com.acme.spi.PaymentProvider;
}
```

```java
ServiceLoader<PaymentProvider> loader = ServiceLoader.load(PaymentProvider.class);
for (PaymentProvider p : loader) {
    if (p.supports("alipay")) return p.pay(req);
}
```

> 传统 classpath 方式需 `META-INF/services/` 文件；模块方式用 `provides ... with`。

## 6. jlink：裁剪运行时

```bash
jlink --module-path $JAVA_HOME/jmods:mods \
      --add-modules com.acme.app \
      --strip-debug \
      --compress=2 \
      --no-header-files \
      --no-man-pages \
      --output myruntime

# 体积可从 200MB 降到 40-60MB
myruntime/bin/java --module com.acme.app/com.acme.app.Main
```

## 7. jpackage：打包安装包

```bash
jpackage --type dmg \
         --name MyApp \
         --input target/ \
         --main-jar app.jar \
         --main-class com.acme.app.Main \
         --runtime-image myruntime \
         --icon app.icns
```

支持：exe、msi（Windows）、dmg、pkg（macOS）、deb、rpm（Linux）。

## 8. 反射与强封装

Java 9+ 默认禁止反射访问其他模块的非 public 成员：

```java
Field f = String.class.getDeclaredField("value");
f.setAccessible(true);      // InaccessibleObjectException
```

解决：

```bash
--add-opens java.base/java.lang=ALL-UNNAMED
--add-exports java.base/sun.nio.ch=ALL-UNNAMED
--add-modules jdk.unsupported
```

> 迁移到 Java 17+ 时，很多老框架（反射操作 JDK 内部类）需要加这些参数。

## 9. 未命名模块与自动模块

| 类型 | 说明 |
|---|---|
| 命名模块 | 有 `module-info.java` |
| 自动模块 | 无 module-info 但在 module-path 上 |
| 未命名模块 | classpath 上的所有代码 |

规则：

- 未命名模块可以读取所有模块（兼容旧代码）。
- 命名模块**不能** requires 未命名模块（只能通过 `--add-reads`）。
- 自动模块 exports 所有包、requires 所有模块。

## 10. 迁移策略

### 策略 1：暂不模块化（推荐多数项目）

- 保持 classpath，仅升级 JDK。
- 用 `--add-opens` 解决反射问题。
- Spring Boot 应用通常不需要 module-info。

### 策略 2：自底向上

1. 先模块化基础库（无依赖的 common）。
2. 逐层向上（依赖关系决定顺序）。
3. 应用层最后（可保持未命名模块）。

### 策略 3：自顶向下

先给应用加 module-info（`requires` 所有依赖，用自动模块），再逐步细化。

### 常见问题

| 问题 | 解决 |
|---|---|
| 三方库无 module-info | 用自动模块，或找 `module-info` 补丁库 |
| 反射失败 | `opens` 或 `--add-opens` |
| 拆分包（split package） | 两个模块导出同名包 → 合并或重命名 |
| 资源访问 | `getResource` 在模块内需 `opens` 或 `exports` |
| JUnit 测试 | 测试代码在未命名模块，需 `--add-reads` |

## 11. 模块化收益与代价

**收益**：

- 强封装（真正的 private）。
- 可靠的依赖声明（启动即验证）。
- 可裁剪运行时（jlink）。
- 更快的启动（无需扫描 classpath）。
- 更清晰的架构边界。

**代价**：

- 迁移成本高（三方库兼容）。
- 反射框架需额外配置。
- 学习曲线。
- 生态支持不完整（很多库仍用 classpath）。

## 12. 实战：多模块项目

```
project/
├── common/                  module com.acme.common
├── domain/                  module com.acme.domain (requires common)
├── infra/                   module com.acme.infra (requires domain, java.sql)
└── app/                     module com.acme.app (requires infra)
```

`domain/src/main/java/module-info.java`：

```java
module com.acme.domain {
    requires transitive com.acme.common;
    exports com.acme.domain.model;
    exports com.acme.domain.service;
}
```

`app/src/main/java/module-info.java`：

```java
module com.acme.app {
    requires com.acme.infra;
    requires java.logging;
    opens com.acme.app.config to spring.core;   // 若用 Spring（通常不用模块化）
}
```

## 13. 常见坑

1. 在 module-info 中 `requires` 了自动模块（名字由 jar 名推导，易错）。
2. 忘记 `opens` 导致 Jackson/Hibernate 反射失败。
3. split package（两个模块同包）→ 编译错误。
4. `exports` 了包但类依赖未导出的类型（编译报错）。
5. 测试代码访问模块内包（需 `--add-reads`/`--add-opens`）。
6. 用 `Class.forName` 加载模块内类（需正确模块名）。
7. 资源文件路径（模块内用 `getResourceAsStream` 需包路径）。
8. jlink 漏了 `--add-modules` 导致运行时缺模块。
9. 认为模块化能替代依赖管理工具（Maven 仍负责下载）。
10. Spring Boot fat jar 与模块化冲突（嵌套 jar 不是模块）。

## 14. 速查

| 需求 | 指令 |
|---|---|
| 依赖模块 | `requires X` |
| 传递依赖 | `requires transitive X` |
| 导出 API | `exports pkg` |
| 允许反射 | `opens pkg` |
| 服务提供 | `provides S with Impl` |
| 服务使用 | `uses S` |
| 裁剪运行时 | `jlink` |
| 打包安装程序 | `jpackage` |
| 开放 JDK 内部 | `--add-opens` |
| 查看模块 | `java --list-modules` |
| 描述模块 | `java --describe-module X` |
