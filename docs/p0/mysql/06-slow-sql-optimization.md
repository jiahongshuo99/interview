# 慢 SQL 定位与优化

## 面试定位

慢 SQL 优化考察的是线上排查能力，不是背几条索引原则。面试常见追问：

- 线上接口变慢，怎么判断是不是 SQL 慢。
- 慢 SQL 从哪里找。
- 拿到 SQL 后怎么分析。
- 有索引为什么还慢。
- 深分页、排序、join、count、批量写怎么优化。
- Java 后端里哪些代码写法容易制造慢 SQL。

回答主线：先定位慢 SQL 和影响范围，再看执行计划和真实执行，再结合索引、扫描行数、回表、排序、锁等待、数据量、返回行数、应用调用方式做优化。不要上来就说“加索引”。

## 定位入口

### 应用侧

先确认慢在哪里：

- 接口 RT。
- 应用日志耗时。
- APM 链路追踪。
- 数据库调用耗时。
- 连接池等待耗时。
- 下游接口耗时。

Java 后端常见指标：

- Tomcat/Netty 工作线程是否耗尽。
- HikariCP 活跃连接、等待连接、超时次数。
- MyBatis SQL 执行耗时日志。
- 事务耗时。
- 慢请求 trace。

### MySQL 慢日志

开启和查看：

```sql
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'long_query_time';
SHOW VARIABLES LIKE 'slow_query_log_file';
SHOW VARIABLES LIKE 'log_queries_not_using_indexes';
```

常见配置：

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;
```

生产环境是否临时调整要结合权限和规范。

慢日志关注：

- `Query_time`
- `Lock_time`
- `Rows_sent`
- `Rows_examined`
- SQL 原文

`Rows_examined` 很大但 `Rows_sent` 很小，通常说明扫描过滤效率差。

### performance_schema 和 sys

MySQL 8.0 常用：

```sql
SELECT *
FROM sys.statement_analysis
ORDER BY total_latency DESC
LIMIT 10;
```

查 digest：

```sql
SELECT digest_text, count_star, avg_timer_wait, sum_rows_examined, sum_rows_sent
FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC
LIMIT 10;
```

优势是能聚合相同模式 SQL，而不是只看单条慢日志。

## 分析流程

拿到慢 SQL 后建议按这个流程：

```text
确认 SQL 和参数
  -> 确认表结构、索引、数据量、数据分布
  -> EXPLAIN / EXPLAIN FORMAT=JSON
  -> 必要时 EXPLAIN ANALYZE
  -> 判断扫描、回表、排序、临时表、join、锁等待
  -> 设计优化方案
  -> 在测试或影子环境验证
  -> 观察线上指标
```

基础信息：

```sql
SHOW CREATE TABLE t_order\G
SHOW INDEX FROM t_order\G
SELECT COUNT(*) FROM t_order;
```

执行计划：

```sql
EXPLAIN SELECT ...
EXPLAIN FORMAT=JSON SELECT ...
```

统计信息：

```sql
ANALYZE TABLE t_order;
```

## 常见慢 SQL 类型

### 未命中索引

问题 SQL：

```sql
SELECT *
FROM t_order
WHERE user_id = 1001
  AND status = 1
ORDER BY created_at DESC
LIMIT 20;
```

可考虑索引：

```sql
CREATE INDEX idx_user_status_time
ON t_order (user_id, status, created_at);
```

判断点：

- `WHERE` 等值列是否能作为前缀。
- `ORDER BY` 是否能利用索引顺序。
- 查询列是否可以覆盖。

### 索引列上做函数

问题 SQL：

```sql
SELECT *
FROM t_order
WHERE DATE(created_at) = '2026-01-01';
```

优化：

```sql
SELECT *
FROM t_order
WHERE created_at >= '2026-01-01'
  AND created_at < '2026-01-02';
```

### 隐式类型转换

字段是数字，参数传字符串，或者字段是字符串，参数传数字，都可能影响索引使用或结果。

```sql
-- phone 是 varchar
SELECT *
FROM t_user
WHERE phone = 13800000000;
```

应改为：

```sql
SELECT *
FROM t_user
WHERE phone = '13800000000';
```

### 前置模糊匹配

```sql
SELECT *
FROM t_user
WHERE name LIKE '%明';
```

B+Tree 无法从左侧定位。可考虑：

- 改业务为前缀搜索。
- 使用倒排索引或搜索引擎。
- 冗余反转字段支持后缀搜索。
- 小数据量接受扫描。

### 深分页

问题：

```sql
SELECT *
FROM t_order
WHERE user_id = 1001
ORDER BY created_at DESC
LIMIT 100000, 20;
```

MySQL 需要跳过前 100000 行，成本高。

优化 1：基于游标翻页。

```sql
SELECT *
FROM t_order
WHERE user_id = 1001
  AND created_at < '2026-06-01 12:00:00'
