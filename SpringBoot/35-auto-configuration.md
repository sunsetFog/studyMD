# 35 · 自动配置原理

## 一、入口：三个注解的合成

```java
@SpringBootApplication
  = @SpringBootConfiguration      // 配置类身份
  + @EnableAutoConfiguration      // 开启自动配置（核心）
  + @ComponentScan                // 扫描业务包
```

`@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 导入选择器——**SpringBoot 的魔法在 Selector 里**。

## 二、自动配置执行流程

```
1. 启动时 AutoConfigurationImportSelector 执行
2. 读取 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
   （Boot 2.7+ 新位置，旧版是 spring.factories）
   ——文件列出全部候选自动配置类（200+ 个）
3. 去重、排除（exclude 指定的）、过滤
4. 逐个跑条件注解判断：@ConditionalOnClass / OnMissingBean / OnProperty ...
5. 条件全满足的配置类其 @Bean 方法执行 → Bean 注册
6. 用户自定义 Bean 优先：@ConditionalOnMissingBean 保证不覆盖
```

## 三、条件注解详解

| 注解 | 语义 |
|---|---|
| @ConditionalOnClass(X.class) | classpath 有该类才生效（**starter 的桥梁**） |
| @ConditionalOnMissingClass | 无该类 |
| @ConditionalOnBean / OnMissingBean | 容器有/无该 Bean |
| @ConditionalOnProperty(name, havingValue) | 配置开关 |
| @ConditionalOnWebApplication | Web 环境 |
| @ConditionalOnResource | 资源存在 |

**自动配置类的典型结构**：

```java
@AutoConfiguration                          // Boot 2.7+，替代 @Configuration
@ConditionalOnClass(DataSource.class)       // 引了 jdbc 依赖才装配
@EnableConfigurationProperties(DataSourceProperties.class)   // 绑定配置
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean               // 用户自定义则让位（关键扩展点）
    public DataSource dataSource(DataSourceProperties props) {
        return DataSourceBuilder.create()
                .url(props.getUrl())
                .build();
    }
}
```

## 四、自动配置 vs 用户配置的博弈

```
用户 Bean（@Component/@Bean）   ← 最高优先级，自动配置让路（OnMissingBean）
配置文件属性                    ← properties 绑定覆盖默认值
自动配置默认值                  ← 兜底
```

这就是「**约定优于配置，但配置随时可覆盖**」的实现：`application.yml` 写了 `spring.datasource.url`，properties 绑定进 DataSourceProperties，自动配置用它建 Bean；你若自己 @Bean 一个 DataSource，自动配置直接退场。

## 五、自定义 starter（原理落地）

### 命名规范

- 官方：`spring-boot-starter-xxx`
- 自定义：`xxx-spring-boot-starter`（如 `sms-spring-boot-starter`）

### 结构

```
sms-spring-boot-starter/
├── pom.xml（依赖 spring-boot-autoconfigure，打成普通 jar）
└── src/main/
    ├── java/com/example/sms/
    │   ├── SmsService.java              # 功能实现
    │   ├── SmsProperties.java           # @ConfigurationProperties(prefix="sms")
    │   └── SmsAutoConfiguration.java    # 自动配置
    └── resources/META-INF/spring/
        └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
        └── 内容一行：com.example.sms.SmsAutoConfiguration
```

```java
@AutoConfiguration
@ConditionalOnClass(SmsClient.class)
@EnableConfigurationProperties(SmsProperties.class)
public class SmsAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(name = "sms.enabled", havingValue = "true", matchIfMissing = true)
    public SmsService smsService(SmsProperties props) {
        return new SmsService(props);
    }
}
```

使用方只需：引 starter + 配置 `sms.app-key=xxx` → `@Autowired SmsService` 可用。

### 多模块拆分（规范做法）

```
xxx-spring-boot-starter      # 空聚合模块，只做依赖引入
xxx-spring-boot-autoconfigure  # 自动配置逻辑
```

## 六、调试自动配置

```yaml
debug: true        # 或启动 --debug
```

输出 CONDITION EVALUATION REPORT：
```
Positive matches（生效的自动配置）
Negative matches（被条件排除的，附原因）
```

排查「某个 Bean 为什么没装配」第一步就开 debug 看报告。

## 七、从「怎么配」到「为什么」的知识闭环

| 现象 | 原理对应 |
|---|---|
| 引 starter-web 就有 Spring MVC | DispatcherServletAutoConfiguration + 内嵌 Tomcat（37 篇） |
| application.yml 的端口能改 server.port | properties 绑定 ServerProperties → ServletWebServerFactory |
| 自己定义 DataSource 后默认配置消失 | @ConditionalOnMissingBean |
| 引了不兼容的库启动报错 | OnClass 判断失败或自动配置内异常 |

## 常见坑

- **坑 1**：自定义类不在 @ComponentScan 包内又没写进 imports → 不装配
- **坑 2**：@ConditionalOnMissingBean 判断时机问题 → 同一自动配置类内顺序敏感；跨配置类用 @AutoConfigureBefore/After
- **坑 3**：starter 里同时依赖 spring-boot-starter（含 autoconfigure）导致重复配置类 → starter 模块只放依赖声明
- **坑 4**：Boot 2.7 迁移到 3 时还往 spring.factories 写自动配置 → 已不读；迁移到新 imports 文件

## Checklist

- [ ] 能画出自动配置执行的六步流程
- [ ] 理解 OnMissingBean 的「用户优先」机制
- [ ] 会写一个标准 starter（imports 文件位置正确）
- [ ] 排查装配问题先开 debug 看条件报告
