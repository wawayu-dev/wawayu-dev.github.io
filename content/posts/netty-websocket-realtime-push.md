---
title: "Netty + WebSocket 实现实时消息推送：从原理到生产实战"
date: 2025-04-18
tags: ["Netty", "分布式系统", "高并发", "最佳实践"]
categories: ["技术"]
description: "深入讲解如何使用 Netty 和 WebSocket 构建高性能实时消息推送系统，涵盖协议升级、心跳机制、集群部署等生产级实践"
draft: false
---

在现代互联网应用中，实时消息推送已成为标配功能。从即时通讯、在线客服到实时监控大屏，都离不开高性能的消息推送能力。传统的 HTTP 轮询方式存在延迟高、资源浪费等问题，而 WebSocket 协议的出现完美解决了这些痛点。

本文将深入讲解如何使用 Netty 框架实现基于 WebSocket 的实时消息推送系统，从协议原理到生产实战，帮助你构建一个高性能、高可用的推送服务。

### 本文亮点
- [x] 掌握 WebSocket 协议原理与握手流程
- [x] 学会使用 Netty 构建 WebSocket 服务器
- [x] 理解心跳机制与连接管理策略
- [x] 掌握集群部署与消息路由方案
- [x] 了解生产环境的性能优化技巧

---

## 为什么选择 Netty + WebSocket

### WebSocket 协议优势

相比传统的 HTTP 轮询，WebSocket 具有以下优势：

| 特性 | HTTP 轮询 | WebSocket |
|------|----------|-----------|
| 通信方式 | 半双工 | 全双工 |
| 连接开销 | 每次请求都需要建立连接 | 一次握手，持久连接 |
| 实时性 | 取决于轮询间隔（秒级） | 毫秒级 |
| 服务器压力 | 高（频繁的 HTTP 请求） | 低（长连接） |
| 带宽消耗 | 高（每次都有 HTTP 头） | 低（只传输数据帧） |

### Netty 框架优势

Netty 是一个高性能的异步事件驱动网络框架，具有以下特点：

1. **高性能**：基于 NIO，支持百万级并发连接
2. **易用性**：提供了丰富的编解码器和处理器
3. **稳定性**：经过大量生产环境验证
4. **扩展性**：灵活的 Pipeline 机制

> **架构思考：** 选择技术方案时，不仅要看技术本身的优势，更要考虑团队的技术栈、运维成本和社区生态。Netty 在 Java 生态中已经成为事实标准，Dubbo、RocketMQ、Elasticsearch 等知名项目都在使用。

---

## WebSocket 协议原理

### 握手流程

WebSocket 基于 HTTP 协议进行握手，然后升级为 WebSocket 协议：

```
客户端请求：
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

服务端响应：
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### 数据帧格式

WebSocket 使用帧（Frame）来传输数据，主要包括：

- **文本帧**：传输 UTF-8 编码的文本
- **二进制帧**：传输二进制数据
- **控制帧**：Ping、Pong、Close

---

## Netty WebSocket 服务器实现

### 项目依赖

```xml
<dependencies>
    <!-- Netty -->
    <dependency>
        <groupId>io.netty</groupId>
        <artifactId>netty-all</artifactId>
        <version>4.1.100.Final</version>
    </dependency>
    
    <!-- JSON 处理 -->
    <dependency>
        <groupId>com.alibaba.fastjson2</groupId>
        <artifactId>fastjson2</artifactId>
        <version>2.0.43</version>
    </dependency>
</dependencies>
```

### 服务器启动类

```java
@Slf4j
@Component
public class WebSocketServer {
    
    private final int port = 8888;
    private EventLoopGroup bossGroup;
    private EventLoopGroup workerGroup;
    
