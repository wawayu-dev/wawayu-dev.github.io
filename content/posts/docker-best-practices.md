---
title: "Docker 容器化部署最佳实践：从 Dockerfile 到生产环境"
date: 2024-09-10
tags: ["Docker", "CI/CD", "最佳实践", "Linux"]
categories: ["技术"]
description: "掌握 Docker 容器化的核心技术，从镜像构建到生产部署的完整实践"
draft: false
---

容器化已经成为现代应用部署的标准方式。Docker 让应用部署变得简单、可靠、可移植。本文讲解 Docker 在生产环境中的最佳实践。

### 本文亮点
- [x] 掌握 Dockerfile 的编写技巧与优化方法
- [x] 理解 Docker 镜像的分层构建原理
- [x] 学会使用多阶段构建减少镜像体积
- [x] 了解 Docker Compose 编排多服务环境

---

## 为什么需要容器化？

**传统部署的痛点**

1. 环境不一致："在我机器上能跑"
2. 依赖管理复杂：JDK、Tomcat、MySQL 版本不一致
3. 部署流程繁琐：手动上传、解压、配置、启动
4. 资源利用率低：一台服务器只跑一个应用

**容器化的优势**

1. 环境一致性：开发、测试、生产环境完全一致
2. 快速部署：秒级启动，一键部署
3. 资源隔离：每个容器独立运行，互不影响
4. 易于扩展：水平扩展只需启动更多容器

---

## Dockerfile 基础

**最简单的 Dockerfile**

```dockerfile
FROM openjdk:11-jre-slim
COPY app.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

**构建镜像**

```bash
docker build -t myapp:1.0 .
```

**运行容器**

```bash
docker run -d -p 8080:8080 myapp:1.0
```

---

## Dockerfile 最佳实践

### 1. 选择合适的基础镜像

**不好的做法**

```dockerfile
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y openjdk-11-jdk
```

镜像大小：600MB+

**好的做法**

```dockerfile
FROM openjdk:11-jre-slim
```

镜像大小：200MB

**更好的做法：使用 Alpine**

```dockerfile
FROM openjdk:11-jre-alpine
```

镜像大小：150MB

> **选择原则：** 优先使用官方镜像，选择最小的满足需求的版本（slim > alpine）

---

### 2. 利用构建缓存

Docker 构建镜像时，会缓存每一层。如果某一层没有变化，会直接使用缓存。

**不好的做法**

```dockerfile
FROM openjdk:11-jre-slim
COPY . /app
RUN cd /app && ./mvnw clean package
```

每次代码变化，都要重新下载依赖。

**好的做法：分离依赖和代码**

```dockerfile
FROM openjdk:11-jre-slim

# 先复制依赖文件
COPY pom.xml /app/
COPY mvnw /app/
COPY .mvn /app/.mvn
WORKDIR /app
RUN ./mvnw dependency:go-offline

# 再复制代码
COPY src /app/src
RUN ./mvnw clean package -DskipTests

ENTRYPOINT ["java", "-jar", "/app/target/app.jar"]
```

这样，只有 pom.xml 变化时才会重新下载依赖。

---

### 3. 多阶段构建

**问题：** 构建工具（Maven、Gradle）会增加镜像体积。

**解决方案：** 使用多阶段构建，只保留运行时需要的文件。

```dockerfile
# 第一阶段：构建
FROM maven:3.8-openjdk-11 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# 第二阶段：运行
FROM openjdk:11-jre-slim
WORKDIR /app
COPY --from=builder /app/target/app.jar .
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**效果：**
- 构建阶段镜像：800MB
- 最终镜像：200MB

---

### 4. 减少镜像层数

**不好的做法**

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get clean
```

每个 RUN 都会创建一层，增加镜像体积。

**好的做法：合并命令**

```dockerfile
RUN apt-get update && \
    apt-get install -y curl vim && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

---

### 5. 使用 .dockerignore

类似 .gitignore，避免将不必要的文件复制到镜像中。

```
# .dockerignore
target/
*.log
.git/
.idea/
*.md
```

---

### 6. 不要在容器中存储数据

容器是临时的，数据应该存储在外部。

**使用 Volume 挂载数据**

```bash
docker run -d -v /data/mysql:/var/lib/mysql mysql:8.0
```

**使用命名卷**

```bash
docker volume create mysql-data
docker run -d -v mysql-data:/var/lib/mysql mysql:8.0
```

---

### 7. 健康检查

```dockerfile
FROM openjdk:11-jre-slim
COPY app.jar /app.jar

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "/app.jar"]
```

Docker 会定期检查容器健康状态，不健康的容器会被重启。

---

### 8. 使用非 root 用户

**安全风险：** 默认情况下，容器内的进程以 root 用户运行。

**解决方案：** 创建普通用户

```dockerfile
FROM openjdk:11-jre-slim

# 创建用户
RUN groupadd -r appuser && useradd -r -g appuser appuser

# 复制文件
COPY app.jar /app.jar
RUN chown appuser:appuser /app.jar

# 切换用户
USER appuser

ENTRYPOINT ["java", "-jar", "/app.jar"]
```

