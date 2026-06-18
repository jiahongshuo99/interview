# binlog、redo log、undo log

## 面试定位

三大日志是 MySQL 事务、复制、恢复的核心。面试官常问：

- binlog、redo log、undo log 分别做什么。
- redo log 和 binlog 有什么区别。
- 为什么需要两阶段提交。
- undo log 是否会写入 redo log。
- 崩溃恢复时怎么保证数据和 binlog 一致。
- 主从复制依赖哪个日志。

回答主线：redo log 是 InnoDB 的物理重做日志，用于崩溃恢复和持久性；undo log 是 InnoDB 的回滚日志和 MVCC 版本来源；binlog 是 MySQL Server 层的逻辑日志，用于复制、归档和基于时间点恢复。事务提交时通过 redo 和 binlog 的两阶段提交保证一致。

## 三类日志总览

| 日志 | 所属层 | 主要内容 | 主要用途 |
| --- | --- | --- | --- |
| redo log | InnoDB 引擎层 | 数据页物理修改 | 崩溃恢复、持久性 |
| undo log | InnoDB 引擎层 | 修改前旧版本 | 回滚、MVCC |
| binlog | Server 层 | 逻辑变更事件 | 主从复制、归档恢复 |

一句话区分：

- redo log 保证“提交后的页修改能恢复”。
- undo log 保证“事务能回滚，快照读能读旧版本”。
- binlog 保证“变更能复制和重放”。

## redo log

redo log 是重做日志，记录数据页的物理修改。

为什么需要 redo：

- 如果每次提交都刷完整数据页，随机 IO 成本很高。
- 有 redo 后，可以先顺序写日志，再异步刷脏页。
- 崩溃后通过 redo 把已提交修改恢复回来。

写流程简化：

```text
修改 Buffer Pool 中的数据页
  -> 生成 redo log record
  -> 写入 redo log buffer
  -> 提交时按策略写入/刷盘 redo log file
  -> 后台刷脏页
```

WAL，Write-Ahead Logging：

```text
先写日志，再写数据页
```

相关配置：

```sql
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
SHOW VARIABLES LIKE 'innodb_redo_log_capacity';
```

## redo log buffer 与刷盘策略

`innodb_flush_log_at_trx_commit`：

- `1`：每次事务提交都写 redo 并刷盘，持久性最好。
- `2`：每次提交写到 OS cache，每秒刷盘。
- `0`：每秒写入并刷盘。

面试回答：

- 设置为 `1` 时，MySQL 进程崩溃和机器崩溃都更不容易丢已提交事务。
- 设置为 `2` 或 `0` 可提升性能，但机器崩溃可能丢最近事务。

## checkpoint

redo log 空间有限，不能无限追加。

checkpoint 表示：某个位置之前的脏页修改已经刷到磁盘，不再需要依赖 redo 恢复。

如果脏页刷得慢，redo 空间逼近上限，会触发更积极刷脏，影响写入性能。

## undo log

undo log 记录修改前的旧版本。

用途：

- 事务回滚。
- MVCC 快照读。

示例：

```sql
UPDATE account
SET balance = 200
WHERE id = 1;
```

undo 中记录旧值：

```text
id=1, old balance=100
```

如果回滚，就按 undo 恢复。

如果另一个事务的 ReadView 看不到新版本，就沿 undo 版本链读旧版本。

undo log 本身也要持久化，它的变更也会受 redo 保护。

## purge 与 undo

事务提交后，undo 不一定立即删除。

如果还有事务 ReadView 需要旧版本，undo 必须保留。

当没有活跃快照需要这些版本，purge 线程清理 undo。

长事务会导致：

- undo 保留时间变长。
- History list length 增长。
- 磁盘空间上涨。
- 查询旧版本链成本增加。

查看：

```sql
SHOW ENGINE INNODB STATUS\G
SELECT * FROM information_schema.innodb_trx\G
```

## binlog

binlog 是 MySQL Server 层的二进制日志。

用途：

- 主从复制。
- 基于时间点恢复。
- 审计和数据同步。
- CDC 订阅。

查看配置：

```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'sync_binlog';
```

查看当前 binlog：

```sql
SHOW MASTER STATUS;
```

MySQL 8.0 新命令：

```sql
SHOW BINARY LOG STATUS;
```

## binlog 格式

### STATEMENT

记录 SQL 原文。

优点：

- 日志量小。

缺点：

- 非确定性函数、触发器、自增、数据分布差异可能导致主从不一致。

### ROW

记录每行变更前后内容。

优点：

- 复制更准确。
- 适合大多数现代业务。
- CDC 更容易解析。

缺点：

- 大批量更新日志量大。

### MIXED

