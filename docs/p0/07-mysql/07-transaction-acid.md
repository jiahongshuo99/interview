# 事务 ACID

## 面试定位

事务 ACID 是数据库基础，但面试通常会追到底层实现：

- 原子性靠什么保证。
- 持久性是不是靠 binlog。
- 隔离性和锁、MVCC 有什么关系。
- 一致性到底是谁保证的。
- commit 时发生了什么。
- Java `@Transactional` 和数据库事务有什么边界。

回答主线：ACID 是事务的四个目标。InnoDB 主要通过 undo log 保证原子性，通过锁和 MVCC 保证隔离性，通过 redo log 和刷盘策略保证持久性；一致性是最终目标，由数据库约束、事务机制和业务代码共同保证。

## 事务是什么

事务是一组数据库操作的逻辑单元，要么整体成功，要么整体失败。

示例：

```sql
START TRANSACTION;

UPDATE account
SET balance = balance - 100
WHERE id = 1;

UPDATE account
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

如果中间失败：

```sql
ROLLBACK;
```

在 Java 中，Spring `@Transactional` 本质也是围绕数据库连接执行：

```text
获取连接
  -> setAutoCommit(false)
  -> 执行业务 SQL
  -> commit 或 rollback
  -> 释放连接
```

## Atomicity 原子性

原子性指事务内操作要么全部成功，要么全部失败。

InnoDB 主要靠 undo log 实现回滚。

更新一行时，InnoDB 会记录修改前的旧版本：

```text
update row
  -> 写 undo log，记录旧值
  -> 修改 Buffer Pool 中的数据页
  -> 提交或回滚
```

如果事务回滚，InnoDB 根据 undo log 把数据恢复到修改前状态。

示例：

```sql
START TRANSACTION;

UPDATE account
SET balance = balance - 100
WHERE id = 1;

-- 发现余额不足或后续失败
ROLLBACK;
```

回滚时使用 undo log 撤销已经执行的修改。

## Consistency 一致性

一致性指事务执行前后，数据库都满足约束和业务规则。

一致性不是单个日志直接保证的，而是多种机制共同达成：

- 数据库约束：主键、唯一键、外键、非空、检查约束。
- 事务原子性：失败时能回滚。
- 隔离性：并发事务不破坏正确性。
- 持久性：提交结果不丢。
- 业务代码：余额不能扣成负数、库存不能超卖等业务规则。

示例：

```sql
CREATE TABLE account (
    id BIGINT PRIMARY KEY,
    balance DECIMAL(10, 2) NOT NULL,
    CHECK (balance >= 0)
) ENGINE = InnoDB;
```

即使数据库有约束，复杂业务一致性仍通常需要业务代码和正确事务边界。

## Isolation 隔离性

隔离性指并发事务之间互不干扰到约定程度。

InnoDB 主要通过两类机制实现：

- 锁：行锁、间隙锁、临键锁、意向锁。
- MVCC：通过 undo log 和 ReadView 支持快照读。

隔离级别决定能避免哪些并发异常：

- 读未提交。
- 读已提交。
- 可重复读。
- 串行化。

普通 `SELECT` 多数是 MVCC 快照读：

```sql
SELECT *
FROM account
WHERE id = 1;
```

当前读会读取最新已提交版本并加锁：

```sql
SELECT *
FROM account
WHERE id = 1
FOR UPDATE;
```

## Durability 持久性

持久性指事务提交后，即使数据库崩溃，结果也不应该丢失。

InnoDB 主要依赖 redo log。

写入流程简化：

```text
修改数据页
  -> 生成 redo log
  -> commit 时按策略刷 redo
  -> 后台再把脏页刷回磁盘
