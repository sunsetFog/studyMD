# 12 · 外部化配置：yml、多环境与配置优先级

## 1. 配置源与优先级（低 → 高）

```
 1 默认属性 SpringApplication.setDefaultProperties
 2 @Configuration 类上的 @PropertySource
 3 配置文件 application.yml / .properties
 4 RandomValuePropertySource（random.*）
 5 操作系统环境变量
 6 Java 系统属性（System.getProperties）
 7 JNDI 属性
 8 ServletContext / ServletConfig 参数
 9 SPRING_APPLICATION_JSON（内嵌 JSON）
10 命令行参数 --key=value          ★ 最高
11 测试环境 @TestPropertySource / @DynamicPropertySource
```

> 记忆口诀：**命令行 > 环境变量 > 配置文件**。容器化部署时环境变量（`SPRING_DATASOURCE_URL`）非常好用。

松散规则：环境变量用下划线大写 `SPRING_DATASOURCE_URL` ↔ 配置 `spring.datasource.url`。

## 2. yml vs properties

| 对比 | yml | properties |
|---|---|---|
| 层级 | 缩进，清晰 | 全名，冗长 |
| 数组 | 支持 | 支持但丑 |
| 中文 | 支持 UTF-8 | 需转义 |

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo
    username: root
server:
  port: 8080
my:
  list:
    - a
    - b
  map:
    k1: v1
```

> 注意：同目录**同时存在** `application.yml` 与 `application.properties` 时，**properties 优先级更高**（Boot 2.x 起）。

## 3. 多环境（Profile）三种写法

### ① 多文件（推荐）

```
application.yml          公共
application-dev.yml
application-test.yml
application-prod.yml
```
```yaml
# application.yml
spring:
  profiles:
    active: dev
```

### ② 单文件文档块

```yaml
server:
  port: 8080
---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8081
---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```

### ③ Profile 专属配置组（Boot 2.4+）

```yaml
spring:
  profiles:
    group:
      prod: proddb,prodmq      # 激活 prod 时同时激活 proddb、prodmq
```

激活方式优先级：`spring.profiles.active`（配置文件） < `SPRING_PROFILES_ACTIVE`（环境变量） < `--spring.profiles.active`（命令行）。

## 4. 配置文件加载位置（低 → 高）

```
1 classpath:/
2 classpath:/config/
3 file:./
4 file:./config/
5 file:./config/*/      （子目录）
6 外部指定：--spring.config.location=/etc/app/
            --spring.config.additional-location=/etc/app/   ★ 追加而非覆盖
```

> 生产推荐：`java -jar app.jar --spring.config.additional-location=/opt/app/config/`，jar 内保留默认配置，外部只覆盖差异。

Boot 2.4+ 支持 `spring.config.import`：

```yaml
spring:
  config:
    import:
      - optional:file:./local.yml
      - optional:nacos:app.yml       # 配置中心
      - classpath:extra.yml
```

## 5. 读取配置的四种方式

```java
// ① @Value：单个值，支持默认值与 SpEL
@Value("${app.name:demo}") private String name;
@Value("${app.port:#{8080}}") private int port;
@Value("#{'${my.list}'.split(',')}") private List<String> list;

// ② Environment
@Autowired Environment env;
env.getProperty("app.name", String.class, "demo");

// ③ @ConfigurationProperties：★ 推荐，类型安全、可校验、可分组
@ConfigurationProperties(prefix = "app.sms")
@Validated @Data
public class SmsProperties { @NotBlank private String key; }

// ④ 构造器绑定（record，不可变）
@ConfigurationProperties("app.pool")
public record PoolProps(int max, Duration timeout) {}
```

| 方式 | 松散绑定 | 元数据/提示 | 校验 | 适用场景 |
|---|---|---|---|---|
| `@Value` | 部分 | 无 | 无 | 少量零散值 |
| `@ConfigurationProperties` | ✅ | ✅ | ✅ | 一组相关配置 |

## 6. 配置加密与敏感信息

- 数据库密码不要明文：用 **Jasypt** 或 **K8s Secret / Vault / Nacos 加密配置**。
- Jasypt 示例：

```xml
<dependency><groupId>com.github.ulisesbocchio</groupId>
           <artifactId>jasypt-spring-boot-starter</artifactId><version>3.0.5</version></dependency>
```
```yaml
jasypt:
  encryptor:
    password: ${JASYPT_PASSWORD}     # 从环境变量读，别进代码库
spring:
  datasource:
    password: ENC(AbCdEf123==)
```

- Actuator 默认会隐藏敏感值，但 `/env` 端点**不要暴露到公网**。

## 7. 配置刷新

- 原生配置**不支持运行时热更新**（除 `@ConfigurationProperties` 配合 Spring Cloud `@RefreshScope` / Nacos 自动刷新）。
- Spring Cloud 环境用 Nacos Config 可自动刷新（见 [45](45-config-center.md)）。

## 8. 随机数与占位符引用

```yaml
my:
  secret: ${random.value}
  id: ${random.int[1000,9999]}
  uuid: ${random.uuid}
app:
  desc: ${spring.application.name}@${server.port}    # 引用其他配置
```

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| yml 缩进错误 | 配置不生效/启动失败 | 用空格，禁止 Tab；同层对齐 |
| `@Value` 注入 static | null | 不支持，改实例字段或 setter |
| `@Value` 无默认值 | 启动报 `Could not resolve placeholder` | 写 `:default` |
| 配置改了不生效 | 被命令行/环境变量覆盖 | 检查所有来源，看 `/actuator/env` |
| `application.properties` 覆盖 yml | 疑惑 | 同目录 properties 优先 |
| 数组写法错误 | 只解析出一项 | 用 `- ` 列表或 `a[0].x=` |
| 中文乱码（properties） | 乱码 | 用 yml，或 IDE 设 UTF-8 + `\u` 转义 |

## 10. 面试速答

- **Boot 配置优先级**：命令行 > 环境变量 > 配置文件 > 默认值；同目录 properties 优先于 yml。
- **@Value 与 @ConfigurationProperties**：前者单个值、无校验；后者按前缀绑定整组、支持松散绑定与 JSR 校验，推荐。
- **多环境怎么切**：`spring.profiles.active` + `application-{profile}.yml`，生产用命令行或环境变量覆盖。
