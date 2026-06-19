# 死锁原因和排查

## 面试定位

死锁题考察线上故障处理能力。面试官常问：

- 什么是死锁。
- InnoDB 如何发现死锁。
- 死锁和锁等待超时有什么区别。
- 常见死锁原因有哪些。
- 线上出现 deadlock 怎么排查。
- Java 业务如何降低死锁概率。

回答主线：死锁是多个事务互相等待对方持有的锁，形成等待环。InnoDB 有死锁检测，会选择代价较小的事务回滚。处理死锁不能只说“重试”，还要分析加锁顺序、索引、事务范围和 SQL 访问路径。

## 什么是死锁

典型死锁：

```text
T1 持有 A，等待 B
T2 持有 B，等待 A
```

两个事务都无法继续。

SQL 示例：

```sql
-- T1
START TRANSACTION;
UPDATE account SET balance = balance - 10 WHERE id = 1;
UPDATE account SET balance = balance + 10 WHERE id = 2;

-- T2
START TRANSACTION;
UPDATE account SET balance = balance - 10 WHERE id = 2;
UPDATE account SET balance = balance + 10 WHERE id = 1;
```

如果 T1 先锁 `id=1`，T2 先锁 `id=2`，再交叉等待，就可能死锁。

## 死锁和锁等待超时

### 死锁

存在等待环，InnoDB 检测到后主动回滚一个事务。

常见错误：

```text
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

### 锁等待超时

没有检测到死锁，事务只是一直等待某个锁，超过 `innodb_lock_wait_timeout`。

常见错误：

```text
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

查看超时配置：

```sql
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
```

## InnoDB 死锁检测

InnoDB 会维护锁等待关系。当发现等待环时，选择一个事务作为 victim 回滚。

选择依据通常和回滚代价有关，例如修改行数较少的事务更可能被回滚。

死锁检测配置：

```sql
SHOW VARIABLES LIKE 'innodb_deadlock_detect';
```

高并发热点行场景下，死锁检测本身也可能带来开销，但一般业务不建议随意关闭。

## 常见死锁原因

### 访问顺序不一致

最典型。

```text
线程 1：先更新订单，再更新库存
线程 2：先更新库存，再更新订单
```

优化：统一访问顺序。

```text
总是先按 id 升序锁库存，再锁订单
```

### 批量更新顺序不固定

```sql
UPDATE account
SET balance = balance + 1
WHERE id IN (3, 1, 2);
```

不同事务传入 ID 顺序不同，实际加锁顺序可能不同。

优化：业务侧排序后逐条或分批处理，保持顺序一致。

### 缺少索引导致锁范围扩大

```sql
UPDATE t_order
SET status = 2
WHERE user_id = 1001
  AND status = 1;
```

如果没有合适索引，可能扫描并锁住大量记录，增加死锁概率。

优化：给更新条件建立合适联合索引。

### 范围锁和插入冲突

RR 下范围当前读可能加 next-key lock。

```sql
SELECT *
FROM t_order
WHERE user_id = 1001
  AND created_at >= '2026-01-01'
FOR UPDATE;
```

其他事务插入该范围记录可能等待，复杂交叉等待可能死锁。

### 二级索引更新

更新二级索引列时，InnoDB 需要维护二级索引和聚簇索引。

不同 SQL 走不同索引路径，可能导致加锁顺序不一致。

### 唯一键冲突和插入

并发插入相同唯一键、`INSERT ... ON DUPLICATE KEY UPDATE`、唯一性检查都可能涉及锁等待和死锁。

### 外键约束

外键检查会访问父表或子表，并加相关锁。复杂事务中可能引入额外锁顺序。

## 排查入口

### SHOW ENGINE INNODB STATUS

```sql
SHOW ENGINE INNODB STATUS\G
```

关注：

- `LATEST DETECTED DEADLOCK`
- 两个或多个事务的 SQL。
- 持有什么锁。
- 等待什么锁。
- 使用哪个索引。
- 事务修改了多少行。
- 哪个事务被回滚。

### 打印所有死锁日志

```sql
SHOW VARIABLES LIKE 'innodb_print_all_deadlocks';
SET GLOBAL innodb_print_all_deadlocks = ON;
```

开启后死锁会记录到 MySQL 错误日志。生产环境要注意日志量。

