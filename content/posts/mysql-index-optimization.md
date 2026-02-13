---
title: "MySQL 索引优化：从 Explain 到执行计划"
date: 2024-11-15
tags: ["MySQL", "索引优化", "SQL调优", "性能优化"]
categories: ["技术"]
description: "系统掌握 MySQL 索引优化方法，从 B+ 树原理到执行计划分析"
draft: false
---

SQL 性能优化是后端开发的必备技能。一条慢查询可能拖垮整个系统，而合理的索引设计能让查询速度提升百倍。本文系统讲解 MySQL 索引优化的方法论。

### 本文亮点
- [x] 理解 B+ 树索引的底层原理
- [x] 掌握 Explain 执行计划的分析方法
- [x] 学会联合索引的最左前缀匹配规则
- [x] 了解覆盖索引与索引下推优化

---

## B+ 树索引原理

**为什么选择 B+ 树？**

常见的数据结构对比：

| 数据结构 | 查询时间复杂度 | 问题 |
|---------|--------------|------|
| 数组 | O(n) | 查询慢 |
| 二叉搜索树 | O(log n) | 可能退化为链表 |
| 平衡二叉树 | O(log n) | 树高太高，IO 次数多 |
| B 树 | O(log n) | 非叶子节点存储数据，浪费空间 |
| B+ 树 | O(log n) | 叶子节点存储数据，非叶子节点只存索引 |

**B+ 树的特点**

1. 所有数据存储在叶子节点
2. 叶子节点通过指针连接，支持范围查询
3. 非叶子节点只存储索引，可以存储更多的键值
4. 树的高度低，IO 次数少


**InnoDB 的聚簇索引**

InnoDB 的主键索引是聚簇索引，叶子节点存储完整的行数据。

```
主键索引（聚簇索引）
       10
      /  \
     5    15
    / \   / \
   1  7  12 18
   |  |  |  |
  [完整行数据]
```

二级索引（非主键索引）的叶子节点存储主键值，需要回表查询。

```
二级索引（name 索引）
      "Bob"
      /   \
   "Alice" "Charlie"
     |       |
    [主键1] [主键3]
     ↓       ↓
   回表查询完整数据
```

> **架构思考：** 聚簇索引的设计体现了"局部性原理"。相邻的主键值存储在相邻的磁盘位置，范围查询时可以顺序读取，减少随机 IO。这也是为什么推荐使用自增主键的原因。

---

## Explain 执行计划分析

**基本用法**

```sql
EXPLAIN SELECT * FROM user WHERE age = 25;
```

**关键字段解析**

**1. type（访问类型）**

性能从好到坏：

- system：表只有一行数据（系统表）
- const：通过主键或唯一索引查询，最多返回一行
- eq_ref：唯一索引扫描，对于每个索引键，表中只有一条记录匹配
- ref：非唯一索引扫描，返回匹配某个单独值的所有行
- range：索引范围扫描（BETWEEN、>、<）
- index：全索引扫描
- ALL：全表扫描（最差）

```sql
-- const
EXPLAIN SELECT * FROM user WHERE id = 1;

-- ref
EXPLAIN SELECT * FROM user WHERE name = 'Alice';

-- range
EXPLAIN SELECT * FROM user WHERE age BETWEEN 20 AND 30;

-- ALL
EXPLAIN SELECT * FROM user WHERE email LIKE '%@gmail.com';
```

**2. key（实际使用的索引）**

如果为 NULL，说明没有使用索引。

**3. rows（扫描的行数）**

预估需要扫描的行数，越少越好。

**4. Extra（额外信息）**

- Using index：使用了覆盖索引，不需要回表
- Using where：使用了 WHERE 过滤
- Using filesort：需要额外的排序操作（性能差）
- Using temporary：需要创建临时表（性能差）

---

## 索引失效的 10 种场景

**1. 使用函数或表达式**

```sql
-- 索引失效
SELECT * FROM user WHERE YEAR(create_time) = 2024;

-- 优化
SELECT * FROM user WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';
```

**2. 隐式类型转换**