---

## 优化 Spring Boot 应用的 Dockerfile

**完整示例**

```dockerfile
# 第一阶段：构建
FROM maven:3.8-openjdk-11-slim AS builder
WORKDIR /build

# 复制依赖文件
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .
RUN ./mvnw dependency:go-offline -B

# 复制源码并构建
COPY src src
RUN ./mvnw package -DskipTests -B

# 解压 jar 包（利用 Spring Boot 的分层特性）
RUN mkdir -p target/dependency && \
    cd target/dependency && \
    jar -xf ../*.jar

# 第二阶段：运行
FROM openjdk:11-jre-slim
WORKDIR /app

# 创建非 root 用户
RUN groupadd -r spring && useradd -r -g spring spring

# 分层复制（利用 Docker 缓存）
COPY --from=builder /build/target/dependency/BOOT-INF/lib /app/lib
COPY --from=builder /build/target/dependency/META-INF /app/META-INF
COPY --from=builder /build/target/dependency/BOOT-INF/classes /app

# 修改所有者
RUN chown -R spring:spring /app

# 切换用户
USER spring

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# 启动应用
ENTRYPOINT ["java", "-cp", "/app:/app/lib/*", "com.example.Application"]
```

**优化效果：**
- 利用分层缓存，依赖变化时不需要重新构建整个应用
- 使用非 root 用户，提升安全性
- 添加健康检查，提升可靠性

---

## Docker Compose：编排多服务

**场景：** 应用需要 MySQL、Redis、Nginx

**docker-compose.yml**

```yaml
version: '3.8'

services:
  # MySQL 数据库
  mysql:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: myapp
    volumes:
      - mysql-data:/var/lib/mysql
    ports:
      - "3306:3306"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis 缓存
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # Spring Boot 应用
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: myapp
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/myapp
      SPRING_REDIS_HOST: redis
    ports:
      - "8080:8080"
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped

  # Nginx 反向代理
  nginx:
    image: nginx:alpine
    container_name: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "80:80"
    depends_on:
      - app
    networks:
      - app-network

volumes:
  mysql-data:

networks:
  app-network:
    driver: bridge
```

**启动所有服务**

```bash
docker-compose up -d
```

**查看日志**

```bash
docker-compose logs -f app
```

**停止所有服务**

```bash
docker-compose down
```

---

## 生产环境部署

### 1. 使用环境变量

**不要在镜像中硬编码配置**

```dockerfile
# 不好
ENV SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/myapp
```

**使用环境变量**

```bash
docker run -d \
  -e SPRING_DATASOURCE_URL=jdbc:mysql://prod-mysql:3306/myapp \
  -e SPRING_DATASOURCE_PASSWORD=prod_password \
  myapp:1.0
```

### 2. 资源限制

```bash
docker run -d \
  --memory="512m" \
  --cpus="1.0" \
  myapp:1.0
```

### 3. 日志管理

**使用日志驱动**

```bash
docker run -d \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp:1.0
```

### 4. 自动重启

```bash
docker run -d --restart=unless-stopped myapp:1.0
```

重启策略：
- `no`：不自动重启（默认）
- `on-failure`：失败时重启
- `always`：总是重启
- `unless-stopped`：除非手动停止，否则总是重启

---

## CI/CD 集成

**GitLab CI 示例**

```yaml
# .gitlab-ci.yml
stages:
  - build
  - deploy

build:
  stage: build
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker tag myapp:$CI_COMMIT_SHA myapp:latest
    - docker push myapp:$CI_COMMIT_SHA
    - docker push myapp:latest

deploy:
  stage: deploy
  script:
    - docker pull myapp:$CI_COMMIT_SHA
    - docker stop myapp || true
    - docker rm myapp || true
    - docker run -d --name myapp -p 8080:8080 myapp:$CI_COMMIT_SHA
  only:
    - main
```

---

## 常见问题与解决方案

### 1. 容器时区问题

```dockerfile
FROM openjdk:11-jre-slim
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
```

### 2. 容器内无法访问宿主机服务

使用 `host.docker.internal`（Mac/Windows）或 `172.17.0.1`（Linux）

```yaml
environment:
  SPRING_DATASOURCE_URL: jdbc:mysql://host.docker.internal:3306/myapp
```

### 3. 镜像构建慢

- 使用国内镜像源
- 利用构建缓存
- 使用多阶段构建

### 4. 容器内存溢出

设置 JVM 参数：

```dockerfile
ENTRYPOINT ["java", "-Xmx512m", "-Xms512m", "-jar", "/app.jar"]
```

---

## 总结与思考

Docker 容器化的核心要点：

- 选择合适的基础镜像，减少镜像体积
- 利用构建缓存和多阶段构建，提升构建速度
- 使用非 root 用户，提升安全性
- 使用 Docker Compose 编排多服务
- 在生产环境中设置资源限制和健康检查

容器化不仅仅是技术，更是一种思维方式。它让我们重新思考应用的部署和运维方式。

下次当你需要部署应用时，不妨试试 Docker，体验容器化带来的便利。