### performance_schema

死锁瞬间通常很快被处理，事后未必能在锁表里看到。但锁等待可查：

```sql
SELECT *
FROM performance_schema.data_locks;

SELECT *
FROM performance_schema.data_lock_waits;
```

### 应用日志

Java 侧要记录：

- SQL 或 mapper 方法。
- 参数。
- 事务入口。
- traceId。
- 重试次数。
- 执行耗时。

只有数据库死锁日志，没有业务参数时，复盘会困难。

## 排查步骤

建议步骤：

1. 收集死锁错误时间点和业务 trace。
2. 查看 `SHOW ENGINE INNODB STATUS` 的最近死锁。
3. 找到互相等待的 SQL、表、索引、锁模式。
4. 对照表结构和索引，判断访问路径。
5. 找到两个事务的业务入口和调用顺序。
6. 判断是否缺索引、顺序不一致、范围过大、事务过长。
7. 制定优化和重试策略。

## 预防手段

### 固定加锁顺序

多个资源按固定顺序访问。

```java
List<Long> ids = new ArrayList<>(request.getAccountIds());
Collections.sort(ids);
for (Long id : ids) {
    accountMapper.lockById(id);
}
```

### 缩短事务

事务内只放必要数据库操作。

避免：

- 远程调用。
- 大文件处理。
- 复杂计算。
- 用户交互等待。

### 建好索引

更新、删除、加锁查询必须有合适索引。

```sql
EXPLAIN UPDATE ...
EXPLAIN SELECT ... FOR UPDATE;
```

### 小批量提交

大批量更新分批处理：

```sql
UPDATE t_order
SET status = 9
WHERE status = 1
  AND id > ?
ORDER BY id
LIMIT 500;
```

### 使用乐观锁

```sql
UPDATE account
SET balance = balance - 100,
    version = version + 1
WHERE id = 1
  AND version = 10;
```

更新行数为 0 时由业务重试或返回失败。

### 死锁重试

死锁不能完全避免，业务应对可重试事务做有限重试。

注意：

- 只重试幂等或可安全重放的事务。
- 设置最大重试次数。
- 加随机退避。
- 保留日志。

## Java 示例

伪代码：

```java
public void transfer(long fromId, long toId, BigDecimal amount) {
    List<Long> ids = Stream.of(fromId, toId)
            .sorted()
            .toList();

    transactionTemplate.execute(status -> {
        Account first = accountMapper.selectForUpdate(ids.get(0));
        Account second = accountMapper.selectForUpdate(ids.get(1));
        // 根据 from/to 扣减和增加余额
        return null;
    });
}
```

重点是锁定顺序稳定，不随转出转入方向变化。

## 常见追问

### 死锁一定是 bug 吗

不一定。高并发数据库中死锁可能偶发出现，但频繁死锁通常说明访问顺序、索引、事务范围或热点设计有问题。

### InnoDB 会自动解决死锁吗

会检测并回滚一个事务，但业务层仍会收到异常。应用要决定是否重试、补偿或返回失败。

### 只加索引能解决死锁吗

不一定。索引可以减少锁范围，但访问顺序不一致、事务过长、热点行竞争仍可能死锁。

### 为什么死锁日志只看到一条最近的

`SHOW ENGINE INNODB STATUS` 只保留最近一次死锁信息。要长期收集可以开启 `innodb_print_all_deadlocks` 写错误日志，或依赖数据库平台采集。

## 易错点

- 认为死锁只发生在显式 `SELECT FOR UPDATE`。普通 `UPDATE`、`DELETE` 也会加锁。
- 只做重试，不分析根因。
- 忽略 SQL 走的索引不同会导致加锁顺序不同。
- 忽略缺索引导致锁范围扩大。
- 在事务内调用外部服务，放大锁持有时间。
- 认为锁等待超时和死锁是同一个问题。

## 自检清单

- 能否定义死锁和锁等待超时。
- 能否解释 InnoDB 死锁检测和 victim 回滚。
- 能否读懂 `LATEST DETECTED DEADLOCK`。
- 能否列出访问顺序不一致、缺索引、范围锁、唯一键冲突等原因。
- 能否给出固定顺序、缩短事务、补索引、批量拆分、有限重试等方案。
- 能否说明 Java 业务如何记录死锁排查所需上下文。
