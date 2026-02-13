---
title: "Redis 缓存架构：从穿透到雪崩的全链路防护"
date: 2025-03-20
tags: ["Redis", "缓存", "高并发", "最佳实践"]
categories: ["技术"]
description: "深入理解缓存穿透、击穿、雪崩问题，构建高可用的Redis缓存架构"
draft: false
---

在高并发系统中，缓存是提升性能的关键。但缓存使用不当会引发穿透、击穿、雪崩等问题，导致数据库崩溃。本文将深入分析这些问题的原理，并提供完整的解决方案。

### 本文亮点
- [x] 理解缓存穿透、击穿、雪崩的本质区别
- [x] 掌握布隆过滤器、互斥锁等防护手段
- [x] 学会设计高可用的缓存架构
- [x] 了解缓存预热、降级等最佳实践

---

## 缓存的三大问题

### 问题对比

| 问题 | 原因 | 影响 | 解决方案 |
|------|------|------|----------|
| 缓存穿透 | 查询不存在的数据 | 大量请求打到DB | 布隆过滤器、空值缓存 |
| 缓存击穿 | 热点key过期 | 瞬间大量请求打到DB | 互斥锁、永不过期 |
| 缓存雪崩 | 大量key同时过期 | DB压力骤增 | 过期时间随机化、多级缓存 |

---

## 缓存穿透：查询不存在的数据

### 问题场景

```java
// 恶意攻击：查询不存在的用户ID
for (int i = -1000000; i < 0; i--) {
    User user = getUser(i);  // 缓存和DB都没有，每次都查DB
}
```

**流程：**
```
请求 → 查缓存（未命中）→ 查数据库（未命中）→ 返回null
```

每次请求都会穿透缓存，直接打到数据库。

### 解决方案1：布隆过滤器

**原理：** 用位数组判断元素是否存在，存在一定的误判率。

```java
@Configuration
public class BloomFilterConfig {
    
    @Bean
    public BloomFilter<Long> userBloomFilter() {
        // 预期元素数量：1000万，误判率：0.01%
        BloomFilter<Long> bloomFilter = BloomFilter.create(
            Funnels.longFunnel(),
            10000000,
            0.0001
        );
        
        // 初始化：将所有用户ID加入布隆过滤器
        List<Long> userIds = userMapper.selectAllIds();
        userIds.forEach(bloomFilter::put);
        
        return bloomFilter;
    }
}
```

```java
@Service
public class UserService {
    
    @Autowired
    private BloomFilter<Long> userBloomFilter;
    
    @Autowired
    private RedisTemplate<String, User> redisTemplate;
    
    @Autowired
    private UserMapper userMapper;
    
    public User getUser(Long userId) {
        // 1. 布隆过滤器判断
        if (!userBloomFilter.mightContain(userId)) {
            return null;  // 一定不存在，直接返回
        }
        
        // 2. 查缓存
        String key = "user:" + userId;
        User user = redisTemplate.opsForValue().get(key);
        if (user != null) {
            return user;
        }
        
        // 3. 查数据库
        user = userMapper.selectById(userId);
        if (user != null) {
            redisTemplate.opsForValue().set(key, user, 1, TimeUnit.HOURS);
        }
        
        return user;
    }
}
```

**优点：**
- 内存占用小（1000万数据只需约12MB）
- 查询速度快（O(k)，k为哈希函数个数）

**缺点：**
- 存在误判（可能判断存在但实际不存在）
- 无法删除元素

### 解决方案2：缓存空值

```java
public User getUser(Long userId) {
    String key = "user:" + userId;
    
    // 1. 查缓存
    User user = redisTemplate.opsForValue().get(key);
    if (user != null) {
        // 空对象标记
        if (user.getId() == null) {
            return null;
        }
        return user;
    }
    
    // 2. 查数据库
    user = userMapper.selectById(userId);
    
    // 3. 缓存结果（包括null）
    if (user == null) {
        // 缓存空对象，设置较短的过期时间
        User emptyUser = new User();
        redisTemplate.opsForValue().set(key, emptyUser, 5, TimeUnit.MINUTES);
    } else {
        redisTemplate.opsForValue().set(key, user, 1, TimeUnit.HOURS);
    }
    
    return user;
}
```

**优点：**
- 实现简单
- 可以防止同一个不存在的key反复查询

**缺点：**
- 占用缓存空间
- 可能缓存大量无效数据

### 解决方案3：布隆过滤器 + 空值缓存

