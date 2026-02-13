---
title: "分布式 ID 生成方案对比与选型指南"
date: 2024-07-05
tags: ["分布式系统", "分布式ID", "架构设计", "高并发"]
categories: ["技术"]
description: "从雪花算法到美团 Leaf，深入理解分布式 ID 的设计思想与实现方案"
draft: false
---

在分布式系统中，生成全局唯一 ID 是一个基础需求。数据库自增 ID 在分库分表后无法使用，UUID 虽然简单但不适合作为主键。本文对比主流的分布式 ID 生成方案。

### 本文亮点
- [x] 理解分布式 ID 的核心要求
- [x] 掌握雪花算法的实现与时钟回拨问题
- [x] 了解美团 Leaf 方案的设计思想
- [x] 学会根据业务场景选择合适方案

---

## 分布式 ID 的核心要求

1. 全局唯一性：不能重复
2. 趋势递增：方便排序和索引
3. 高性能：生成速度快
4. 高可用：服务不能挂
5. 信息安全：不能泄露业务信息

---

## 方案一：数据库自增 ID

```sql
CREATE TABLE id_generator (
    id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    stub CHAR(1) NOT NULL DEFAULT '',
    UNIQUE KEY stub (stub)
);

INSERT INTO id_generator (stub) VALUES ('a');
SELECT LAST_INSERT_ID();
```

优点：简单，趋势递增

缺点：

- 性能瓶颈：数据库压力大
- 单点故障：数据库挂了就无法生成 ID
- 扩展性差：分库分表后需要设置不同的步长

---

## 方案二：UUID

```java
String id = UUID.randomUUID().toString();
```

优点：简单，本地生成，性能高

缺点：

- 无序：不利于数据库索引
- 长度长：36 个字符，占用空间大
- 不适合作为主键：B+ 树索引性能差

---

## 方案三：雪花算法（Snowflake）

Twitter 开源的分布式 ID 生成算法，生成 64 位 long 型 ID。

**ID 结构**

```
0 - 41位时间戳 - 10位机器ID - 12位序列号
```

- 1 位：符号位，固定为 0
- 41 位：时间戳（毫秒级），可用 69 年
- 10 位：机器 ID（5 位数据中心 ID + 5 位机器 ID），支持 1024 台机器
- 12 位：序列号，同一毫秒内可生成 4096 个 ID

**实现代码**

```java
public class SnowflakeIdGenerator {
    
    private final long twepoch = 1288834974657L;  // 起始时间戳
    private final long workerIdBits = 5L;
    private final long datacenterIdBits = 5L;
    private final long sequenceBits = 12L;
    
    private final long workerIdShift = sequenceBits;
    private final long datacenterIdShift = sequenceBits + workerIdBits;
    private final long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);
    
    private long workerId;
    private long datacenterId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;
    
    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();
        
        // 时钟回拨检测
        if (timestamp < lastTimestamp) {
            throw new RuntimeException("Clock moved backwards");
        }
        
        // 同一毫秒内，序列号自增
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            if (sequence == 0) {
                // 序列号用完，等待下一毫秒
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0L;
        }
        
        lastTimestamp = timestamp;
        
        // 组装 ID
        return ((timestamp - twepoch) << timestampLeftShift)
            | (datacenterId << datacenterIdShift)
            | (workerId << workerIdShift)
            | sequence;
    }
    
    private long tilNextMillis(long lastTimestamp) {
        long timestamp = System.currentTimeMillis();
        while (timestamp <= lastTimestamp) {
            timestamp = System.currentTimeMillis();
        }
        return timestamp;
    }
}
```

**时钟回拨问题**

如果系统时间被回拨，会导致 ID 重复。

解决方案：

1. 拒绝生成 ID，抛出异常
2. 等待时钟追上
3. 使用扩展位记录时钟回拨次数

---

## 方案四：美团 Leaf

美团开源的分布式 ID 生成服务，支持两种模式：

**Leaf-segment：号段模式**

从数据库批量获取 ID 号段，缓存在内存中分配。

```sql
CREATE TABLE leaf_alloc (
    biz_tag VARCHAR(128) NOT NULL,
    max_id BIGINT NOT NULL,
    step INT NOT NULL,
    PRIMARY KEY (biz_tag)
);
```

流程：

1. 服务启动时，从数据库获取号段（如 1-1000）
2. 在内存中分配 ID
3. 号段用完后，再次从数据库获取下一个号段

优点：

- 减少数据库访问次数
- 支持动态调整步长

**Leaf-snowflake：雪花模式**

基于 Zookeeper 生成机器 ID，解决了雪花算法的机器 ID 分配问题。

---

## 方案对比

| 方案 | 性能 | 可用性 | 趋势递增 | 复杂度 |
|------|------|--------|----------|--------|
| 数据库自增 | 低 | 低 | 是 | 低 |
| UUID | 高 | 高 | 否 | 低 |
| 雪花算法 | 高 | 高 | 是 | 中 |
| Leaf-segment | 中 | 中 | 是 | 中 |
| Leaf-snowflake | 高 | 高 | 是 | 高 |

---

## 选型建议

- 小型项目：数据库自增 ID
- 对顺序无要求：UUID
- 高性能要求：雪花算法
- 需要集中管理：Leaf

---

## 总结与思考

分布式 ID 生成是分布式系统的基础问题，不同方案适用于不同场景。

- 雪花算法性能高，但需要解决时钟回拨和机器 ID 分配问题
- Leaf 提供了更完善的解决方案，适合生产环境

下次当你需要生成分布式 ID 时，根据业务特点选择合适的方案。
