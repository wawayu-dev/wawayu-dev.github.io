---
title: "JVM OOM 问题排查实战：从 Heap Dump 到根因分析"
date: 2024-10-25
tags: ["Java", "JVM调优", "内存泄漏", "问题排查"]
categories: ["技术"]
description: "掌握 JVM 内存模型与 GC 机制，学会使用 MAT 工具分析 Heap Dump"
draft: false
---

生产环境的 OOM（Out Of Memory）是后端开发最头疼的问题之一。本文通过真实案例，讲解如何定位和解决 JVM 内存问题。

### 本文亮点
- [x] 理解 JVM 内存模型与 GC 机制
- [x] 掌握 Heap Dump 的生成与分析方法
- [x] 学会使用 MAT 工具定位内存泄漏
- [x] 了解常见的内存泄漏场景与解决方案

---

## JVM 内存模型

**堆内存（Heap）**

- 年轻代（Young Generation）
  - Eden 区：新对象分配的区域
  - Survivor 区：S0 和 S1，存放经过一次 GC 后存活的对象
- 老年代（Old Generation）：存放长期存活的对象

**非堆内存**

- 方法区（Metaspace）：存放类信息、常量、静态变量
- 栈（Stack）：存放局部变量、方法调用
- 直接内存（Direct Memory）：NIO 使用的堆外内存


**GC 机制**

- Minor GC：清理年轻代
- Major GC：清理老年代
- Full GC：清理整个堆内存

**对象晋升**

1. 新对象在 Eden 区分配
2. Eden 区满了，触发 Minor GC
3. 存活的对象移动到 Survivor 区
4. 经过多次 GC 后，对象晋升到老年代

> **架构思考：** JVM 的分代收集体现了"大部分对象朝生夕死"的特点。年轻代使用复制算法，快速回收短命对象；老年代使用标记-清除或标记-整理算法，处理长期存活的对象。这种设计在性能和内存利用率之间找到了平衡。

---

## OOM 的常见类型

**1. java.lang.OutOfMemoryError: Java heap space**

堆内存不足，最常见的 OOM。

原因：

- 内存泄漏：对象无法被 GC 回收
- 内存溢出：对象太多，超过堆内存大小

**2. java.lang.OutOfMemoryError: GC overhead limit exceeded**

GC 占用了 98% 的时间，但只回收了不到 2% 的内存。

**3. java.lang.OutOfMemoryError: Metaspace**

方法区（元空间）内存不足。

原因：

- 加载了太多的类
- 动态生成类（如 CGLIB 代理）

**4. java.lang.OutOfMemoryError: Direct buffer memory**

直接内存不足，通常是 NIO 使用不当。

---

## 案例一：ThreadLocal 导致的内存泄漏

**问题现象**

生产环境的应用内存持续增长，最终 OOM。

**排查步骤**

**1. 生成 Heap Dump**

```bash
# 方式一：OOM 时自动生成
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof -jar app.jar

# 方式二：手动生成
jmap -dump:format=b,file=/tmp/heapdump.hprof <pid>
```

**2. 使用 MAT 分析**

下载 Eclipse Memory Analyzer（MAT）工具，打开 Heap Dump 文件。

**3. 查看 Dominator Tree**

Dominator Tree 显示占用内存最多的对象。

发现：Thread 对象占用了 2GB 内存。

**4. 查看 Thread 对象的引用链**

展开 Thread 对象，发现 threadLocals 字段中有大量 Entry 对象。

**5. 查看 Entry 的 value**

发现 value 是业务对象（User List），包含 10 万条数据。

**根因**

代码中使用了 ThreadLocal，但没有调用 remove()，导致内存泄漏。

```java
@Service
public class UserService {
    
    private static final ThreadLocal<List<User>> cache = new ThreadLocal<>();
    
    public void processUsers() {
        List<User> users = userMapper.selectAll();
        cache.set(users);
        
        // 处理业务逻辑
        doSomething(users);
        
        // 忘记调用 remove()
    }
}
```

**解决方案**

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

---

## 案例二：大对象导致的 OOM

**问题现象**

导出 Excel 时，应用 OOM。

**排查步骤**

**1. 查看 GC 日志**

```bash
java -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/tmp/gc.log -jar app.jar
```

GC 日志显示：Full GC 频繁，但回收效果不明显。

**2. 分析 Heap Dump**

发现大量的 ArrayList 对象，每个包含 10 万条数据。

**根因**

使用传统的 POI 导出 Excel，一次性加载所有数据到内存。

```java
public void exportExcel() {
    List<User> users = userMapper.selectAll();  // 查询 10 万条数据
    
    Workbook workbook = new XSSFWorkbook();
    Sheet sheet = workbook.createSheet();
    
    for (int i = 0; i < users.size(); i++) {
        Row row = sheet.createRow(i);
        // 填充数据
    }
    
    workbook.write(outputStream);
}
```

**解决方案：使用 EasyExcel 流式写入**