    @PostConstruct
    public void start() throws InterruptedException {
        bossGroup = new NioEventLoopGroup(1);
        workerGroup = new NioEventLoopGroup();
        
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new WebSocketChannelInitializer())
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);
            
            ChannelFuture future = bootstrap.bind(port).sync();
            log.info("WebSocket 服务器启动成功，端口：{}", port);
            
            // 等待服务器关闭
            future.channel().closeFuture().sync();
        } finally {
            shutdown();
        }
    }
    
    @PreDestroy
    public void shutdown() {
        if (bossGroup != null) {
            bossGroup.shutdownGracefully();
        }
        if (workerGroup != null) {
            workerGroup.shutdownGracefully();
        }
        log.info("WebSocket 服务器已关闭");
    }
}
```

### Channel 初始化器

```java
public class WebSocketChannelInitializer extends ChannelInitializer<SocketChannel> {
    
    @Override
    protected void initChannel(SocketChannel ch) {
        ChannelPipeline pipeline = ch.pipeline();
        
        // HTTP 编解码器
        pipeline.addLast(new HttpServerCodec());
        // HTTP 聚合器（将多个 HTTP 消息聚合成一个完整的消息）
        pipeline.addLast(new HttpObjectAggregator(65536));
        // HTTP 压缩
        pipeline.addLast(new HttpContentCompressor());
        
        // WebSocket 协议处理器（处理握手、Close、Ping、Pong）
        pipeline.addLast(new WebSocketServerProtocolHandler("/ws", null, true));
        
        // 心跳检测（读空闲 60 秒，写空闲 0 秒，读写空闲 0 秒）
        pipeline.addLast(new IdleStateHandler(60, 0, 0, TimeUnit.SECONDS));
        
        // 自定义业务处理器
        pipeline.addLast(new WebSocketHandler());
    }
}
```

### 业务处理器

```java
@Slf4j
@ChannelHandler.Sharable
public class WebSocketHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {
    
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        // 连接建立时
        Channel channel = ctx.channel();
        log.info("客户端连接：{}", channel.id().asShortText());
        
        // 将 Channel 加入管理器
        ChannelManager.addChannel(channel);
    }
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) {
        // 接收到客户端消息
        String content = msg.text();
        log.info("收到消息：{}", content);
        
        // 解析消息
        Message message = JSON.parseObject(content, Message.class);
        
        // 处理不同类型的消息
        switch (message.getType()) {
            case "auth":
                handleAuth(ctx, message);
                break;
            case "heartbeat":
                handleHeartbeat(ctx);
                break;
            case "message":
                handleMessage(ctx, message);
                break;
            default:
                log.warn("未知消息类型：{}", message.getType());
        }
    }
    
    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        // 连接断开时
        Channel channel = ctx.channel();
        log.info("客户端断开：{}", channel.id().asShortText());
        
        // 从管理器中移除
        ChannelManager.removeChannel(channel);
    }
    
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        // 处理空闲事件
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            if (event.state() == IdleState.READER_IDLE) {
                log.warn("客户端 {} 读空闲，关闭连接", ctx.channel().id().asShortText());
                ctx.close();
            }
        }
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        log.error("连接异常：{}", cause.getMessage(), cause);
        ctx.close();
    }
    
    /**
     * 处理认证消息
     */
    private void handleAuth(ChannelHandlerContext ctx, Message message) {
        String userId = message.getUserId();
        Channel channel = ctx.channel();
        
        // 绑定用户 ID 和 Channel
        ChannelManager.bindUser(userId, channel);
        
        // 发送认证成功消息
        Message response = Message.builder()
                .type("auth_response")
                .content("认证成功")
                .build();
        ctx.writeAndFlush(new TextWebSocketFrame(JSON.toJSONString(response)));
        
        log.info("用户 {} 认证成功", userId);
    }
    
    /**
     * 处理心跳消息
     */
    private void handleHeartbeat(ChannelHandlerContext ctx) {
        Message response = Message.builder()
                .type("heartbeat_response")
                .content("pong")
                .build();
        ctx.writeAndFlush(new TextWebSocketFrame(JSON.toJSONString(response)));
    }
    
    /**
     * 处理业务消息
     */
    private void handleMessage(ChannelHandlerContext ctx, Message message) {
        // 根据业务逻辑处理消息
        log.info("处理业务消息：{}", message.getContent());
        
        // 示例：广播消息给所有在线用户
        ChannelManager.broadcast(message);
    }
}
```

---

## 连接管理与消息推送

### Channel 管理器

```java
@Slf4j
public class ChannelManager {
    