```java
public User getUser(Long userId) {
    // 1. 布隆过滤器快速判断
    if (!userBloomFilter.mightContain(userId)) {
        return null;
    }
    
    String key = "user:" + userId;
    
    // 2. 查缓存（包括空值）
    User user = redisTemplate.opsForValue().get(key);
    if (user != null) {
        return user.getId() == null ? null : user;
    }
    
    // 3. 查数据库
    user = userMapper.selectById(userId);
    
    // 4. 缓存结果
    if (user == null) {
        User emptyUser = new User();
        redisTemplate.opsForValue().set(key, emptyUser, 5, TimeUnit.MINUTES);
    } else {
        redisTemplate.opsForValue().set(key, user, 1, TimeUnit.HOURS);
    }
    
    return user;
}
```

> **架构思考：** 布隆过滤器适合过滤大部分不存在的请求，空值缓存适合处理少量不存在但被频繁查询的key。两者结合使用，可以达到最佳效果。

---

## 缓存击穿：热点key过期

### 问题场景

```
热点商品详情页，缓存过期瞬间，1000个并发请求同时打到数据库
```

**流程：**
```
时刻T：缓存过期
时刻T+1ms：1000个请求同时查缓存（未命中）→ 同时查数据库
```

### 解决方案1：互斥锁

```java
public Product getProduct(Long productId) {
    String key = "product:" + productId;
    
    // 1. 查缓存
    Product product = redisTemplate.opsForValue().get(key);
    if (product != null) {
        return product;
    }
    
    // 2. 获取互斥锁
    String lockKey = "lock:product:" + productId;
    Boolean locked = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS);
    
    if (locked) {
        try {
            // 3. 双重检查
            product = redisTemplate.opsForValue().get(key);
            if (product != null) {
                return product;
            }
            
            // 4. 查数据库
            product = productMapper.selectById(productId);
            
            // 5. 写入缓存
            if (product != null) {
                redisTemplate.opsForValue().set(key, product, 1, TimeUnit.HOURS);
            }
            
            return product;
        } finally {
            // 6. 释放锁
            redisTemplate.delete(lockKey);
        }
    } else {
        // 7. 未获取到锁，等待后重试
        try {
            Thread.sleep(50);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return getProduct(productId);  // 递归重试
    }
}
```

**优点：**
- 保证只有一个线程查询数据库
- 实现简单

**缺点：**
- 其他线程需要等待
- 可能导致请求堆积

### 解决方案2：逻辑过期

```java
@Data
public class CacheData<T> {
    private T data;
    private LocalDateTime expireTime;
}
```

```java
public Product getProduct(Long productId) {
    String key = "product:" + productId;
    
    // 1. 查缓存（永不过期）
    CacheData<Product> cacheData = redisTemplate.opsForValue().get(key);
    
    if (cacheData == null) {
        // 缓存未命中，查数据库
        return loadAndCache(productId);
    }
    
    // 2. 判断逻辑过期
    if (cacheData.getExpireTime().isBefore(LocalDateTime.now())) {
        // 3. 异步更新缓存
        String lockKey = "lock:product:" + productId;
        Boolean locked = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS);
        
        if (locked) {
            CompletableFuture.runAsync(() -> {
                try {
                    loadAndCache(productId);
                } finally {
                    redisTemplate.delete(lockKey);
                }
            });
        }
    }
    
    // 4. 返回旧数据
    return cacheData.getData();
}

private Product loadAndCache(Long productId) {
    Product product = productMapper.selectById(productId);
    if (product != null) {
        CacheData<Product> cacheData = new CacheData<>();
        cacheData.setData(product);
        cacheData.setExpireTime(LocalDateTime.now().plusHours(1));
        
        redisTemplate.opsForValue().set("product:" + productId, cacheData);
    }
    return product;
}
```

**优点：**
- 不会阻塞请求
- 始终返回数据（即使是旧数据）

**缺点：**
- 可能返回过期数据
- 实现复杂

### 解决方案3：热点数据永不过期

```java
public Product getProduct(Long productId) {
    String key = "product:" + productId;
    
    // 热点商品永不过期
    Product product = redisTemplate.opsForValue().get(key);
    if (product != null) {
        return product;
    }
    
    // 查数据库并缓存（不设置过期时间）
    product = productMapper.selectById(productId);
    if (product != null) {
        redisTemplate.opsForValue().set(key, product);
    }
    
    return product;
}

// 通过消息队列异步更新缓存
@RabbitListener(queues = "product.update")
public void updateProductCache(Long productId) {
    Product product = productMapper.selectById(productId);
    redisTemplate.opsForValue().set("product:" + productId, product);
}
```

