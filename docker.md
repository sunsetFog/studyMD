# Docker 知识点总结

## 1. 简介

Docker 是一个容器化平台，把应用及其依赖打包成镜像，在任何地方一致地运行。

容器 vs 虚拟机：
- 容器共享宿主机内核，启动快（秒级）、占用小
- 虚拟机有完整操作系统，隔离性强但更重

核心概念：
- 镜像（Image）：只读模板，用于创建容器
- 容器（Container）：镜像的运行实例
- 仓库（Registry）：存放镜像的地方

## 2. 安装

```
# 检查是否安装成功
docker --version
docker version
docker info
```

## 3. 镜像 Image

镜像是分层存储的只读模板，包含运行应用所需的全部文件。

```
# 列出本地镜像
docker images

# 搜索镜像
docker search nginx

# 拉取镜像
docker pull nginx:latest

# 删除镜像
docker rmi nginx:latest

# 给镜像打标签
docker tag nginx:latest mynginx:1.0
```

## 4. 容器 Container

容器是镜像的运行实例，可启动、停止、删除。

```
# 运行容器
docker run -d --name mynginx -p 80:80 nginx

# 查看运行中的容器
docker ps

# 查看所有容器（含已停止）
docker ps -a

# 停止容器
docker stop mynginx

# 启动已停止的容器
docker start mynginx

# 重启容器
docker restart mynginx

# 删除容器（需先停止）
docker rm mynginx
```

## 5. run 常用参数

```
docker run [选项] 镜像 [命令]

常用选项：
-d            # 后台运行（detached）
--name        # 指定容器名
-p 8080:80    # 端口映射（宿主机:容器）
-v /data:/var # 挂载数据卷（宿主机:容器）
-e KEY=value  # 设置环境变量
--restart=always   # 开机/异常自动重启
--rm          # 容器退出后自动删除
-it           # 交互式终端
```

```
# 示例：运行 MySQL
docker run -d \
  --name mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -v /data/mysql:/var/lib/mysql \
  mysql:8.0
```

## 6. 进入容器

```
# 在运行中的容器里执行命令
docker exec -it mynginx bash

# 直接执行单条命令
docker exec mynginx ls /etc

# 交互式进入
docker exec -it mynginx sh
```

## 7. 查看日志与信息

```
# 查看容器日志
docker logs mynginx

# 实时跟踪日志（-f 类似 tail -f）
docker logs -f mynginx

# 查看最后 100 行
docker logs --tail 100 mynginx

# 查看容器详细信息
docker inspect mynginx

# 查看容器资源占用
docker stats

# 查看容器内进程
docker top mynginx
```

## 8. 拷贝文件

```
# 宿主机 → 容器
docker cp ./file.txt mynginx:/tmp/file.txt

# 容器 → 宿主机
docker cp mynginx:/tmp/file.txt ./file.txt
```

## 9. Dockerfile 简介

Dockerfile 是构建镜像的脚本，描述镜像的构建步骤。

```
# 简单示例：构建一个 nginx 镜像
FROM nginx:alpine                    # 基础镜像
COPY index.html /usr/share/nginx/html/   # 复制文件
EXPOSE 80                           # 声明端口
CMD ["nginx", "-g", "daemon off;"]  # 启动命令
```

## 10. Dockerfile 指令详解

```
FROM        # 指定基础镜像（必须第一行）
FROM openjdk:17-jdk-alpine

WORKDIR     # 设置工作目录
WORKDIR /app

COPY        # 复制宿主机文件到镜像
COPY target/app.jar /app/app.jar

ADD         # 同 COPY，但支持 URL 和自动解压 tar
ADD app.tar.gz /app/

RUN         # 构建时执行命令
RUN apt-get update && apt-get install -y curl

ENV         # 设置环境变量
ENV JAVA_OPTS="-Xmx512m"

EXPOSE      # 声明容器监听端口（元数据，不真正映射）
EXPOSE 8080

CMD         # 容器启动时的默认命令（可被覆盖）
CMD ["java", "-jar", "app.jar"]

ENTRYPOINT  # 容器入口命令（不易被覆盖）
ENTRYPOINT ["java", "-jar"]

VOLUME      # 声明匿名数据卷
VOLUME /data
```

## 11. CMD 与 ENTRYPOINT 区别

- CMD：默认命令，运行容器时传入命令会覆盖它
- ENTRYPOINT：入口命令，不会被覆盖，CMD 作为其参数

```
# 只使用 CMD
FROM ubuntu
CMD ["echo", "hello"]
# docker run 镜像 echo world  → 覆盖 CMD，输出 world

# ENTRYPOINT + CMD 组合
FROM ubuntu
ENTRYPOINT ["echo"]
CMD ["hello"]
# docker run 镜像            → 输出 hello
# docker run 镜像 world      → 输出 world（CMD 被替换）
```

## 12. 构建镜像

```
# 构建镜像（-t 指定名称和标签）
docker build -t myapp:1.0 .

# 指定 Dockerfile
docker build -t myapp:1.0 -f Dockerfile.prod .

# 查看构建历史
docker history myapp:1.0
```

## 13. 多阶段构建

减小镜像体积，把构建和运行分开。

```
# 阶段一：构建
FROM maven:3.8-jdk-11 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# 阶段二：运行（只保留产物）
FROM openjdk:11-jre-slim
WORKDIR /app
COPY --from=build /app/target/app.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

## 14. 数据卷 Volume

持久化数据，容器删除后数据不丢失。

```
# 创建数据卷
docker volume create mydata

