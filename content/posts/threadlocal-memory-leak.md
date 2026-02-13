---
title: "ThreadLocal 原理与内存泄漏排查实战"
date: 2024-02-20
tags: ["Java", "并发编程", "内存泄漏", "问题排查"]
categories: ["技术"]
description: "深入 ThreadLocal 的实现机制，理解弱引用设计，掌握线程池环境下的内存泄漏排查"
draft: false
---

ThreadLocal 是 Java 并发编程中的重要工具，它为每个线程提供独立的变量副本。但很多人只知道怎么用，不知道它的实现原理，更不知道它可能导致内存泄漏。本文深入剖析 ThreadLocal 的底层机制。

### 本文亮点
- [x] 理解 ThreadLocal 的实现原理与 ThreadLocalMap 结构
- [x] 掌握弱引用的设计意图与内存泄漏的根因
- [x] 学会在线程池环境下正确使用 ThreadLocal
- [x] 了解 InheritableThreadLocal 的使用场景

---

## ThreadLocal 的使用场景

ThreadLocal 常用于以下场景：

**场景一：用户上下文传递**

```java
public class UserContext {
    private static final ThreadLocal<User> context = new ThreadLocal<>();
    
    public static void setUser(User user) {
        context.set(user);
    }
    
    public static User getUser() {
        return context.get();
    }
    
    public static void remove() {
        context.remove();
    }
}

// 在 Controller 中设置用户信息
@RestController
public class UserController {
    
    @GetMapping("/user/info")
    public UserDTO getUserInfo() {
        User user = UserContext.getUser();
        return UserDTO.from(user);
    }
}
```

**场景二：数据库连接管理**

```java
public class ConnectionManager {
    private static final ThreadLocal<Connection> connectionHolder = new ThreadLocal<>();
    
    public static Connection getConnection() {
        Connection conn = connectionHolder.get();
        if (conn == null) {
            conn = DriverManager.getConnection(url, user, password);
            connectionHolder.set(conn);
        }
        return conn;
    }
    
    public static void closeConnection() {
        Connection conn = connectionHolder.get();
        if (conn != null) {
            conn.close();
            connectionHolder.remove();
        }
    }
}
```

**场景三：SimpleDateFormat 线程安全**

SimpleDateFormat 不是线程安全的，可以用 ThreadLocal 为每个线程创建独立实例：

```java
public class DateUtil {
    private static final ThreadLocal<SimpleDateFormat> formatter = 
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
    
    public static String format(Date date) {
        return formatter.get().format(date);
    }
}
```

---

## ThreadLocal 的实现原理

**ThreadLocalMap：真正的存储结构**

ThreadLocal 本身不存储数据，数据存储在 Thread 对象的 threadLocals 字段中：

```java
public class Thread {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

ThreadLocalMap 是 ThreadLocal 的静态内部类，类似于 HashMap，但实现更简单：

```java
static class ThreadLocalMap {
    
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
        
        Entry(ThreadLocal<?> k, Object v) {
            super(k);  // key 是弱引用
            value = v;
        }
    }
    
    private Entry[] table;
    private int size = 0;
    private int threshold;
}
```

关键点：Entry 的 key 是 ThreadLocal 的弱引用，value 是强引用。

**set 方法的实现**

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

流程：

1. 获取当前线程的 ThreadLocalMap
2. 如果 map 不存在，创建一个新的 map
3. 将 ThreadLocal 作为 key，value 作为值存入 map

**get 方法的实现**

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            return (T) e.value;
        }
    }
    return setInitialValue();
}

private T setInitialValue() {
    T value = initialValue();  // 默认返回 null
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
    return value;
}
```

> **架构思考：** ThreadLocal 的设计巧妙之处在于：数据存储在 Thread 对象中，而不是 ThreadLocal 对象中。这样，每个线程访问自己的数据时，不需要加锁，避免了线程竞争。

---

## 弱引用：为什么 key 是弱引用？

**四种引用类型**

- 强引用：普通的对象引用，只要引用存在，对象就不会被回收
- 软引用：内存不足时会被回收
- 弱引用：下次 GC 时一定会被回收
- 虚引用：无法通过引用获取对象，只用于跟踪对象被回收的状态

**为什么 key 是弱引用？**

假设 key 是强引用：

```java
ThreadLocal<User> userThreadLocal = new ThreadLocal<>();
userThreadLocal.set(new User());

// 业务代码不再使用 userThreadLocal
userThreadLocal = null;
```

此时，虽然 userThreadLocal 变量被置为 null，但 ThreadLocalMap 中的 Entry 仍然持有 ThreadLocal 的强引用，导致 ThreadLocal 对象无法被回收。

如果 key 是弱引用：

```java
userThreadLocal = null;
// 下次 GC 时，ThreadLocal 对象会被回收
// ThreadLocalMap 中的 Entry 的 key 变成 null
```

这样，ThreadLocal 对象可以被及时回收，避免内存泄漏。

**但是，value 仍然是强引用！**

即使 key 被回收，value 仍然存在，这就是内存泄漏的根源。

