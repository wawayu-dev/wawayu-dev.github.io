---
title: "拼团交易平台系统：DDD 架构下的分布式实战"
date: 2024-03-15
tags: ["DDD", "分布式", "RabbitMQ", "Redis", "ShardingSphere"]
categories: ["项目"]
description: "从零到一设计高并发拼团系统，解决库存超卖、分布式事务、海量订单存储等核心问题"
draft: false
---

在电商业务高速发展的背景下，拼团模式因其社交裂变属性成为流量增长的利器。但看似简单的"凑够 N 人成团"背后，隐藏着库存超卖、分布式事务、海量数据存储等一系列技术挑战。本文复盘我在拼团交易平台项目中的架构设计与技术选型思考。

### 项目亮点
- [x] 基于 DDD 思想的领域模型设计，实现业务逻辑与技术实现的解耦
- [x] Redis 分布式锁 + 分库分表解决高并发库存扣减问题
- [x] RabbitMQ 异步解耦实现单步流水线，提升系统吞吐量
- [x] ShardingSphere 分库分表支撑海量订单数据存储与查询

---

## 业务场景与技术挑战

拼团交易的核心流程看似简单：用户发起拼团 → 其他用户参团 → 达到人数后成团 → 扣减库存 → 通知支付。但在高并发场景下，这个流程会遇到三个致命问题：

**库存超卖：** 100 件商品，1000 人同时下单，如何保证不会超卖？传统的数据库行锁在分布式环境下失效。

**分布式事务：** 订单创建、库存扣减、支付通知分布在不同服务，如何保证数据一致性？

**海量数据：** 订单表单日新增百万级数据，如何保证查询性能不随数据量增长而劣化？

这些问题倒逼我们必须从架构层面重新思考系统设计。

---

## 架构设计：DDD 思想的落地实践

传统的三层架构（Controller-Service-DAO）在复杂业务场景下会导致 Service 层逻辑臃肿，业务规则散落在各处。我们采用 DDD（领域驱动设计）思想，将系统划分为三个核心领域：

**订单域（Order）：** 负责拼团订单的生命周期管理，包括订单创建、状态流转、超时取消等。

**库存域（Inventory）：** 负责商品库存的扣减、回滚、预占等操作，是整个系统的核心瓶颈。

**支付域（Payment）：** 负责支付流程的编排，包括支付通知、退款处理等。

每个领域都有自己的聚合根（Aggregate Root）和领域服务（Domain Service）。比如订单域的聚合根是 `Order`，它封装了订单的所有业务规则：

```java
public class Order {
    private OrderId id;
    private UserId userId;
    private OrderStatus status;
    private List<OrderItem> items;
    
    // 业务规则：只有待支付状态才能取消
    public void cancel() {
        if (this.status != OrderStatus.PENDING_PAYMENT) {
            throw new OrderCannotCancelException();
        }
        this.status = OrderStatus.CANCELLED;
        // 发布领域事件：订单已取消
        DomainEventPublisher.publish(new OrderCancelledEvent(this.id));
    }
}
```

这种设计的好处是：业务规则内聚在领域对象中，而不是散落在 Service 层的各种 if-else 里。当产品经理说"增加一个拼团失败自动退款的逻辑"时，我们只需要监听 `OrderCancelledEvent` 事件，而不需要改动订单取消的核心逻辑。

> **架构思考：** DDD 的本质是"用代码表达业务语言"。当你的代码中出现 `Order.cancel()` 而不是 `orderService.updateStatus(orderId, 3)`，业务人员也能看懂代码在做什么。这种"通用语言"（Ubiquitous Language）是 DDD 最大的价值。

---

## 核心难点：分布式锁解决库存超卖

库存扣减是整个系统的性能瓶颈。最简单的方案是在数据库层面加行锁：

```sql
UPDATE inventory SET stock = stock - 1 WHERE product_id = ? AND stock > 0
```

但这种方案在高并发下会导致大量线程阻塞在数据库锁上，TPS 上不去。我们的优化思路是：用 Redis 分布式锁 + 预扣减机制。

**第一步：Redis 预扣减**

用户下单时，先在 Redis 中扣减库存（原子操作）：

```java
public boolean preDeductStock(String productId, int quantity) {
    String key = "stock:" + productId;
    Long remaining = redisTemplate.opsForValue().decrement(key, quantity);
    return remaining != null && remaining >= 0;
}
```

如果 Redis 扣减成功，说明有库存，允许创建订单。如果失败，直接返回"库存不足"，不需要访问数据库。

**第二步：异步同步到数据库**

通过 RabbitMQ 发送消息，异步将库存变更同步到 MySQL：

```java
rabbitTemplate.convertAndSend("inventory.exchange", "stock.deduct", 
    new StockDeductMessage(productId, quantity, orderId));
```

消费者监听消息，更新数据库库存：

```java
@RabbitListener(queues = "stock.deduct.queue")
public void handleStockDeduct(StockDeductMessage msg) {
    inventoryMapper.deductStock(msg.getProductId(), msg.getQuantity());
}
```

**第三步：Redisson 看门狗机制**

为了防止 Redis 和数据库数据不一致，我们用 Redisson 的分布式锁保护整个扣减流程：

