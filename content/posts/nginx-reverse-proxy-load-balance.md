---
title: "Nginx 反向代理与负载均衡实战：从入门到生产优化"
date: 2025-06-25
tags: ["Nginx", "Linux", "最佳实践"]
categories: ["技术"]
description: "深入讲解 Nginx 反向代理和负载均衡的原理与实践，涵盖配置优化、健康检查、会话保持等生产级方案"
draft: false
---

Nginx 作为高性能的 Web 服务器和反向代理服务器，已经成为互联网架构中的标配组件。无论是静态资源服务、反向代理还是负载均衡，Nginx 都能提供出色的性能和稳定性。

本文将从实际场景出发，深入讲解 Nginx 反向代理和负载均衡的配置与优化，帮助你构建高可用的服务架构。

### 本文亮点
- [x] 理解反向代理与正向代理的区别
- [x] 掌握 Nginx 负载均衡的多种策略
- [x] 学会配置健康检查与故障转移
- [x] 了解会话保持的实现方案
- [x] 掌握生产环境的性能调优技巧

---

## 反向代理基础

### 正向代理 vs 反向代理

**正向代理**：客户端知道目标服务器，通过代理服务器访问
```
客户端 → 代理服务器 → 目标服务器
（客户端配置代理）
```

**反向代理**：客户端不知道真实服务器，代理服务器决定转发到哪台服务器
```
客户端 → 反向代理 → 后端服务器集群
（服务端配置代理）
```

### 反向代理的优势

1. **负载均衡**：将请求分发到多台服务器
2. **隐藏真实服务器**：提高安全性
3. **SSL 卸载**：在代理层处理 HTTPS
4. **缓存静态资源**：减轻后端压力
5. **统一入口**：便于管理和监控

> **架构思考：** 反向代理不仅是流量分发器，更是架构中的关键节点。通过反向代理，可以实现灰度发布、A/B 测试、流量控制等高级功能。

---

## Nginx 反向代理配置

### 基础配置

```nginx
# /etc/nginx/nginx.conf

user nginx;
worker_processes auto;  # 自动设置为 CPU 核心数
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;  # 每个 worker 进程的最大连接数
    use epoll;  # Linux 下使用 epoll
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    '$request_time $upstream_response_time';
    
    access_log /var/log/nginx/access.log main;
    
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # 包含其他配置文件
    include /etc/nginx/conf.d/*.conf;
}
```

### 简单反向代理

```nginx
# /etc/nginx/conf.d/api.conf

server {
    listen 80;
    server_name api.example.com;
    
    location / {
        # 代理到后端服务器
        proxy_pass http://192.168.1.100:8080;
        
        # 设置请求头
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### 路径重写

```nginx
server {
    listen 80;
    server_name api.example.com;
    
    # 将 /api/v1/users 代理到 http://backend/users
    location /api/v1/ {
        proxy_pass http://backend/;  # 注意末尾的 /
        proxy_set_header Host $host;
    }
    
    # 将 /old-api 重写为 /new-api
    location /old-api {
        rewrite ^/old-api/(.*)$ /new-api/$1 break;
        proxy_pass http://backend;
    }
}
```

---

## 负载均衡策略

### 1. 轮询（Round Robin）

默认策略，按顺序依次分配请求：

```nginx
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}

server {
    listen 80;
    server_name api.example.com;
    
    location / {
        proxy_pass http://backend;
    }
}
```

### 2. 加权轮询（Weighted Round Robin）

根据服务器性能分配不同权重：

```nginx
upstream backend {
    server 192.168.1.101:8080 weight=3;  # 权重 3
    server 192.168.1.102:8080 weight=2;  # 权重 2
    server 192.168.1.103:8080 weight=1;  # 权重 1
}
```

权重为 3:2:1，意味着每 6 个请求中，101 服务器处理 3 个，102 处理 2 个，103 处理 1 个。

### 3. IP Hash

根据客户端 IP 进行哈希，同一 IP 的请求总是分配到同一台服务器：

```nginx
upstream backend {
    ip_hash;
    
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}
```

**适用场景**：需要会话保持的应用（如购物车、登录状态）

**缺点**：
- 负载可能不均衡（IP 分布不均）
- 无法应对服务器故障（IP 重新哈希）

### 4. 最少连接（Least Connections）

将请求分配给当前连接数最少的服务器：

```nginx
upstream backend {
    least_conn;
    
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}
```

**适用场景**：请求处理时间差异较大的应用

### 5. 一致性哈希（需要第三方模块）

```nginx
upstream backend {
    hash $request_uri consistent;
    
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}
```

**适用场景**：缓存服务器集群

### 负载均衡策略对比

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 轮询 | 简单、均衡 | 无会话保持 | 无状态服务 |
| 加权轮询 | 考虑服务器性能 | 无会话保持 | 服务器性能不同 |
| IP Hash | 会话保持 | 负载可能不均 | 有状态服务 |
| 最少连接 | 动态均衡 | 计算开销大 | 长连接服务 |
| 一致性哈希 | 缓存命中率高 | 实现复杂 | 缓存集群 |

---

## 健康检查与故障转移

### 被动健康检查

Nginx 默认的健康检查机制，当请求失败时标记服务器为不可用：

```nginx
upstream backend {
    server 192.168.1.101:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.102:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.103:8080 max_fails=3 fail_timeout=30s;
}
```

参数说明：
- `max_fails=3`：连续失败 3 次后标记为不可用
- `fail_timeout=30s`：标记为不可用后，30 秒内不再分配请求

### 主动健康检查（Nginx Plus 或第三方模块）

使用 `nginx_upstream_check_module` 模块：

```nginx
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
    
    check interval=3000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "HEAD /health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}