```sql
-- phone 是 VARCHAR 类型，索引失效
SELECT * FROM user WHERE phone = 13800138000;

-- 优化
SELECT * FROM user WHERE phone = '13800138000';
```

**3. 使用 OR 连接**

```sql
-- 如果 age 没有索引，整个查询不走索引
SELECT * FROM user WHERE name = 'Alice' OR age = 25;

-- 优化：使用 UNION
SELECT * FROM user WHERE name = 'Alice'
UNION
SELECT * FROM user WHERE age = 25;
```

**4. LIKE 以 % 开头**

```sql
-- 索引失效
SELECT * FROM user WHERE name LIKE '%Alice';

-- 优化：% 放在后面
SELECT * FROM user WHERE name LIKE 'Alice%';
```

**5. 不等于（!= 或 <>）**

```sql
-- 索引失效
SELECT * FROM user WHERE age != 25;

-- 优化：使用范围查询
SELECT * FROM user WHERE age < 25 OR age > 25;
```

**6. IS NULL 或 IS NOT NULL**

```sql
-- 可能索引失效（取决于 NULL 值的比例）
SELECT * FROM user WHERE age IS NULL;

-- 优化：避免字段为 NULL，使用默认值
```

**7. NOT IN 和 NOT EXISTS**

```sql
-- 索引失效
SELECT * FROM user WHERE id NOT IN (1, 2, 3);

-- 优化：使用 LEFT JOIN
SELECT u.* FROM user u
LEFT JOIN (SELECT 1 AS id UNION SELECT 2 UNION SELECT 3) t ON u.id = t.id
WHERE t.id IS NULL;
```

**8. 联合索引不满足最左前缀**

```sql
-- 联合索引 (name, age, city)

-- 走索引
SELECT * FROM user WHERE name = 'Alice';
SELECT * FROM user WHERE name = 'Alice' AND age = 25;

-- 不走索引
SELECT * FROM user WHERE age = 25;
SELECT * FROM user WHERE city = 'Beijing';
```

**9. 范围查询后的字段不走索引**

```sql
-- 联合索引 (name, age, city)

-- age 走索引，city 不走索引
SELECT * FROM user WHERE name = 'Alice' AND age > 20 AND city = 'Beijing';
```

**10. 数据量太小**

如果表的数据量很小（几百行），MySQL 可能选择全表扫描而不是索引。

---

## 联合索引的最左前缀匹配

**规则**

联合索引 (a, b, c) 相当于创建了三个索引：

- (a)
- (a, b)
- (a, b, c)

**案例分析**

```sql
-- 创建联合索引
CREATE INDEX idx_name_age_city ON user(name, age, city);

-- 走索引
SELECT * FROM user WHERE name = 'Alice';
SELECT * FROM user WHERE name = 'Alice' AND age = 25;
SELECT * FROM user WHERE name = 'Alice' AND age = 25 AND city = 'Beijing';
SELECT * FROM user WHERE name = 'Alice' AND city = 'Beijing';  -- name 走索引，city 不走

-- 不走索引
SELECT * FROM user WHERE age = 25;
SELECT * FROM user WHERE city = 'Beijing';
SELECT * FROM user WHERE age = 25 AND city = 'Beijing';
```

**索引顺序的选择**

区分度高的字段放在前面。

```sql
-- 假设 name 有 1000 个不同值，age 有 50 个不同值，city 有 10 个不同值
-- 应该创建 (name, age, city)，而不是 (city, age, name)
```

---

## 覆盖索引

覆盖索引是指查询的字段都在索引中，不需要回表查询。

**案例**

```sql
-- 创建索引
CREATE INDEX idx_name_age ON user(name, age);

-- 覆盖索引（不需要回表）
SELECT name, age FROM user WHERE name = 'Alice';

-- 需要回表
SELECT * FROM user WHERE name = 'Alice';
```

**优化建议**

如果查询只需要少数几个字段，可以创建联合索引覆盖这些字段。

```sql
-- 优化前
SELECT id, name, age FROM user WHERE name = 'Alice';

-- 创建覆盖索引
CREATE INDEX idx_name_age ON user(name, age);

-- 优化后（不需要回表）
SELECT id, name, age FROM user WHERE name = 'Alice';
```