---

## 内存泄漏：线程池环境下的陷阱

**问题场景**

```java
@Service
public class UserService {
    
    private static final ThreadLocal<List<User>> cache = new ThreadLocal<>();
    
    public void processUsers() {
        List<User> users = userMapper.selectAll();  // 假设有 10 万条数据
        cache.set(users);
        
        // 处理业务逻辑
        doSomething(users);
        
        // 忘记调用 remove()
    }
}
```

在 Tomcat 线程池环境下：

1. 请求到达，线程池分配线程 A 处理请求
2. 线程 A 执行 processUsers()，将 10 万条数据存入 ThreadLocal
3. 请求结束，线程 A 返回线程池，但 ThreadLocal 中的数据没有清理
4. 下次请求复用线程 A 时，ThreadLocal 中仍然保存着上次的数据

随着请求增多，每个线程的 ThreadLocal 都会积累大量数据，最终导致 OOM。

**内存泄漏的根因**

```
Thread (强引用)
  ↓
ThreadLocalMap (强引用)
  ↓
Entry[] (强引用)
  ↓
Entry.key (弱引用) → ThreadLocal (可能被回收)
Entry.value (强引用) → User List (无法被回收)
```

即使 ThreadLocal 对象被回收，value 仍然被 Entry 强引用，无法被 GC 回收。

**正确的使用方式**

```java
@Service
public class UserService {
    
    private static final ThreadLocal<List<User>> cache = new ThreadLocal<>();
    
    public void processUsers() {
        try {
            List<User> users = userMapper.selectAll();
            cache.set(users);
            doSomething(users);
        } finally {
            cache.remove();  // 必须在 finally 中清理
        }
    }
}
```

**ThreadLocal 的自动清理机制**

ThreadLocalMap 在 set、get、remove 时会触发清理：

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    
    // 清理 key 为 null 的 Entry
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;
    
    // 继续扫描后续的 Entry
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        }
    }
    return i;
}
```

但这种清理是被动的，不能完全依赖。最佳实践是手动调用 remove()。

> **避坑提示：** 在使用线程池的环境下，ThreadLocal 必须在 finally 块中调用 remove()。否则会导致内存泄漏和数据混乱（下一个请求可能读到上一个请求的数据）。

---

## InheritableThreadLocal：父子线程传递

ThreadLocal 的数据无法在父子线程之间传递：

```java
ThreadLocal<String> threadLocal = new ThreadLocal<>();
threadLocal.set("parent");

new Thread(() -> {
    System.out.println(threadLocal.get());  // 输出 null
}).start();
```

InheritableThreadLocal 解决了这个问题：

```java
InheritableThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();
inheritableThreadLocal.set("parent");

new Thread(() -> {
    System.out.println(inheritableThreadLocal.get());  // 输出 parent
}).start();
```

**实现原理**

Thread 类有两个 ThreadLocalMap 字段：

```java
public class Thread {
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
}
```

创建子线程时，会复制父线程的 inheritableThreadLocals：

```java
private void init(ThreadGroup g, Runnable target, String name, long stackSize) {
    Thread parent = currentThread();
    
    if (parent.inheritableThreadLocals != null) {
        this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    }
}
```

**注意事项**

InheritableThreadLocal 只在创建子线程时复制数据，如果使用线程池，子线程会被复用，数据不会更新。

解决方案：使用阿里开源的 TransmittableThreadLocal（TTL）。

---

## 实战案例：排查 ThreadLocal 内存泄漏

**问题现象**

生产环境的应用内存持续增长，最终 OOM。通过 MAT 分析 Heap Dump，发现大量 ThreadLocalMap.Entry 对象。

**排查步骤**

1. 查看 Dominator Tree，发现 Thread 对象占用了大量内存
2. 展开 Thread 对象，发现 threadLocals 字段中有大量 Entry
3. 查看 Entry 的 value，发现是业务对象（User List）
4. 通过引用链找到创建 ThreadLocal 的代码位置

**修复方案**

在所有使用 ThreadLocal 的地方，添加 finally 块调用 remove()：

```java
try {
    threadLocal.set(value);
    // 业务逻辑
} finally {
    threadLocal.remove();
}
```

**预防措施**

1. Code Review 重点关注 ThreadLocal 的使用
2. 使用 Alibaba Java Coding Guidelines 插件，自动检测 ThreadLocal 未调用 remove()
3. 在单元测试中模拟线程池环境，检测内存泄漏

---

## 总结与思考

ThreadLocal 的核心原理：

- 数据存储在 Thread 对象的 ThreadLocalMap 中
- key 是 ThreadLocal 的弱引用，value 是强引用
- 弱引用设计避免了 ThreadLocal 对象的内存泄漏，但 value 仍可能泄漏

使用 ThreadLocal 的最佳实践：

- 在 finally 块中调用 remove()
- 避免存储大对象
- 使用 try-with-resources 模式封装 ThreadLocal

下次当你使用 ThreadLocal 时，记得问自己：我在 finally 中调用 remove() 了吗？