    // 所有连接的 Channel
    private static final ChannelGroup CHANNELS = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
    
    // 用户 ID 与 Channel 的映射
    private static final ConcurrentHashMap<String, Channel> USER_CHANNELS = new ConcurrentHashMap<>();
    
    /**
     * 添加 Channel
     */
    public static void addChannel(Channel channel) {
        CHANNELS.add(channel);
        log.info("当前在线连接数：{}", CHANNELS.size());
    }
    
    /**
     * 移除 Channel
     */
    public static void removeChannel(Channel channel) {
        CHANNELS.remove(channel);
        
        // 移除用户绑定
        USER_CHANNELS.entrySet().removeIf(entry -> entry.getValue().equals(channel));
        
        log.info("当前在线连接数：{}", CHANNELS.size());
    }
    
    /**
     * 绑定用户和 Channel
     */
    public static void bindUser(String userId, Channel channel) {
        USER_CHANNELS.put(userId, channel);
        log.info("用户 {} 绑定成功，当前在线用户数：{}", userId, USER_CHANNELS.size());
    }
    
    /**
     * 推送消息给指定用户
     */
    public static void pushToUser(String userId, Message message) {
        Channel channel = USER_CHANNELS.get(userId);
        if (channel != null && channel.isActive()) {
            channel.writeAndFlush(new TextWebSocketFrame(JSON.toJSONString(message)));
            log.info("推送消息给用户 {}：{}", userId, message.getContent());
        } else {
            log.warn("用户 {} 不在线或连接已断开", userId);
        }
    }
    
    /**
     * 广播消息给所有用户
     */
    public static void broadcast(Message message) {
        TextWebSocketFrame frame = new TextWebSocketFrame(JSON.toJSONString(message));
        CHANNELS.writeAndFlush(frame);
        log.info("广播消息给 {} 个用户：{}", CHANNELS.size(), message.getContent());
    }
    
    /**
     * 获取在线用户数
     */
    public static int getOnlineUserCount() {
        return USER_CHANNELS.size();
    }
}
```

### 消息推送服务

```java
@Slf4j
@Service
public class MessagePushService {
    
    /**
     * 推送消息给指定用户
     */
    public void pushToUser(String userId, String content) {
        Message message = Message.builder()
                .type("push")
                .content(content)
                .timestamp(System.currentTimeMillis())
                .build();
        
        ChannelManager.pushToUser(userId, message);
    }
    
    /**
     * 推送消息给多个用户
     */
    public void pushToUsers(List<String> userIds, String content) {
        Message message = Message.builder()
                .type("push")
                .content(content)
                .timestamp(System.currentTimeMillis())
                .build();
        
        userIds.forEach(userId -> ChannelManager.pushToUser(userId, message));
    }
    
    /**
     * 广播消息给所有在线用户
     */
    public void broadcast(String content) {
        Message message = Message.builder()
                .type("broadcast")
                .content(content)
                .timestamp(System.currentTimeMillis())
                .build();
        
        ChannelManager.broadcast(message);
    }
}
```

---

## 心跳机制与连接保活

### 为什么需要心跳

在长连接场景下，客户端和服务端之间可能存在以下问题：

1. **网络中断检测**：网络异常时，TCP 连接不会立即断开
2. **NAT 超时**：NAT 设备会清理长时间无数据传输的连接
3. **负载均衡超时**：负载均衡器可能会关闭空闲连接

### 心跳实现方案

**服务端心跳检测**：

```java
// 在 ChannelInitializer 中添加
pipeline.addLast(new IdleStateHandler(60, 0, 0, TimeUnit.SECONDS));
```

**客户端心跳发送**（JavaScript 示例）：

```javascript
class WebSocketClient {
    constructor(url) {
        this.url = url;
        this.ws = null;
        this.heartbeatTimer = null;
        this.reconnectTimer = null;
    }
    
