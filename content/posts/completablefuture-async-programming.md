---
title: "CompletableFuture 异步编程实战指南"
date: 2024-03-25
tags: ["Java", "并发编程", "多线程", "最佳实践"]
categories: ["技术"]
description: "掌握 Java 8 异步编程的核心工具，从链式调用到异常处理的完整实践"
draft: false
---

在微服务架构中，一个接口往往需要调用多个下游服务。如果串行调用，响应时间会累加；如果用传统的 Future，代码会变得复杂难维护。CompletableFuture 提供了优雅的异步编程方案。

### 本文亮点
- [x] 掌握 CompletableFuture 的链式调用与组合操作
- [x] 学会异常处理与超时控制
- [x] 理解与线程池的配合使用
- [x] 了解异步编程的最佳实践

---

## 传统 Future 的痛点

JDK 1.5 引入的 Future 接口只能通过 get() 方法阻塞等待结果：

```java
ExecutorService executor = Executors.newFixedThreadPool(10);

Future<String> future = executor.submit(() -> {
    Thread.sleep(1000);
    return "Hello";
});

// 阻塞等待结果
String result = future.get();
```

问题：

1. get() 会阻塞当前线程，无法异步处理结果
2. 无法组合多个异步任务
3. 异常处理不方便

CompletableFuture 解决了这些问题。

---

## CompletableFuture 基础用法

**创建 CompletableFuture**

```java
// 方式一：手动完成
CompletableFuture<String> future = new CompletableFuture<>();
future.complete("Hello");

// 方式二：异步执行
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "Hello";
});

// 方式三：无返回值的异步执行
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    System.out.println("Hello");
});
```

**获取结果**

```java
// 阻塞等待（不推荐）
String result = future.get();

// 阻塞等待，带超时
String result = future.get(1, TimeUnit.SECONDS);

// 非阻塞获取，如果未完成返回默认值
String result = future.getNow("default");

// 阻塞等待，不抛出受检异常
String result = future.join();
```

---

## 链式调用：优雅的异步流水线

**thenApply：转换结果**

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "Hello";
}).thenApply(s -> {
    return s + " World";
}).thenApply(s -> {
    return s.toUpperCase();
});

System.out.println(future.join());  // HELLO WORLD
```

**thenAccept：消费结果**

```java
CompletableFuture.supplyAsync(() -> {
    return "Hello";
}).thenAccept(s -> {
    System.out.println(s);  // 不返回结果
});
```

**thenRun：执行后续操作**

```java
CompletableFuture.supplyAsync(() -> {
    return "Hello";
}).thenRun(() -> {
    System.out.println("Done");  // 不关心前面的结果
});
```

**thenCompose：扁平化嵌套的 CompletableFuture**

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "user123";
}).thenCompose(userId -> {
    // 根据 userId 查询用户信息（返回 CompletableFuture）
    return getUserInfo(userId);
});

CompletableFuture<User> getUserInfo(String userId) {
    return CompletableFuture.supplyAsync(() -> {
        return userMapper.selectById(userId);
    });
}
```

> **架构思考：** thenApply 和 thenCompose 的区别类似于 Stream 的 map 和 flatMap。thenApply 用于简单的转换，thenCompose 用于扁平化嵌套的异步操作。

---

## 组合多个异步任务

**thenCombine：合并两个任务的结果**

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    return "Hello";
});

CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    return "World";
});

CompletableFuture<String> result = future1.thenCombine(future2, (s1, s2) -> {
    return s1 + " " + s2;
});

System.out.println(result.join());  // Hello World
```

**allOf：等待所有任务完成**

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Task1");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "Task2");
CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> "Task3");

CompletableFuture<Void> allFutures = CompletableFuture.allOf(future1, future2, future3);

// 等待所有任务完成
allFutures.join();

// 获取所有结果
List<String> results = Stream.of(future1, future2, future3)
    .map(CompletableFuture::join)
    .collect(Collectors.toList());
```

**anyOf：等待任意一个任务完成**

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    Thread.sleep(1000);
    return "Task1";
});

CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    Thread.sleep(500);
    return "Task2";
});

CompletableFuture<Object> anyFuture = CompletableFuture.anyOf(future1, future2);