ORDER BY created_at DESC
LIMIT 20;
```

优化 2：延迟关联。

```sql
SELECT o.*
FROM t_order o
JOIN (
    SELECT id
    FROM t_order
    WHERE user_id = 1001
    ORDER BY created_at DESC
    LIMIT 100000, 20
) x ON o.id = x.id;
```

前提是子查询能使用覆盖索引。

### 排序和临时表

典型问题：

```sql
SELECT status, COUNT(*)
FROM t_order
WHERE created_at >= '2026-01-01'
GROUP BY status
ORDER BY COUNT(*) DESC;
```

可能出现：

- `Using temporary`
- `Using filesort`

优化方向：

- 控制时间范围。
- 设计合适索引。
- 预聚合到统计表。
- 离线计算。
- 避免在主链路做大范围统计。

### join 慢

问题：

```sql
SELECT *
FROM t_order o
JOIN t_user u ON o.user_id = u.id
WHERE o.status = 1;
```

检查：

- 驱动表过滤后行数是否小。
- 被驱动表 join 字段是否有索引。
- join 字段类型和字符集是否一致。
- 是否返回过多列。

被驱动表缺索引时可能出现 `Using join buffer`。

### `count(*)` 慢

InnoDB 不像 MyISAM 维护精确总行数。`COUNT(*)` 通常要扫描索引。

优化方向：

- 使用更小的二级索引扫描。
- 加条件缩小范围。
- 使用缓存或统计表。
- 弱一致场景使用近似计数。
- 避免频繁实时全表 count。

### 大事务导致慢

慢不一定是 SQL 本身执行慢，也可能在等锁。

排查：

```sql
SHOW ENGINE INNODB STATUS\G
SELECT * FROM information_schema.innodb_trx\G
```

MySQL 8.0：

```sql
SELECT *
FROM performance_schema.data_locks;

SELECT *
FROM performance_schema.data_lock_waits;
```

优化：

- 缩短事务。
- 固定访问顺序。
- 批量操作分段提交。
- 避免事务内调用外部接口。
- 确保更新条件有索引。

## 索引设计策略

### 按查询模式设计

不要孤立地看单列选择性，要看完整 SQL：

```sql
WHERE user_id = ?
  AND status = ?
ORDER BY created_at DESC
LIMIT ?
```

候选索引：

```sql
KEY idx_user_status_time (user_id, status, created_at)
```

它同时支持：

- 用户过滤。
- 状态过滤。
- 时间排序。
- LIMIT 提前停止。

### 控制索引数量

过多索引的问题：

- 写入变慢。
- 占用磁盘。
- 统计和优化器选择更复杂。
- DDL 成本变高。

删除索引前要确认：

- 是否有线上 SQL 使用。
- 是否被唯一约束依赖。
- 是否被外键依赖。
- 是否可以由更长联合索引前缀替代。

## SQL 改写技巧

### 只查必要列

```sql
-- 不推荐
SELECT *
FROM t_order
WHERE user_id = 1001;

-- 推荐
SELECT id, status, amount, created_at
FROM t_order
WHERE user_id = 1001;
```

收益：

- 降低网络传输。
- 有机会覆盖索引。
- 避免读取大字段。

### 分批处理

大批量更新：

```sql
UPDATE t_order
SET status = 9
WHERE created_at < '2025-01-01'
LIMIT 1000;
```

循环执行并控制间隔，避免长事务、锁范围过大和主从延迟。

### 避免 N+1 查询

Java 代码中常见：

```java
for (Long userId : userIds) {
    orderMapper.listByUserId(userId);
}
```

优化：

```sql
SELECT *
FROM t_order
WHERE user_id IN (?, ?, ?);
```

注意 `IN` 列表也不能无限大，需要分批。

## Java 后端常见坑

- MyBatis 循环里查库，造成 N+1。
- `PageHelper` 深分页无业务上限。
- 接口默认返回全量列表。
- 事务包住远程调用，导致锁持有时间长。
- 批量插入逐条提交。
- SQL 参数类型和字段类型不一致。
- 动态 SQL 漏掉过滤条件。
- 读写分离下读到从库延迟数据，误判为查询慢或数据不一致。

## 优化验证

优化前后要对比：

- 执行计划。
- 实际耗时。
- 扫描行数。
- 返回行数。
- CPU、IO、锁等待。
- Buffer Pool 命中。
- 主从延迟。
- 应用接口 RT。

不要只在很小测试数据上验证索引效果。

## 常见追问

### 加索引就能解决慢 SQL 吗

不一定。慢 SQL 可能来自锁等待、排序、临时表、返回数据太多、网络传输、应用 N+1、事务过长。索引只是最常见手段之一。

### 为什么有索引还是慢

可能原因：

- 命中行数太多。
- 大量回表。
- 索引顺序不匹配排序。
- 统计信息不准。
- 低选择性索引被放弃。
- 等锁。
- 查询列太多或包含大字段。

### 是否要使用 `FORCE INDEX`

谨慎。它可以临时绕过优化器误判，但数据分布变化后可能变成负优化。优先考虑统计信息、索引设计和 SQL 改写。

### 慢日志里 `Lock_time` 很高怎么办

说明等待锁时间明显。需要查长事务、锁等待、更新条件是否命中索引、事务内是否有外部调用、是否有批量更新。

## 易错点

- 一看到慢 SQL 就加单列索引。
- 只看平均耗时，不看 P95/P99 和尖刺。
- 只看数据库，不看连接池等待和应用 N+1。
- 忽略 `Rows_examined` 和 `Rows_sent` 的差距。
- 忽略线上数据分布和测试环境差异。
- 用 `SELECT *` 破坏覆盖索引。
- 在事务内做慢 RPC 或大批量循环处理。

## 自检清单

- 能否说出慢 SQL 的应用侧和数据库侧定位入口。
- 能否根据慢日志字段判断扫描多、返回多还是锁等待。
- 能否用 `EXPLAIN` 分析索引、回表、排序和临时表。
- 能否给出深分页、函数索引失效、隐式转换、join 慢的优化方案。
- 能否解释为什么低选择性索引可能不被使用。
- 能否结合 Java 代码说出 N+1、连接池、事务过长等问题。
