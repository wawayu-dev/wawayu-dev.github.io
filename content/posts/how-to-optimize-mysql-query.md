---
title: "MySQL 慢查询优化实战案例"
date: 2024-12-10
tags: ["MySQL", "性能优化", "SQL调优", "问题排查"]
categories: ["技术"]
description: "从 5 秒到 50ms 的优化之路，真实案例讲解 SQL 性能调优方法"
draft: false
---

一条慢查询可能拖垮整个系统。本文通过真实的生产案例，讲解如何定位和优化慢查询，让你掌握 SQL 性能调优的完整方法论。

### 本文亮点
- [x] 掌握慢查询日志的分析方法
- [x] 学会使用 Explain 定位性能瓶颈
- [x] 了解常见的 SQL 优化技巧
- [x] 理解数据库层面的性能优化

---

## 案例背景

某电商系统的订单列表查询接口响应时间达到 5 秒，用户体验极差。

**原始 SQL**

```sql
SELECT o.*, u.username, u.phone
FROM `order` o
LEFT JOIN user u ON o.user_id = u.id
WHERE o.status IN (1, 2, 3)
  AND o.create_time >= '2024-01-01'
  AND u.city = 'Beijing'
ORDER BY o.create_time DESC
LIMIT 20;
```

数据量：

- order 表：100 万条
- user 表：50 万条

---

## 第一步：开启慢查询日志

**配置 MySQL**

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
```

**分析慢查询日志**

```bash
# 使用 mysqldumpslow 分析
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
```

输出：

```
Count: 1000  Time=5.23s (5230s)  Lock=0.00s (0s)  Rows=20.0 (20000)
SELECT o.*, u.username, u.phone FROM order o LEFT JOIN user u ...
```

---

## 第二步：Explain 分析执行计划

```sql
EXPLAIN SELECT o.*, u.username, u.phone
FROM `order` o
LEFT JOIN user u ON o.user_id = u.id
WHERE o.status IN (1, 2, 3)
  AND o.create_time >= '2024-01-01'
  AND u.city = 'Beijing'
ORDER BY o.create_time DESC
LIMIT 20;
```


**执行计划分析**

| id | table | type | key | rows | Extra |
|----|-------|------|-----|------|-------|
| 1 | o | ALL | NULL | 1000000 | Using where; Using filesort |
| 1 | u | eq_ref | PRIMARY | 1 | Using where |

问题：

1. order 表全表扫描（type = ALL）
2. 扫描了 100 万行数据
3. 需要额外的排序操作（Using filesort）

---

## 第三步：优化索引

**分析查询条件**

- WHERE：status、create_time
- JOIN：user_id
- ORDER BY：create_time

**创建联合索引**

```sql
CREATE INDEX idx_status_time ON `order`(status, create_time);
```

**再次 Explain**

| id | table | type | key | rows | Extra |
|----|-------|------|-----|------|-------|
| 1 | o | range | idx_status_time | 50000 | Using index condition |
| 1 | u | eq_ref | PRIMARY | 1 | Using where |

改进：

- type 从 ALL 变成 range
- rows 从 100 万降低到 5 万
- 去掉了 Using filesort

执行时间：从 5 秒降低到 800ms

---

## 第四步：优化 JOIN

**问题分析**

LEFT JOIN 会先查询 order 表的所有数据，然后再关联 user 表。但实际上我们只需要 city = 'Beijing' 的用户。

**优化方案：改为 INNER JOIN**

```sql
SELECT o.*, u.username, u.phone
FROM `order` o
INNER JOIN user u ON o.user_id = u.id
WHERE o.status IN (1, 2, 3)
  AND o.create_time >= '2024-01-01'
  AND u.city = 'Beijing'
ORDER BY o.create_time DESC
LIMIT 20;
```

**为 user 表创建索引**

```sql
CREATE INDEX idx_city ON user(city);
```

**再次 Explain**

| id | table | type | key | rows | Extra |
|----|-------|------|-----|------|-------|
| 1 | u | ref | idx_city | 10000 | Using where |
| 1 | o | ref | idx_user_status_time | 100 | Using where |

执行时间：从 800ms 降低到 200ms

---

## 第五步：避免 SELECT *

**问题**

SELECT * 会查询所有字段，包括不需要的大字段（如订单详情 JSON）。

**优化**

```sql
SELECT o.id, o.order_no, o.amount, o.status, o.create_time,
       u.username, u.phone
FROM `order` o
INNER JOIN user u ON o.user_id = u.id
WHERE o.status IN (1, 2, 3)
  AND o.create_time >= '2024-01-01'
  AND u.city = 'Beijing'