```java
public void exportExcel() {
    EasyExcel.write(outputStream, User.class)
        .registerWriteHandler(new LongestMatchColumnWidthStyleStrategy())
        .sheet("用户列表")
        .doWrite(() -> {
            // 分批查询
            int pageSize = 1000;
            int pageNum = 1;
            List<User> users;
            List<User> allUsers = new ArrayList<>();
            
            do {
                users = userMapper.selectPage(pageNum, pageSize);
                allUsers.addAll(users);
                pageNum++;
            } while (users.size() == pageSize);
            
            return allUsers;
        });
}
```

更好的方案：使用 EasyExcel 的分批写入

```java
public void exportExcel() {
    ExcelWriter excelWriter = EasyExcel.write(outputStream, User.class).build();
    WriteSheet writeSheet = EasyExcel.writerSheet("用户列表").build();
    
    int pageSize = 1000;
    int pageNum = 1;
    List<User> users;
    
    do {
        users = userMapper.selectPage(pageNum, pageSize);
        excelWriter.write(users, writeSheet);
        pageNum++;
    } while (users.size() == pageSize);
    
    excelWriter.finish();
}
```

---

## 案例三：静态集合导致的内存泄漏

**问题现象**

应用运行一段时间后，内存持续增长。

**根因**

使用静态集合缓存数据，但没有清理机制。

```java
public class CacheManager {
    
    private static final Map<String, Object> cache = new HashMap<>();
    
    public static void put(String key, Object value) {
        cache.put(key, value);
    }
    
    public static Object get(String key) {
        return cache.get(key);
    }
}
```

**解决方案一：使用 WeakHashMap**

```java
private static final Map<String, Object> cache = new WeakHashMap<>();
```

WeakHashMap 的 key 是弱引用，当 key 没有强引用时，会被 GC 回收。

**解决方案二：使用 Guava Cache**

```java
private static final Cache<String, Object> cache = CacheBuilder.newBuilder()
    .maximumSize(10000)  // 最大缓存数量
    .expireAfterWrite(10, TimeUnit.MINUTES)  // 写入后 10 分钟过期
    .build();
```

**解决方案三：使用 Redis**

将缓存放到 Redis，避免占用 JVM 内存。

---

## 常见的内存泄漏场景

**1. ThreadLocal 未清理**

在线程池环境下，ThreadLocal 必须调用 remove()。

**2. 静态集合**

静态集合的生命周期与应用相同，容易导致内存泄漏。

**3. 监听器未注销**

```java
// 注册监听器
eventBus.register(listener);

// 忘记注销
// eventBus.unregister(listener);
```

**4. 数据库连接未关闭**

```java
Connection conn = DriverManager.getConnection(url);
// 忘记关闭
// conn.close();
```

**5. IO 流未关闭**

```java
FileInputStream fis = new FileInputStream(file);
// 忘记关闭
// fis.close();
```

使用 try-with-resources 自动关闭资源：

```java
try (FileInputStream fis = new FileInputStream(file)) {
    // 使用流
}
```

---

## JVM 参数调优

**堆内存设置**

```bash
-Xms4g  # 初始堆内存
-Xmx4g  # 最大堆内存（建议与 Xms 相同，避免动态扩容）
```

**年轻代设置**

```bash
-Xmn2g  # 年轻代大小（建议为堆内存的 1/3 到 1/2）
```

**元空间设置**

```bash
-XX:MetaspaceSize=256m  # 初始元空间大小
-XX:MaxMetaspaceSize=512m  # 最大元空间大小
```

**GC 日志**

```bash
-XX:+PrintGCDetails  # 打印 GC 详细信息
-XX:+PrintGCDateStamps  # 打印 GC 时间戳
-Xloggc:/tmp/gc.log  # GC 日志文件
```

**OOM 时生成 Heap Dump**

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof
```

**GC 收集器选择**

```bash
# G1 收集器（推荐）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200  # 最大 GC 停顿时间

# ZGC（JDK 11+）
-XX:+UseZGC
```

---

## 监控与预警

**1. 使用 JVM 监控工具**

- JConsole：JDK 自带的监控工具
- VisualVM：可视化监控工具
- Arthas：阿里开源的诊断工具

**2. 使用 APM 工具**

- SkyWalking：分布式追踪系统
- Prometheus + Grafana：监控与可视化

**3. 设置内存告警**

当堆内存使用率超过 80% 时，发送告警通知。

---

## 总结与思考

JVM OOM 排查的方法论：

1. 生成 Heap Dump
2. 使用 MAT 分析内存占用
3. 查看 Dominator Tree，找到占用内存最多的对象
4. 分析引用链，找到根因
5. 修复代码，避免内存泄漏

常见的内存泄漏场景：

- ThreadLocal 未清理
- 静态集合无限增长
- 监听器未注销
- 资源未关闭

预防措施：

- Code Review 重点关注资源管理
- 使用 try-with-resources 自动关闭资源
- 定期分析 Heap Dump，提前发现问题
- 设置合理的 JVM 参数和监控告警

下次当你的应用 OOM 时，按照这个方法论一步步排查，一定能找到问题所在。