```

redo log 是物理日志，记录数据页的修改，用于崩溃恢复。

关键配置：

```sql
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
SHOW VARIABLES LIKE 'sync_binlog';
```

`innodb_flush_log_at_trx_commit` 常见值：

- `1`：每次提交都写入并刷盘 redo，持久性最好，开销更高。
- `2`：每次提交写入 OS cache，每秒刷盘，崩溃可能丢最近数据。
- `0`：每秒写入和刷盘，性能更好，持久性更弱。

## commit 时发生什么

简化流程：

```text
事务执行 SQL
  -> 生成 undo log
  -> 修改 Buffer Pool 数据页
  -> 生成 redo log
  -> 写 binlog
  -> redo commit
  -> 返回提交成功
```

涉及主从复制时，还要考虑 binlog 和 redo log 的两阶段提交，保证崩溃恢复后 InnoDB 数据和 binlog 一致。

## autocommit

MySQL 默认：

```sql
SHOW VARIABLES LIKE 'autocommit';
```

`autocommit = 1` 时，每条 SQL 默认都是一个独立事务。

显式事务：

```sql
START TRANSACTION;
UPDATE ...
COMMIT;
```

或者：

```sql
SET autocommit = 0;
UPDATE ...
COMMIT;
```

Java 后端要注意连接池复用连接，事务属性必须在归还连接前正确恢复。

## 保存点

保存点允许事务内局部回滚。

```sql
START TRANSACTION;

UPDATE account SET balance = balance - 100 WHERE id = 1;

SAVEPOINT sp1;

UPDATE account SET balance = balance + 100 WHERE id = 2;

ROLLBACK TO SAVEPOINT sp1;

COMMIT;
```

Spring 的 `Propagation.NESTED` 在支持时通常基于数据库保存点。

## Java 后端事务注意点

### 事务边界

事务应该覆盖需要原子提交的数据库操作，但不要包太大。

不推荐：

```java
@Transactional
public void createOrder() {
    orderMapper.insert(order);
    remoteInventoryClient.lock(); // 远程调用
    payClient.pay();              // 远程调用
    orderMapper.update(order);
}
```

问题：

- 数据库连接占用时间长。
- 锁持有时间长。
- 远程超时会拖垮事务。
- 死锁和连接池耗尽风险增加。

### 异常回滚

Spring 默认只对运行时异常和错误回滚。受检异常需要配置：

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder() throws Exception {
}
```

### 自调用失效

同一个类内部方法调用不会经过 Spring 代理，事务注解可能不生效。

## 常见追问

### 原子性和持久性分别靠什么

原子性主要靠 undo log 回滚；持久性主要靠 redo log 崩溃恢复。binlog 主要用于复制和归档恢复，不是 InnoDB 崩溃恢复的核心日志。

### 一致性是数据库保证的吗

数据库保证基础约束和事务语义，但业务一致性需要业务代码、约束、锁、隔离级别、幂等和补偿机制共同保证。

### 普通查询会加锁吗

InnoDB 默认隔离级别下，普通 `SELECT` 通常是快照读，不加行锁。`SELECT ... FOR UPDATE`、`UPDATE`、`DELETE` 是当前读，会加锁。

### `COMMIT` 返回成功就一定不丢吗

取决于刷盘配置和硬件。`innodb_flush_log_at_trx_commit = 1` 且存储可靠时持久性最强。配置为 `0` 或 `2` 时，操作系统或机器崩溃可能丢最近事务。

## 易错点

- 误以为 ACID 是某个单一机制保证的。
- 误以为持久性靠 binlog。InnoDB 崩溃恢复主要靠 redo log。
- 误以为一致性完全由数据库自动保证。业务规则仍要业务代码保证。
- 误以为普通 `SELECT` 都会加锁。
- 误以为 Spring 注解存在就一定有事务，自调用、异常类型、传播行为都会影响。
- 误把长事务当成更安全，实际长事务会拖累锁、undo、主从延迟和连接池。

## 自检清单

- 能否分别说明 ACID 四个属性。
- 能否说出 undo、redo、锁、MVCC 分别支撑哪个属性。
- 能否解释 autocommit 和显式事务的区别。
- 能否说明保存点和嵌套事务的关系。
- 能否讲清楚 Java `@Transactional` 的边界和常见失效场景。
- 能否说明事务为什么不能包住耗时远程调用。
