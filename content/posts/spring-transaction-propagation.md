---
title: "Spring 事务传播机制与失效场景完全解析"
date: 2024-05-15
tags: ["Spring", "MySQL", "分布式事务", "最佳实践"]
categories: ["技术"]
description: "深入理解 @Transactional 的 7 种传播行为，掌握事务失效的常见场景与解决方案"
draft: false
---

Spring 事务管理是后端开发的基础技能，但很多人只会在方法上加 @Transactional，却不知道事务传播机制和失效场景。本文系统讲解 Spring 事务的核心原理。

### 本文亮点
- [x] 理解 7 种事务传播行为的区别与适用场景
- [x] 掌握事务失效的 5 种常见场景
- [x] 学会编程式事务的使用方法
- [x] 了解分布式事务的解决方案

---

## 事务的 ACID 特性

- Atomicity（原子性）：事务中的操作要么全部成功，要么全部失败
- Consistency（一致性）：事务执行前后，数据保持一致状态
- Isolation（隔离性）：并发事务之间互不干扰
- Durability（持久性）：事务提交后，数据永久保存

---

## 7 种事务传播行为

**REQUIRED（默认）**

如果当前存在事务，加入该事务；如果不存在，创建新事务。

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
    // 操作 1
    methodB();  // 加入 methodA 的事务
}

@Transactional(propagation = Propagation.REQUIRED)
public void methodB() {
    // 操作 2
}
```

如果 methodB 抛出异常，methodA 和 methodB 的操作都会回滚。

**REQUIRES_NEW**

无论当前是否存在事务，都创建新事务。如果当前存在事务，挂起当前事务。

```java
@Transactional
public void methodA() {
    // 操作 1
    methodB();  // 创建新事务，挂起 methodA 的事务
    // 操作 3
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void methodB() {
    // 操作 2
}
```

如果 methodB 抛出异常，只回滚操作 2，操作 1 和操作 3 不受影响。

**NESTED**

如果当前存在事务，创建嵌套事务；如果不存在，等同于 REQUIRED。

```java
@Transactional
public void methodA() {
    // 操作 1
    try {
        methodB();  // 创建嵌套事务（保存点）
    } catch (Exception e) {
        // methodB 回滚，但 methodA 可以继续
    }
    // 操作 3
}

@Transactional(propagation = Propagation.NESTED)
public void methodB() {
    // 操作 2
}
```

嵌套事务基于数据库的 Savepoint 机制实现。

**SUPPORTS**

如果当前存在事务，加入该事务；如果不存在，以非事务方式执行。

适用于查询方法。

**NOT_SUPPORTED**

以非事务方式执行，如果当前存在事务，挂起当前事务。

**MANDATORY**

必须在事务中执行，如果当前不存在事务，抛出异常。

**NEVER**

必须在非事务中执行，如果当前存在事务，抛出异常。

> **架构思考：** 事务传播行为的设计体现了"灵活性与安全性的平衡"。REQUIRED 保证了数据一致性，REQUIRES_NEW 提供了事务隔离，NESTED 支持部分回滚。不同的业务场景需要不同的传播行为。

---

## 事务失效的 5 种场景

**场景一：方法不是 public**

@Transactional 只对 public 方法生效。

```java
@Service
public class UserService {
    
    @Transactional
    private void addUser(User user) {  // 事务失效
        userMapper.insert(user);
    }
}
```

原因：Spring AOP 基于代理，只能代理 public 方法。

**场景二：方法内部调用**

同一个类中，方法 A 调用方法 B，方法 B 的事务失效。

```java
@Service
public class UserService {
    
    public void methodA() {
        methodB();  // 直接调用，不走代理
    }
    
    @Transactional
    public void methodB() {  // 事务失效
        userMapper.insert(user);
    }
}
```

原因：内部调用不走代理对象，而是直接调用 this.methodB()。

解决方案：

```java
@Service
public class UserService {
    
    @Autowired
    private UserService self;  // 注入自己
    
    public void methodA() {
        self.methodB();  // 通过代理调用
    }
    
    @Transactional
    public void methodB() {
        userMapper.insert(user);
    }
}
```

**场景三：异常被捕获**

```java
@Transactional
public void addUser(User user) {
    try {
        userMapper.insert(user);
        int i = 1 / 0;  // 抛出异常
    } catch (Exception e) {
        log.error("异常", e);  // 异常被捕获，事务不回滚
    }
}
```

解决方案：手动回滚

```java
@Transactional
public void addUser(User user) {
    try {
        userMapper.insert(user);
        int i = 1 / 0;
    } catch (Exception e) {
        log.error("异常", e);
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

**场景四：异常类型不匹配**

@Transactional 默认只回滚 RuntimeException 和 Error。

```java
@Transactional
public void addUser(User user) throws Exception {
    userMapper.insert(user);
    throw new Exception("checked exception");  // 事务不回滚
}
```

解决方案：指定回滚的异常类型

```java
@Transactional(rollbackFor = Exception.class)
public void addUser(User user) throws Exception {
    userMapper.insert(user);
    throw new Exception("checked exception");
}
```

**场景五：数据库不支持事务**

MyISAM 引擎不支持事务，只有 InnoDB 支持。

---

## 编程式事务

声明式事务（@Transactional）简单易用，但不够灵活。编程式事务提供了更细粒度的控制。

```java
@Service
public class UserService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void addUser(User user) {
        transactionTemplate.execute(status -> {
            try {
                userMapper.insert(user);
                // 其他操作
                return true;
            } catch (Exception e) {
                status.setRollbackOnly();
                return false;
            }
        });
    }
}
```

---

## 分布式事务

单体应用的事务由数据库保证，但在微服务架构中，一个业务可能涉及多个服务和数据库，需要分布式事务。

**解决方案一：两阶段提交（2PC）**

协调者先询问所有参与者是否可以提交，如果都同意，再通知提交。

缺点：性能差，存在单点故障。

**解决方案二：TCC**

Try-Confirm-Cancel，业务层面的两阶段提交。

- Try：预留资源
- Confirm：确认提交
- Cancel：取消回滚

**解决方案三：Seata AT 模式**

自动记录 SQL 的前后镜像，失败时自动回滚。

**解决方案四：最终一致性**

通过消息队列 + 补偿机制实现最终一致性。

---

## 总结与思考

Spring 事务的核心是 AOP 代理，理解代理机制才能避免事务失效。

- 7 种传播行为适用于不同场景
- 事务失效的 5 种场景要牢记
- 分布式事务要根据业务特点选择方案

下次当你的事务不生效时，检查一下是不是踩了这些坑。