---

## 缓存雪崩：大量key同时过期

### 问题场景

```
凌晨2点，系统重启，所有缓存同时加载，设置1小时过期
凌晨3点，所有缓存同时过期，大量请求打到数据库
```

### 解决方案1：过期时间随机化

```java
public void setCache(String key, Object value, long baseExpireSeconds) {
    // 在基础过期时间上增加随机值
    long randomSeconds = ThreadLocalRandom.current().nextLong(0, 300);
    long expireSeconds = baseExpireSeconds + randomSeconds;
    
    redisTemplate.opsForValue().set(key, value, expireSeconds, TimeUnit.SECONDS);
}
```

```java
// 使用示例
setCache("user:1", user, 3600);  // 3600 ~ 3900秒
setCache("user:2", user, 3600);  // 3600 ~ 3900秒
```

### 解决方案2：多级缓存

```
请求 → 本地缓存（Caffeine）→ Redis → 数据库
```

```java
@Configuration
public class CacheConfig {
    
    @Bean
    public Cache<String, Object> localCache() {
        return Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .build();
    }
}
```

```java
@Service
public class UserService {
    
    @Autowired
    private Cache<String, Object> localCache;
    
    @Autowired
    private RedisTemplate<String, User> redisTemplate;
    
    @Autowired
    private UserMapper userMapper;
    
    public User getUser(Long userId) {
        String key = "user:" + userId;
        
        // 1. 查本地缓存
        User user = (User) localCache.getIfPresent(key);
        if (user != null) {
            return user;
        }
        
        // 2. 查Redis
        user = redisTemplate.opsForValue().get(key);
        if (user != null) {
            localCache.put(key, user);
            return user;
        }
        
        // 3. 查数据库
        user = userMapper.selectById(userId);
        if (user != null) {
            // 写入Redis和本地缓存
            redisTemplate.opsForValue().set(key, user, 1, TimeUnit.HOURS);
            localCache.put(key, user);
        }
        
        return user;
    }
}
```

### 解决方案3：缓存预热

```java
@Component
public class CacheWarmUp implements ApplicationRunner {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private ProductMapper productMapper;
    
    @Override
    public void run(ApplicationArguments args) {
        log.info("开始缓存预热...");
        
        // 加载热点商品
        List<Product> hotProducts = productMapper.selectHotProducts(100);
        for (Product product : hotProducts) {
            String key = "product:" + product.getId();
            long expireSeconds = 3600 + ThreadLocalRandom.current().nextLong(0, 300);
            redisTemplate.opsForValue().set(key, product, expireSeconds, TimeUnit.SECONDS);
        }
        
        log.info("缓存预热完成，共加载 {} 个商品", hotProducts.size());
    }
}
```

### 解决方案4：限流降级

```java
@Service
public class ProductService {
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    @Autowired
    private ProductMapper productMapper;
    
    /**
     * 使用 Sentinel 限流
     */
    @SentinelResource(
        value = "getProduct",
        blockHandler = "getProductBlockHandler",
        fallback = "getProductFallback"
    )
    public Product getProduct(Long productId) {
        String key = "product:" + productId;
        
        // 查缓存
        Product product = redisTemplate.opsForValue().get(key);
        if (product != null) {
            return product;
        }
        
        // 查数据库
        product = productMapper.selectById(productId);
        if (product != null) {
            redisTemplate.opsForValue().set(key, product, 1, TimeUnit.HOURS);
        }
        
        return product;
    }
    
    /**
     * 限流降级处理
     */
    public Product getProductBlockHandler(Long productId, BlockException ex) {
        log.warn("触发限流: productId={}", productId);
        // 返回默认商品或提示信息
        return getDefaultProduct();
    }
    
    /**
     * 异常降级处理
     */
    public Product getProductFallback(Long productId, Throwable ex) {
        log.error("查询异常: productId={}", productId, ex);
        // 返回默认商品
        return getDefaultProduct();
    }
}
```

---

## 缓存更新策略

### 策略对比

| 策略 | 一致性 | 实现复杂度 | 适用场景 |
|------|--------|------------|----------|
| Cache Aside | 弱一致性 | 简单 | 读多写少 |
| Read/Write Through | 强一致性 | 复杂 | 读写均衡 |
| Write Behind | 最终一致性 | 复杂 | 写多读少 |

### Cache Aside（旁路缓存）

