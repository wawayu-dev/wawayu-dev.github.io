---
title: "MySQL 事务隔离级别与 MVCC 机制深度解析"
date: 2024-11-28
tags: ["MySQL", "数据库", "并发编程", "源码分析"]
categories: ["技术"]
description: "深入理解 MySQL 的并发控制机制，掌握 MVCC 的 ReadView 原理"
draft: false
---

在高并发场景下，多个事务同时访问数据库，如何保证数据的一致性？MySQL 通过事务隔离级别和 MVCC 机制解决了这个问题。本文深入剖析 MySQL 的并发控制原理。

### 本文亮点
- [x] 理解四种事务隔离级别的区别与适用场景
- [x] 掌握 MVCC 的 ReadView 机制
- [x] 学会解决幻读问题的方法
- [x] 了解锁机制与 MVCC 的配合

---

## 事务的并发问题

**脏读（Dirty Read）**

事务 A 读取了事务 B 未提交的数据，事务 B 回滚后，事务 A 读到的数据是无效的。

```
时间线：
T1: 事务 A 开始
T2: 事务 B 开始，将 balance 从 100 改为 200
T3: 事务 A 读取 balance = 200（脏读）
T4: 事务 B 回滚，balance 恢复为 100
T5: 事务 A 提交
```

**不可重复读（Non-Repeatable Read）**

事务 A 两次读取同一数据，结果不一致（因为事务 B 修改并提交了数据）。

```
时间线：
T1: 事务 A 读取 balance = 100
T2: 事务 B 将 balance 改为 200 并提交
T3: 事务 A 再次读取 balance = 200（不可重复读）
```

**幻读（Phantom Read）**

事务 A 两次查询，第二次查询多出了新的行（因为事务 B 插入了新数据）。

```
时间线：
T1: 事务 A 查询 age > 20 的用户，结果 2 条
T2: 事务 B 插入一条 age = 25 的用户并提交
T3: 事务 A 再次查询 age > 20 的用户，结果 3 条（幻读）
```

---

## 四种事务隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---------|------|-----------|------|
| READ UNCOMMITTED | 可能 | 可能 | 可能 |
| READ COMMITTED | 不可能 | 可能 | 可能 |
| REPEATABLE READ（默认） | 不可能 | 不可能 | 可能 |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 |

**查看当前隔离级别**

```sql
SELECT @@transaction_isolation;
```

**设置隔离级别**

```sql
-- 会话级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 全局级别
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

---

## READ UNCOMMITTED：读未提交

**特点：** 事务可以读取其他事务未提交的数据。

**问题：** 会出现脏读、不可重复读、幻读。

**适用场景：** 几乎不使用，数据一致性无法保证。

**演示脏读**

```sql
-- 事务 A
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT balance FROM account WHERE id = 1;  -- 100

-- 事务 B
START TRANSACTION;
UPDATE account SET balance = 200 WHERE id = 1;

-- 事务 A
SELECT balance FROM account WHERE id = 1;  -- 200（脏读）

-- 事务 B
ROLLBACK;

-- 事务 A
SELECT balance FROM account WHERE id = 1;  -- 100
COMMIT;
```

---

## READ COMMITTED：读已提交

**特点：** 事务只能读取其他事务已提交的数据。

**问题：** 会出现不可重复读、幻读。

**适用场景：** Oracle、PostgreSQL 的默认隔离级别。

**演示不可重复读**

```sql
-- 事务 A
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT balance FROM account WHERE id = 1;  -- 100

-- 事务 B
START TRANSACTION;
UPDATE account SET balance = 200 WHERE id = 1;
COMMIT;

-- 事务 A
SELECT balance FROM account WHERE id = 1;  -- 200（不可重复读）
COMMIT;
```

---

## REPEATABLE READ：可重复读

**特点：** 事务内多次读取同一数据，结果一致。

**问题：** 理论上会出现幻读，但 MySQL 通过 MVCC + Next-Key Lock 解决了大部分幻读问题。

**适用场景：** MySQL 的默认隔离级别，适合大多数场景。

**演示可重复读**

```sql
-- 事务 A
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM account WHERE id = 1;  -- 100

-- 事务 B
START TRANSACTION;
UPDATE account SET balance = 200 WHERE id = 1;
COMMIT;

-- 事务 A
SELECT balance FROM account WHERE id = 1;  -- 100（可重复读）
COMMIT;
```

---

## SERIALIZABLE：串行化

**特点：** 事务串行执行，完全避免并发问题。

**问题：** 性能最差，并发度最低。

**适用场景：** 对数据一致性要求极高的场景（如金融系统）。

**实现方式：** 对读取的每一行数据都加锁。

```sql
-- 事务 A
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
SELECT * FROM account WHERE id = 1;  -- 加共享锁

