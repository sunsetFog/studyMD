# 02 · 安装 JDK 与第一个程序

## 1. 选择发行版

| 发行版 | 提供方 | 特点 |
|---|---|---|
| Oracle JDK | Oracle | 商用需授权（NFTC 之后免费但有限制） |
| **Eclipse Temurin** | Adoptium | 社区首选，免费，推荐 |
| Amazon Corretto | AWS | 长期免费，云上常用 |
| Azul Zulu | Azul | 支持广泛，有 Prime 商业版 |
| GraalVM | Oracle | 支持 Native Image、多语言 |
| OpenJDK | 社区 | 上游源码，各发行版基于它构建 |

推荐：**Temurin 21 LTS** 或 **Corretto 21**。

## 2. 安装（macOS）

```bash
# Homebrew
brew install --cask temurin@21
/usr/libexec/java_home -V          # 列出已安装 JDK
export JAVA_HOME=$(/usr/libexec/java_home -v 21)
export PATH="$JAVA_HOME/bin:$PATH"
```

Linux（Debian/Ubuntu）：

```bash
sudo apt install -y openjdk-21-jdk
sudo update-alternatives --config java
```

## 3. 环境变量

| 变量 | 作用 |
|---|---|
| `JAVA_HOME` | JDK 根目录，Maven/Gradle/IDEA 依赖它 |
| `PATH` | 加入 `$JAVA_HOME/bin` 才能直接用 `javac` |
| `CLASSPATH` | 类路径（**建议不设置**，用 `-cp` 显式指定，避免污染） |
| `JAVA_OPTS` / `_JAVA_OPTIONS` | 默认 JVM 参数（不同工具读取规则不同，慎用） |

验证：

```bash
java -version
javac -version
echo $JAVA_HOME
```

## 4. JDK 自带工具

| 命令 | 用途 |
|---|---|
| `javac` | 编译 `.java` → `.class` |
| `java` | 启动 JVM 运行类 / jar / 单文件源码 |
| `jar` | 打包（`jar --create --file app.jar -C out .`） |
| `javadoc` | 生成 API 文档 |
| `javap` | 反汇编字节码 |
| `jshell` | 交互式 REPL（Java 9+） |
| `jlink` | 裁剪自定义运行时 |
| `jpackage` | 打包成 exe/dmg/deb |
| `jps/jstat/jmap/jstack` | 运行时诊断（见 JVM 篇） |

## 5. 编译与运行细节

```bash
# 标准流程
javac -d out src/com/example/Hello.java
java -cp out com.example.Hello arg1 arg2

# 指定语言级别（避免用到高版本 API）
javac --release 17 -d out src/**/*.java

# 打成可执行 jar
jar --create --file hello.jar --main-class com.example.Hello -C out .
java -jar hello.jar

# 单文件源码模式（Java 11+）
java Hello.java
```

### 类路径规则

- `-cp` / `-classpath` 支持目录和 jar，多个用 `:`（Windows 用 `;`）分隔。
- JVM 查找类顺序：**bootstrap → extension/platform → classpath**（详见类加载篇）。
- 同名类在多个 jar 中出现时，**先找到的生效**（jar 顺序敏感，常见「依赖冲突」根因）。

## 6. jshell 快速实验

```java
jshell> int x = 1 + 2
x ==> 3
jshell> List.of(1,2,3).stream().map(i -> i * 2).toList()
$2 ==> [2, 4, 6]
/exit
```

适合验证 API 行为，不适合写多文件工程。

## 7. 目录结构约定（Maven 标准）

```
project/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/            # 源码
    │   └── resources/       # 配置、静态资源
    └── test/
        ├── java/            # 测试代码
        └── resources/
```

编译产物在 `target/classes`、`target/test-classes`。

## 8. Hello World 的完整拆解

```java
package com.example;              // 1. 包声明，必须首行（注释除外）

import java.time.LocalDate;       // 2. 导入，java.lang 无需导入

public class Hello {              // 3. 类名与文件名一致（public 类）
    public static void main(String[] args) {   // 4. 入口签名固定
        System.out.println("Hello, " + args.length + " args, today " + LocalDate.now());
    }
}
```

要点：

- `public static void main(String[] args)` 是 JVM 约定入口；`String... args` 也可。
- 一个文件只能有**一个 public 顶级类**，但可有多个非 public 类。
- 类名首字母大写（帕斯卡），包名全小写、反向域名。

## 9. 常见启动错误

| 报错 | 原因 | 解决 |
|---|---|---|
| `Could not find or load main class` | 包名/类路径不对 | 用全限定名 + 正确 `-cp` |
| `UnsupportedClassVersionError` | 编译版本高于运行 JVM | 升级 JVM 或 `--release` 降级 |
| `NoClassDefFoundError` | 运行期缺依赖 jar | 检查 classpath/依赖冲突 |
| `javac: invalid target release` | JDK 版本低于 target | 换 JDK |
| 中文乱码 | 源文件编码与编译编码不一致 | `javac -encoding UTF-8`，文件存 UTF-8 |

## 10. 推荐工具链

- **IDE**：IntelliJ IDEA（社区版免费）
- **构建**：Maven / Gradle
- **版本管理**：SDKMAN!（`sdk install java 21.0.5-tem`）
- **诊断**：Arthas、JFR、VisualVM