---

## 索引下推（ICP）

MySQL 5.6 引入的优化技术，将部分 WHERE 条件下推到存储引擎层过滤。

**案例**

```sql
-- 联合索引 (name, age)
SELECT * FROM user WHERE name LIKE 'A%' AND age = 25;
```

**没有索引下推**

1. 存储引擎根据 name LIKE 'A%' 找到所有匹配的记录
2. 返回给 Server 层
3. Server 层过滤 age = 25

**有索引下推**

1. 存储引擎根据 name LIKE 'A%' 找到匹配的记录
2. 在存储引擎层直接过滤 age = 25
3. 只返回符合条件的记录给 Server 层

减少了回表次数，提升了性能。

---

## 分页查询优化

**问题：深度分页**

```sql
-- 查询第 100 万条数据
SELECT * FROM user ORDER BY id LIMIT 1000000, 10;
```

MySQL 需要扫描 100 万 + 10 条数据，然后丢弃前 100 万条，性能很差。

**优化方案一：使用子查询**

```sql
SELECT * FROM user
WHERE id >= (SELECT id FROM user ORDER BY id LIMIT 1000000, 1)
LIMIT 10;
```

子查询使用覆盖索引，只返回 id，然后通过 id 查询完整数据。

**优化方案二：记录上次查询的最大 ID**

```sql
-- 第一次查询
SELECT * FROM user ORDER BY id LIMIT 10;

-- 第二次查询（假设上次最大 ID 是 10）
SELECT * FROM user WHERE id > 10 ORDER BY id LIMIT 10;
```

这种方式只适用于 ID 连续的场景。

---

## 索引设计的最佳实践

**1. 选择合适的字段创建索引**

- WHERE、ORDER BY、GROUP BY、JOIN 的字段
- 区分度高的字段（不要给性别创建索引）
- 字段长度小的字段

**2. 使用前缀索引**

对于长字符串字段，可以只索引前几个字符。

```sql
CREATE INDEX idx_email ON user(email(10));
```

**3. 避免冗余索引**

如果已经有联合索引 (name, age)，就不需要单独创建 (name) 索引。

**4. 定期分析和优化索引**

```sql
-- 查看索引使用情况
SHOW INDEX FROM user;

-- 删除未使用的索引
DROP INDEX idx_unused ON user;
```

**5. 使用自增主键**

避免页分裂，提升插入性能。

---

## 实战案例：慢查询优化

**问题 SQL**

```sql
SELECT * FROM order
WHERE user_id = 123 AND status = 1 AND create_time > '2024-01-01'
ORDER BY create_time DESC
LIMIT 10;
```

执行时间：3 秒

**分析**

```sql
EXPLAIN SELECT * FROM order
WHERE user_id = 123 AND status = 1 AND create_time > '2024-01-01'
ORDER BY create_time DESC
LIMIT 10;
```

结果：type = ALL，rows = 1000000，Extra = Using filesort

**优化步骤**

1. 创建联合索引

```sql
CREATE INDEX idx_user_status_time ON order(user_id, status, create_time);
```

2. 再次分析

```sql
EXPLAIN SELECT * FROM order
WHERE user_id = 123 AND status = 1 AND create_time > '2024-01-01'
ORDER BY create_time DESC
LIMIT 10;
```

结果：type = range，rows = 100，Extra = Using index condition

执行时间：50ms

性能提升：60 倍

---

## 总结与思考

MySQL 索引优化的核心要点：

- 理解 B+ 树的原理，知道为什么索引能提升性能
- 掌握 Explain 的使用，能够分析执行计划
- 避免索引失效的 10 种场景
- 合理设计联合索引，利用最左前缀匹配
- 使用覆盖索引和索引下推优化查询

索引不是越多越好，每个索引都会占用存储空间，并且影响写入性能。要根据业务场景合理设计索引。

下次当你的 SQL 查询很慢时，不妨用 Explain 分析一下，看看是否走了索引。
