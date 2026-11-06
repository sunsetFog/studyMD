# 04 · IDEA、Maven 与 Gradle

## 1. IntelliJ IDEA 必备操作

| 场景 | 快捷键（macOS） |
|---|---|
| 全局搜索类 | `⌘ O` |
| 全局搜索文件 | `⇧ ⌘ O` |
| 全局文本搜索 | `⇧ ⌘ F` |
| 查找用法 | `⌥ F7` |
| 跳转定义 | `⌘ B` |
| 生成代码（getter/构造器） | `⌘ N` |
| 重构重命名 | `⇧ F6` |
| 格式化 | `⌥ ⌘ L` |
| 优化 import | `⌃ ⌥ O` |
| 运行 | `⌃ R` |
| 调试 | `⌃ D` |
| 求值表达式 | `⌥ F8`（Debug 中） |
| 提取变量/方法 | `⌥ ⌘ V` / `⌥ ⌘ M` |

调试技巧：

- **条件断点**：右键断点设置条件，避免循环里狂点。
- **Evaluate Expression**：可执行任意代码，包括调用方法改状态。
- **Drop Frame**：回退栈帧重跑当前方法。
- **远程调试**：`-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005`。

## 2. Maven 核心概念

### 2.1 坐标

```xml
<groupId>com.acme</groupId>
<artifactId>order-service</artifactId>
<version>1.0.0-SNAPSHOT</version>
<packaging>jar</packaging>
```

- `SNAPSHOT`：快照版本，可被重复覆盖发布。
- `RELEASE`：正式版本，发布后不可变（Nexus 会拒绝覆盖）。

### 2.2 生命周期

**三套独立生命周期**，执行后面的阶段会自动执行前面的：

| 生命周期 | 阶段（顺序） |
|---|---|
| clean | pre-clean → clean → post-clean |
| **default** | validate → initialize → compile → test → package → verify → install → deploy |
| site | pre-site → site → post-site → site-deploy |

常用：

```bash
mvn clean package            # 清理 + 编译 + 测试 + 打包
mvn clean package -DskipTests           # 跳过测试执行
mvn clean package -Dmaven.test.skip=true # 跳过测试编译
mvn install -DskipTests      # 安装到本地仓库 ~/.m2/repository
mvn dependency:tree          # 依赖树
mvn dependency:tree -Dincludes=org.slf4j   # 只看某依赖
mvn help:effective-pom       # 查看生效的 pom（排查继承）
mvn versions:display-dependency-updates
```

### 2.3 依赖作用域

| scope | 编译 | 测试 | 运行 | 打包 | 典型用途 |
|---|---|---|---|---|---|
| `compile`（默认） | ✔ | ✔ | ✔ | ✔ | 通用库 |
| `provided` | ✔ | ✔ | ✘ | ✘ | servlet-api、lombok |
| `runtime` | ✘ | ✔ | ✔ | ✔ | JDBC 驱动 |
| `test` | ✘ | ✔ | ✘ | ✘ | junit、mockito |
| `system` | ✔ | ✔ | ✔ | ✘ | 本地 jar（不推荐） |
| `import` | — | — | — | — | BOM 导入（仅 dependencyManagement） |

### 2.4 依赖冲突与调解

Maven 采用**「最短路径优先，同深度先声明优先」**：

```
A → B → C 1.0
A → C 2.0        ← 深度 1，胜出
```

排查三板斧：

```bash
mvn dependency:tree -Dverbose -Dincludes=com.fasterxml.jackson.core
mvn dependency:analyze          # 找出未使用/未声明依赖
```

解决方式：

```xml
<!-- 1. 排除 -->
<exclusions>
  <exclusion>
    <groupId>commons-logging</groupId>
    <artifactId>commons-logging</artifactId>
  </exclusion>
</exclusions>

<!-- 2. 统一版本（推荐 BOM） -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.3.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

### 2.5 常用插件

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <configuration>
        <release>21</release>
        <parameters>true</parameters>   <!-- 保留参数名，Spring/MyBatis 需要 -->
      </configuration>
    </plugin>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <version>3.2.5</version>
    </plugin>
  </plugins>
</build>
```

打可执行 fat jar（Spring Boot）：

```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

### 2.6 settings.xml 关键配置

```xml
<settings>
  <localRepository>~/.m2/repository</localRepository>
  <mirrors>
    <mirror>
      <id>aliyun</id>
      <mirrorOf>central</mirrorOf>
      <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
  </mirrors>
  <servers>
    <server>
      <id>releases</id>
      <username>deploy</username>
      <password>******</password>
    </server>
  </servers>
</settings>
```

## 3. Gradle 基础

### 3.1 build.gradle（Kotlin DSL 推荐）

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.3.0"
}

group = "com.acme"
version = "1.0.0"

java {
    toolchain { languageVersion = JavaLanguageVersion.of(21) }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    compileOnly("org.projectlombok:lombok:1.18.32")
    annotationProcessor("org.projectlombok:lombok:1.18.32")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
}

tasks.test {
    useJUnitPlatform()
}
```

### 3.2 配置阶段 vs 执行阶段

- `implementation`：不传递给消费者（编译不可见），**优于 `api`**，减少重编译。
- `api`：暴露给下游（仅 `java-library` 插件）。
- 常用命令：

```bash
./gradlew build --scan
./gradlew dependencies --configuration runtimeClasspath
./gradlew clean test
./gradlew bootRun
./gradlew --refresh-dependencies
```

### 3.3 Maven vs Gradle

| 维度 | Maven | Gradle |
|---|---|---|
| 配置 | XML，约定优于配置 | Groovy/Kotlin DSL，灵活 |
| 性能 | 一般（无增量缓存） | 快（增量构建、构建缓存、daemon） |
| 学习成本 | 低 | 中 |
| 生态 | 最广 | Android 事实标准 |
| 可读性 | 冗长但统一 | 简洁但各项目风格差异大 |

## 4. 多模块工程

```
parent/
├── pom.xml            <packaging>pom</packaging>
├── common/            # 工具、DTO
├── domain/            # 领域模型，依赖 common
├── infra/             # 数据访问，依赖 domain
└── app/               # 启动模块，依赖 infra
```

`parent/pom.xml`：

```xml
<modules>
  <module>common</module>
  <module>domain</module>
  <module>infra</module>
  <module>app</module>
</modules>
```

子模块：

```xml
<parent>
  <groupId>com.acme</groupId>
  <artifactId>parent</artifactId>
  <version>1.0.0</version>
</parent>
```

原则：**依赖方向单向**（app → infra → domain → common），禁止循环依赖。

## 5. 常见问题

| 问题 | 原因 | 解决 |
|---|---|---|
| `Could not resolve dependencies` | 私服地址/网络/坐标错 | `mvn -U` 强制更新，检查 settings.xml |
| 编译报「找不到符号」但代码存在 | 依赖作用域错（provided/test） | 调整 scope |
| Lombok 不生效 | 未开注解处理 / 版本不兼容 JDK | 加 `annotationProcessor`，升级 Lombok |
| 打包后运行 `no main manifest attribute` | 未配置 main-class | 用 shade/spring-boot 插件 |
| IDEA 与命令行结果不一致 | IDEA 用自己的编译器/缓存 | 改用 Maven 编译或 Reimport |
