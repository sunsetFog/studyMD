# 01 · 环境搭建

## 一、JDK / JRE / JVM

```
JDK = JRE + 开发工具（javac/javadoc/jdb...）
JRE = JVM + 核心类库
JVM = 字节码虚拟机（「一次编译，到处运行」的基石）
```

```java
Hello.java --javac--> Hello.class（字节码）--JVM--> 各平台运行
```

## 二、版本选择（2026 视角）

| 版本 | 定位 |
|---|---|
| Java 8 | 存量巨大，老项目维护 |
| Java 11 LTS | 部分企业过渡版本 |
| **Java 17 LTS** | SpringBoot 3 最低要求，主流基准 |
| **Java 21 LTS** | 最新 LTS：虚拟线程、Record Pattern |
| Java 25 LTS | 2025 年发布的新 LTS 线 |

**新项目选 21**；生态兼容性优先（如老框架）选 17。SpringBoot 3.x / 最新 Spring 全家桶均支持两者。

## 三、安装

### macOS（推荐 SDKMAN 或 Homebrew）

```bash
# SDKMAN：多版本管理，切换方便
curl -s "https://get.sdkman.io" | bash
sdk install java 21.0.5-tem        # Temurin 发行版
sdk use java 21.0.5-tem

# Homebrew
brew install --cask temurin@21
```

### Windows / Linux

- 下载 Temurin（Eclipse Adoptium，推荐）或 Oracle JDK
- 配置 `JAVA_HOME` 并把 `%JAVA_HOME%\bin` 加入 PATH

### 验证

```bash
java -version      # openjdk version "21.0.5" ...
javac -version
```

**发行版怎么选**：Temurin（免费开源，社区主流）、Oracle JDK（商用需授权）、GraalVM（含原生镜像能力）、Amazon Corretto / Alibaba Dragonwell（云厂商优化）。纯语法学习用 Temurin 即可——**所有发行版都基于 OpenJDK，功能等价**。

## 四、IDE

| 工具 | 说明 |
|---|---|
| IntelliJ IDEA（推荐） | 社区版够用（Community），Ultimate 支持 Spring |
| VS Code | Extension Pack for Java，轻量 |
| Vim/Emacs | LSP（jdtls） |

IDEA 必开设置：`Settings → Editor → General → Auto Import`（自动导包）、显示行号、编码 UTF-8。

## 五、第一个程序

```java
// Hello.java —— 文件名必须与 public 类名一致
public class Hello {
    public static void main(String[] args) {   // JVM 入口，签名固定
        System.out.println("Hello, Java " + System.getProperty("java.version"));
    }
}
```

```bash
javac Hello.java   # 编译 → Hello.class
java Hello         # 运行（Java 11+ 可直接 java Hello.java 单文件运行）
```

**关键理解**：
- `main` 的 `static`：JVM 无需创建对象即可调用入口
- `String[] args`：命令行参数
- 一个源文件只能有一个 public 类，且类名 = 文件名

## 六、JShell（快速实验）

Java 9+ 内置 REPL，验证语法片段无需建项目：

```bash
jshell
jshell> 2 + 2
$1 ==> 4
jshell> var list = List.of("a", "b")
jshell> /exit
```

## 七、项目脚手架

- **Maven**（国内主流）：`mvn archetype:generate` 或 IDEA New Project → Maven
- **Gradle**：`gradle init`
- 详见 38 篇工具链

## 常见坑

- **坑 1**：文件名与 public 类名不一致 → 编译错误
- **坑 2**：`java Hello.class` → 错误！直接 `java Hello`
- **坑 3**：Windows 命令行中文乱码 → `chcp 65001` 切 UTF-8
- **坑 4**：多版本 JDK 冲突 → 确认 JAVA_HOME 指向；用 SDKMAN 管理
- **坑 5**：IDEA 的 JDK 与系统 JAVA_HOME 不一致 → Project Structure 里统一

## Checklist

- [ ] JDK 17/21 安装完成，java -version 正确
- [ ] IDEA 配置好 JDK 与编码
- [ ] 第一个程序编译运行成功
- [ ] 理解 JDK/JRE/JVM 关系与字节码跨平台机制
