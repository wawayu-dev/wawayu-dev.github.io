---
title: "Redis 分布式锁的生产级实践：从原理到 Redisson"
date: 2024-08-15
tags: ["Redis", "分布式锁", "高并发", "最佳实践"]
categories: ["技术"]
description: "深入理解 Redis 分布式锁的实现原理，掌握 Redisson 看门狗机制与锁优化方案"
draft: false
---

在分布式系统中，多个服务实例可能同时访问共享资源，需要分布式锁来保证数据一致性。Redis 是实现分布式锁的常用方案，但很多人只会用 SETNX，不知道其中的坑。本文深入剖析 Redis 分布式锁的实现。

### 本文亮点
- [x] 理解 Redis 分布式锁的实现原理与常见问题
- [x] 掌握 Redisson 看门狗机制的设计思想
- [x] 学会高并发场景下的锁优化方案
- [x] 了解分布式锁的最佳实践

---

## 为什么需要分布式锁？

**场景一：库存扣减**

```java
// 错误示例：没有加锁
public void deductStock(Long productId, int quantity) {
    Stock stock = stockMapper.selectById(productId);
    if (stock.getQuantity() >= quantity) {
        stock.setQuantity(stock.getQuantity() - quantity);
        stockMapper.updateById(stock);
    }
}
```

在高并发场景下，多个线程同时读取库存，可能导致超卖。

**场景二：定时任务去重**

多个服务实例部署时，定时任务会重复执行，需要分布式锁保证只有一个实例执行。

---

## 基础实现：SETNX + EXPIRE

**版本一：最简单的实现**

```java
public boolean tryLock(String key) {
    return redisTemplate.opsForValue().setIfAbsent(key, "1");
}

public void unlock(String key) {
    redisTemplate.delete(key);
}
```

问题：如果获取锁后，服务宕机，锁永远不会释放，导致死锁。

**版本二：加上过期时间**

```java
public boolean tryLock(String key, long expireTime) {
    Boolean result = redisTemplate.opsForValue().setIfAbsent(key, "1");
    if (result) {
        redisTemplate.expire(key, expireTime, TimeUnit.SECONDS);
    }
    return result;
}
```

问题：SETNX 和 EXPIRE 不是原子操作，如果在两者之间宕机，仍然会死锁。

**版本三：原子操作**

```java
public boolean tryLock(String key, long expireTime) {
    return redisTemplate.opsForValue()
        .setIfAbsent(key, "1", expireTime, TimeUnit.SECONDS);
}
```

Redis 2.6.12 之后，SET 命令支持 NX 和 EX 参数，保证原子性。

```redis
SET lock_key unique_value NX EX 30
```

- NX：只在键不存在时设置
- EX：设置过期时间（秒）

---

## 进阶问题：锁误删

**问题场景**

```
1. 线程 A 获取锁，设置过期时间 30 秒
2. 线程 A 执行业务逻辑，耗时 35 秒
3. 30 秒后，锁自动过期，线程 B 获取锁
4. 线程 A 执行完毕，删除锁（误删了线程 B 的锁）
```

**解决方案：锁的唯一标识**

```java
public boolean tryLock(String key, String requestId, long expireTime) {
    return redisTemplate.opsForValue()
        .setIfAbsent(key, requestId, expireTime, TimeUnit.SECONDS);
}

public void unlock(String key, String requestId) {
    String value = redisTemplate.opsForValue().get(key);
    if (requestId.equals(value)) {
        redisTemplate.delete(key);
    }
}
```

每个线程使用唯一的 requestId（如 UUID），释放锁时先判断是否是自己的锁。

但这里仍有问题：GET 和 DELETE 不是原子操作。

**Lua 脚本保证原子性**

