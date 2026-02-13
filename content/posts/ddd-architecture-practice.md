---
title: "领域驱动设计（DDD）落地实践：从理论到代码"
date: 2024-09-25
tags: ["DDD", "架构设计", "微服务", "最佳实践"]
categories: ["技术"]
description: "深入理解 DDD 的核心概念，掌握聚合根、领域事件、限界上下文的设计方法"
draft: false
---

DDD（Domain-Driven Design）是一套复杂业务系统的设计方法论。但很多人学了 DDD 的概念，却不知道如何落地。本文通过一个电商订单系统的案例，讲解 DDD 的实战应用。

### 本文亮点
- [x] 理解 DDD 的核心概念：实体、值对象、聚合根、领域服务
- [x] 掌握限界上下文的划分方法
- [x] 学会用领域事件实现服务解耦
- [x] 了解 DDD 分层架构的代码组织

---

## 为什么需要 DDD？

传统的三层架构（Controller-Service-DAO）在简单业务场景下够用，但当业务复杂度增加时，会出现几个问题：

**Service 层逻辑臃肿**

所有业务逻辑都堆在 Service 层，一个 OrderService 可能有几千行代码。

**业务规则散落各处**

订单的状态流转规则可能分散在多个 Service 方法中，难以维护。

**领域知识流失**

代码中充斥着 `updateStatus(orderId, 3)`，没人知道 3 代表什么状态。

DDD 通过领域建模，让代码更贴近业务，提升可维护性。

---

## DDD 核心概念

**实体（Entity）**

有唯一标识的对象，生命周期内标识不变。比如订单、用户。

```java
public class Order {
    private OrderId id;  // 唯一标识
    private UserId userId;
    private OrderStatus status;
    private List<OrderItem> items;
    
    // 业务方法
    public void pay() {
        if (this.status != OrderStatus.PENDING_PAYMENT) {
            throw new IllegalStateException("订单状态不允许支付");
        }
        this.status = OrderStatus.PAID;
    }
}
```

**值对象（Value Object）**

没有唯一标识，通过属性值判断相等性。比如地址、金额。

```java
public class Money {
    private final BigDecimal amount;
    private final Currency currency;
    
    public Money(BigDecimal amount, Currency currency) {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("金额不能为负数");
        }
        this.amount = amount;
        this.currency = currency;
    }
    
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("货币类型不一致");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

值对象是不可变的，所有操作都返回新对象。

**聚合根（Aggregate Root）**

聚合是一组相关对象的集合，聚合根是聚合的入口。外部只能通过聚合根访问聚合内的对象。

```java
public class Order {  // 聚合根
    private OrderId id;
    private List<OrderItem> items;  // 聚合内的实体
    
    // 外部不能直接修改 items，必须通过聚合根的方法
    public void addItem(Product product, int quantity) {
        OrderItem item = new OrderItem(product, quantity);
        this.items.add(item);
    }
}
```

聚合根的作用是保护聚合的不变性（Invariant）。

**领域服务（Domain Service）**

不属于任何实体的业务逻辑，放在领域服务中。

```java
public class PricingService {
    
    public Money calculateTotalPrice(Order order) {
        Money total = Money.zero();
        for (OrderItem item : order.getItems()) {
            Money itemPrice = item.getProduct().getPrice().multiply(item.getQuantity());
            total = total.add(itemPrice);
        }
        return total;
    }
}
```

> **架构思考：** 实体和值对象的区别在于是否有唯一标识。订单有订单号，是实体；金额没有唯一标识，是值对象。判断标准是：如果两个对象的所有属性都相同，它们是否应该被视为同一个对象？

---

## 限界上下文：划分领域边界

一个复杂系统包含多个子域，每个子域有自己的领域模型。限界上下文（Bounded Context）定义了领域模型的边界。

**案例：电商系统的限界上下文**

- **订单上下文：** 订单、订单项、订单状态
- **库存上下文：** 商品、库存、库存扣减
- **支付上下文：** 支付单、支付方式、支付状态
- **物流上下文：** 物流单、物流状态、物流公司

每个上下文有自己的领域模型。比如"商品"在不同上下文中的含义不同：

- 订单上下文：商品是订单项的一部分，关注商品名称、价格
- 库存上下文：商品是库存管理的对象，关注库存数量、仓库位置

**上下文映射（Context Mapping）**

不同上下文之间需要协作，通过上下文映射定义协作方式：

- **共享内核（Shared Kernel）：** 两个上下文共享部分领域模型
- **客户-供应商（Customer-Supplier）：** 一个上下文依赖另一个上下文
- **防腐层（Anti-Corruption Layer）：** 通过适配器隔离外部系统

案例：订单上下文依赖库存上下文

```java
// 订单上下文中的防腐层
public class InventoryAdapter {
    
    @Autowired
    private InventoryClient inventoryClient;
    
