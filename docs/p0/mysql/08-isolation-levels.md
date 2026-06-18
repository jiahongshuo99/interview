# 隔离级别

## 面试定位

隔离级别是事务题的核心。面试官通常会问：

- 四种隔离级别分别解决什么问题。
- 脏读、不可重复读、幻读分别是什么。
- MySQL 默认隔离级别是什么。
- InnoDB 可重复读为什么还能避免很多幻读。
- 快照读和当前读在隔离级别下有什么区别。
- `READ COMMITTED` 和 `REPEATABLE READ` 的 ReadView 时机有什么不同。

回答主线：隔离级别定义并发事务之间可见性的强弱。InnoDB 通过 MVCC 支持快照读，通过锁和 next-key lock 支持当前读。讨论 MySQL 隔离级别时必须区分快照读和当前读。

## 并发异常

### 脏读

读到了其他事务未提交的数据。

```text
T1 update balance = 0, not commit
T2 read balance = 0
T1 rollback
```

T2 读到的数据从未真正提交。

### 不可重复读

同一事务内两次读取同一行，结果不同，因为另一个事务提交了修改。

```text
T1 read balance = 100
T2 update balance = 200, commit
T1 read balance = 200
```

### 幻读

同一事务内按同一条件两次查询，第二次看到了第一次没有的新增或删除行。

```text
T1 select count(*) where age > 18 = 10
T2 insert age = 20, commit
T1 select count(*) where age > 18 = 11
```

幻读关注的是符合条件的记录集合变化。

## 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 特点 |
| --- | --- | --- | --- | --- |
| Read Uncommitted | 可能 | 可能 | 可能 | 基本不用 |
| Read Committed | 避免 | 可能 | 可能 | Oracle 常见默认，MySQL 可选 |
| Repeatable Read | 避免 | 避免 | InnoDB 当前读配合锁可避免很多幻读 | MySQL InnoDB 默认 |
| Serializable | 避免 | 避免 | 避免 | 隔离最强，并发最低 |

查看隔离级别：

```sql
SELECT @@transaction_isolation;
```

设置当前会话：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

设置下一个事务：

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
```

## Read Uncommitted

读未提交允许读到其他事务未提交的数据。

问题：

- 脏读。
- 结果不可依赖。
- 工程中很少使用。

示例：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

如果 T1 修改未提交，T2 可能读到这个未提交版本。

## Read Committed

读已提交只能读到其他事务已经提交的数据。

特点：

- 避免脏读。
- 可能不可重复读。
- 每条快照读语句生成新的 ReadView。

示例：

```text
T1 begin
T1 select balance -> 100
T2 update balance = 200, commit
T1 select balance -> 200
```

在 RC 下，T1 第二次快照读会看到 T2 已提交结果。

## Repeatable Read

可重复读是 MySQL InnoDB 默认隔离级别。

特点：

- 避免脏读。
- 对快照读避免不可重复读。
- 同一事务内快照读通常复用第一次快照读创建的 ReadView。
- 当前读通过锁读取最新已提交数据，并可能加 next-key lock 避免幻读。

示例：

```text
T1 begin
T1 select balance -> 100
T2 update balance = 200, commit
T1 select balance -> 100
```

T1 普通 `SELECT` 仍看到事务开始后第一次快照读对应的版本。

注意：如果 T1 使用当前读：

```sql
SELECT *
FROM account
WHERE id = 1
FOR UPDATE;
```

它会读取最新已提交版本并加锁，不是读取旧快照。

## Serializable

串行化隔离级别最强。

特点：

- 事务像串行执行。
- 普通读也可能加锁。
- 并发能力低。
- 死锁和锁等待概率上升。

在高并发 Java 业务中很少全局使用，通常用更细粒度的锁、唯一约束、乐观锁或业务幂等解决一致性问题。

## 快照读和当前读

讨论 InnoDB 隔离级别必须区分两类读。

### 快照读

普通一致性读：

```sql
SELECT *
FROM t_order
WHERE id = 1;
```

特点：

- 基于 MVCC。
- 不加行锁。
- 读的是符合 ReadView 的历史版本。

### 当前读

读取最新已提交版本，并加锁。

```sql
SELECT *
FROM t_order
WHERE id = 1
FOR UPDATE;

SELECT *
FROM t_order
WHERE id = 1
LOCK IN SHARE MODE;

UPDATE t_order
SET status = 2
WHERE id = 1;