```java
public void unlock(String key, String requestId) {
    String script = 
        "if redis.call('get', KEYS[1]) == ARGV[1] then " +
        "    return redis.call('del', KEYS[1]) " +
        "else " +
        "    return 0 " +
        "end";
    
    redisTemplate.execute(
        new DefaultRedisScript<>(script, Long.class),
        Collections.singletonList(key),
        requestId
    );
}
```

Lua 脚本在 Redis 中是原子执行的，保证了判断和删除的原子性。

> **架构思考：** 分布式锁的核心问题是"原子性"。SETNX + EXPIRE 不是原子的，需要用 SET NX EX；GET + DELETE 不是原子的，需要用 Lua 脚本。这种思想在很多场景都适用：要么全部成功，要么全部失败。

---

## Redisson：生产级的分布式锁

Redisson 是 Redis 官方推荐的 Java 客户端，提供了开箱即用的分布式锁。

**基本使用**

```java
@Autowired
private RedissonClient redissonClient;

public void doSomething() {
    RLock lock = redissonClient.getLock("myLock");
    try {
        // 尝试加锁，最多等待 10 秒，锁自动过期时间 30 秒
        boolean isLocked = lock.tryLock(10, 30, TimeUnit.SECONDS);
        if (isLocked) {
            // 执行业务逻辑
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        lock.unlock();
    }
}
```

**看门狗机制（Watchdog）**

如果不指定锁的过期时间，Redisson 会启动看门狗机制：

```java
lock.lock();  // 默认过期时间 30 秒
```

看门狗会每隔 10 秒（过期时间的 1/3）检查锁是否还被持有，如果是，自动续期 30 秒。

**看门狗的实现原理**

```java
private void scheduleExpirationRenewal(long threadId) {
    ExpirationEntry entry = new ExpirationEntry();
    ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
    
    if (oldEntry != null) {
        oldEntry.addThreadId(threadId);
    } else {
        entry.addThreadId(threadId);
        renewExpiration();
    }
}

private void renewExpiration() {
    ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
    if (ee == null) {
        return;
    }
    
    Timeout task = commandExecutor.getConnectionManager().newTimeout(timeout -> {
        // 续期锁
        RFuture<Boolean> future = renewExpirationAsync(threadId);
        future.onComplete((res, e) -> {
            if (e != null) {
                return;
            }
            if (res) {
                // 续期成功，继续调度下一次续期
                renewExpiration();
            }
        });
    }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
    
    ee.setTimeout(task);
}
```

看门狗的好处：

- 避免业务执行时间过长导致锁过期
- 避免手动设置过期时间不准确

> **避坑提示：** 看门狗只在未指定过期时间时生效。如果你调用 `lock.tryLock(10, 30, TimeUnit.SECONDS)`，看门狗不会启动，锁会在 30 秒后自动过期。

---

## 可重入锁

Redisson 的锁是可重入的，同一个线程可以多次获取同一把锁。

```java
public void methodA() {
    RLock lock = redissonClient.getLock("myLock");
    lock.lock();
    try {
        methodB();  // 可以再次获取锁
    } finally {
        lock.unlock();
    }
}

public void methodB() {
    RLock lock = redissonClient.getLock("myLock");
    lock.lock();
    try {
        // 业务逻辑
    } finally {
        lock.unlock();
    }
}
```

**实现原理：Hash 结构**

```redis
HSET myLock thread-1 1  # 第一次加锁
HINCRBY myLock thread-1 1  # 第二次加锁，计数器 +1
```

释放锁时，计数器 -1，只有计数器为 0 时才真正释放锁。

---

## 高并发优化：分段锁

在秒杀场景下，所有请求竞争同一把锁，性能很差。可以使用分段锁优化。

**场景：库存扣减**

```java
public void deductStock(Long productId, int quantity) {
    // 根据商品 ID 分段
    int segment = (int) (productId % 10);
    String lockKey = "stock_lock:" + productId + ":" + segment;
    
    RLock lock = redissonClient.getLock(lockKey);
    lock.lock();
    try {
        // 扣减库存
        Stock stock = stockMapper.selectById(productId);
        if (stock.getQuantity() >= quantity) {
            stock.setQuantity(stock.getQuantity() - quantity);
            stockMapper.updateById(stock);
        }
    } finally {
        lock.unlock();
    }
}
```

