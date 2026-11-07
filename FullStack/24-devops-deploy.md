# 24 - DevOps 与部署（全栈整合）

## 1. 部署架构总览

```
用户 → CDN（静态资源）→ Nginx（反向代理/负载均衡）
     → 前端静态文件（Nginx 托管 dist）
     → 后端服务（多实例）→ MySQL 主从 / Redis / MQ
```

## 2. Nginx 核心配置（必考）

```nginx
server {
  listen 80;
  # 前端 SPA：history 路由回退
  location / { root /var/www/dist; try_files $uri $uri/ /index.html; }
  # 静态资源强缓存
  location /assets/ { expires 365d; add_header Cache-Control "immutable"; }
  # API 反向代理
  location /api/ { proxy_pass http://backend; proxy_set_header Host $host; }
  # WebSocket
  location /ws/ { proxy_pass http://backend; proxy_http_version 1.1;
                  proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade"; }
}
upstream backend { server 10.0.0.1:8080; server 10.0.0.2:8080; keepalive 32; }
```

- 负载均衡策略：轮询/权重/ip_hash（会话粘滞，尽量不用）/least_conn
- gzip/brotli 压缩、限流 limit_req、HTTPS 配置

## 3. Docker（必考⭐）

```dockerfile
# 后端多阶段构建
FROM maven:3.9-eclipse-temurin-17 AS build
COPY . . && RUN mvn -pl app -am package -DskipTests
FROM eclipse-temurin:17-jre
COPY --from=build /app/target/app.jar app.jar
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75","-jar","app.jar"]

# 前端
FROM node:20 AS build
RUN npm ci && npm run build
FROM nginx:alpine
COPY --from=build /dist /usr/share/nginx/html
```

- 镜像分层缓存：依赖层与代码层分离（先 COPY pom 再 COPY src）
- 容器化 Java：内存感知（UseContainerSupport 默认开）、优雅停机（SIGTERM + kill grace period）
- docker-compose 本地一键起 MySQL/Redis/后端/前端

## 4. CI/CD 流水线（⭐）

```
push → CI：lint/test/build（Docker 镜像，打版本 tag）
     → CD：推镜像仓库 → 部署 dev → 自动化测试 → 手动审批 → staging → prod（滚动/蓝绿/灰度）
```

- Git Flow 简化为 trunk-based + feature flag（现代主流）
- 发布策略：
  | 策略 | 特点 |
  |---|---|
  | 滚动 | 逐个替换，省资源，短暂版本混合 |
  | 蓝绿 | 双环境切换，回滚快，资源×2 |
  | 灰度/金丝雀 | 小比例流量验证（按用户/地区），生产标配 |

- 数据库变更独立于代码发布（Flyway 迁移脚本，向后兼容两阶段：先加列→发码→后删列）

## 5. 监控告警体系（⭐）

```
指标：Prometheus 采集（JVM/HTTP QPS/RT/错误率）+ Grafana 看板
日志：应用 JSON 日志 → Filebeat → ELK / Loki，按 traceId 串联
链路：OpenTelemetry/SkyWalking 全链路 trace
告警：分级（P0 电话 / P1 群通知），基于 SLO（错误预算）
```

- 前端监控：web-vitals + JS 错误上报 + sourcemap 还原（见 10）
- 黄金四指标：延迟/流量/错误/饱和度

## 6. 环境与配置管理

- 配置分离：环境变量/配置中心（Nacos），禁明文密码入库（Vault/KMS）
- 12-Factor：配置、日志（stdout）、无状态、可丢弃
- Secrets 管理：CI 注入、轮换

## 常见追问

1. 线上发布后接口大量 500 怎么办？——回滚优先（镜像版本回退），再排查；配置回滚开关
2. HTTPS 证书怎么上？——ACME 自动续期（certbot）或云证书托管
3. 前端发版后用户缓存旧版本？——index.html no-cache + 资源 hash + 运行时检测版本提示刷新

## Checklist

- [ ] Nginx 能写 SPA 回退与 WS 代理
- [ ] Dockerfile 多阶段构建与分层缓存
- [ ] 发布三策略 + 灰度是生产标配
- [ ] 监控黄金四指标能背