    connect() {
        this.ws = new WebSocket(this.url);
        
        this.ws.onopen = () => {
            console.log('WebSocket 连接成功');
            this.startHeartbeat();
            
            // 发送认证消息
            this.send({
                type: 'auth',
                userId: 'user123',
                token: 'xxx'
            });
        };
        
        this.ws.onmessage = (event) => {
            const message = JSON.parse(event.data);
            console.log('收到消息：', message);
            
            // 处理不同类型的消息
            switch (message.type) {
                case 'auth_response':
                    console.log('认证成功');
                    break;
                case 'push':
                    this.handlePush(message);
                    break;
                case 'heartbeat_response':
                    // 收到心跳响应
                    break;
            }
        };
        
        this.ws.onclose = () => {
            console.log('WebSocket 连接关闭');
            this.stopHeartbeat();
            this.reconnect();
        };
        
        this.ws.onerror = (error) => {
            console.error('WebSocket 错误：', error);
        };
    }
    
    send(message) {
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify(message));
        }
    }
    
    startHeartbeat() {
        this.heartbeatTimer = setInterval(() => {
            this.send({
                type: 'heartbeat',
                timestamp: Date.now()
            });
        }, 30000); // 每 30 秒发送一次心跳
    }
    
    stopHeartbeat() {
        if (this.heartbeatTimer) {
            clearInterval(this.heartbeatTimer);
            this.heartbeatTimer = null;
        }
    }
    
    reconnect() {
        if (this.reconnectTimer) {
            return;
        }
        
        this.reconnectTimer = setTimeout(() => {
            console.log('尝试重新连接...');
            this.reconnectTimer = null;
            this.connect();
        }, 5000); // 5 秒后重连
    }
    
    handlePush(message) {
        // 处理推送消息
        console.log('收到推送：', message.content);
    }
    
    close() {
        this.stopHeartbeat();
        if (this.ws) {
            this.ws.close();
        }
    }
}

// 使用示例
const client = new WebSocketClient('ws://localhost:8888/ws');
client.connect();
```

> **避坑提示：** 心跳间隔的设置需要权衡：间隔太短会增加服务器压力，间隔太长可能无法及时发现连接断开。建议客户端心跳间隔为 30 秒，服务端超时时间为 60 秒。

---

## 集群部署与消息路由

### 单机架构的局限

单机 WebSocket 服务器存在以下问题：

1. **连接数限制**：单机最多支持几万到十几万连接
2. **单点故障**：服务器宕机导致所有连接断开
3. **消息路由**：无法推送消息给其他服务器上的用户

### 集群架构方案

```
                    ┌─────────────┐
                    │   Nginx     │
                    │ (负载均衡)   │
                    └──────┬──────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
      ┌─────▼────┐   ┌─────▼────┐   ┌─────▼────┐
      │ WS Server│   │ WS Server│   │ WS Server│
      │   Node1  │   │   Node2  │   │   Node3  │
      └─────┬────┘   └─────┬────┘   └─────┬────┘
            │              │              │
            └──────────────┼──────────────┘
                           │
                    ┌──────▼──────┐
                    │    Redis    │
                    │ (消息路由)   │
                    └─────────────┘
```

### Nginx 配置

```nginx
upstream websocket {
    # IP Hash 策略，确保同一用户连接到同一台服务器
    ip_hash;
    
    server 192.168.1.101:8888;
    server 192.168.1.102:8888;
    server 192.168.1.103:8888;
}