-- 事务 B
START TRANSACTION;
UPDATE account SET balance = 200 WHERE id = 1;  -- 等待事务 A 释放锁
```

---

## MVCC：多版本并发控制

**核心思想：** 为每个事务提供数据的快照，不同事务看到不同版本的数据。

**优势：**
- 读不加锁，写不阻塞读
- 提升并发性能

**实现机制：**
1. 隐藏字段：DB_TRX_ID、DB_ROLL_PTR、DB_ROW_ID
2. Undo Log：记录数据的历史版本
3. ReadView：判断数据版本的可见性

---

## 隐藏字段

InnoDB 为每一行数据添加了三个隐藏字段：

| 字段 | 说明 |
|------|------|
| DB_TRX_ID | 最后修改该行的事务 ID |
| DB_ROLL_PTR | 指向 Undo Log 的指针 |
| DB_ROW_ID | 行 ID（如果没有主键） |

**示例**

```
原始数据：
id | name  | age | DB_TRX_ID | DB_ROLL_PTR
1  | Alice | 20  | 100       | NULL

事务 101 修改：
id | name  | age | DB_TRX_ID | DB_ROLL_PTR
1  | Alice | 25  | 101       | → Undo Log (age=20, TRX_ID=100)

事务 102 修改：
id | name  | age | DB_TRX_ID | DB_ROLL_PTR
1  | Alice | 30  | 102       | → Undo Log (age=25, TRX_ID=101)
```

通过 Undo Log，可以构建版本链，回溯到任意历史版本。

---

## ReadView：可见性判断

ReadView 是事务开始时创建的快照，包含以下信息：

| 字段 | 说明 |
|------|------|
| m_ids | 当前活跃的事务 ID 列表 |
| min_trx_id | 最小的活跃事务 ID |
| max_trx_id | 下一个要分配的事务 ID |
| creator_trx_id | 创建 ReadView 的事务 ID |

**可见性规则**

对于数据行的 DB_TRX_ID：

1. 如果 `DB_TRX_ID < min_trx_id`，说明该版本在 ReadView 创建前已提交，可见
2. 如果 `DB_TRX_ID >= max_trx_id`，说明该版本在 ReadView 创建后才开始，不可见
3. 如果 `min_trx_id <= DB_TRX_ID < max_trx_id`：
   - 如果 `DB_TRX_ID` 在 `m_ids` 中，说明该事务还未提交，不可见
   - 如果 `DB_TRX_ID` 不在 `m_ids` 中，说明该事务已提交，可见
4. 如果 `DB_TRX_ID == creator_trx_id`，说明是当前事务自己修改的，可见

如果当前版本不可见，通过 DB_ROLL_PTR 找到 Undo Log 中的上一个版本，继续判断。

---

## READ COMMITTED 与 REPEATABLE READ 的区别

**READ COMMITTED**

每次 SELECT 都创建新的 ReadView。

```
时间线：
T1: 事务 A 开始（TRX_ID=100）
T2: 事务 A 第一次 SELECT，创建 ReadView1（m_ids=[100]）
T3: 事务 B（TRX_ID=101）修改数据并提交
T4: 事务 A 第二次 SELECT，创建 ReadView2（m_ids=[100]）
    此时事务 101 已提交，不在 m_ids 中，可见
```

结果：两次 SELECT 结果不同（不可重复读）。

**REPEATABLE READ**

事务开始时创建 ReadView，整个事务期间复用。

```
时间线：
T1: 事务 A 开始（TRX_ID=100），创建 ReadView（m_ids=[100]）
T2: 事务 A 第一次 SELECT
T3: 事务 B（TRX_ID=101）修改数据并提交
T4: 事务 A 第二次 SELECT，复用 ReadView（m_ids=[100]）
    事务 101 在 ReadView 创建后才提交，不可见
```

结果：两次 SELECT 结果相同（可重复读）。

> **架构思考：** MVCC 的设计体现了"空间换时间"的思想。通过 Undo Log 保存历史版本，牺牲了存储空间，换取了读写并发的性能提升。这种思想在很多系统中都有应用，比如 Git 的版本控制、Redis 的 RDB 快照。

---

## 幻读问题

**什么是幻读？**

事务 A 两次查询，第二次查询多出了新的行。

**MVCC 能解决幻读吗？**

部分解决。对于快照读（普通 SELECT），MVCC 可以解决幻读。但对于当前读（SELECT FOR UPDATE、UPDATE、DELETE），MVCC 无法解决幻读。

**演示幻读**

```sql
-- 事务 A
START TRANSACTION;
SELECT * FROM user WHERE age > 20;  -- 2 条记录

