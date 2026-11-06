# 02 · 快速上手

## 一、环境准备

| 组件 | 版本要求 |
|---|---|
| JDK | 17+（推荐 21 LTS） |
| Maven | 3.8+（或用内置 mvnw） |
| IDE | IntelliJ IDEA（装 Spring 插件） |

验证：`java -version`、`mvn -v`

## 二、创建项目的三种方式

1. **Spring Initializr**（推荐）：https://start.spring.io 或 IDEA 内置向导
2. **Maven 手动搭建**：写 parent 即可
3. **CLI**：`spring init -d web,data-jpa demo`

## 三、首个应用

### pom.xml 核心

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.4</version>
    <relativePath/>
</parent>

<properties>
    <java.version>17</java.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

> `parent` 的作用：统一依赖版本仲裁 + 插件默认配置 + 资源过滤（application.yml 中可用 `@..@` 占位）。若公司有自定义 parent，改用 `dependencyManagement` 导入 `spring-boot-dependencies` BOM。

### 启动类

```java
@SpringBootApplication  // = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

三个注解（详见 35/36 篇原理）：
- `@SpringBootConfiguration`：本质是 `@Configuration`，标记配置类
- `@EnableAutoConfiguration`：开启自动配置
- `@ComponentScan`：扫描**启动类所在包及子包**

### 第一个接口

```java
@RestController
public class HelloController {

    @GetMapping("/hello")
    public Map<String, String> hello(@RequestParam(defaultValue = "world") String name) {
        return Map.of("message", "hello " + name);
    }
}
```

启动运行：`mvn spring-boot:run`，访问 `http://localhost:8080/hello?name=spring`

## 四、标准项目结构

```
com.example.demo
├── DemoApplication.java
├── controller/        # Web 层：参数接收、响应封装
├── service/           # 业务层：接口 + impl
├── mapper/  (或 repository/)  # 数据层
├── entity/  (或 domain/)      # 数据库实体
├── dto/               # 传输对象（入参/出参）
├── config/            # 配置类
├── common/            # 统一响应、异常、工具
│    ├── result/
│    ├── exception/
│    └── util/
└── aspect/            # 切面
resources
├── application.yml
├── application-dev.yml / application-prod.yml
├── mapper/            # MyBatis XML
└── static/ templates/
```

**分层铁律**：
- Controller 不写业务，Service 不碰 HttpServletRequest
- Entity 不出 Service 层，出参一律 DTO（防敏感字段泄露 + 解耦表结构）
- 依赖方向：controller → service → mapper，禁止反向

## 五、常用 starter 速查

| starter | 用途 |
|---|---|
| spring-boot-starter-web | MVC + Tomcat + Jackson |
| spring-boot-starter-validation | 参数校验 |
| spring-boot-starter-data-jpa | JPA/Hibernate |
| mybatis-spring-boot-starter | MyBatis（第三方） |
| spring-boot-starter-data-redis | Redis |
| spring-boot-starter-security | 安全 |
| spring-boot-starter-actuator | 监控端点 |
| spring-boot-starter-test | 测试全家桶 |

> spring 官方 starter 命名 `spring-boot-starter-*`，第三方 `*-spring-boot-starter`（如 `mybatis-spring-boot-starter`）。

## 六、devtools 热重载

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

改动后 `Ctrl+F9`（Build）触发重启，原理是双 ClassLoader 快速重启（非 JVM 热替换）。生产包会自动禁用。

## 常见坑

- **坑 1**：启动类放包外层导致 Bean 扫描不到 → 启动类必须在根包
- **坑 2**：改了配置/代码没生效 → 确认是否在 target/classes 编译输出；IDEA 开启 `Build project automatically`
- **坑 3**：端口占用 `Web server failed to start. Port 8080...` → `server.port` 修改或 `lsof -i:8080` 杀进程
- **坑 4**：Lombok 在 IDEA 不生效 → 装插件 + 开启 annotation processing

## Checklist

- [ ] 会用 Initializr 创建项目并解释 parent/starter 作用
- [ ] 启动类位于根包，三层结构清晰
- [ ] 第一个接口可正常访问
- [ ] devtools 已配置（开发环境）
