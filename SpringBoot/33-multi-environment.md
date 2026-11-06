# 33 · 多环境与配置中心（Nacos）

## 一、环境矩阵与职责

| 环境 | 目标 | 配置来源 | 数据 |
|---|---|---|---|
| dev | 本地开发 | 本地 yml | 随便造 |
| test | QA 验证 | CI 注入 | 测试库 |
| staging | 预发验证 | 配置中心 | 类生产脱敏 |
| prod | 生产 | 配置中心 + 密钥系统 | 生产库 |

**原则**：jar 环境无关；一切差异外置（03 篇优先级体系）。

## 二、纯文件多环境（中小项目）

```
application.yml           # 公共 + spring.profiles.active: @profiles.active@
application-dev.yml
application-prod.yml
```

Maven profile 联动打包：

```xml
<profiles>
    <profile>
        <id>dev</id>
        <properties><profiles.active>dev</profiles.active></profiles.active></properties>
        <activation><activeByDefault>true</activeByDefault></activation>
    </profile>
    <profile>
        <id>prod</id>
        <properties><profiles.active>prod</profiles.active></properties>
    </profile>
</profiles>
<build>
  <resources>
    <resource><directory>src/main/resources</directory><filtering>true</filtering></resource>
  </resources>
</build>
```

打包：`mvn package -Pprod`。**生产更推荐不改包，用启动参数覆盖**。

## 三、配置中心：Nacos（生产主流）

### 为什么要配置中心

- 多实例配置一处修改全局生效（动态刷新）
- 敏感配置托管（权限/加密）
- 环境隔离（namespace/group）
- 变更审计

### 集成

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yml 或 application.yml（Boot 2.4+ 用 spring.config.import）
spring:
  application:
    name: order-service
  config:
    import:
      - nacos:order-service.yaml        # 主配置
      - nacos:common.yaml               # 公共配置
  cloud:
    nacos:
      config:
        server-addr: nacos.example.com:8848
        namespace: prod                 # 环境隔离
        group: DEFAULT_GROUP
        username: ${NACOS_USER}
        password: ${NACOS_PASSWORD}
```

### 动态刷新

```java
@Component
@RefreshScope                  // 配置变更时 Bean 重建
public class PriceService {

    @Value("${discount.rate:0.9}")
    private BigDecimal rate;

    public BigDecimal discount(BigDecimal price) {
        return price.multiply(rate);
    }
}
```

注意：
- @RefreshScope 重建 Bean，有状态组件慎用
- @ConfigurationProperties 绑定的配置**天然支持刷新**（无需 @RefreshScope），优先用
- 数据库连接池等变更仍需重启——刷新不等于万能

### 与数据库敏感配置

- Nacos 配置加密（插件/自实现）或 KMS 托管
- 账号权限最小化：生产 namespace 只读权限给应用

## 四、Apollo 对比与选型

| 维度 | Nacos | Apollo |
|---|---|---|
| 定位 | 配置+注册发现一体 | 纯配置中心，功能强 |
| 灰度发布 | 基础 | 原生支持 |
| 权限/审计 | 一般 | 完善 |
| 适用 | SpringCloud Alibaba 栈 | 大型企业配置治理 |

小团队：文件 + Git 管理；中团队：Nacos；大团队/强合规：Apollo。

## 五、配置漂移治理

- 本地文件只留「骨架 + import」，真实值全在配置中心
- Git 仓库管理配置快照（配置即代码，变更有 PR 评审）
- 启动时校验关键配置存在性（快速失败）

```java
@ConfigurationProperties(prefix = "app")
@Validated
public record AppProperties(
        @NotBlank String name,
        @NotNull @Min(1) Integer pageSize) {
}
```

## 常见坑

- **坑 1**：bootstrap.yml 不生效 → Boot 3 用 spring.config.import 方式（或引入 spring-cloud-starter-bootstrap）
- **坑 2**：@RefreshScope 只对直接 @Value/属性绑定生效 → setter 注入的中间对象不刷新
- **坑 3**：namespace/group 配错加载到测试配置 → 生产炸库；启动打印生效数据源验证
- **坑 4**：配置中心挂了应用无法启动 → 本地留兜底配置文件（优先级低于远端）
- **坑 5**：动态刷新的连接池/线程池参数不生效 → 这类资源需重建逻辑，别指望 @RefreshScope

## Checklist

- [ ] jar 环境无关，差异全外置
- [ ] 生产配置中心托管 + 权限最小化
- [ ] 动态刷新方案明确（@ConfigurationProperties 优先）
- [ ] 本地兜底配置防止中心不可用
