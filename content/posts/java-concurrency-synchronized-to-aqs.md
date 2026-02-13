---
title: "Java 并发编程：从 synchronized 到 AQS 的演进"
date: 2024-01-15
tags: ["Java", "并发编程", "锁机制", "源码分析"]
categories: ["技术"]
description: "深入理解 Java 锁机制的底层实现，从 synchronized 的锁升级到 AQS 的核心原理"
draft: false
---

在 Java 并发编程中，锁是保证线程安全的核心机制。从最早的 synchronized 到 JDK 1.5 引入的 AQS 框架，Java 的锁机制经历了巨大的演进。理解这个演进过程，是掌握 Java 并发编程的关键。

### 本文亮点
- [x] 理解 synchronized 的锁升级过程（偏向锁、轻量级锁、重量级锁）
- [x] 掌握 AQS 的核心原理与应用场景
- [x] 学会选择 ReentrantLock 还是 synchronized
- [x] 了解 CAS 操作与自旋锁的实现

---

## synchronized：从重量级到轻量级

在 JDK 1.6 之前，synchronized 是重量级锁，每次加锁都需要操作系统的 mutex 互斥量，涉及用户态和内核态的切换，性能很差。

JDK 1.6 对 synchronized 进行了大量优化，引入了锁升级机制：偏向锁 → 轻量级锁 → 重量级锁。

**对象头：锁信息的存储位置**

Java 对象在内存中的布局包括三部分：对象头、实例数据、对齐填充。锁信息存储在对象头的 Mark Word 中。

Mark Word 的结构（64 位 JVM）：

```
|--------------------------------------------------------------|
| 锁状态   | 25bit | 31bit | 1bit | 4bit | 1bit偏向锁 | 2bit锁标志 |
|--------------------------------------------------------------|
| 无锁     | unused | hashcode | unused | age | 0 | 01 |
| 偏向锁   | ThreadID | Epoch | unused | age | 1 | 01 |
| 轻量级锁 | 指向栈中锁记录的指针 | 00 |
| 重量级锁 | 指向互斥量的指针 | 10 |
```

**偏向锁：单线程场景的优化**

大部分情况下，锁不仅不存在多线程竞争，而且总是由同一个线程多次获取。偏向锁的思想是：第一次获取锁时，在对象头中记录线程 ID，后续该线程再次获取锁时，只需要判断 Mark Word 中的线程 ID 是否是自己，无需 CAS 操作。

```java
public class BiasedLockDemo {
    private static Object lock = new Object();
    
    public static void main(String[] args) {
        // 第一次加锁：升级为偏向锁
        synchronized (lock) {
            System.out.println("第一次加锁");
        }
        
        // 第二次加锁：直接获取偏向锁，无需 CAS
        synchronized (lock) {
            System.out.println("第二次加锁");
        }
    }
}
```

偏向锁的撤销：当其他线程尝试获取锁时，偏向锁会撤销，升级为轻量级锁。

**轻量级锁：多线程交替执行的优化**

当有两个线程交替获取锁（没有竞争），使用轻量级锁。轻量级锁通过 CAS 操作将 Mark Word 替换为指向栈中锁记录的指针。

```java
// 线程 A
synchronized (lock) {
    // 执行业务逻辑
}

// 线程 B（在 A 释放锁后才获取）
synchronized (lock) {
    // 执行业务逻辑
}
```

轻量级锁的加锁流程：

1. 在栈帧中创建锁记录（Lock Record）
2. 将对象头的 Mark Word 复制到锁记录中
3. 通过 CAS 将对象头的 Mark Word 替换为指向锁记录的指针
4. 如果 CAS 成功，获取锁；如果失败，自旋重试

**重量级锁：多线程竞争的最终方案**

当自旋次数过多（默认 10 次），或者有多个线程同时竞争锁，轻量级锁会升级为重量级锁。重量级锁会阻塞等待的线程，通过操作系统的 mutex 实现。

```java
// 多个线程同时竞争
for (int i = 0; i < 10; i++) {
    new Thread(() -> {
        synchronized (lock) {
            // 执行业务逻辑
        }
    }).start();
}
```

> **架构思考：** synchronized 的锁升级体现了"适应性优化"的思想。在不同的竞争场景下，使用不同的锁实现，在性能和正确性之间找到平衡。这种思想在很多系统中都有应用，比如 MySQL 的自适应哈希索引、JVM 的分层编译。

---

## AQS：并发工具的基石

AQS（AbstractQueuedSynchronizer）是 JDK 1.5 引入的并发框架，ReentrantLock、Semaphore、CountDownLatch 等工具都基于 AQS 实现。

**AQS 的核心思想**

AQS 维护了一个 volatile int state 变量和一个 FIFO 队列。

- state：表示同步状态（0 表示未锁定，1 表示已锁定）
- FIFO 队列：存储等待获取锁的线程

**独占模式：ReentrantLock 的实现**

