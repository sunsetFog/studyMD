# 45 · 配置中心：Nacos Config 与 Spring Cloud Config

## 1. 为什么需要配置中心

| 问题 | 配置中心解决 |
|---|---|
| 配置散落各服务 | 集中管理 |
| 改配置要重启 | **动态刷新** |
| 多环境管理混乱 | 命名空间/分组 |
| 无审计 | 版本历史、回滚 |
| 敏感信息明文 | 加密 + 权限 |

## 2. Nacos Config（推荐）

### 依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

### 配置（必须用 bootstrap）

```xml
<dependency><groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-bootstrap</artifactId></dependency>
```

```yaml
# bootstrap.yml  ★ 配置中心配置必须放这里（早于 application.yml 加载）
spring:
  application:
    name: order-service
  profiles:
    active: dev
  cloud:
    nacos:
      server-addr: localhost:8848
      username: nacos
      password: nacos
      config:
        namespace: dev-ns-id          # 命名空间 ID
        group: DEFAULT_GROUP
        file-extension: yaml
        refresh-enabled: true
        # 共享配置（公共数据源、Redis 等）
        shared-configs:
          - data-id: common-datasource.yaml
            group: COMMON
            refresh: true
        extension-configs:
          - data-id: common-redis.yaml
            refresh: true
```

### DataId 规则

```
${prefix}-${spring.profiles.active}.${file-extension}
默认 prefix = spring.application.name
例：order-service-dev.yaml
优先级：order-service-dev.yaml > order-service.yaml > extension > shared
```

### 动态刷新

```java
@RestController
@RefreshScope                       // ★ 加了才会动态刷新
public class DemoController {
    @Value("${app.feature.x:false}") private boolean featureX;
}

// 或推荐用 @ConfigurationProperties（自动刷新，无需 @RefreshScope）
@Component
@ConfigurationProperties(prefix = "app.feature")
@RefreshScope
@Data
public class FeatureProps { private boolean x; }
```

```java
// 监听配置变更
@NacosConfigListener(dataId = "order-service.yaml", groupId = "DEFAULT_GROUP")
public void onChange(String config) { log.info("配置变更: {}", config); }
```

> `@Value` + `@RefreshScope` 会重建 Bean；`@ConfigurationProperties` 更轻量，优先使用。

### 配置加密

Nacos 支持 `cipher-aes` 前缀加密（需插件）或自行用 Jasypt。

## 3. Spring Cloud Config（官方，Git 后端）

### Server

```xml
<dependency><groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-config-server</artifactId></dependency>
```
```java
@SpringBootApplication @EnableConfigServer
```
```yaml
server: {port: 8888}
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/xxx/config-repo
          search-paths: '{application}'
          default-label: main
          username: xxx
          password: xxx
```

### Client

```yaml
spring:
  application: {name: order-service}
  cloud:
    config:
      uri: http://localhost:8888
      label: main
      profile: dev
      fail-fast: true
      retry:
        max-attempts: 6
```

刷新：`POST /actuator/refresh`（单个实例）或通过 Spring Cloud Bus（广播到所有实例）。

> 对比：Nacos 有控制台、支持推送（长轮询）、无需额外 Bus；Config 基于 Git，天然版本管理，但需 Bus 才能广播。

## 4. Apollo（携程，功能最全）

特点：配置修改后**实时推送**（HTTP 长轮询）、灰度发布、权限审计、多环境多集群、客户端缓存到本地文件（服务端挂了也能启动）。

```xml
<dependency><groupId>com.ctrip.framework.apollo</groupId>
           <artifactId>apollo-client</artifactId><version>2.2.0</version></dependency>
```
```yaml
apollo:
  meta: http://localhost:8080
  bootstrap:
    enabled: true
    namespaces: application,datasource
app:
  id: order-service
```

## 5. 三种方案对比

| 维度 | Nacos | Spring Cloud Config | Apollo |
|---|---|---|---|
| 存储 | 内嵌 DB + 文件 | Git | DB |
| 控制台 | ✅ | ❌ | ✅ |
| 实时推送 | ✅ 长轮询 | 需 Bus | ✅ |
| 灰度 | 简单 | ❌ | ✅ |
| 权限审计 | 基础 | Git 自带 | ✅ |
| 运维成本 | 低（一个组件） | 中（+Bus+MQ） | 中 |
| 推荐 | **国内首选** | 官方轻方案 | 大厂/强治理 |

## 6. 配置规范建议

- **公共配置**下沉（数据源模板、Redis、MQ、日志）为 shared config。
- **环境隔离**用 namespace（dev/test/prod），而非 profile 文件堆砌。
- **敏感配置**加密，禁止明文入库代码库。
- **变更要灰度**：先改一个实例验证，再全量。
- **本地缓存兜底**：Nacos 客户端会缓存到 `~/nacos/config`，服务端挂了仍能启动（生产可加 `-Dnacos.standalone=true` 无关）。
- 配置项**必须有默认值**，防止配置缺失启动失败。

## 7. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 配置不生效 | 写在 application.yml | Nacos 配置要放 **bootstrap.yml**；或 Boot 2.4+ 用 `spring.config.import=nacos:xxx.yaml` |
| 动态刷新无效 | 改了没变 | 加 `@RefreshScope` 或用 `@ConfigurationProperties` |
| DataId 拼错 | 404 无配置 | 严格 `${name}-${profile}.${ext}` |
| namespace 用名称 | 找不到 | 用 ID |
| 中文乱码 | 乱码 | 页面编辑用 UTF-8；文件后缀 yaml 而非 yml（注意一致） |
| 配置中心挂了服务起不来 | 启动失败 | 开启本地缓存；关键配置本地兜底 |
| 刷新导致 Bean 重建 | 短暂不可用 | 了解 `@RefreshScope` 是重建 Bean，非热替换字段 |

## 8. 面试速答

- **配置中心解决什么**：集中管理、动态刷新、环境隔离、审计回滚、敏感加密。
- **Nacos 动态刷新原理**：客户端长轮询（默认 30s 超时）对比配置 MD5，变更即拉取并发布 `RefreshEvent`，`@RefreshScope` 的 Bean 被重建。
- **bootstrap.yml 为什么必须**：配置中心地址需要在应用配置加载**之前**就绪，bootstrap 上下文先于主上下文加载（Boot 2.4+ 也可用 `spring.config.import`）。
- **@Value 与 @ConfigurationProperties 刷新差异**：前者需 `@RefreshScope`（重建 Bean），后者由 `ConfigurationPropertiesRebinder` 重新绑定，更轻量。