server {
    listen 80;
    
    location /status {
        check_status;
        access_log off;
    }
}
```

参数说明：
- `interval=3000`：每 3 秒检查一次
- `rise=2`：连续成功 2 次标记为可用
- `fall=3`：连续失败 3 次标记为不可用
- `timeout=1000`：超时时间 1 秒
- `type=http`：使用 HTTP 协议检查

### 备用服务器

```nginx
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
    server 192.168.1.104:8080 backup;  # 备用服务器
}
```

只有当所有主服务器都不可用时，才会使用备用服务器。

### 故障转移配置

```nginx
location / {
    proxy_pass http://backend;
    proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
    proxy_next_upstream_tries 2;  # 最多尝试 2 次
    proxy_next_upstream_timeout 10s;  # 总超时时间 10 秒
}
```

---

## 会话保持方案

### 方案一：IP Hash

```nginx
upstream backend {
    ip_hash;
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}
```

**优点**：配置简单
**缺点**：负载可能不均衡

### 方案二：Cookie 绑定

```nginx
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    
    sticky cookie srv_id expires=1h domain=.example.com path=/;
}
```

Nginx 会在响应中设置一个 Cookie，后续请求根据 Cookie 路由到同一台服务器。

### 方案三：Session 共享（推荐）

不依赖 Nginx，在应用层实现 Session 共享：

**使用 Redis 存储 Session**：

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {
    
    @Bean
    public LettuceConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory();
    }
}
```

**使用 JWT Token**：

```java
@RestController
public class AuthController {
    
    @PostMapping("/login")
    public Result login(@RequestBody LoginRequest request) {
        // 验证用户名密码
        User user = userService.authenticate(request.getUsername(), request.getPassword());
        
        // 生成 JWT Token
        String token = JwtUtil.generateToken(user.getId());
        
        return Result.success(token);
    }
}
```

> **架构思考：** 会话保持的最佳实践是在应用层实现无状态化。通过 Redis 共享 Session 或使用 JWT Token，可以实现真正的水平扩展，而不依赖于负载均衡器的特定功能。

---

## HTTPS 配置

### SSL 证书配置

```nginx
server {
    listen 443 ssl http2;
    server_name api.example.com;
    
    # SSL 证书
    ssl_certificate /etc/nginx/ssl/api.example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/api.example.com.key;
    
    # SSL 协议和加密套件
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers on;
    
    # SSL 会话缓存
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=31536000" always;
    
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name api.example.com;
    return 301 https://$server_name$request_uri;
}
```

### 免费 SSL 证书（Let's Encrypt）

```bash
# 安装 Certbot
sudo yum install certbot python3-certbot-nginx

# 获取证书
sudo certbot --nginx -d api.example.com

# 自动续期
sudo certbot renew --dry-run
```

---

## 性能优化

### 1. 连接优化

```nginx
http {
    # 长连接配置
    keepalive_timeout 65;
    keepalive_requests 100;
    
    # 上游服务器长连接
    upstream backend {
        server 192.168.1.101:8080;
        server 192.168.1.102:8080;
        
        keepalive 32;  # 保持 32 个空闲连接
    }
    
    server {
        location / {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
}
```

### 2. 缓存配置

```nginx
# 定义缓存路径
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m;

server {
    location / {
        proxy_pass http://backend;
        
        # 启用缓存
        proxy_cache my_cache;
        proxy_cache_valid 200 304 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_key $scheme$proxy_host$request_uri;
        
        # 缓存头信息
        add_header X-Cache-Status $upstream_cache_status;
    }
    
    # 静态资源缓存
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        proxy_pass http://backend;
        proxy_cache my_cache;
        proxy_cache_valid 200 30d;
        expires 30d;
    }
}
```

