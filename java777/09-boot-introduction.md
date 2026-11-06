# 09 · Spring Boot 入门：约定优于配置与 Starter

## 1. Boot 解决了什么

| Spring 的痛点 | Boot 的方案 |
|---|---|
| 大量 XML/Java 配置 | **自动配置**（Auto Configuration） |
| 依赖版本冲突 | **Starter + BOM 依赖管理** |
| 需要外置 Tomcat | **内嵌服务器**（Tomcat/Jetty/Undertow） |
| 监控/健康检查要自己搭 | **Actuator** |
| 配置散落 | **统一 application.yml + 外部化配置** |

核心理念：**约定优于配置（Convention over Configuration）**——你只管写业务，框架按默认值把一切配好；要改就改配置。

## 2. 三个核心特性

```
① 自动配置：根据 classpath 上的类、已有 Bean、配置文件，推断并注册需要的 Bean
② 起步依赖 Starter：一个依赖搞定一整块功能（web / data-jpa / redis / security）
③ 内嵌容器 + 可执行 jar：java -jar 直接跑
```

## 3. SpringBootApplication 注解拆解

```java
@SpringBootApplication
public class App { public static void main(String[] a) { SpringApplication.run(App.class, a); } }
```

等价于三个注解：

```java
@SpringBootConfiguration     // 就是 @Configuration，标记该类为配置类（且是主配置源）
@EnableAutoConfiguration     // ★ 开启自动配置（导入 AutoConfigurationImportSelector）
@ComponentScan               // ★ 扫描「启动类所在包及其子包」
```

> **启动类必须放在根包**（如 `com.example`），否则子模块的 `@Service` 扫不到——这是最常见的启动即失败原因。
> 需要扫描外部包：加 `@ComponentScan(basePackages = {...})` 或 `@SpringBootApplication(scanBasePackages = {...})`。

## 4. 父 POM 与依赖管理

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.5</version>
</parent>
```

它做了什么：
- 继承 `spring-boot-dependencies`（**BOM**，锁定了 300+ 常用依赖的兼容版本）
- 默认 Java 17、UTF-8 编码
- Maven 插件配置（repackage、资源过滤）

不用 parent 时：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.3.5</version>
            <type>pom</type><scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 5. 常用 Starter 一览

| Starter | 功能 |
|---|---|
| `spring-boot-starter-web` | Spring MVC + Tomcat + Jackson |
| `spring-boot-starter-webflux` | WebFlux + Netty |
| `spring-boot-starter-data-jpa` | Hibernate + Spring Data JPA |
| `spring-boot-starter-data-redis` | Lettuce + RedisTemplate |
| `spring-boot-starter-data-mongodb` / `-elasticsearch` | NoSQL |
| `spring-boot-starter-jdbc` | HikariCP + JdbcTemplate |
| `spring-boot-starter-validation` | Hibernate Validator |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-aop` | AOP + AspectJ |
| `spring-boot-starter-amqp` | RabbitMQ |
| `spring-boot-starter-actuator` | 监控 |
| `spring-boot-starter-test` | JUnit5 + Mockito + AssertJ + MockMvc |
| `spring-boot-starter-log4j2` | 替换 Logback |
| `spring-boot-starter-cache` | 缓存抽象 |
| `spring-boot-starter-mail` | 邮件 |
| `spring-boot-starter-quartz` | 定时任务 |
| `spring-boot-starter-oauth2-resource-server` | JWT 资源服务器 |
| `mybatis-spring-boot-starter` | MyBatis（第三方） |

命名规范：
- 官方：`spring-boot-starter-xxx`
- 第三方：`xxx-spring-boot-starter`（如 `mybatis-spring-boot-starter`）

## 6. 排除自动配置

```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class,
                                  RedisAutoConfiguration.class})
```

```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

> 排除 `DataSourceAutoConfiguration` 可让无数据库的项目正常启动（否则报 `Failed to configure a DataSource`）。

## 7. 内嵌容器切换

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion><groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-starter-tomcat</artifactId></exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

| 容器 | 特点 |
|---|---|
| Tomcat | 默认，生态最成熟 |
| Jetty | 轻量，长连接友好 |
| Undertow | 高性能，内存占用低 |

## 8. 打包与运行

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <excludes><exclude><groupId>org.projectlombok</groupId>
                          <artifactId>lombok</artifactId></exclude></excludes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

```bash
mvn clean package                    # 生成 fat jar
java -jar app.jar
java -jar app.jar --spring.profiles.active=prod
java -jar app.jar --server.port=9090
java -Dspring.config.location=/etc/app/ -jar app.jar
```

jar 结构：`BOOT-INF/classes`（你的代码）+ `BOOT-INF/lib`（依赖）+ `org/springframework/boot/loader`（启动器）。

## 9. 启动过程一览

```
new SpringApplication()
  ├── 推断应用类型（SERVLET / REACTIVE / NONE）
  ├── 加载 ApplicationContextInitializer / ApplicationListener（spring.factories）
  └── 推断主类
run()
  ├── 计时开始
  ├── 发布 ApplicationStartingEvent
  ├── 准备 Environment（读取配置、决定 profile）
  ├── 打印 Banner
  ├── createApplicationContext
  ├── prepareContext（加载启动类 BeanDefinition）
  ├── refreshContext  ★ 自动配置在此生效（AutoConfigurationImportSelector）
  ├── 调用 ApplicationRunner / CommandLineRunner
  └── 发布 ApplicationReadyEvent
```

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 启动类位置不对 | Bean 扫不到 | 移到根包或显式 `scanBasePackages` |
| 无数据源却引入 JPA | 启动失败 | 排除 `DataSourceAutoConfiguration` 或加 H2 |
| 端口被占用 | `Port 8080 was already in use` | `server.port` 或 `lsof -i :8080` |
| 依赖版本冲突 | `NoSuchMethodError` | 不要手动覆盖 Boot 管理的版本 |
| `mvn package` 无主清单 | `no main manifest attribute` | 加 `spring-boot-maven-plugin` |
| Boot 2 迁移 Boot 3 | 一堆 `javax` 报错 | 全局替换 `javax.*` → `jakarta.*`，JDK 17+ |

## 11. 面试速答

- **Boot 的核心优势**：自动配置、起步依赖、内嵌容器、Actuator、外部化配置。
- **Starter 是什么**：一个聚合了某功能所需全部依赖 + 自动配置类的 Maven 模块（POM 里只有一个 `pom` 依赖，真正的自动配置在 `spring-boot-autoconfigure` 里）。
- **为什么不用配版本**：parent 继承了 `spring-boot-dependencies` BOM，锁定兼容版本。