    public boolean checkStock(ProductId productId, int quantity) {
        // 调用库存上下文的接口
        InventoryDTO inventory = inventoryClient.getInventory(productId.getValue());
        return inventory.getStock() >= quantity;
    }
}
```

防腐层的作用是：隔离外部系统的变化，保护本上下文的领域模型。

---

## 领域事件：实现服务解耦

领域事件是领域中发生的重要事情，比如"订单已创建"、"订单已支付"。

**发布领域事件**

```java
public class Order {
    
    public void pay() {
        if (this.status != OrderStatus.PENDING_PAYMENT) {
            throw new IllegalStateException("订单状态不允许支付");
        }
        this.status = OrderStatus.PAID;
        
        // 发布领域事件
        DomainEventPublisher.publish(new OrderPaidEvent(this.id, this.userId));
    }
}
```

**监听领域事件**

```java
@Component
public class OrderEventListener {
    
    @Autowired
    private InventoryService inventoryService;
    
    @EventListener
    public void handleOrderPaid(OrderPaidEvent event) {
        // 订单支付后，扣减库存
        inventoryService.deduct(event.getOrderId());
    }
}
```

领域事件的好处：

- 解耦：订单上下文不需要知道库存上下文的存在
- 扩展性：新增监听器不需要修改订单代码
- 审计：事件流记录了系统的所有变化

> **架构思考：** 领域事件是事件驱动架构（EDA）的基础。它让系统从"调用驱动"变成"事件驱动"，提升了系统的松耦合性。类似的思想还有：消息队列、发布订阅模式、CQRS。

---

## DDD 分层架构

DDD 推荐的分层架构：

```
┌─────────────────────────────────┐
│  用户接口层（User Interface）    │  Controller、DTO
├─────────────────────────────────┤
│  应用层（Application）           │  ApplicationService、事件发布
├─────────────────────────────────┤
│  领域层（Domain）                │  Entity、ValueObject、DomainService
├─────────────────────────────────┤
│  基础设施层（Infrastructure）    │  Repository、MQ、缓存
└─────────────────────────────────┘
```

**用户接口层**

处理 HTTP 请求，调用应用层服务。

```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    
    @Autowired
    private OrderApplicationService orderApplicationService;
    
    @PostMapping
    public OrderDTO createOrder(@RequestBody CreateOrderRequest request) {
        OrderId orderId = orderApplicationService.createOrder(request);
        return orderApplicationService.getOrder(orderId);
    }
}
```

**应用层**

编排领域对象，处理事务和事件发布。

```java
@Service
public class OrderApplicationService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private DomainEventPublisher eventPublisher;
    
    @Transactional
    public OrderId createOrder(CreateOrderRequest request) {
        // 创建订单聚合
        Order order = Order.create(request.getUserId(), request.getItems());
        
        // 保存订单
        orderRepository.save(order);
        
        // 发布领域事件
        eventPublisher.publish(new OrderCreatedEvent(order.getId()));
        
        return order.getId();
    }
}
```

**领域层**

核心业务逻辑，不依赖任何外部框架。

```java
public class Order {
    private OrderId id;
    private UserId userId;
    private OrderStatus status;
    private List<OrderItem> items;
    
    public static Order create(UserId userId, List<OrderItem> items) {
        if (items.isEmpty()) {
            throw new IllegalArgumentException("订单项不能为空");
        }
        Order order = new Order();
        order.id = OrderId.generate();
        order.userId = userId;
        order.status = OrderStatus.PENDING_PAYMENT;
        order.items = items;
        return order;
    }
    
    public void pay() {
        if (this.status != OrderStatus.PENDING_PAYMENT) {
            throw new IllegalStateException("订单状态不允许支付");
        }
        this.status = OrderStatus.PAID;
    }
}
```

**基础设施层**

实现领域层定义的接口，处理持久化、消息队列等。

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    
    @Autowired
    private OrderMapper orderMapper;
    
    @Override
    public void save(Order order) {
        OrderPO po = OrderConverter.toPO(order);
        orderMapper.insert(po);
    }
    
    @Override
    public Order findById(OrderId id) {
        OrderPO po = orderMapper.selectById(id.getValue());
        return OrderConverter.toDomain(po);
    }
}
```

> **避坑提示：** 领域层不应该依赖基础设施层。Repository 接口定义在领域层，实现在基础设施层。这是依赖倒置原则（DIP）的体现。

---

## DDD 的适用场景

DDD 不是银弹，它适合复杂业务系统，不适合简单的 CRUD 应用。

**适合 DDD 的场景：**

- 业务规则复杂，需要频繁变更
- 多个团队协作开发
- 系统需要长期演进

**不适合 DDD 的场景：**

- 简单的增删改查
- 业务逻辑很少
- 项目周期短，不需要长期维护

---

## 总结与思考

DDD 的核心是：用代码表达业务语言，让代码更贴近业务。

- 通过实体、值对象、聚合根建模领域对象
- 通过限界上下文划分领域边界
- 通过领域事件实现服务解耦
- 通过分层架构隔离业务逻辑和技术实现

DDD 不是技术，而是思维方式。它要求开发者深入理解业务，与业务专家共同建模。这是一个持续迭代的过程。

下次当你的 Service 层代码超过 500 行时，不妨试试 DDD，重新审视你的领域模型。