### 3. Gzip 压缩

```nginx
http {
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss 
               application/rss+xml font/truetype font/opentype 
               application/vnd.ms-fontobject image/svg+xml;
    gzip_disable "msie6";
}
```

### 4. 限流配置

```nginx
http {
    # 定义限流区域
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
    
    server {
        location /api/ {
            # 限制请求速率：每秒 10 个请求，突发 20 个
            limit_req zone=api_limit burst=20 nodelay;
            
            # 限制并发连接数：每个 IP 最多 10 个连接
            limit_conn conn_limit 10;
            
            proxy_pass http://backend;
        }
    }
}
```

### 5. Worker 进程优化

```nginx
# 自动设置 worker 进程数为 CPU 核心数
worker_processes auto;

# 绑定 worker 进程到 CPU 核心
worker_cpu_affinity auto;

# 每个 worker 进程的最大文件描述符数
worker_rlimit_nofile 65535;

events {
    # 每个 worker 进程的最大连接数
    worker_connections 10240;
    
    # 使用 epoll 事件模型
    use epoll;
    
    # 尽可能多地接受连接
    multi_accept on;
}
```

---

## 监控与日志

### 访问日志分析

```nginx
# 自定义日志格式
log_format detailed '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '$request_time $upstream_response_time '
                    '$upstream_addr $upstream_status';

access_log /var/log/nginx/access.log detailed;
```

### 使用 GoAccess 分析日志

```bash
# 安装 GoAccess
sudo yum install goaccess

# 实时分析日志
goaccess /var/log/nginx/access.log -o /var/www/html/report.html --log-format=COMBINED --real-time-html
```

### Nginx 状态监控

```nginx
server {
    listen 8080;
    server_name localhost;
    
    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }
}
```

访问 `http://localhost:8080/nginx_status` 查看状态：

```
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106
```

### Prometheus 监控

使用 `nginx-prometheus-exporter`：

```bash
# 下载并运行 exporter
docker run -p 9113:9113 nginx/nginx-prometheus-exporter:latest \
  -nginx.scrape-uri=http://nginx:8080/nginx_status
```

---

## 生产环境最佳实践

### 1. 配置文件组织

```
/etc/nginx/
├── nginx.conf              # 主配置文件
├── conf.d/
│   ├── api.conf           # API 服务配置
│   ├── web.conf           # Web 服务配置
│   └── admin.conf         # 管理后台配置
├── upstream/
│   ├── api-backend.conf   # API 后端服务器列表
│   └── web-backend.conf   # Web 后端服务器列表
└── ssl/
    ├── api.example.com.crt
    └── api.example.com.key
```

### 2. 配置检查与重载

```bash
# 检查配置文件语法
sudo nginx -t

# 重载配置（不中断服务）
sudo nginx -s reload

# 查看 Nginx 版本和编译参数
nginx -V
```

### 3. 安全加固

```nginx
# 隐藏 Nginx 版本号
server_tokens off;

# 限制请求方法
if ($request_method !~ ^(GET|POST|PUT|DELETE|HEAD)$) {
    return 405;
}

# 防止点击劫持
add_header X-Frame-Options "SAMEORIGIN" always;

# XSS 防护
add_header X-XSS-Protection "1; mode=block" always;

# 内容类型嗅探防护
add_header X-Content-Type-Options "nosniff" always;

# CSP 策略
add_header Content-Security-Policy "default-src 'self'" always;
```

### 4. 日志轮转

```bash
# /etc/logrotate.d/nginx

/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 nginx adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

> **避坑提示：** 生产环境中，务必做好以下几点：1) 定期备份配置文件；2) 使用版本控制管理配置；3) 配置变更前先在测试环境验证；4) 设置监控告警；5) 准备回滚方案。

---

## 总结与思考

本文深入讲解了 Nginx 反向代理和负载均衡的配置与优化，从基础概念到生产实践：

- **反向代理**：理解原理，掌握基础配置和路径重写
- **负载均衡**：掌握多种策略，根据场景选择合适方案
- **健康检查**：实现故障检测和自动转移
- **会话保持**：了解多种方案，推荐应用层无状态化
- **HTTPS 配置**：SSL 证书配置和安全加固
- **性能优化**：连接优化、缓存、压缩、限流等
- **监控日志**：日志分析和状态监控

在实际应用中，Nginx 的作用远不止于此。通过合理配置，还可以实现：

- **灰度发布**：根据条件将部分流量导向新版本
- **A/B 测试**：将不同用户导向不同版本
- **流量控制**：限流、熔断、降级
- **安全防护**：WAF、DDoS 防护

下次当你需要搭建高可用服务架构时，Nginx 将是你的得力助手。