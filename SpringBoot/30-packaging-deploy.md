# 30 · 打包与部署

## 一、可执行 jar（标准方式）

```bash
mvn clean package -DskipTests
java -jar target/demo-1.0.0.jar --spring.profiles.active=prod
```

fat jar 结构（Boot 3.x）：`BOOT-INF/classes/` 业务类 + `BOOT-INF/lib/` 依赖 + `org.springframework.boot.loader.launch.JarLauncher` 自定义加载器。

> 生产可用 **Spring Boot Layertools** 分层 jar：依赖层与业务层分离，Docker 构建缓存友好（31 篇）。

## 二、war 部署（传统 Tomcat）

```java
public class App extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(DemoApplication.class);
    }
}
```

```xml
<packaging>war</packaging>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```

> 前后端分离时代首选 jar，war 仅存量需求。

## 三、启动参数与环境变量

```bash
# 配置覆盖优先级最高
java -jar app.jar \
  --spring.profiles.active=prod \
  --server.port=8080 \
  --spring.datasource.url=... \
  -Xms1g -Xmx1g -XX:+UseG1GC \
  -Djava.security.egd=file:/dev/./urandom
```

标准启动脚本（systemd 亦可）：

```bash
#!/bin/bash
APP=app.jar
PID=$(pgrep -f "$APP")
case "$1" in
  start) nohup java -jar $APP --spring.profiles.active=prod > /dev/null 2>&1 & echo "started" ;;
  stop)  kill -15 $PID; echo "stopping gracefully" ;;
  status) [ -n "$PID" ] && echo "running pid=$PID" || echo "stopped" ;;
esac
```

## 四、优雅停机（重要）

```yaml
server:
  shutdown: graceful            # Boot 2.3+
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

流程：收到 SIGTERM → 停止接受新请求 → 等待处理中的请求完成（最长 30s）→ 容器销毁（@PreDestroy 执行，线程池 waitForTasksToComplete 生效）。**发送 kill -15 而非 kill -9**。

## 五、Nginx 反向代理（前后端分离标准拓扑）

```nginx
upstream backend { server 127.0.0.1:8080; }

server {
    listen 80;
    server_name demo.example.com;

    location /api/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 20m;
    }

    # 前端静态资源（Vue/React build 产物）
    location / {
        root /var/www/dist;
        try_files $uri $uri/ /index.html;     # SPA history 路由回退
    }
}
```

## 六、CI/CD 流水线

```
push → lint & unit test → package → build image → push registry
     → 部署 dev（自动）→ 部署 prod（审批 + 滚动/蓝绿）
```

滚动发布要点：
- 启动探针通过才接流量（32 篇 Actuator health）
- 停机走优雅停机
- 数据库迁移先行且向下兼容（Flyway/Liquibase，加列不删列）

## 七、Flyway 数据库版本管理

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

```
resources/db/migration/
  V1__init_schema.sql
  V2__add_order_table.sql
```

应用启动自动按版本号执行，`flyway_schema_history` 表记录。**规则：迁移文件一旦合入禁止修改**。

## 常见坑

- **坑 1**：jar 里读资源文件 `new File("src/main/resources/...")` → 生产不存在；用 `ClassPathResource`/Stream
- **坑 2**：kill -9 杀进程 → 优雅停机失效、请求被硬断
- **坑 3**：fat jar 解压后运行 → loader 结构破坏；必须 `java -jar`
- **坑 4**：静态资源 gzip/缓存头缺失 → nginx 配置 gzip 与 expires
- **坑 5**：多实例滚动发布时定时任务重复执行 → 分布式锁（19 篇）

## Checklist

- [ ] jar 启动脚本标准化（参数外置）
- [ ] 优雅停机 + systemd/supervisor 托管
- [ ] Nginx 静态资源 + API 反代拓扑
- [ ] Flyway 管理库表变更
- [ ] CI 流水线含测试与镜像构建