```java
// 读操作
public User getUser(Long userId) {
    String key = "user:" + userId;
    
    // 1. 查缓存
    User user = redisTemplate.opsForValue().get(key);
    if (user != null) {
        return user;
    }
    
    // 2. 查数据库
    user = userMapper.selectById(userId);
    
    // 3. 写入缓存
    if (user != null) {
        redisTemplate.opsForValue().set(key, user, 1, TimeUnit.HOURS);
    }
    
    return user;
}

// 写操作
public void updateUser(User user) {
    // 1. 更新数据库
    userMapper.updateById(user);
    
    // 2. 删除缓存
    String key = "user:" + user.getId();
    redisTemplate.delete(key);
}
```

**为什么是删除缓存而不是更新缓存？**

1. 更新缓存可能浪费（更新后可能没人查询）
2. 并发更新可能导致数据不一致

### 延迟双删

```java
public void updateUser(User user) {
    String key = "user:" + user.getId();
    
    // 1. 删除缓存
    redisTemplate.delete(key);
    
    // 2. 更新数据库
    userMapper.updateById(user);
    
    // 3. 延迟删除缓存
    CompletableFuture.runAsync(() -> {
        try {
            Thread.sleep(500);  // 延迟500ms
            redisTemplate.delete(key);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    });
}
```

**为什么需要延迟双删？**

防止以下场景的数据不一致：
```
时刻1：线程A删除缓存
时刻2：线程B查询缓存（未命中）
时刻3：线程B查询数据库（旧数据）
时刻4：线程A更新数据库
时刻5：线程B写入缓存（旧数据）
```

延迟双删可以删除线程B写入的旧数据。

---

## 缓存监控

### 关键指标

```java
@Component
public class CacheMonitor {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Scheduled(fixedDelay = 60000)
    public void monitor() {
        RedisConnection connection = redisTemplate.getConnectionFactory().getConnection();
        Properties info = connection.info();
        
        // 1. 内存使用
        long usedMemory = Long.parseLong(info.getProperty("used_memory"));
        long maxMemory = Long.parseLong(info.getProperty("maxmemory"));
        double memoryUsageRate = (double) usedMemory / maxMemory * 100;
        
        // 2. 命中率
        long hits = Long.parseLong(info.getProperty("keyspace_hits"));
        long misses = Long.parseLong(info.getProperty("keyspace_misses"));
        double hitRate = (double) hits / (hits + misses) * 100;
        
        // 3. 连接数
        int connectedClients = Integer.parseInt(info.getProperty("connected_clients"));
        
        log.info("Redis监控 - 内存使用率: {}%, 命中率: {}%, 连接数: {}", 
            memoryUsageRate, hitRate, connectedClients);
        
        // 4. 告警
        if (memoryUsageRate > 80) {
            log.warn("Redis内存使用率过高: {}%", memoryUsageRate);
        }
        if (hitRate < 80) {
            log.warn("Redis命中率过低: {}%", hitRate);
        }
    }
}
```

---

## 最佳实践

**1. 合理设置过期时间**

- 热点数据：1-2小时
- 普通数据：30分钟-1小时
- 冷数据：5-10分钟

**2. 使用合适的数据结构**

- String：简单key-value
- Hash：对象存储
- List：列表、队列
- Set：去重、交集
- ZSet：排行榜

**3. 避免大key**

- 单个key不超过10KB
- 集合元素不超过5000个
- 使用SCAN代替KEYS

**4. 设置内存淘汰策略**

```conf
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

**5. 监控告警**

- 内存使用率 > 80%
- 命中率 < 80%
- 慢查询 > 10ms

> **避坑提示：** 缓存不是万能的，不要为了缓存而缓存。只有高频访问、变化不频繁的数据才适合缓存。对于实时性要求高的数据，直接查数据库可能更合适。

---

## 总结与思考

Redis 缓存架构的核心要点：

- 使用布隆过滤器防止缓存穿透
- 使用互斥锁或逻辑过期防止缓存击穿
- 使用过期时间随机化和多级缓存防止缓存雪崩
- 使用 Cache Aside 模式更新缓存
- 监控缓存命中率和内存使用率

缓存是提升性能的利器，但也会带来复杂度。在设计缓存架构时要权衡：

- 是否真的需要缓存？
- 缓存什么数据？
- 如何保证数据一致性？
- 如何应对缓存失效？

下次当你的系统面临高并发挑战时，不妨试试本文介绍的缓存架构方案，让系统性能更上一层楼。
