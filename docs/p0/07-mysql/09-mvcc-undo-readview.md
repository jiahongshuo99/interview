# MVCC、undo log、ReadView

## 面试定位

MVCC 是 MySQL 事务面试的高频深水区。面试官常问：

- MVCC 解决什么问题。
- undo log 和 MVCC 是什么关系。
- ReadView 里有哪些字段。
- RC 和 RR 的 ReadView 生成时机有什么不同。
- 一行数据多个版本如何判断可见性。
- 长事务为什么会导致 undo 膨胀。

回答主线：MVCC 通过记录历史版本和可见性规则，让读写在很多场景下不互相阻塞。InnoDB 每行有事务 ID 和回滚指针，旧版本存在 undo log 版本链中；快照读通过 ReadView 判断哪个版本对当前事务可见。

## MVCC 是什么

MVCC，全称 Multi-Version Concurrency Control，多版本并发控制。

目标：

- 提高读写并发。
- 普通 `SELECT` 不阻塞写。
- 写不阻塞普通快照读。
- 支持 RC、RR 等隔离级别的一致性读。

示例：

```text
T1 正在更新某行但未提交
T2 普通 SELECT 仍可以读取旧版本
```

如果没有 MVCC，读写更容易互相阻塞。

## InnoDB 隐藏列

InnoDB 聚簇索引记录中有隐藏信息：

- `DB_TRX_ID`：最近一次修改该行的事务 ID。
- `DB_ROLL_PTR`：回滚指针，指向 undo log 中的旧版本。
- `DB_ROW_ID`：没有显式主键时生成的隐藏行 ID。

简化结构：

```text
row
  -> user columns
  -> DB_TRX_ID
  -> DB_ROLL_PTR
```

每次更新都会生成一个新版本，并把旧版本信息写入 undo log。

## undo log

undo log 有两个主要作用：

- 回滚：事务失败时按 undo 撤销修改。
- MVCC：快照读需要通过 undo 版本链读取历史版本。

更新示例：

```sql
UPDATE account
SET balance = 200
WHERE id = 1;
```

版本链简化：

```text
current row: balance=200, trx_id=20, roll_ptr -> undo
undo:        balance=100, trx_id=10, roll_ptr -> older undo
```

快照读如果看不到 `trx_id=20` 版本，就沿 `roll_ptr` 找旧版本。

## ReadView

ReadView 是快照读时创建的可见性视图。

核心字段可以这样理解：

- `creator_trx_id`：创建该 ReadView 的事务 ID。
- `m_ids`：创建 ReadView 时活跃且未提交的事务 ID 集合。
- `min_trx_id`：活跃事务 ID 中最小值。
- `max_trx_id`：下一个将要分配的事务 ID，可以理解为高水位。

判断某个版本 `row_trx_id` 是否可见：

1. 如果 `row_trx_id == creator_trx_id`，说明是当前事务自己改的，可见。
2. 如果 `row_trx_id < min_trx_id`，说明在 ReadView 创建前已提交，可见。
3. 如果 `row_trx_id >= max_trx_id`，说明是 ReadView 创建后才出现的事务，不可见。
4. 如果 `row_trx_id` 在 `m_ids` 中，说明创建 ReadView 时还没提交，不可见。
5. 否则说明创建 ReadView 前已经提交，可见。

如果当前版本不可见，就沿 undo 版本链找上一个版本继续判断。

## RC 和 RR 的 ReadView 时机

### Read Committed

每条快照读语句都会创建新的 ReadView。

```text
T1 begin
T1 select -> ReadView A
T2 update commit
T1 select -> ReadView B，能看到 T2 提交
```

所以 RC 可以避免脏读，但可能不可重复读。

### Repeatable Read

同一事务内第一次快照读创建 ReadView，后续快照读复用。

```text
T1 begin
T1 select -> ReadView A
T2 update commit
T1 select -> still ReadView A，看不到 T2 后续提交
```

所以 RR 下普通快照读可重复。

## 快照读和当前读

### 快照读

普通 `SELECT`：

```sql
SELECT *
FROM account
WHERE id = 1;
```

特点：

- 使用 MVCC。
- 不加行锁。
- 根据 ReadView 判断版本可见性。

### 当前读

当前读读取最新已提交版本，并加锁。

```sql
SELECT *
FROM account
WHERE id = 1
FOR UPDATE;

UPDATE account
SET balance = balance - 100
WHERE id = 1;

DELETE FROM account
WHERE id = 1;
```

