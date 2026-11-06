# 22 · 自定义 Starter 与 @Enable 模式

## 1. Starter 的两部分

```
xxx-spring-boot-starter        只是一个空 POM，聚合依赖（不含代码）
xxx-spring-boot-autoconfigure  真正的自动配置代码（也可合并到 starter 中）
```

| 组成 | 内容 |
|---|---|
| `AutoConfiguration` 类 | `@AutoConfiguration` + `@Conditional*` + `@Bean` |
| `XxxProperties` | `@ConfigurationProperties` 配置绑定 |
| `spring/...AutoConfiguration.imports` | **Boot 3 必需**的注册文件 |
| `spring-configuration-metadata.json` | 可选，IDE 提示 |
| `spring.factories` | Boot 2 方式（3.x 已废弃用于自动配置） |

## 2. 实战：短信 Starter

### ① starter POM（空壳）

```xml
<project>
  <artifactId>sms-spring-boot-starter</artifactId>
  <dependencies>
    <dependency>
      <groupId>com.example</groupId>
      <artifactId>sms-spring-boot-autoconfigure</artifactId>
      <version>1.0.0</version>
    </dependency>
  </dependencies>
</project>
```

### ② autoconfigure 模块

```xml
<dependencies>
  <dependency><groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-autoconfigure</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-configuration-processor</artifactId><optional>true</optional></dependency>
</dependencies>
```

### ③ 属性类

```java
@ConfigurationProperties(prefix = "sms")
@Data
public class SmsProperties {
    /** 是否启用 */
    private boolean enabled = true;
    private String accessKey;
    private String secret;
    private Duration timeout = Duration.ofSeconds(3);
    private int maxRetry = 2;
}
```

### ④ 核心服务

```java
public class SmsTemplate {
    private final SmsProperties props;
    public SmsTemplate(SmsProperties props) { this.props = props; }
    public void send(String phone, String tplId, Map<String,String> params) { /* 调 SDK */ }
}
```

### ⑤ 自动配置类

```java
@AutoConfiguration                                  // Boot 3
@ConditionalOnProperty(prefix = "sms", name = "enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(SmsProperties.class)
public class SmsAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                        // ★ 用户自定义则跳过
    public SmsTemplate smsTemplate(SmsProperties props) {
        return new SmsTemplate(props);
    }
}
```

### ⑥ 注册文件（Boot 3 关键）

```
src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```
内容：
```
com.example.sms.autoconfigure.SmsAutoConfiguration
```

> Boot 2 写法（了解即可）：`META-INF/spring.factories` 中
> `org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.example.sms.SmsAutoConfiguration`

### ⑦ 使用方

```yaml
sms:
  access-key: AKxxx
  secret: SKxxx
  timeout: 5s
```
```java
@Service
@RequiredArgsConstructor
public class NotifyService {
    private final SmsTemplate smsTemplate;      // 直接可用
}
```

## 3. @Enable 模式

两种实现方式：

```java
// 方式1：@Import 直接导入配置类
@Target(TYPE) @Retention(RUNTIME)
@Import(SmsConfiguration.class)
public @interface EnableSms {}

// 方式2：ImportSelector（可按条件/配置动态选择）
@Target(TYPE) @Retention(RUNTIME)
@Import(SmsImportSelector.class)
public @interface EnableSms { String mode() default "aliyun"; }

public class SmsImportSelector implements ImportSelector {
    public String[] selectImports(AnnotationMetadata meta) {
        Map<String,Object> attrs = meta.getAnnotationAttributes(EnableSms.class.getName());
        String mode = (String) attrs.get("mode");
        return new String[]{"aliyun".equals(mode) ? AliyunSmsConfig.class.getName() : TencentSmsConfig.class.getName()};
    }
}

// 方式3：ImportBeanDefinitionRegistrar（可自定义 BeanDefinition，最灵活）
public class MapperRegistrar implements ImportBeanDefinitionRegistrar {
    public void registerBeanDefinitions(AnnotationMetadata meta, BeanDefinitionRegistry registry) { ... }
}
// MyBatis 的 @MapperScan → MapperScannerRegistrar 就是这么做的
```