将库存分成 10 段，每段独立加锁，提升并发度。

**读写锁**

如果读多写少，可以使用读写锁：

```java
RReadWriteLock rwLock = redissonClient.getReadWriteLock("myLock");

// 读锁（共享锁）
RLock readLock = rwLock.readLock();
readLock.lock();
try {
    // 读操作
} finally {
    readLock.unlock();
}

// 写锁（排他锁）
RLock writeLock = rwLock.writeLock();
writeLock.lock();
try {
    // 写操作
} finally {
    writeLock.unlock();
}
```

多个线程可以同时持有读锁，但写锁是排他的。

---

## 红锁（RedLock）：多节点的分布式锁

单节点 Redis 存在单点故障问题，RedLock 通过多个独立的 Redis 节点实现高可用。

**RedLock 算法**

1. 获取当前时间戳
2. 依次向 N 个 Redis 节点请求加锁
3. 如果超过半数节点加锁成功，且总耗时小于锁的过期时间，则认为加锁成功
4. 如果加锁失败，向所有节点释放锁

**Redisson 实现**

```java
RLock lock1 = redissonClient1.getLock("myLock");
RLock lock2 = redissonClient2.getLock("myLock");
RLock lock3 = redissonClient3.getLock("myLock");

RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
redLock.lock();
try {
    // 业务逻辑
} finally {
    redLock.unlock();
}
```

RedLock 的争议：

- Martin Kleppmann 认为 RedLock 不安全（时钟漂移问题）
- Redis 作者 Antirez 认为 RedLock 在大多数场景下够用

实际生产中，单节点 + 主从复制 + 哨兵模式已经足够。

---

## 最佳实践

**1. 设置合理的过期时间**

过期时间太短，业务还没执行完锁就过期了；过期时间太长，宕机后锁长时间不释放。

建议：根据业务执行时间设置，或使用 Redisson 的看门狗机制。

**2. 使用 try-finally 释放锁**

```java
lock.lock();
try {
    // 业务逻辑
} finally {
    lock.unlock();
}
```

**3. 避免长时间持有锁**

锁的粒度要尽量小，只锁必要的代码块。

```java
// 不好：锁的粒度太大
lock.lock();
try {
    queryDatabase();  // 耗时操作
    updateCache();
    sendMessage();
} finally {
    lock.unlock();
}

// 好：只锁必要的代码
queryDatabase();
lock.lock();
try {
    updateCache();
} finally {
    lock.unlock();
}
sendMessage();
```

**4. 锁降级**

如果 Redis 不可用，可以降级为本地锁或直接放行。

```java
RLock lock = redissonClient.getLock("myLock");
try {
    boolean isLocked = lock.tryLock(1, 30, TimeUnit.SECONDS);
    if (!isLocked) {
        // 降级处理
        log.warn("获取分布式锁失败，降级为本地锁");
        synchronized (this) {
            // 业务逻辑
        }
    }
} catch (Exception e) {
    log.error("分布式锁异常", e);
    // 降级处理
}
```

---

## 总结与思考

Redis 分布式锁的核心要点：

- 使用 SET NX EX 保证加锁的原子性
- 使用 Lua 脚本保证解锁的原子性
- 使用唯一标识避免锁误删
- 使用 Redisson 简化实现，利用看门狗机制自动续期

分布式锁不是银弹，它会降低系统性能。在设计时要权衡：

- 是否真的需要分布式锁？能否通过数据库的乐观锁解决？
- 锁的粒度是否合理？能否通过分段锁提升并发度？
- 是否需要高可用？单节点是否够用？

下次当你需要分布式锁时，不妨先问问自己：这个场景真的需要强一致性吗？

