# 行锁、间隙锁、临键锁

## 面试定位

InnoDB 锁是 MySQL 面试中最容易暴露理解深度的部分。常见问题：

- InnoDB 行锁锁的是什么。
- 记录锁、间隙锁、临键锁有什么区别。
- 为什么范围查询会锁住不存在的记录。
- 什么情况下会锁一行，什么情况下会锁一段范围。
- 索引缺失为什么可能导致锁范围扩大。
- 插入意向锁和间隙锁是什么关系。

回答主线：InnoDB 行锁基于索引实现，锁的不是抽象的 Java 对象或整行内存，而是索引记录、记录之间的间隙或二者组合。RR 下当前读为了避免幻读，会对范围加 next-key lock。

## 锁分类总览

常见锁：

- 表级意向锁：IS、IX。
- 记录锁：record lock。
- 间隙锁：gap lock。
- 临键锁：next-key lock。
- 插入意向锁：insert intention lock。
- 自增锁：auto-inc lock。

本篇重点是行锁相关的三类：

```text
record lock: 锁索引记录本身
gap lock: 锁两个索引记录之间的间隙
next-key lock: record lock + gap lock
```

## InnoDB 行锁锁的是索引

InnoDB 通过索引定位记录，所以行锁加在索引上。

示例：

```sql
CREATE TABLE t_user (
    id BIGINT PRIMARY KEY,
    age INT NOT NULL,
    name VARCHAR(64),
    KEY idx_age (age)
) ENGINE = InnoDB;
```

当前读：

```sql
START TRANSACTION;
SELECT *
FROM t_user
WHERE id = 10
FOR UPDATE;
```

如果 `id = 10` 存在，通常锁主键索引上这条记录。

如果通过二级索引更新：

```sql
UPDATE t_user
SET name = 'a'
WHERE age = 20;
```

可能涉及：

- 二级索引 `idx_age` 上的锁。
- 回表后聚簇索引记录上的锁。

## 记录锁

记录锁锁住已存在的索引记录。

示例：

```sql
START TRANSACTION;
SELECT *
FROM t_user
WHERE id = 10
FOR UPDATE;
```

另一个事务：

```sql
UPDATE t_user
SET name = 'b'
WHERE id = 10;
```

会被阻塞，直到第一个事务提交或回滚。

记录锁用于防止其他事务修改或删除同一条记录。

## 间隙锁

间隙锁锁住索引记录之间的间隙，不锁具体记录。

假设 `age` 索引中有：

```text
10, 20, 30
```

间隙包括：

```text
(-∞, 10)
(10, 20)
(20, 30)
(30, +∞)
```

如果事务锁住 `(10, 20)` 这个 gap，其他事务不能在这个 gap 中插入 `age = 15` 的记录。

间隙锁主要用于防止幻读。

## 临键锁 next-key lock

临键锁是记录锁加前面的间隙锁。

如果索引值是：

```text
10, 20, 30
```

next-key lock 范围可以理解为：

```text
(-∞, 10]
(10, 20]
(20, 30]
(30, +∞)
```

它既锁记录，也锁记录前的间隙。

RR 下范围当前读常用 next-key lock：

```sql
START TRANSACTION;
SELECT *
FROM t_user
WHERE age BETWEEN 10 AND 20
FOR UPDATE;
```

这样可以防止其他事务插入新的 `age = 15` 记录造成幻读。

## 插入意向锁

插入意向锁是插入前对目标 gap 加的一种意向锁。

它表示：我要在这个间隙里插入一条记录。

特点：

- 插入意向锁之间通常兼容，只要插入位置不同。
- 插入意向锁和已有 gap lock、next-key lock 会冲突。
- 它不是表级意向锁。

示例：

```text
索引已有 10 和 20
T1 持有 gap lock (10, 20)
T2 INSERT age = 15，需要插入意向锁，被 T1 阻塞
```

## 唯一索引等值查询

如果通过唯一索引等值查询命中一条存在记录，通常只需要记录锁。

```sql
SELECT *
FROM t_user
WHERE id = 10
FOR UPDATE;
```

因为唯一索引能精确定位，不需要锁范围防止同条件插入另一条记录。

如果查询的唯一键不存在，可能锁住应该插入位置的间隙，防止其他事务插入同一键。

```sql
SELECT *
FROM t_user
WHERE id = 15
FOR UPDATE;
```