MySQL 根据 SQL 类型在 statement 和 row 间选择。

工程中常见推荐是 ROW。

## redo log 和 binlog 区别

| 对比 | redo log | binlog |
| --- | --- | --- |
| 所属 | InnoDB | MySQL Server |
| 内容 | 物理页修改 | 逻辑变更事件 |
| 用途 | 崩溃恢复 | 复制和归档恢复 |
| 写入方式 | 循环写 | 追加写 |
| 是否引擎特有 | 是 | 否 |
| 是否支持主从复制 | 不直接用于复制 | 是复制核心 |

常见误区：binlog 不是 InnoDB 崩溃恢复的核心，redo log 也不是主从复制的日志。

## 两阶段提交

如果一个事务既要写 InnoDB 数据，又要写 binlog，就需要保证二者一致。

否则可能出现：

- 数据提交了，binlog 没写，主库有数据，从库没数据。
- binlog 写了，数据没提交，从库有数据，主库没数据。

MySQL 使用 redo log 和 binlog 的两阶段提交。

简化流程：

```text
1. InnoDB 写 redo log，状态 prepare
2. Server 写 binlog
3. InnoDB 写 redo log，状态 commit
```

崩溃恢复时：

- redo prepare 且 binlog 存在：提交事务。
- redo prepare 但 binlog 不存在：回滚事务。
- redo commit：提交事务。

这样保证主库恢复结果和 binlog 可复制结果一致。

## 崩溃场景

### redo prepare 前崩溃

事务未进入提交阶段，回滚。

### redo prepare 后、binlog 前崩溃

binlog 不存在，恢复时回滚。

### binlog 后、redo commit 前崩溃

binlog 已存在，redo 是 prepare，恢复时提交。

### redo commit 后崩溃

事务提交，恢复时重做。

## SQL 示例

查看日志相关配置：

```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'sync_binlog';
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
SHOW VARIABLES LIKE 'innodb_redo_log_capacity';
```

查看 binlog 文件：

```sql
SHOW BINARY LOGS;
SHOW BINLOG EVENTS IN 'mysql-bin.000001' LIMIT 10;
```

使用命令行解析 binlog：

```bash
mysqlbinlog --base64-output=decode-rows -vv mysql-bin.000001
```

## 排查与优化

### 写入慢

关注：

- redo 刷盘压力。
- binlog 刷盘策略。
- 磁盘 IO。
- 大事务产生大量日志。
- 主从复制压力。

配置权衡：

```sql
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
SHOW VARIABLES LIKE 'sync_binlog';
```

`sync_binlog = 1` 表示每次提交都同步 binlog，复制一致性和持久性更强，开销更高。

### undo 膨胀

查长事务：

```sql
SELECT *
FROM information_schema.innodb_trx\G
```

处理：

- 结束异常长事务。
- 优化批处理事务大小。
- 避免长时间开启事务只读不提交。

### 主从恢复

binlog 用于按位置或 GTID 恢复：

```text
全量备份
  -> 找到备份位点
  -> 重放 binlog 到目标时间点
```

## 常见追问

### update 一行会写哪些日志

通常会写 undo log 记录旧版本，写 redo log 记录页修改，提交时写 binlog 记录逻辑变更。实际还涉及内存 buffer、脏页和刷盘策略。

### undo log 会被 redo log 保护吗

会。undo log 本身也是 InnoDB 数据，需要持久化，它的页修改也会记录 redo。

### redo log 已经能恢复，为什么还要 binlog

redo 是 InnoDB 私有物理日志，循环写，用于崩溃恢复。binlog 是 Server 层逻辑日志，追加写，用于复制、归档和时间点恢复。

### binlog row 格式会记录所有列吗

取决于配置和事件类型。ROW 格式记录行变更镜像，具体记录完整行还是最小列集合受 `binlog_row_image` 影响。

## 易错点

- 误以为 binlog 用于 InnoDB 崩溃恢复。
- 误以为 redo log 用于主从复制。
- 误以为 undo log 提交后马上删除。
- 误以为两阶段提交是分布式事务两阶段提交，二者不是同一个层面的概念。
- 误以为 `COMMIT` 一定同步所有数据页到磁盘，实际通常先保证日志。
- 忽略刷盘配置对持久性和性能的权衡。

## 自检清单

- 能否一句话区分 binlog、redo log、undo log。
- 能否说明 redo log 为什么能提升写性能并保证崩溃恢复。
- 能否解释 undo log 的回滚和 MVCC 作用。
- 能否说明 binlog 的复制和恢复用途。
- 能否画出 redo prepare、binlog、redo commit 的两阶段提交流程。
- 能否分析崩溃发生在不同阶段时如何恢复。
