# 31 · Docker 化与容器编排

## 一、Dockerfile（多阶段构建 + 分层缓存）

```dockerfile
# 阶段 1：构建
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline          # 依赖层缓存：pom 不变不重新下载
COPY src ./src
RUN mvn clean package -DskipTests

# 阶段 2：运行
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# 安全：非 root 运行
RUN addgroup -S app && adduser -S app -G app
USER app

COPY --from=build /app/target/*.jar app.jar

# JVM 容器感知参数
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"
ENV TZ=Asia/Shanghai

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

要点：
- **多阶段构建**：镜像里无 Maven/源码，体积小
- **非 root 用户**：安全基线
- **MaxRAMPercentage**：容器内 JVM 按配额自动分配堆（不要写死 -Xmx 与 limit 冲突）
- JRE 而非 JDK 运行时

### Layertools 分层（依赖与代码分离，重建更快）

```dockerfile
FROM eclipse-temurin:17-jre AS extract
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
RUN java -Djarmode=tools -jar app.jar extract --layers

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=extract /app/app/dependencies/ ./
COPY --from=extract /app/app/spring-boot-loader/ ./
COPY --from=extract /app/app/snapshot-dependencies/ ./
COPY --from=extract /app/app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

## 二、docker-compose 本地环境（一键起全套依赖）

```yaml
version: "3.8"
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/demo
      - SPRING_DATA_REDIS_HOST=redis
      - SPRING_RABBITMQ_HOST=rabbitmq
    depends_on:
      mysql: { condition: service_healthy }
      redis: { condition: service_started }
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: demo
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass 123456
  rabbitmq:
    image: rabbitmq:3.13-management
    ports: ["15672:15672"]     # 管理界面
volumes:
  mysql-data:
```

```bash
docker compose up -d
docker compose logs -f app
```

> 应用连 MySQL 注意 host 用服务名 `mysql` 而非 localhost；启动顺序用 healthcheck + depends_on condition。

## 三、Kubernetes 部署清单（核心配置）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  replicas: 3
  strategy:
    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }   # 平滑滚动
  template:
    spec:
      containers:
        - name: app
          image: registry/demo:1.0.0
          resources:
            requests: { cpu: "500m", memory: 512Mi }
            limits: { cpu: "2", memory: 1Gi }
          livenessProbe:                                 # 挂了重启
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 30
          readinessProbe:                                # 就绪才接流量
            httpGet: { path: /actuator/health/readiness, port: 8080 }
          lifecycle:
            preStop:
              exec: { command: ["sh", "-c", "sleep 10"] }  # 等网关摘流量再停机
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
      terminationGracePeriodSeconds: 45
```

**K8s + SpringBoot 三件套**：
1. `management.endpoint.health.probes.enabled=true` 开启 liveness/readiness
2. preStop sleep → 等待负载均衡摘除（配合优雅停机）
3. 定时任务/锁注意多副本语义（19 篇）

## 四、镜像与安全清单

- 基础镜像固定版本 + 定期重建（CVE 修复）
- 不在镜像里放密钥（构建参数会留在层里）→ K8s Secret/配置中心
- `.dockerignore` 排除 target/.git
- 镜像扫描（Trivy）接入 CI

## 常见坑

- **坑 1**：容器内 JVM 堆超内存 limit → OOMKilled；用 MaxRAMPercentage
- **坑 2**：docker-compose depends_on 只保证启动顺序不保证就绪 → healthcheck + condition
- **坑 3**：日志不输出文件只打 stdout → 容器化标准实践，交给采集器
- **坑 4**：时区默认 UTC → 设置 TZ 环境变量
- **坑 5**：readiness 未配置 → 启动期间收到请求报错

## Checklist

- [ ] 多阶段构建、非 root、分层缓存
- [ ] JVM 参数容器感知
- [ ] liveness/readiness/probes 完整
- [ ] 优雅停机 + preStop 摘流量