ReentrantLock 的加锁流程：

```java
public class ReentrantLock {
    
    private final Sync sync;
    
    public void lock() {
        sync.acquire(1);
    }
    
    abstract static class Sync extends AbstractQueuedSynchronizer {
        
        final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            
            // 如果 state == 0，说明锁未被占用
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 如果是当前线程持有锁，支持重入
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                setState(nextc);
                return true;
            }
            return false;
        }
    }
}
```

如果 tryAcquire 失败，线程会被加入等待队列：

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    
    // 快速尝试在队尾添加
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    
    // 如果失败，通过自旋添加
    enq(node);
    return node;
}
```

等待队列是一个双向链表，每个节点包含线程引用和等待状态。

**共享模式：Semaphore 的实现**

Semaphore 允许多个线程同时访问资源，通过 state 表示可用许可数。

```java
public class Semaphore {
    
    private final Sync sync;
    
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    
    abstract static class Sync extends AbstractQueuedSynchronizer {
        
        final int tryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                
                // 如果许可不足，返回负数
                if (remaining < 0) {
                    return remaining;
                }
                
                // 通过 CAS 扣减许可
                if (compareAndSetState(available, remaining)) {
                    return remaining;
                }
            }
        }
    }
}
```

共享模式下，释放锁时会唤醒队列中的所有等待线程。

> **避坑提示：** AQS 的 state 是 volatile 的，保证了可见性，但不保证原子性。所以修改 state 时必须使用 CAS 操作。

---

## ReentrantLock vs synchronized

**功能对比**

| 特性 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 锁的实现 | JVM 层面 | JDK 层面 |
| 可重入性 | 支持 | 支持 |
| 公平锁 | 不支持 | 支持 |
| 可中断 | 不支持 | 支持 |
| 超时获取 | 不支持 | 支持 |
| 条件变量 | 单个（wait/notify） | 多个（Condition） |

**性能对比**

在 JDK 1.6 之后，synchronized 经过优化，性能已经和 ReentrantLock 相当。在低竞争场景下，synchronized 甚至更快（因为有锁消除和锁粗化优化）。

**使用建议**

- 优先使用 synchronized：代码简洁，JVM 会自动优化
- 需要高级特性时使用 ReentrantLock：公平锁、可中断、超时获取

**实战案例：可中断锁**

```java
public class InterruptibleLockDemo {
    
    private final ReentrantLock lock = new ReentrantLock();
    
    public void doWork() throws InterruptedException {
        // 可中断地获取锁
        lock.lockInterruptibly();
        try {
            // 执行业务逻辑
            Thread.sleep(10000);
        } finally {
            lock.unlock();
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        InterruptibleLockDemo demo = new InterruptibleLockDemo();
        
        Thread t1 = new Thread(() -> {
            try {
                demo.doWork();
            } catch (InterruptedException e) {
                System.out.println("线程被中断");
            }
        });
        
        t1.start();
        Thread.sleep(1000);
        t1.interrupt();  // 中断线程
    }
}
```

如果使用 synchronized，线程无法被中断，只能等待锁释放。

---

## CAS 与自旋锁

CAS（Compare And Swap）是实现无锁并发的基础。

**CAS 的原理**

CAS 包含三个操作数：内存位置 V、预期值 A、新值 B。只有当 V 的值等于 A 时，才会将 V 更新为 B。

```java
public class AtomicInteger {
    
    private volatile int value;
    
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
    
    public final int incrementAndGet() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next)) {
                return next;
            }
        }
    }
}
```

CAS 的优势：无需加锁，避免了线程阻塞和上下文切换。

CAS 的问题：

1. ABA 问题：值从 A 变成 B，再变回 A，CAS 无法感知
2. 自旋开销：如果长时间 CAS 失败，会浪费 CPU
3. 只能保证单个变量的原子性

**自旋锁的实现**

```java
public class SpinLock {
    
    private AtomicReference<Thread> owner = new AtomicReference<>();
    
    public void lock() {
        Thread current = Thread.currentThread();
        // 自旋等待，直到 CAS 成功
        while (!owner.compareAndSet(null, current)) {
        }
    }
    
    public void unlock() {
        Thread current = Thread.currentThread();
        owner.compareAndSet(current, null);
    }
}
```

自旋锁适合锁持有时间很短的场景，避免了线程阻塞的开销。

---

## 总结与思考

Java 锁机制的演进体现了性能优化的核心思想：在不同场景下使用不同的策略。

- synchronized：从重量级锁优化为锁升级机制，适应不同竞争场景
- AQS：提供了灵活的并发工具框架，支持独占和共享模式
- CAS：无锁并发的基础，避免了线程阻塞

理解这些机制，才能在实际开发中选择合适的并发工具，写出高性能的并发代码。

下次当你需要加锁时，不妨想想：这个场景适合 synchronized 还是 ReentrantLock？是否可以用 CAS 实现无锁并发？