DELETE FROM t_order
WHERE id = 1;
```

特点：

- 读最新版本。
- 加锁。
- 在 RR 下范围当前读可能使用 next-key lock。

## ReadView 生成时机

### RC

每条快照读语句生成新的 ReadView。

```text
T1 begin
T1 select -> ReadView A
T2 commit
T1 select -> ReadView B
```

所以第二次能看到 T2 已提交结果。

### RR

同一事务内第一次快照读生成 ReadView，后续快照读复用。

```text
T1 begin
T1 select -> ReadView A
T2 commit
T1 select -> still ReadView A
```

所以可以重复读。

注意：如果事务开始后先执行当前读，再执行普通 `SELECT`，ReadView 创建时机仍要以第一次快照读为准。

## 幻读在 InnoDB 中怎么处理

对快照读：

- RR 下同一 ReadView，普通 `SELECT` 看不到其他事务后续插入的数据。
- 因此快照读层面不会出现传统意义的幻读。

对当前读：

- 范围查询需要阻止其他事务在范围内插入新记录。
- InnoDB 在 RR 下通过 next-key lock 锁住记录和间隙，避免当前读幻读。

示例：

```sql
-- T1
START TRANSACTION;
SELECT *
FROM t_order
WHERE user_id = 1001
FOR UPDATE;

-- T2 试图插入 user_id = 1001 的记录，可能被阻塞
INSERT INTO t_order(id, user_id, status, created_at)
VALUES(100, 1001, 1, NOW());
```

前提是查询条件能走合适索引。没有索引时，锁范围可能扩大。

## 隔离级别选择

MySQL 默认 RR，很多业务直接使用默认值。

选择 RC 的原因可能是：

- 降低 gap lock 带来的锁冲突。
- 更接近每条语句读最新已提交数据的直觉。
- 某些高并发写入场景减少锁等待。

选择 RR 的原因：

- 默认行为稳定。
- 快照读可重复。
- 配合 next-key lock 防止范围当前读幻读。

工程上不要只靠隔离级别兜底，还要结合：

- 唯一约束。
- 乐观锁版本号。
- 幂等键。
- 正确索引。
- 明确加锁读。

## SQL 示例

查看当前隔离级别：

```sql
SELECT @@global.transaction_isolation;
SELECT @@session.transaction_isolation;
```

设置事务隔离级别：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT * FROM t_order WHERE id = 1;
COMMIT;
```

当前读：

```sql
START TRANSACTION;
SELECT *
FROM t_order
WHERE id = 1
FOR UPDATE;
COMMIT;
```

共享当前读：

```sql
SELECT *
FROM t_order
WHERE id = 1
LOCK IN SHARE MODE;
```

MySQL 8.0 也支持：

```sql
SELECT *
FROM t_order
WHERE id = 1
FOR SHARE;
```

## 常见追问

### MySQL RR 能完全避免幻读吗

要分场景。普通快照读在同一事务内看同一个 ReadView，不会看到后续插入。当前读在 RR 下通过 next-key lock 防止范围内插入。没有合适索引时锁范围可能扩大，但不是靠“凭空记住查询结果”实现。

### RC 下会有 gap lock 吗

RC 下通常减少 gap lock 使用，但外键检查、唯一性检查等场景仍可能出现间隙相关锁。不要简单说 RC 完全没有 gap lock。

### RR 下为什么当前读能读到其他事务已提交的新值

当前读不走旧 ReadView，它读取最新已提交版本并加锁。`UPDATE`、`DELETE`、`SELECT FOR UPDATE` 都属于当前读。

### 隔离级别越高越好吗

不是。隔离越强，并发越低，锁等待和死锁风险越高。工程上要在一致性和吞吐之间权衡。

## 易错点

- 不区分快照读和当前读，导致解释 RR 时前后矛盾。
- 误以为 RR 下任何读都永远读旧版本。当前读不是。
- 误以为 RC 完全没有间隙锁。
- 误以为 Serializable 是解决并发问题的常用方案。
- 误以为隔离级别可以替代唯一约束和幂等设计。

## 自检清单

- 能否定义脏读、不可重复读、幻读。
- 能否列出四种隔离级别的差异。
- 能否说明 MySQL InnoDB 默认隔离级别。
- 能否解释 RC 和 RR 下 ReadView 生成时机。
- 能否区分快照读和当前读。
- 能否解释 RR 下 next-key lock 如何防当前读幻读。