-- 事务 B
START TRANSACTION;
INSERT INTO user (name, age) VALUES ('Charlie', 25);
COMMIT;

-- 事务 A
SELECT * FROM user WHERE age > 20;  -- 仍然 2 条（MVCC 解决了快照读的幻读）

-- 但是
SELECT * FROM user WHERE age > 20 FOR UPDATE;  -- 3 条（当前读出现幻读）
```

**解决方案：Next-Key Lock**

Next-Key Lock = Record Lock（行锁） + Gap Lock（间隙锁）

```sql
-- 事务 A
START TRANSACTION;
SELECT * FROM user WHERE age > 20 FOR UPDATE;  -- 锁定 age > 20 的所有行和间隙

-- 事务 B
INSERT INTO user (name, age) VALUES ('Charlie', 25);  -- 等待（被间隙锁阻塞）
```

间隙锁锁定的是索引记录之间的间隙，防止其他事务插入数据。

---

## 锁机制

**共享锁（S Lock）**

读锁，多个事务可以同时持有。

```sql
SELECT * FROM user WHERE id = 1 LOCK IN SHARE MODE;
```

**排他锁（X Lock）**

写锁，只有一个事务可以持有。

```sql
SELECT * FROM user WHERE id = 1 FOR UPDATE;
UPDATE user SET age = 25 WHERE id = 1;
DELETE FROM user WHERE id = 1;
```

**行锁（Record Lock）**

锁定单个索引记录。

**间隙锁（Gap Lock）**

锁定索引记录之间的间隙，防止插入。

**Next-Key Lock**

行锁 + 间隙锁，锁定索引记录及其前面的间隙。

**锁的兼容性**

|  | S Lock | X Lock |
|--|--------|--------|
| S Lock | 兼容 | 冲突 |
| X Lock | 冲突 | 冲突 |

---

## 死锁

**场景**

```sql
-- 事务 A
START TRANSACTION;
UPDATE user SET age = 25 WHERE id = 1;  -- 锁定 id=1

-- 事务 B
START TRANSACTION;
UPDATE user SET age = 30 WHERE id = 2;  -- 锁定 id=2

-- 事务 A
UPDATE user SET age = 26 WHERE id = 2;  -- 等待事务 B 释放锁

-- 事务 B
UPDATE user SET age = 31 WHERE id = 1;  -- 等待事务 A 释放锁（死锁）
```

**检测与解决**

MySQL 会自动检测死锁，并回滚其中一个事务。

**避免死锁**

1. 按相同顺序访问资源
2. 缩短事务时间
3. 降低隔离级别
4. 使用乐观锁

---

## 实战案例：秒杀系统的库存扣减

**方案一：悲观锁**

```sql
START TRANSACTION;
SELECT stock FROM product WHERE id = 1 FOR UPDATE;  -- 加排他锁
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

优点：数据一致性强
缺点：并发性能差

**方案二：乐观锁**

```sql
-- 查询库存和版本号
SELECT stock, version FROM product WHERE id = 1;

-- 更新时检查版本号
UPDATE product SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = #{oldVersion};
```

优点：并发性能好
缺点：高并发下更新失败率高

**方案三：Redis + 数据库**

```java
// 1. Redis 预扣减
Long stock = redisTemplate.opsForValue().decrement("stock:1");
if (stock < 0) {
    redisTemplate.opsForValue().increment("stock:1");
    return "库存不足";
}

// 2. 异步更新数据库
rabbitTemplate.convertAndSend("order.exchange", "order.created", orderId);
```

优点：性能最好
缺点：实现复杂，需要保证最终一致性

---

## 总结与思考

MySQL 事务隔离级别与 MVCC 的核心要点：

- 四种隔离级别解决不同的并发问题
- MVCC 通过 ReadView 和 Undo Log 实现多版本并发控制
- READ COMMITTED 每次 SELECT 创建新 ReadView
- REPEATABLE READ 事务开始时创建 ReadView 并复用
- Next-Key Lock 解决当前读的幻读问题

理解这些机制，才能在实际开发中选择合适的隔离级别和锁策略。

下次当你遇到并发问题时，不妨想想：这是脏读、不可重复读还是幻读？应该用什么隔离级别和锁机制？
