# 63 · Docker 打包与镜像优化

## 1. Dockerfile 基础

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/app.jar app.jar
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75"
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

构建与运行：

```bash
docker build -t order-service:1.0 .
docker run -d -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/demo \
  --name order order-service:1.0
```

## 2. 多阶段构建（编译与运行分离）

```dockerfile
# 阶段1：构建
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn -B -q dependency:go-offline          # ★ 依赖层缓存
COPY src ./src
RUN mvn -B -q clean package -DskipTests

# 阶段2：运行
FROM eclipse-temurin:17-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=build /build/target/*.jar app.jar
USER app                                      # ★ 非 root
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError"
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s \
  CMD wget -q -O- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["sh","-c","java $JAVA_OPTS -jar /app/app.jar"]
```

## 3. Boot 分层镜像（推荐）

```bash
java -Djarmode=layertools -jar app.jar extract    # 拆成 layers
```

```dockerfile
FROM eclipse-temurin:17-jre-alpine AS builder
WORKDIR /app
COPY target/app.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
# 按变化频率从低到高 COPY，最大化层缓存
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
ENTRYPOINT ["java","org.springframework.boot.loader.launch.JarLauncher"]
```

层顺序（Boot 默认 4 层）：`dependencies` → `spring-boot-loader` → `snapshot-dependencies` → `application`。

> 改代码只重建 application 层（几十 KB），依赖层全部命中缓存 → **推送/拉取极其快**。
> Maven 插件可直接构建镜像：`mvn spring-boot:build-image`（用 Buildpacks，无需 Dockerfile）。

## 4. 镜像瘦身要点

| 手段 | 效果 |
|---|---|
| 用 `*-jre-alpine` / `distroless` 基础镜像 | 300MB → 100MB |
| 多阶段构建（不带 Maven/源码） | 大幅减小 |
| 分层（依赖与代码分离） | 加速增量 |
| 清理 apt 缓存、`/tmp` | 减小 |
| 只 COPY 必要文件（`.dockerignore`） | 减小 + 安全 |
| 用 `jlink` 定制 JRE | 40~60MB |
| GraalVM Native Image | 20~50MB + 极快启动 |

```dockerfile
# 自定义 JRE（JDK 17+）
RUN jlink --add-modules java.base,java.sql,java.naming,java.desktop,java.management,java.security.jgss,java.instrument \
          --strip-debug --no-man-pages --no-header-files --compress=2 --output /javaruntime
FROM alpine:3.19
COPY --from=build /javaruntime /opt/java
ENV PATH=/opt/java/bin:$PATH
```

`.dockerignore`：
```
target/
.git/
.idea/
*.log
src/main/resources/application-prod.yml
```

## 5. JVM 容器化参数

```
-XX:+UseContainerSupport          # Boot 3/JDK 10+ 默认开启，读取 cgroup 限制
-XX:MaxRAMPercentage=75.0         # 堆占容器内存比例（不要用固定 -Xmx）
-XX:InitialRAMPercentage=50.0
-XX:+ExitOnOutOfMemoryError       # OOM 立即退出，让编排重启
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps
-Xss512k
-XX:MaxMetaspaceSize=256m
```

> **不要**用 `-Xmx4g` 硬编码；容器 limit 与 JVM 堆要留余量（堆外内存、元空间、线程栈、DirectBuffer）。

## 6. 时区、字体、证书

```dockerfile
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
# 如需验证码字体
RUN apk add --no-cache fontconfig ttf-dejavu
# 如需自定义 CA
COPY cert.pem /usr/local/share/ca-certificates/ && update-ca-certificates
```

## 7. 健康检查与优雅停机

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 --start-period=60s \
  CMD curl -fsS http://localhost:8080/actuator/health || exit 1
```
- 应用 `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=30s`。
- `docker stop` 发 SIGTERM，JVM 触发 shutdown hook。
- `ENTRYPOINT` 用 `exec` 形式，保证 PID 1 能收到信号。

```dockerfile
ENTRYPOINT ["java","-jar","/app/app.jar"]     # ✅ exec 形式
# 或用 tini/dumb-init 处理僵尸进程
```

## 8. 安全建议

- 非 root 用户运行（`USER app`）。
- 不要打进密钥（用环境变量/Secret 挂载）。
- 镜像扫描（Trivy、Clair）。
- 使用可信基础镜像、定期更新。
- 只读文件系统（`--read-only`）+ tmpfs。

## 9. docker-compose 一键启依赖

```yaml
version: "3.8"
services:
  mysql:
    image: mysql:8.0
    environment: {MYSQL_ROOT_PASSWORD: root, MYSQL_DATABASE: demo}
    ports: ["3306:3306"]
    volumes: ["mysql-data:/var/lib/mysql"]
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    ports: ["6379:6379"]
  nacos:
    image: nacos/nacos-server:v2.3.2
    environment:
      MODE: standalone
    ports: ["8848:8848"]
  app:
    build: .
    depends_on: [mysql, redis]
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/demo
    ports: ["8080:8080"]
volumes: {mysql-data: {}}
```

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 容器 OOMKilled | 进程被杀 | 堆比例调小；`MaxRAMPercentage` |
| 时区错 | 日志差 8 小时 | 设 TZ |
| 中文乱码/方块 | 字体缺失 | 装字体，设 `file.encoding=UTF-8` |
| 启动慢 | 熵不足/类加载 | `-Djava.security.egd=file:/dev/./urandom`；AOT/CDS |
| 改代码构建慢 | 每次重下依赖 | 分层 + `.dockerignore` + 依赖缓存 |
| PID 1 收不到 SIGTERM | 无法优雅停机 | exec 形式 ENTRYPOINT 或用 tini |
| 镜像过大 | 推送慢 | 多阶段 + alpine/jlink |
| 容器内端口 | 连不上 | `server.port` 与 EXPOSE 一致；`0.0.0.0` 监听 |

## 11. 面试速答

- **多阶段构建**：构建阶段用 JDK+Maven 编译，运行阶段只拷 jar 到 JRE 镜像，减小体积、减少攻击面。
- **Boot 分层镜像**：把 jar 拆成依赖层与代码层，改代码只重建代码层，极大加速构建与分发。
- **容器里 JVM 参数怎么设**：`UseContainerSupport` + `MaxRAMPercentage`（百分比而非固定值）+ `ExitOnOutOfMemoryError`。
- **为什么不能用 root 运行**：安全风险（容器逃逸影响主机），应创建非特权用户。
- **Native Image 有什么好处**：启动毫秒级、内存占用低、镜像小；代价是构建慢、反射/动态代理需配置。