| 方式 | 能力 |
|---|---|
| `@Import(Config.class)` | 固定导入 |
| `ImportSelector` | 按条件返回类名数组 |
| `DeferredImportSelector` | 延迟到所有配置解析后再导入（自动配置用） |
| `ImportBeanDefinitionRegistrar` | 直接操作 BeanDefinition（扫描注册 Mapper/Feign） |

## 4. 条件注解进阶

```java
@ConditionalOnClass(name = "com.aliyun.dysmsapi20170525.Client")   // 不引入 SDK 就不生效
@ConditionalOnMissingBean(SmsTemplate.class)
@ConditionalOnSingleCandidate(RedisConnectionFactory.class)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@ConditionalOnResource(resources = "classpath:sms-config.xml")
```

自定义条件：

```java
public class OnLinuxCondition implements Condition {
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata meta) {
        return ctx.getEnvironment().getProperty("os.name").toLowerCase().contains("linux");
    }
}
@Conditional(OnLinuxCondition.class)
```

## 5. 失败分析器（启动失败时给出友好提示）

```java
public class SmsFailureAnalyzer extends AbstractFailureAnalyzer<SmsConfigException> {
    @Override protected FailureAnalysis analyze(Throwable root, SmsConfigException cause) {
        return new FailureAnalysis("短信配置错误：" + cause.getMessage(),
                "请检查 sms.access-key / sms.secret 是否正确", cause);
    }
}
```
```
META-INF/spring.factories
org.springframework.boot.diagnostics.FailureAnalyzer=com.example.SmsFailureAnalyzer
```
> `spring.factories` 在 Boot 3 中仍用于 `FailureAnalyzer`、`EnvironmentPostProcessor`、`ApplicationListener`、`EnableAutoConfiguration` **之外**的扩展点。

## 6. EnvironmentPostProcessor（早期配置处理）

```java
public class MyEnvPostProcessor implements EnvironmentPostProcessor {
    public void postProcessEnvironment(ConfigurableEnvironment env, SpringApplication app) {
        // 在容器刷新前读取/修改配置，可用于解密、加载远程配置
    }
}
```

## 7. 测试 Starter

```java
@SpringBootTest
class SmsAutoConfigurationTest {
    @Autowired(required = false) SmsTemplate template;

    @Test void shouldLoad() { assertThat(template).isNotNull(); }
}

// 更专业的做法：用 ApplicationContextRunner
new ApplicationContextRunner()
    .withConfiguration(AutoConfigurations.of(SmsAutoConfiguration.class))
    .withPropertyValues("sms.access-key=AK")
    .run(ctx -> assertThat(ctx).hasSingleBean(SmsTemplate.class));
```

## 8. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| Boot 3 中 starter 不生效 | 自动配置未加载 | 检查 `.imports` 文件路径与内容（不是 `spring.factories`） |
| `@AutoConfiguration` 被扫到 | 重复注册/条件失效 | 自动配置类不要放在 `@ComponentScan` 范围 |
| `@ConditionalOnMissingBean` 顺序错 | 判断结果不符预期 | 自动配置是 Deferred，用户配置先解析；用户类里不要用 `@ConditionalOnBean` |
| 属性绑定为 null | prefix 不匹配 | 检查 yml 层级与 `@EnableConfigurationProperties` |
| 依赖冲突 | 用户项目版本被污染 | starter 依赖尽量 `<optional>true</optional>` 或只依赖 autoconfigure |
| 用户自定义 Bean 被覆盖 | 覆盖顺序 | 自动配置用 `@ConditionalOnMissingBean` |

## 9. 面试速答

- **自定义 Starter 步骤**：写 `XxxProperties` + `XxxAutoConfiguration`（带 `@ConditionalOnMissingBean`）+ `META-INF/spring/...AutoConfiguration.imports` + 空壳 starter POM。
- **Boot 2 与 3 注册方式差异**：2 用 `spring.factories` 的 `EnableAutoConfiguration` 键，3 用 `AutoConfiguration.imports` 文件。
- **@Enable 注解原理**：底层是 `@Import`，可导入配置类、`ImportSelector`（按条件返回类名）、`ImportBeanDefinitionRegistrar`（编程注册 BeanDefinition）。
