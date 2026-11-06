# 03 · 配置体系

## 一、配置文件格式与优先级

SpringBoot 支持 `properties` / `yaml` / `yml`。同一位置同时存在时 **properties 优先于 yml**。

加载优先级（高 → 低，后者被覆盖）：

```
1. 命令行参数          --server.port=9090
2. 环境变量            SERVER_PORT=9090
3. jar 包外 application-{profile}.yml
4. jar 包外 application.yml
5. jar 包内 application-{profile}.yml
6. jar 包内 application.yml
7. @PropertySource 指定的文件
8. 默认属性（SpringApplication.setDefaultProperties）
```

> 外部化配置的意义：**一个 jar 包，不同环境换不同外置配置即可**，无需重新打包。

### YAML 语法要点

```yaml
server:
  port: 8080          # 冒号后必须有空格
  servlet:
    context-path: /

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: "123456"      # 特殊字符/纯数字建议加引号
    driver-class-name: com.mysql.cj.jdbc.Driver

logging:
  level:
    com.example.demo: debug   # 层级写法
```

## 二、读取配置的四种方式

### 1. @Value（零散单值）

```java
@Value("${app.upload-dir}")
private String uploadDir;

// 默认值
@Value("${app.timeout:30}")
private int timeout;
```

### 2. @ConfigurationProperties（推荐：一组配置绑定为一个对象）

```yaml
app:
  jwt:
    secret: my-secret-key-please-change-32bytes
    expire-minutes: 120
    header: Authorization
```

```java
@Component
@ConfigurationProperties(prefix = "app.jwt")
@Data
public class JwtProperties {
    private String secret;
    private int expireMinutes = 60;   // 支持默认值
    private String header = "Authorization";
}
```

> 优点：类型安全、IDE 提示、可校验、支持 relaxed-binding（`expire-minutes` 自动映射 `expireMinutes`）。

### 3. Environment（动态 key）

```java
@Autowired
private Environment env;
String port = env.getProperty("server.port", "8080");
```

### 4. @Configuration + @Bean 注册第三方对象

```java
@Configuration
@EnableConfigurationProperties(JwtProperties.class)
public class AppConfig { ... }
```

## 三、Profile 多环境

### 方式一：多文件

```
application.yml            # 公共配置 + spring.profiles.active
application-dev.yml        # 开发
application-test.yml       # 测试
application-prod.yml       # 生产
```

```yaml
# application.yml
spring:
  profiles:
    active: dev    # 默认环境；生产由启动参数覆盖
```

```bash
java -jar app.jar --spring.profiles.active=prod
```

### 方式二：单文件多段（少用）

```yaml
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8080
---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```

### 按环境装配 Bean

```java
@Profile("prod")
@Bean
public DataSource prodDataSource() { ... }
```

## 四、配置占位符与随机数

```yaml
app:
  name: demo
  desc: "应用：${app.name}"            # 引用其他配置
  secret: ${random.uuid}               # 随机 UUID
  port: ${random.int[1024,65535]}
  db-password: ${DB_PASSWORD:default}  # 引用环境变量，带默认值
```

## 五、配置文件加密：jasypt

```yaml
spring:
  datasource:
    password: ENC(xJ8s9dFk3L...)
```

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
```

启动时传密钥：`-Djasypt.encryptor.password=xxx`。**密钥绝不能写进配置文件**。

## 六、配置中心（简述，详见 33 篇）

配置量大、需要动态刷新时，把配置外移到 Nacos/Apollo：

```yaml
spring:
  config:
    import: nacos:app-config.yaml
```

配合 `@RefreshScope` 实现不重启更新。

## 常见坑

- **坑 1**：yml 缩进用 Tab → 解析失败。**YAML 只允许空格**
- **坑 2**：密码含 `#`、`:` 未加引号 → 被当注释/分隔符截断
- **坑 3**：@Value 在 static 字段或无 Spring 管理的类中不生效
- **坑 4**：`spring.profiles.active` 写在 profile 分段文件里 → 无效（active 只能在主配置/命令行指定）
- **坑 5**：@ConfigurationProperties 类没加 `@Component`（或没 @EnableConfigurationProperties）→ 注入为 null

## Checklist

- [ ] 统一使用 yml，公共配置在 application.yml，差异在 profile 文件
- [ ] 一组相关配置用 @ConfigurationProperties 绑定，不散用 @Value
- [ ] 生产敏感信息走环境变量或 jasypt，不落明文
- [ ] 生产通过外置配置文件/命令行覆盖，jar 保持环境无关
