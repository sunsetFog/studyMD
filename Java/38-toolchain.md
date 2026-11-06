# 38 · 工具链：Maven、Lombok、测试与质量

## 一、Maven：依赖与构建标准

### 坐标与生命周期

```xml
<groupId>com.example</groupId>      <!-- 组织倒置域名 -->
<artifactId>demo</artifactId>       <!-- 项目名 -->
<version>1.0.0</version>            <!-- SNAPSHOT 开发版 / RELEASE 正式版 -->
<packaging>jar</packaging>
```

```
生命周期：validate → compile → test → package → verify → install → deploy
常用命令：
mvn clean package          # 打包
mvn clean install          # 装入本地仓库
mvn test                   # 跑测试
mvn dependency:tree        # 依赖树（排查冲突神器）
```

### 依赖管理三件套

```xml
<!-- 1. 传递依赖与作用域 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.3.0</version>
    <scope>runtime</scope>      <!-- compile/test/provided/runtime/system -->
</dependency>

<!-- 2. 统一版本仲裁（多模块/BOM） -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.3.4</version>
            <type>pom</type><scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- 3. 排除冲突依赖 -->
<exclusions><exclusion>
    <groupId>org.slf4j</groupId><artifactId>slf4j-log4j12</artifactId>
</exclusion></exclusions>
```

**依赖冲突原则**：最短路径优先 + 先声明优先。`NoSuchMethodError` 十有八九是版本冲突 → dependency:tree 排除。

### 多模块工程

```
parent（统一版本/插件）
├── common        工具/DTO
├── domain        领域模型
├── service       业务
└── web           启动入口
```

## 二、Gradle 对比（了解级）

```groovy
// 构建脚本更灵活（Kotlin/Groovy DSL），Android 生态默认
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
// 国内企业主流仍是 Maven，新项目可评估 Gradle
```

## 三、Lombok：样板代码终结者

```java
@Data                     // getter+setter+toString+equals+hashCode（注意 equals 陷阱）
@Getter @Setter           // 精细控制
@Builder                  // 建造者
@AllArgsConstructor @NoArgsConstructor
@RequiredArgsConstructor   // final 字段构造器 —— 构造注入标配
@Value                    // 不可变类（替代简单 record 场景）
@Slf4j                    // 日志对象
@SneakyThrows             // 偷抛受检异常（慎用）
```

**注意**：`@Data` 在含可变集合/父类时 equals 语义复杂 → 领域对象手写或用 record；DTO/配置类放心用。

## 四、单元测试（JUnit 5 + AssertJ + Mockito）

```java
class CalculatorTest {

    @Test
    @DisplayName("除数为零抛异常")
    void divide_byZero() {
        Calculator c = new Calculator();
        assertThatThrownBy(() -> c.divide(1, 0))
                .isInstanceOf(ArithmeticException.class)
                .hasMessageContaining("zero");
    }

    @ParameterizedTest              // 参数化
    @CsvSource({"2,3,5", "0,0,0", "-1,1,0"})
    void add(int a, int b, int expected) {
        assertThat(new Calculator().add(a, b)).isEqualTo(expected);
    }

    @Test
    void withMock() {
        var repo = mock(UserRepository.class);
        when(repo.findById(1L)).thenReturn(Optional.of(new User("tom")));
        assertThat(service.getName(1L)).isEqualTo("tom");
        verify(repo).findById(1L);
    }
}
```

### JMH 微基准（性能测试唯一正解）

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
public class StringConcatBench {
    @Benchmark
    public String stringBuilder() {
        return new StringBuilder().append("a").append("b").toString();
    }
}
// mvn jmh:benchmark —— 别再手写循环计时比较性能（34 篇 JIT 会骗你）
```

## 五、代码质量工具链

| 工具 | 用途 |
|---|---|
| Checkstyle / Spotless | 风格统一（IDE 配置 or CI 卡点） |
| SpotBugs / SonarQube | 静态缺陷扫描（NPE/资源泄漏/并发问题） |
| Error Prone | 编译期抓常见错误 |
| JaCoCo | 覆盖率报告（CI 门禁） |
| ArchUnit | 架构约束测试（分层规则写进测试） |

```java
// ArchUnit 示例：守护分层
@ArchTest
static ArchRule controllersShouldNotAccessMappers =
    noClasses().that().resideInAPackage("..controller..")
        .should().dependOnClassesThat().resideInAPackage("..mapper..");
```

## 六、调试技巧

```
IDEA：
- 条件断点：右键断点填 condition（只在大 id 时停）
- Evaluate Expression：Alt+F8 现场求值
- 异常断点：任何 NPE 抛出处自动停
- Stream 调试：断点面板 Stream Trace 看每步数据
远程调试：-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
```

## 常见坑

- **坑 1**：依赖冲突排查无头绪 → 先 `mvn dependency:tree -Dincludes=冲突组`
- **坑 2**：@Data 用在含继承的领域对象 → equals 破坏契约；@EqualsAndHashCode(callSuper = true)
- **坑 3**：测试依赖真实 DB/网络 → mock 隔离；集成测试 Testcontainers
- **坑 4**：手写循环基准测性能 → JMH
- **坑 5**：SNAPSHOT 依赖导致构建不稳定 → 发布版锁定

## Checklist

- [ ] dependency:tree 排冲突肌肉记忆
- [ ] Lombok 注解按对象类型正确选择
- [ ] 单测 AAA + 参数化 + mock 边界
- [ ] CI 质量门禁（覆盖率/静态扫描）就位
