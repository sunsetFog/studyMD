# 10 · Boot 快速上手：项目结构与第一个接口

## 1. 创建项目

三种方式：
1. **start.spring.io**（推荐）：选 Boot 3.x、Java 17、Maven、依赖 Web/Lombok。
2. IDEA → New Project → Spring Initializr。
3. 命令行：`curl https://start.spring.io/starter.zip -d dependencies=web,lombok -o demo.zip`。

## 2. 标准目录结构

```
demo/
├── pom.xml
├── src/main/java/com/example/demo/
│   ├── DemoApplication.java        ★ 启动类（放根包）
│   ├── config/                     配置类（MybatisConfig、WebConfig…）
│   ├── controller/                 控制器
│   ├── service/                    业务接口
│   │   └── impl/                   业务实现
│   ├── mapper/ 或 repository/      数据访问
│   ├── domain/
│   │   ├── entity/                 数据库实体（DO）
│   │   ├── dto/                    入参
│   │   └── vo/                     出参
│   ├── convert/                    MapStruct 转换器
│   ├── exception/                  自定义异常 + 全局处理
│   ├── util/                       工具
│   └── common/                     Result、常量
├── src/main/resources/
│   ├── application.yml
│   ├── application-dev.yml
│   ├── application-prod.yml
│   ├── static/  templates/
│   └── mapper/*.xml               MyBatis 映射
└── src/test/java/...
```

> 分层建议：`controller → service（接口+impl）→ mapper → DB`，**禁止 controller 直接调 mapper**（后期加缓存/事务无处安放）。

## 3. 一个完整的 REST 接口

```java
// 统一返回体
@Data @NoArgsConstructor @AllArgsConstructor
public class Result<T> implements Serializable {
    private int code; private String msg; private T data;
    public static <T> Result<T> ok(T data) { return new Result<>(200, "OK", data); }
    public static <T> Result<T> fail(String msg) { return new Result<>(500, msg, null); }
}

@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
@Validated
public class UserController {
    private final UserService userService;

    @GetMapping("/{id}")
    public Result<UserVO> get(@PathVariable Long id) {
        return Result.ok(userService.getById(id));
    }

    @PostMapping
    public Result<Long> create(@Valid @RequestBody UserDTO dto) {
        return Result.ok(userService.create(dto));
    }

    @DeleteMapping("/{id}")
    public Result<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return Result.ok(null);
    }
}
```

## 4. application.yml 最小配置

```yaml
server:
  port: 8080
  servlet:
    context-path: /api        # 可选，注意会影响 Actuator 路径
spring:
  application:
    name: demo
  profiles:
    active: dev
  jackson:
    default-property-inclusion: non_null
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
```

## 5. 自定义 Banner 与关闭

`src/main/resources/banner.txt`：

```
  ___ _ __  _ __ ___  ___
 / __| '_ \| '__/ _ \/ _ \
 \__ \ |_) | | |  __/  __/
 |___/ .__/|_|  \___|\___|
     |_|
${application.version}
```

```yaml
spring:
  main:
    banner-mode: off   # console / log / off
```

## 6. 常用启动参数与配置

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(App.class);
        app.setBannerMode(Banner.Mode.OFF);
        app.setLazyInitialization(true);        // 懒加载，加快启动（生产慎用）
        app.setAdditionalProfiles("dev");
        app.setWebApplicationType(WebApplicationType.SERVLET);  // NONE 可做定时任务
        app.run(args);
    }
}
```

命令行参数优先级**最高**：`--server.port` > `application.yml` > 默认值。

## 7. 热部署（devtools）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional><scope>runtime</scope>
</dependency>
```

- 修改代码自动重启（只重载开发者的类，很快）。
- 配合 IDEA：**Build Project 自动** + `compiler.automake.allow.when.app.running`。
- 生产环境会被自动禁用（fat jar 中 devtools 不生效）。

## 8. 三种运行方式

```bash
mvn spring-boot:run                          # 开发
java -jar target/app.jar                      # 生产
mvn clean package -DskipTests && java -jar ... # 跳过测试
```

IDE 直接运行 `main` 方法（最常用）。

## 9. 基础配置文件模板（可直接抄）

```yaml
# application.yml
spring:
  profiles:
    active: @activatedProperties@          # 配合 Maven profile
---
# application-dev.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 10
  jpa:
    show-sql: true
logging:
  level:
    com.example: debug
```

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 404 | 接口找不到 | 检查 `@RestController`、`@RequestMapping` 路径、`context-path` |
| 中文乱码 | 响应乱码 | `server.servlet.encoding.charset=UTF-8` + MySQL URL 加参数 |
| 端口冲突 | 启动失败 | 改端口或杀进程 |
| 改代码不生效 | 缓存 | devtools 或手动重启；IDEA 需 `Build` |
| 依赖改了不生效 | 类找不到 | Maven `Reload All Projects` |
| Lombok 报红 | 找不到 getter | 安装 Lombok 插件 + 开启 Annotation Processors |
| 时间差 8 小时 | 时区 | `spring.jackson.time-zone: Asia/Shanghai` 与 MySQL `serverTimezone` |

## 11. 面试速答

- **Boot 项目启动做了什么**：加载配置 → 创建上下文 → 执行自动配置 → 启动内嵌 Tomcat → 发布 Ready 事件。
- **Controller 与 RestController**：后者 = `@Controller` + `@ResponseBody`，返回值直接写 JSON。
- **devtools 为什么生产不生效**：以 fat jar 运行时 devtools 自动禁用，`restart` 类加载器不启用。