如果 `15` 不存在，可能锁住 `15` 所在的 gap。

## 普通索引等值查询

普通索引不唯一，即使是等值，也可能锁住范围。

假设 `age = 20` 有多条记录。

```sql
SELECT *
FROM t_user
WHERE age = 20
FOR UPDATE;
```

InnoDB 需要锁住所有 `age = 20` 的记录，并防止新的 `age = 20` 插入，通常会涉及 next-key lock 和 gap lock。

## 无索引条件

如果条件没有合适索引：

```sql
UPDATE t_user
SET name = 'x'
WHERE name = 'Tom';
```

`name` 没有索引时，InnoDB 可能扫描大量记录并对扫描到的记录加锁。RR 下当前读还可能造成更大范围的锁冲突。

工程建议：更新、删除、加锁查询必须确保条件命中合适索引。

## RC 和 RR 下的差异

在 RR 下：

- 当前读范围查询常使用 next-key lock 防止幻读。
- gap lock 使用更明显。

在 RC 下：

- 通常减少 gap lock。
- 但唯一性检查、外键检查等仍可能使用 gap 相关锁。

不要简单说“RC 没有 gap lock”。

## SQL 演示

准备表：

```sql
CREATE TABLE t_lock_demo (
    id BIGINT PRIMARY KEY,
    age INT NOT NULL,
    name VARCHAR(64),
    KEY idx_age (age)
) ENGINE = InnoDB;

INSERT INTO t_lock_demo(id, age, name)
VALUES (1, 10, 'a'), (2, 20, 'b'), (3, 30, 'c');
```

事务 1：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT *
FROM t_lock_demo
WHERE age BETWEEN 10 AND 20
FOR UPDATE;
```

事务 2：

```sql
INSERT INTO t_lock_demo(id, age, name)
VALUES (4, 15, 'd');
```

事务 2 可能被阻塞，因为 `age = 15` 落在被锁住的范围内。

查看锁：

```sql
SELECT *
FROM performance_schema.data_locks\G

SELECT *
FROM performance_schema.data_lock_waits\G
```

## 排查与优化

### 查长事务

```sql
SELECT *
FROM information_schema.innodb_trx\G
```

关注：

- 事务开始时间。
- 事务状态。
- 当前 SQL。
- 持锁时间。

### 查锁等待

MySQL 8.0：

```sql
SELECT *
FROM performance_schema.data_locks;

SELECT *
FROM performance_schema.data_lock_waits;
```

老版本或概览：

```sql
SHOW ENGINE INNODB STATUS\G
```

### 降低锁冲突

常见措施：

- 更新、删除条件加索引。
- 缩短事务时间。
- 固定多表、多行访问顺序。
- 批量操作分批提交。
- 避免事务内远程调用。
- 只锁必要范围。
- 用唯一约束、乐观锁、幂等键减少范围锁依赖。

## 常见追问

### 行锁锁的是整行吗

更准确地说，InnoDB 行锁锁的是索引记录。通过不同索引访问时，可能在二级索引和聚簇索引上都加锁。

### gap lock 锁住的记录能更新吗

gap lock 锁的是间隙，不是已有记录本身。它主要阻止插入。已有记录能否更新取决于是否有记录锁或 next-key lock。

### next-key lock 为什么能防幻读

因为它锁住记录和记录前间隙，其他事务不能在范围内插入新记录，当前读再次执行时不会出现新的符合条件记录。

### 没有索引会锁表吗

不是直接变成表锁，但因为需要扫描大量索引记录，可能对大量记录加锁，效果上接近锁住大范围甚至全表。

## 易错点

- 误以为 InnoDB 行锁和索引无关。
- 误以为 gap lock 锁的是已有记录。
- 误以为 `WHERE id = ?` 不存在时不会加锁。
- 误以为 RC 完全没有 gap lock。
- 误以为普通 `SELECT` 也会加这些行锁。普通快照读通常不加行锁。
- 忽略二级索引更新可能同时锁二级索引和聚簇索引。

## 自检清单

- 能否区分 record lock、gap lock、next-key lock。
- 能否说明 InnoDB 行锁为什么基于索引。
- 能否解释唯一索引等值命中和普通索引范围查询的锁差异。
- 能否说明插入意向锁触发时机。
- 能否用两个 session 演示范围锁阻塞插入。
- 能否通过 `performance_schema.data_locks` 排查锁等待。
