# 11 · 自动配置原理（源码级）

## 1. 一句话概括

> 启动类上的 `@EnableAutoConfiguration` → `AutoConfigurationImportSelector` → 读取候选配置类清单 → 按 `@Conditional` 条件筛选 → 注册到容器。

## 2. 从注解走进源码

```java
@EnableAutoConfiguration
  └── @Import(AutoConfigurationImportSelector.class)
        └── DeferredImportSelector（延迟导入，等用户自己的 @Configuration 先处理完）
```

```java
// AutoConfigurationImportSelector
public String[] selectImports(AnnotationMetadata metadata) {
    AutoConfigurationEntry entry = getAutoConfigurationEntry(metadata);
    return StringUtils.toStringArray(entry.getConfigurations());
}
// 核心三步：
// 1. getCandidateConfigurations  → 加载候选清单
// 2. removeDuplicates + getExclusions（排除项）
// 3. filter(configurations, autoConfigurationMetadata) → @Conditional 过滤
// 4. fireAutoConfigurationImportEvents
```

## 3. 候选清单从哪来

### Boot 2.7 之前

```
META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
...
```

### Boot 2.7 ~ 3.x（当前）

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
...
```

> Boot 3.x **只**读 `.imports` 文件（`spring.factories` 中的 `EnableAutoConfiguration` 项已弃用）。这是面试高频变更点。
> 加载器：`ImportCandidates.load(AutoConfiguration.class, classLoader)`。

你可以查看已生效的配置：
```bash
java -jar app.jar --debug                 # 打印 CONDITION EVALUATION DELTA
# 或
curl localhost:8080/actuator/conditions   # 需开启 actuator
```

## 4. 条件注解（@Conditional 家族）

| 注解 | 生效条件 |
|---|---|
| `@ConditionalOnClass` | classpath 存在指定类 |
| `@ConditionalOnMissingClass` | classpath **不存在**指定类 |
| `@ConditionalOnBean` | 容器中**存在**指定 Bean |
| `@ConditionalOnMissingBean` | 容器中**不存在**指定 Bean ★ 让用户配置优先 |
| `@ConditionalOnProperty` | 配置属性匹配（`prefix`+`name`+`havingValue`+`matchIfMissing`） |
| `@ConditionalOnExpression` | SpEL 为 true |
| `@ConditionalOnWebApplication` | Web 环境（SERVLET/REACTIVE） |
| `@ConditionalOnNotWebApplication` | 非 Web |
| `@ConditionalOnResource` | 资源文件存在 |
| `@ConditionalOnJava` | JDK 版本 |
| `@ConditionalOnSingleCandidate` | 容器中该类型只有一个或有一个 Primary |

```java
@AutoConfiguration                    // Boot 3 推荐：代替 @Configuration + 排序
@ConditionalOnClass(RedisOperations.class)
@ConditionalOnMissingBean(RedisConnectionFactory.class)
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(name = "redisTemplate")   // 用户自定义则跳过
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory f) { ... }
}
```

## 5. 自动配置的顺序

```java
@AutoConfiguration(before = ..., after = ...)
// 或
@AutoConfigureBefore(DataSourceAutoConfiguration.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
```

> `@AutoConfiguration` 不能同时被 `@ComponentScan` 扫到（其包路径刻意避开扫描范围），只能被 `.imports` 导入，避免双重注册。

## 6. 配置属性绑定：@ConfigurationProperties

```java
@ConfigurationProperties(prefix = "app.sms")
@Data
public class SmsProperties {
    private String accessKey;
    private int timeout = 3;          // 默认值
    private List<String> templates;
}
```

```java
@EnableConfigurationProperties(SmsProperties.class)   // 注册
// 或
@ConfigurationPropertiesScan                          // 批量扫描
```

```yaml
app:
  sms:
    access-key: AKxxx       # 松散绑定：accessKey / access-key / ACCESS_KEY 都行
    timeout: 5
    templates: [a, b]
```

绑定规则：
- **松散绑定（Relaxed Binding）**：`accessKey` ↔ `access-key` ↔ `ACCESS_KEY`。
- 支持 `Duration`、`DataSize`（`10s` / `10MB`）。
- 构造器绑定（Boot 3 推荐，配合 `@ConstructorBinding` 或单构造器 record）。
- 校验：类上加 `@Validated` + `@NotBlank`。

> IDE 提示：加 `spring-boot-configuration-processor` 依赖，生成 `spring-configuration-metadata.json`，yml 自动补全。

## 7. 完整流程时序

```
1 启动，refresh()
2 ConfigurationClassPostProcessor 解析 @SpringBootApplication
3 解析 @Import(AutoConfigurationImportSelector)
   —— 因为是 DeferredImportSelector，推迟到所有 @Configuration 解析完之后
4 ImportCandidates 读取所有 jar 的 META-INF/spring/*.AutoConfiguration.imports
5 去重、排除（spring.autoconfigure.exclude / exclude 属性）
6 AutoConfigurationImportFilter（OnClassCondition 等）按条件过滤
   —— 这一步极大加速：用字节码元数据而非加载类
7 逐个解析 @AutoConfiguration 类，其中的 @Bean 再经过 @Conditional 判断
8 注册 BeanDefinition，实例化
```

> 关键：先解析**用户自己的配置**，再解析自动配置，因此 `@ConditionalOnMissingBean` 才能生效。

## 8. 自动配置报告

```yaml
management:
  endpoints:
    web:
      exposure:
        include: conditions,beans,configprops
```

GET `/actuator/conditions` 返回三类：
- `positiveMatches`：生效的（含原因）
- `negativeMatches`：未生效的（含原因）★ 排错利器
- `unconditionalClasses`

## 9. 为什么自动配置类不能放在主扫描包下

Boot 源码中自动配置类都在 `org.springframework.boot.autoconfigure.**`，用户的 `@ComponentScan` 扫的是 `com.xxx`，**天然不重叠**。若你把自己的 `@Configuration` 也写成自动配置并被扫到，会绕过条件判定导致重复注册。

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 自定义 Bean 不生效 | 被自动配置覆盖或反之 | 看 `/actuator/conditions`；`@ConditionalOnMissingBean` 让自定义优先 |
| 自定义扫描包导致自动配置失效 | 大量 Bean 缺失 | 不要手动改 `@ComponentScan` 覆盖默认 |
| Boot 2 的 `spring.factories` 写法在 3.x 无效 | 自定义 starter 不生效 | 改用 `.imports` 文件 |
| `@ConditionalOnBean` 不生效 | 顺序问题 | 自动配置类加 `@AutoConfigureAfter`；用户配置类不要用此注解 |
| 配置属性绑定为 null | yml 缩进/命名错 | 检查 prefix、加 processor 依赖 |
| 排除无效 | 类名写错 | 用全限定类名 |

## 11. 面试速答

- **自动配置原理**：`@EnableAutoConfiguration` 导入 `AutoConfigurationImportSelector`，它从 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 读取候选类，再经 `@Conditional*` 条件过滤，最后注册 BeanDefinition。
- **为什么用户配置优先**：自动配置通过 `DeferredImportSelector` **延迟**到用户配置解析完成后才处理，配合 `@ConditionalOnMissingBean` 实现「用户有就用用户的」。
- **Boot 2 与 Boot 3 的区别**：注册文件从 `META-INF/spring.factories` 变为 `META-INF/spring/*.AutoConfiguration.imports`；`@Configuration` 变 `@AutoConfiguration`。