ORDER BY o.create_time DESC
LIMIT 20;
```

执行时间：从 200ms 降低到 120ms

---

## 第六步：使用覆盖索引

**创建覆盖索引**

```sql
CREATE INDEX idx_status_time_user ON `order`(status, create_time, user_id);
```

**优化后的 Explain**

| id | table | type | key | rows | Extra |
|----|-------|------|-----|------|-------|
| 1 | u | ref | idx_city | 10000 | Using index |
| 1 | o | ref | idx_status_time_user | 100 | Using index |

Extra 变成了 Using index，说明使用了覆盖索引，不需要回表。

执行时间：从 120ms 降低到 50ms

---

## 优化总结

| 优化步骤 | 执行时间 | 优化效果 |
|---------|---------|---------|
| 原始 SQL | 5000ms | - |
| 创建索引 | 800ms | 提升 6.25 倍 |
| 优化 JOIN | 200ms | 提升 4 倍 |
| 避免 SELECT * | 120ms | 提升 1.67 倍 |
| 使用覆盖索引 | 50ms | 提升 2.4 倍 |

总体提升：100 倍

---

## 常见的 SQL 优化技巧

**1. 避免在 WHERE 中使用函数**

```sql
-- 不好
SELECT * FROM `order` WHERE DATE(create_time) = '2024-01-01';

-- 好
SELECT * FROM `order` WHERE create_time >= '2024-01-01' AND create_time < '2024-01-02';
```

**2. 使用 UNION ALL 代替 UNION**

UNION 会去重，需要额外的排序操作。如果确定没有重复数据，使用 UNION ALL。

```sql
-- 不好
SELECT * FROM order WHERE status = 1
UNION
SELECT * FROM order WHERE status = 2;

-- 好
SELECT * FROM order WHERE status = 1
UNION ALL
SELECT * FROM order WHERE status = 2;
```

**3. 使用 EXISTS 代替 IN**

当子查询结果集很大时，EXISTS 性能更好。

```sql
-- 不好
SELECT * FROM user WHERE id IN (SELECT user_id FROM order);

-- 好
SELECT * FROM user u WHERE EXISTS (SELECT 1 FROM order o WHERE o.user_id = u.id);
```

**4. 避免使用子查询**

子查询会创建临时表，性能较差。可以改为 JOIN。

```sql
-- 不好
SELECT * FROM user WHERE id IN (SELECT user_id FROM order WHERE status = 1);

-- 好
SELECT DISTINCT u.* FROM user u
INNER JOIN order o ON u.id = o.user_id
WHERE o.status = 1;
```

**5. 分批处理大数据量**

```sql
-- 不好：一次更新 100 万条数据
UPDATE order SET status = 2 WHERE status = 1;

-- 好：分批更新
UPDATE order SET status = 2 WHERE status = 1 LIMIT 1000;
```

**6. 使用 LIMIT 限制返回数据量**

```sql
-- 不好
SELECT * FROM order WHERE status = 1;

-- 好
SELECT * FROM order WHERE status = 1 LIMIT 100;
```

---

## 数据库层面的优化

**1. 调整 InnoDB 缓冲池大小**

```ini
[mysqld]
innodb_buffer_pool_size = 8G  # 设置为物理内存的 70-80%
```

**2. 开启查询缓存（MySQL 5.7 及以下）**

```ini
[mysqld]
query_cache_type = 1
query_cache_size = 256M
```

注意：MySQL 8.0 已移除查询缓存。

**3. 调整连接数**

```ini
[mysqld]
max_connections = 1000
```

**4. 开启慢查询日志**

```ini
[mysqld]
slow_query_log = 1
long_query_time = 1
```

**5. 定期分析表**

```sql
ANALYZE TABLE order;
```

更新表的统计信息，帮助优化器选择更好的执行计划。

---

## 监控与预警

**1. 使用 Prometheus + Grafana 监控**

监控指标：

- QPS（每秒查询数）
- 慢查询数量
- 连接数
- 缓冲池命中率

**2. 设置慢查询告警**

当慢查询数量超过阈值时，发送告警通知。

**3. 定期审查慢查询日志**

每周分析慢查询日志，找出需要优化的 SQL。

---

## 总结与思考

SQL 性能优化的方法论：

1. 开启慢查询日志，定位问题 SQL
2. 使用 Explain 分析执行计划
3. 创建合适的索引
4. 优化 SQL 语句（避免 SELECT *、优化 JOIN）
5. 数据库层面优化（调整参数、监控）

性能优化是一个持续的过程，需要根据业务变化不断调整。

记住：过早优化是万恶之源。先保证功能正确，再根据监控数据优化性能。

下次当你的 SQL 查询很慢时，按照这个方法论一步步排查，一定能找到问题所在。