```java
RLock lock = redisson.getLock("lock:stock:" + productId);
try {
    lock.lock(10, TimeUnit.SECONDS);
    // 执行扣减逻辑
} finally {
    lock.unlock();
}
```

Redisson 的看门狗机制会自动续期锁，避免业务执行时间过长导致锁提前释放。

> **避坑提示：** Redis 预扣减方案有个隐患：如果订单超时未支付，需要回滚库存。这时必须保证 Redis 和数据库的库存数据最终一致。我们的做法是：定时任务扫描超时订单，同时回滚 Redis 和数据库库存。

---

## 异步解耦：RabbitMQ 单步流水线设计

传统的同步调用链路是：创建订单 → 扣减库存 → 发送支付通知。如果每一步都同步等待，整个接口的 RT（响应时间）会累加，用户体验很差。

我们用 RabbitMQ 实现异步解耦：

1. 用户下单后，立即返回"订单创建成功"
2. 发送消息到 `order.created` 队列
3. 库存服务监听消息，扣减库存后发送 `stock.deducted` 消息
4. 支付服务监听消息，发送支付通知

这种单步流水线设计的好处是：每个服务只关心自己的职责，通过消息驱动实现松耦合。如果库存服务挂了，消息会堆积在队列中，恢复后自动消费，不会丢失数据。

```java
// 订单服务：发布订单创建事件
@Transactional
public void createOrder(OrderDTO dto) {
    Order order = orderMapper.insert(dto);
    rabbitTemplate.convertAndSend("order.exchange", "order.created", 
        new OrderCreatedEvent(order.getId()));
}

// 库存服务：监听订单创建事件
@RabbitListener(queues = "order.created.queue")
public void handleOrderCreated(OrderCreatedEvent event) {
    inventoryService.deductStock(event.getOrderId());
    rabbitTemplate.convertAndSend("inventory.exchange", "stock.deducted", 
        new StockDeductedEvent(event.getOrderId()));
}
```

> **架构思考：** 异步化的代价是调试复杂度增加。当订单创建成功但库存扣减失败时，如何排查问题？我们的做法是：每条消息都带上 `traceId`，通过日志系统串联整个调用链路。

---

## 海量数据：ShardingSphere 分库分表实战

订单表单日新增百万级数据，如果不做分库分表，查询性能会随着数据量增长而急剧下降。我们用 ShardingSphere 实现水平分表：

**分片策略：** 按 `user_id` 取模分 16 张表（`t_order_0` ~ `t_order_15`）

```yaml
shardingRule:
  tables:
    t_order:
      actualDataNodes: ds0.t_order_${0..15}
      tableStrategy:
        standard:
          shardingColumn: user_id
          shardingAlgorithmName: order_mod
  shardingAlgorithms:
    order_mod:
      type: MOD
      props:
        sharding-count: 16
```

**为什么选择 user_id 而不是 order_id？**

因为大部分查询场景是"查询某个用户的订单列表"，按 `user_id` 分片可以保证单个用户的订单都在同一张表，避免跨分片查询。

**跨分片查询优化：**

如果需要按 `order_id` 查询（比如订单详情页），会触发全表扫描。我们的优化方案是：在 Redis 中维护 `order_id -> user_id` 的映射关系，先查 Redis 获取 `user_id`，再精准路由到对应分片。

```java
public Order getOrderById(String orderId) {
    // 先从 Redis 获取 user_id
    String userId = redisTemplate.opsForValue().get("order:user:" + orderId);
    if (userId != null) {
        // 精准路由到对应分片
        return orderMapper.selectByOrderIdAndUserId(orderId, userId);
    }
    // 降级方案：全表扫描
    return orderMapper.selectByOrderId(orderId);
}
```

> **避坑提示：** 分库分表后，分布式事务会变得更复杂。我们的做法是：尽量避免跨分片事务，通过业务设计保证单个事务只操作一个分片。如果实在需要跨分片，使用 Seata 的 AT 模式保证最终一致性。

---

## 项目成果与思考

经过三个月的迭代，拼团交易平台成功上线，核心指标如下：

- 订单创建接口 RT 从 800ms 降低到 120ms
- 库存扣减 TPS 从 500 提升到 3000+
- 支持单日百万级订单数据写入，查询性能稳定在 50ms 以内

这个项目最大的收获不是技术栈的堆砌，而是对分布式系统设计的深度理解：

**没有银弹：** Redis 分布式锁、RabbitMQ 异步、ShardingSphere 分库分表，每个技术方案都有适用场景和代价。选择技术方案的本质是在性能、一致性、复杂度之间做权衡。

**架构演进：** 系统不是一开始就设计成分布式的。我们最初是单体应用 + MySQL 主从，当 TPS 到达瓶颈后才引入 Redis 和 RabbitMQ。过早优化是万恶之源。

**可观测性：** 分布式系统最大的挑战是排查问题。我们在每个关键节点都埋点上报监控数据（订单创建数、库存扣减数、消息堆积数），通过 Grafana 实时监控系统健康度。

如果让我重新设计这个系统，我会在项目初期就引入全链路压测，提前发现性能瓶颈，而不是等到上线后才发现问题。