System.out.println(anyFuture.join());  // Task2（先完成的）
```

---

## 实战案例：并行调用多个服务

假设一个商品详情页需要调用三个服务：

- 商品服务：查询商品信息
- 库存服务：查询库存
- 评论服务：查询评论

**串行调用（慢）**

```java
public ProductDetailDTO getProductDetail(Long productId) {
    Product product = productService.getProduct(productId);  // 100ms
    Stock stock = stockService.getStock(productId);          // 100ms
    List<Comment> comments = commentService.getComments(productId);  // 100ms
    
    return ProductDetailDTO.builder()
        .product(product)
        .stock(stock)
        .comments(comments)
        .build();
}
// 总耗时：300ms
```

**并行调用（快）**

```java
public ProductDetailDTO getProductDetail(Long productId) {
    CompletableFuture<Product> productFuture = CompletableFuture.supplyAsync(() -> 
        productService.getProduct(productId)
    );
    
    CompletableFuture<Stock> stockFuture = CompletableFuture.supplyAsync(() -> 
        stockService.getStock(productId)
    );
    
    CompletableFuture<List<Comment>> commentsFuture = CompletableFuture.supplyAsync(() -> 
        commentService.getComments(productId)
    );
    
    // 等待所有任务完成
    CompletableFuture.allOf(productFuture, stockFuture, commentsFuture).join();
    
    return ProductDetailDTO.builder()
        .product(productFuture.join())
        .stock(stockFuture.join())
        .comments(commentsFuture.join())
        .build();
}
// 总耗时：100ms（并行执行）
```

性能提升：从 300ms 降低到 100ms。

---

## 异常处理

**exceptionally：捕获异常并返回默认值**

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (Math.random() > 0.5) {
        throw new RuntimeException("Error");
    }
    return "Success";
}).exceptionally(ex -> {
    log.error("异常: {}", ex.getMessage());
    return "Default";
});
```

**handle：同时处理正常结果和异常**

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (Math.random() > 0.5) {
        throw new RuntimeException("Error");
    }
    return "Success";
}).handle((result, ex) -> {
    if (ex != null) {
        log.error("异常: {}", ex.getMessage());
        return "Default";
    }
    return result;
});
```

**whenComplete：不改变结果，只做副作用操作**

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return "Success";
}).whenComplete((result, ex) -> {
    if (ex != null) {
        log.error("异常: {}", ex.getMessage());
    } else {
        log.info("结果: {}", result);
    }
});
```

> **避坑提示：** exceptionally 和 handle 会改变 CompletableFuture 的结果，而 whenComplete 不会。选择哪个取决于你是否需要改变结果。

---

## 超时控制

JDK 9 引入了超时控制方法：

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    Thread.sleep(2000);
    return "Success";
}).orTimeout(1, TimeUnit.SECONDS)  // 超时抛出 TimeoutException
  .exceptionally(ex -> {
      return "Timeout";
  });
```

JDK 8 可以通过 CompletableFuture.anyOf 实现：

```java
CompletableFuture<String> taskFuture = CompletableFuture.supplyAsync(() -> {
    Thread.sleep(2000);
    return "Success";
});

CompletableFuture<String> timeoutFuture = new CompletableFuture<>();
scheduler.schedule(() -> {
    timeoutFuture.completeExceptionally(new TimeoutException());
}, 1, TimeUnit.SECONDS);

CompletableFuture<String> result = (CompletableFuture<String>) CompletableFuture.anyOf(
    taskFuture, timeoutFuture
);
```

---

## 与线程池的配合

默认情况下，CompletableFuture 使用 ForkJoinPool.commonPool()，线程数等于 CPU 核心数。

对于 IO 密集型任务，应该使用自定义线程池：

```java
ExecutorService executor = Executors.newFixedThreadPool(20);

CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // IO 密集型任务
    return httpClient.get(url);
}, executor);
```

**注意事项**

1. 不同类型的任务使用不同的线程池（CPU 密集型 vs IO 密集型）
2. 线程池要合理配置大小，避免资源耗尽
3. 应用关闭时要优雅地关闭线程池

```java
@PreDestroy
public void shutdown() {
    executor.shutdown();
    try {
        if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
            executor.shutdownNow();
        }
    } catch (InterruptedException e) {
        executor.shutdownNow();
    }
}
```

---

## 最佳实践

**1. 避免阻塞操作**

不要在 CompletableFuture 的回调中执行阻塞操作，会占用线程池资源。

```java
// 不好：阻塞操作
CompletableFuture.supplyAsync(() -> {
    Thread.sleep(1000);  // 阻塞线程
    return "Result";
});

// 好：使用异步 API
CompletableFuture.supplyAsync(() -> {
    return asyncHttpClient.get(url).join();
});
```

**2. 合理设置超时**

避免任务无限等待，导致资源泄漏。

**3. 异常处理要完善**

确保每个 CompletableFuture 都有异常处理逻辑，避免异常被吞掉。

**4. 避免过度并行**

并行任务数不是越多越好，要根据系统资源和下游服务的承载能力调整。

---

## 总结与思考

CompletableFuture 是 Java 异步编程的利器，它提供了：

- 链式调用：优雅的异步流水线
- 组合操作：灵活的任务编排
- 异常处理：完善的错误处理机制
- 线程池集成：高效的资源利用

掌握 CompletableFuture，能够显著提升系统的并发性能和代码的可维护性。

下次当你需要并行调用多个服务时，不妨试试 CompletableFuture，体验异步编程的魅力。
