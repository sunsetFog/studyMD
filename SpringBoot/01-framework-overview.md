# 01 · Spring 全家盘认知

## 一、三者关系

```
Spring Framework（地基：IoC/AOP/MVC）
    └── SpringBoot（脚手架：自动配置 + 内嵌容器 + 起步依赖）
            └── Spring Cloud（微服务全家桶：注册发现/网关/熔断...）
```

| 维度 | Spring | SpringBoot | Spring Cloud |
|---|---|---|---|
| 定位 | 基础框架 | 快速构建单体应用 | 微服务解决方案 |
| 核心价值 | IoC/AOP | 约定优于配置 | 分布式组件集成 |
| 部署 | war + 外置 Tomcat | 可执行 jar + 内嵌容器 | 各服务独立部署 |

**一句话**：SpringBoot 不是新框架，是 Spring 的「默认值生成器」——把 Spring 繁琐的 XML/JavaConfig 配置按约定自动化。

## 二、SpringBoot 解决的四大痛点

1. **依赖管理**：starter 起步依赖，按需引入，版本由 parent BOM 统一仲裁
2. **自动配置**：classpath 有什么就配什么（见 35 篇原理）
3. **内嵌容器**：Tomcat/Jetty/Undertow 打进 jar，`java -jar` 直接跑
4. **生产特性**：Actuator 健康检查、指标、外部化配置

## 三、版本选型（2026 视角）

| 版本 | 状态 | 说明 |
|---|---|---|
| 3.2/3.3/3.4+ | 主流 | Java 17+，Jakarta 命名空间 |
| 2.7.x | EOL | 大量存量项目，javax 命名空间 |
| 3.5.x LTS | 推荐 | 2025 年发布，长期支持线 |

**SpringBoot 2 → 3 关键迁移点**：
- `javax.*` → `jakarta.*`（Servlet、Validation、Persistence 全部换包名）
- Java 最低 17
- Trailing slash 匹配移除：`/api/user/` 不再匹配 `/api/user`
- Spring Security 写法大改（Lambda DSL）

## 四、技术栈全景图（本文档路线）

```
Web 层：  Spring MVC + Validation + SpringDoc
数据层：  MySQL + MyBatis(-Plus) / JPA + Redis
安全层：  Spring Security + JWT
中间件：  RabbitMQ / Kafka / WebSocket
工程化：  Vite 前端分离部署、Docker、Actuator + Prometheus
```

## 五、与前端协作的心智模型

后端职责收敛为三件事：
1. **契约**：统一响应体 + RESTful 语义（05/07 篇）
2. **鉴权**：无状态 JWT，网关/过滤器层解决（21 篇）
3. **文档**：OpenAPI 自动生成，前端可直接对接（22 篇）

## 常见坑

- **坑 1**：把 SpringBoot 3 项目按 2 的方式引入 `javax.servlet` 依赖 → 编译报错或运行时类找不到
- **坑 2**：Spring Cloud 版本必须与 SpringBoot 版本矩阵对齐（如 Cloud 2023.x ↔ Boot 3.2.x）
- **坑 3**：以为 SpringBoot 是独立框架，绕过 Spring 直接找「SpringBoot 的 API」——所有 API 都来自 Spring

## Checklist

- [ ] 能说清 Spring / SpringBoot / Spring Cloud 的关系与边界
- [ ] 明确项目使用的 Boot 版本与 Java 版本
- [ ] 知道 2→3 迁移的 jakarta 命名空间变化