当前读不使用旧 ReadView 返回历史版本。它要基于最新数据做修改或加锁。

## 版本链示例

初始数据：

```sql
INSERT INTO account(id, balance) VALUES(1, 100);
```

事务 T20：

```sql
START TRANSACTION;
UPDATE account SET balance = 200 WHERE id = 1;
-- 未提交
```

事务 T30：

```sql
START TRANSACTION;
SELECT balance FROM account WHERE id = 1;
```

如果 T30 的 ReadView 判断 T20 未提交，则当前版本 `balance=200` 不可见，会沿 undo 找到 `balance=100` 返回。

## 长事务问题

长事务会长期持有 ReadView，导致旧版本不能被 purge 清理。

影响：

- undo log 膨胀。
- History list length 增长。
- 查询旧版本链变长。
- 磁盘和 Buffer Pool 压力上升。
- DDL 或清理操作受影响。

排查：

```sql
SELECT *
FROM information_schema.innodb_trx\G
```

查看 InnoDB 状态：

```sql
SHOW ENGINE INNODB STATUS\G
```

关注：

- `History list length`
- 长时间运行的事务。
- 事务开始时间。
- 当前执行 SQL。

## purge

事务提交后，undo log 不能立即删除，因为可能仍有旧 ReadView 需要读取历史版本。

当没有活跃 ReadView 需要这些旧版本时，purge 线程才可以清理。

```text
old version
  -> still visible to some ReadView: keep
  -> no ReadView needs it: purge
```

长事务会阻塞 purge 前进。

## 与锁的关系

MVCC 不等于没有锁。

普通快照读：

- 读历史版本。
- 通常不加行锁。
- 不阻塞写。

当前读和写：

- 读取最新版本。
- 加行锁、间隙锁或 next-key lock。
- 可能阻塞其他事务。

所以回答“MVCC 解决了读写冲突”时要补充：主要是普通快照读和写之间，当前读仍然要加锁。

## SQL 示例

查看隔离级别：

```sql
SELECT @@transaction_isolation;
```

模拟 RR 快照：

```sql
-- session 1
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM account WHERE id = 1;

-- session 2
UPDATE account SET balance = balance + 100 WHERE id = 1;
COMMIT;

-- session 1
SELECT balance FROM account WHERE id = 1;
COMMIT;
```

session 1 两次普通 `SELECT` 结果一致。

当前读对比：

```sql
-- session 1
START TRANSACTION;
SELECT balance FROM account WHERE id = 1;

-- session 2
UPDATE account SET balance = balance + 100 WHERE id = 1;
COMMIT;

-- session 1
SELECT balance FROM account WHERE id = 1 FOR UPDATE;
COMMIT;
```

最后的 `FOR UPDATE` 是当前读，可能读到最新已提交版本。

## 常见追问

### MVCC 是否完全不加锁

不是。MVCC 主要服务普通快照读。`UPDATE`、`DELETE`、`SELECT FOR UPDATE` 等当前读仍然加锁。

### undo log 是不是只用于回滚

不是。undo log 既用于事务回滚，也用于 MVCC 读取历史版本。

### RR 下事务一开始就创建 ReadView 吗

不是。通常是第一次快照读时创建 ReadView。只执行更新或当前读时，行为要结合当前读机制理解。

### 为什么长事务会影响数据库

长事务持有旧 ReadView，导致 undo 旧版本不能清理，History list length 增长，影响存储和查询性能。

## 易错点

- 误以为 MVCC 对所有读都生效。当前读不是快照读。
- 误以为 RR 在事务开始瞬间一定创建 ReadView。
- 误以为 undo log 提交后马上可以删除。
- 误以为 ReadView 里保存了所有行的快照。它保存的是事务可见性边界，不是数据副本。
- 误以为 MVCC 可以解决所有并发一致性问题。写写冲突仍靠锁、约束和业务设计。

## 自检清单

- 能否说明 MVCC 解决什么问题。
- 能否画出 `DB_TRX_ID`、`DB_ROLL_PTR` 和 undo 版本链。
- 能否说出 ReadView 的核心字段。
- 能否按规则判断一个版本是否可见。
- 能否解释 RC 和 RR 的 ReadView 生成时机。
- 能否区分快照读和当前读。
- 能否解释长事务为什么导致 undo 膨胀。