server {
    listen 80;
    server_name ws.example.com;
    
    location /ws {
        proxy_pass http://websocket;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### Redis 消息路由

```java
@Slf4j
@Component
public class RedisMessageRouter {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    private static final String CHANNEL_PREFIX = "ws:message:";
    
    @PostConstruct
    public void init() {
        // 订阅 Redis 频道
        redisTemplate.getConnectionFactory().getConnection()
                .subscribe((message, pattern) -> {
                    String msg = new String(message.getBody());
                    handleMessage(msg);
                }, (CHANNEL_PREFIX + "*").getBytes());
    }
    
    /**
     * 发布消息到 Redis
     */
    public void publish(String userId, Message message) {
        String channel = CHANNEL_PREFIX + userId;
        redisTemplate.convertAndSend(channel, JSON.toJSONString(message));
    }
    
    /**
     * 处理从 Redis 接收到的消息
     */
    private void handleMessage(String msg) {
        Message message = JSON.parseObject(msg, Message.class);
        String userId = message.getUserId();
        
        // 如果用户在本节点，直接推送
        ChannelManager.pushToUser(userId, message);
    }
}
```

### 改进的推送服务

```java
@Slf4j
@Service
public class ClusterMessagePushService {
    
    @Autowired
    private RedisMessageRouter redisMessageRouter;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    /**
     * 推送消息给指定用户（集群版）
     */
    public void pushToUser(String userId, String content) {
        Message message = Message.builder()
                .type("push")
                .userId(userId)
                .content(content)
                .timestamp(System.currentTimeMillis())
                .build();
        
        // 先尝试本地推送
        Channel channel = ChannelManager.getUserChannel(userId);
        if (channel != null && channel.isActive()) {
            ChannelManager.pushToUser(userId, message);
        } else {
            // 用户不在本节点，通过 Redis 路由
            redisMessageRouter.publish(userId, message);
        }
    }
    
    /**
     * 广播消息给所有在线用户（集群版）
     */
    public void broadcast(String content) {
        Message message = Message.builder()
                .type("broadcast")
                .content(content)
                .timestamp(System.currentTimeMillis())
                .build();
        
        // 发布到 Redis 广播频道
        redisTemplate.convertAndSend("ws:broadcast", JSON.toJSONString(message));
    }
}
```

> **架构思考：** 在集群架构中，消息路由是核心问题。除了 Redis Pub/Sub，还可以考虑使用 RocketMQ、Kafka 等消息队列。选择时需要考虑：1) 消息可靠性要求；2) 消息顺序性要求；3) 系统复杂度。对于实时性要求高但可靠性要求不高的场景，Redis Pub/Sub 是不错的选择。

---

## 生产环境优化

### 性能优化

**1. 线程池配置**

```java
// Boss 线程组：处理连接请求
EventLoopGroup bossGroup = new NioEventLoopGroup(1);

// Worker 线程组：处理 I/O 操作
// 默认线程数 = CPU 核心数 * 2
EventLoopGroup workerGroup = new NioEventLoopGroup();

// 业务线程池：处理业务逻辑
ThreadPoolExecutor businessExecutor = new ThreadPoolExecutor(
    10, 50, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),
    new ThreadFactoryBuilder().setNameFormat("business-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

**2. 内存优化**

```java
// 使用池化的 ByteBuf
bootstrap.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
bootstrap.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

// 设置接收缓冲区大小
bootstrap.childOption(ChannelOption.RCVBUF_ALLOCATOR, 
    new AdaptiveRecvByteBufAllocator(64, 1024, 65536));
```

**3. 连接数优化**

```java
// 增加 TCP 连接队列长度
bootstrap.option(ChannelOption.SO_BACKLOG, 1024);

// 启用 TCP_NODELAY（禁用 Nagle 算法）
bootstrap.childOption(ChannelOption.TCP_NODELAY, true);

// 启用 SO_KEEPALIVE
bootstrap.childOption(ChannelOption.SO_KEEPALIVE, true);
```

### 监控指标

```java
@Component
public class WebSocketMetrics {
    