# 查看数据卷
docker volume ls

# 使用数据卷挂载
docker run -d -v mydata:/var/lib/mysql mysql

# 查看数据卷详情
docker volume inspect mydata

# 删除数据卷
docker volume rm mydata
```

## 15. 挂载方式对比

```
# 1. 命名卷（推荐，Docker 管理）
-v mydata:/data

# 2. 绑定挂载（宿主机路径）
-v /host/path:/container/path

# 3. 匿名卷（只指定容器路径）
-v /data
```

## 16. 网络

```
# 查看网络
docker network ls

# 创建自定义网络
docker network create mynet

# 容器加入同一网络，可用容器名互相访问
docker run -d --name app --network mynet myapp
docker run -d --name db --network mynet mysql

# 查看网络详情
docker network inspect mynet
```

## 17. Docker Compose

用 YAML 文件编排多个容器。

```
# docker-compose.yml
version: '3.8'

services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: app
    volumes:
      - db_data:/var/lib/mysql
    ports:
      - "3306:3306"

  app:
    build: .                    # 从当前目录 Dockerfile 构建
    ports:
      - "8080:8080"
    depends_on:
      - db                      # 依赖 db 先启动
    environment:
      DB_HOST: db               # 用服务名访问数据库

volumes:
  db_data:                      # 声明数据卷
```

```
# Compose 常用命令
docker-compose up -d        # 后台启动所有服务
docker-compose down         # 停止并删除容器
docker-compose ps           # 查看服务状态
docker-compose logs -f      # 查看日志
docker-compose restart      # 重启
```

## 18. 仓库与推送

```
# 登录仓库
docker login registry.example.com

# 给镜像打标签（包含仓库地址）
docker tag myapp:1.0 registry.example.com/myapp:1.0

# 推送镜像
docker push registry.example.com/myapp:1.0

# 拉取镜像
docker pull registry.example.com/myapp:1.0

# 登出
docker logout
```

## 19. 镜像分层

镜像由多个只读层叠加组成，每层对应 Dockerfile 的一条指令，相同层可共享复用。

```
# 查看镜像分层
docker history nginx

# 好处：构建缓存复用，相同层不必重复下载
```

```
# 优化：把变化频繁的放在后面，充分利用缓存
FROM node:18
COPY package.json .          # 依赖文件变化少，放前面
RUN npm install              # 依赖不变则用缓存
COPY . .                     # 源码变化频繁，放后面
```

## 20. .dockerignore

构建时忽略不需要的文件，类似 .gitignore。

```
# .dockerignore
node_modules
dist
*.log
.git
.env
```

## 21. 常用镜像运行示例

```
# Nginx（挂载静态文件和配置）
docker run -d \
  --name nginx \
  -p 80:80 \
  -v /data/nginx/html:/usr/share/nginx/html \
  -v /data/nginx/conf:/etc/nginx/conf.d \
  nginx

# Redis（带密码）
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7 \
  redis-server --requirepass 123456

# MySQL
docker run -d \
  --name mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -v /data/mysql:/var/lib/mysql \
  mysql:8.0

# Java 应用
docker run -d \
  --name app \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  myapp:1.0
```

## 22. 资源限制

```
# 限制 CPU 和内存
docker run -d \
  --name app \
  --cpus=1.5                 # 最多使用 1.5 个 CPU
  --memory=512m              # 最多使用 512MB 内存
  --memory-swap=1g           # 内存 + 交换空间上限
  myapp:1.0
```

## 23. 清理无用资源

```
# 清理所有停止的容器、未使用的网络、悬空镜像
docker system prune

# 更彻底：含未使用的镜像
docker system prune -a

# 单独清理
docker container prune    # 清理停止的容器
docker image prune        # 清理悬空镜像
docker volume prune       # 清理未使用的卷
```

## 24. 容器状态与生命周期

```
created      # 已创建未启动
running      # 运行中
paused       # 已暂停
stopped      # 已停止
exited       # 已退出
restarting   # 重启中
dead         # 已死亡
```

```
# 暂停与恢复
docker pause mynginx
docker unpause mynginx
```

## 25. 查看容器 IP 与端口

```
# 查看容器 IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mynginx

# 查看端口映射
docker port mynginx
```

## 26. Dockerfile 最佳实践

```
1. 使用官方基础镜像，指定具体版本
   FROM node:18-alpine   # 而非 node:latest

2. 合并 RUN 命令，减少镜像层
   RUN apt-get update && apt-get install -y curl \
       && rm -rf /var/lib/apt/lists/*

3. 使用多阶段构建减小体积

4. 使用 .dockerignore 排除无关文件

5. 非 root 用户运行
   RUN adduser -D appuser
   USER appuser

6. 优先使用 COPY 而非 ADD
```

## 27. 常用命令速查

| 命令 | 作用 |
|------|------|
| docker images | 列出镜像 |
| docker pull | 拉取镜像 |
| docker build | 构建镜像 |
| docker rmi | 删除镜像 |
| docker ps | 列出容器 |
| docker run | 运行容器 |
| docker start/stop/restart | 启停容器 |
| docker rm | 删除容器 |
| docker exec | 进入容器执行命令 |
| docker logs | 查看日志 |
| docker inspect | 查看详情 |
| docker stats | 查看资源占用 |
| docker cp | 拷贝文件 |
| docker volume | 管理数据卷 |
| docker network | 管理网络 |
| docker-compose | 编排容器 |
| docker system prune | 清理资源 |