    private final AtomicLong totalConnections = new AtomicLong(0);
    private final AtomicLong activeConnections = new AtomicLong(0);
    private final AtomicLong totalMessages = new AtomicLong(0);
    
    /**
     * 记录连接建立
     */
    public void recordConnect() {
        totalConnections.incrementAndGet();
        activeConnections.incrementAndGet();
    }
    
    /**
     * 记录连接断开
     */
    public void recordDisconnect() {
        activeConnections.decrementAndGet();
    }
    
    /**
     * 记录消息发送
     */
    public void recordMessage() {
        totalMessages.incrementAndGet();
    }
    
    /**
     * 获取监控数据
     */
    public Map<String, Object> getMetrics() {
        Map<String, Object> metrics = new HashMap<>();
        metrics.put("totalConnections", totalConnections.get());
        metrics.put("activeConnections", activeConnections.get());
        metrics.put("totalMessages", totalMessages.get());
        metrics.put("onlineUsers", ChannelManager.getOnlineUserCount());
        return metrics;
    }
}
```

### 安全加固

**1. 认证鉴权**

```java
private void handleAuth(ChannelHandlerContext ctx, Message message) {
    String token = message.getToken();
    
    // 验证 Token
    if (!validateToken(token)) {
        ctx.writeAndFlush(new TextWebSocketFrame(
            JSON.toJSONString(Message.builder()
                .type("auth_response")
                .code(401)
                .content("认证失败")
                .build())
        ));
        ctx.close();
        return;
    }
    
    // 认证成功，绑定用户
    String userId = getUserIdFromToken(token);
    ChannelManager.bindUser(userId, ctx.channel());
}
```

**2. 消息限流**

```java
@Component
public class MessageRateLimiter {
    
    private final LoadingCache<String, RateLimiter> limiters = CacheBuilder.newBuilder()
            .maximumSize(10000)
            .expireAfterAccess(1, TimeUnit.HOURS)
            .build(new CacheLoader<String, RateLimiter>() {
                @Override
                public RateLimiter load(String key) {
                    // 每秒最多 10 条消息
                    return RateLimiter.create(10.0);
                }
            });
    
    public boolean tryAcquire(String userId) {
        try {
            RateLimiter limiter = limiters.get(userId);
            return limiter.tryAcquire();
        } catch (ExecutionException e) {
            return false;
        }
    }
}
```

**3. XSS 防护**

```java
private String sanitizeContent(String content) {
    // 过滤 HTML 标签和脚本
    return content.replaceAll("<[^>]*>", "")
                  .replaceAll("javascript:", "")
                  .replaceAll("on\\w+\\s*=", "");
}
```

> **避坑提示：** 生产环境中，安全性至关重要。除了上述措施，还需要：1) 使用 WSS（WebSocket over TLS）加密传输；2) 设置合理的消息大小限制；3) 实施 IP 黑白名单；4) 记录审计日志。

---

## 总结与思考

本文深入讲解了如何使用 Netty 和 WebSocket 构建高性能实时消息推送系统，涵盖了从协议原理到生产实战的完整内容：

- **协议原理**：理解 WebSocket 握手流程和数据帧格式
- **服务器实现**：使用 Netty 构建 WebSocket 服务器
- **连接管理**：实现 Channel 管理和消息推送
- **心跳机制**：保证连接可靠性和及时检测断线
- **集群部署**：通过 Redis 实现跨节点消息路由
- **性能优化**：线程池、内存、连接数等多方面优化
- **安全加固**：认证鉴权、限流、XSS 防护

在实际应用中，还需要根据具体业务场景进行调整。例如：

- **即时通讯**：需要考虑消息可靠性、离线消息、消息顺序
- **在线客服**：需要考虑会话管理、消息路由、客服分配
- **实时监控**：需要考虑数据聚合、告警推送、大屏展示

下次当你需要实现实时消息推送功能时，不妨试试 Netty + WebSocket 这个强大的组合。