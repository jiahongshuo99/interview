# EXPLAIN 字段解读

## 面试定位

`EXPLAIN` 是慢 SQL 分析的入口。面试官常问：

- `type` 从好到坏怎么排。
- `key`、`possible_keys`、`key_len` 分别是什么意思。
- `rows` 是真实扫描行数吗。
- `Extra` 里 `Using index`、`Using where`、`Using filesort`、`Using temporary` 各代表什么。
- `EXPLAIN` 和 `EXPLAIN ANALYZE` 有什么区别。
- 如何根据执行计划提出优化方案。

回答主线：`EXPLAIN` 展示的是优化器选择的执行计划，多数字段是估算结果。看执行计划不能只看“有没有用索引”，要结合访问类型、扫描行数、过滤比例、排序临时表、回表和实际执行耗时。

## 基本用法

```sql
EXPLAIN SELECT ...
EXPLAIN FORMAT=JSON SELECT ...
EXPLAIN ANALYZE SELECT ...
```

区别：

- `EXPLAIN`：展示优化器预估计划，不真正执行查询。
- `EXPLAIN FORMAT=JSON`：提供更详细的成本、过滤、回表、排序等信息。
- `EXPLAIN ANALYZE`：会实际执行 SQL，并输出真实耗时和行数，线上慎用。

## 常见字段总览

| 字段 | 含义 |
| --- | --- |
| `id` | 查询块编号，复杂 SQL 中表示执行层级 |
| `select_type` | 查询类型 |
| `table` | 当前访问的表或派生表 |
| `partitions` | 命中的分区 |
| `type` | 访问类型，重要 |
| `possible_keys` | 可能使用的索引 |
| `key` | 实际选择的索引 |
| `key_len` | 使用的索引长度 |
| `ref` | 与索引比较的列或常量 |
| `rows` | 预估扫描行数 |
| `filtered` | 预估过滤后保留比例 |
| `Extra` | 额外执行信息 |

## id

`id` 表示查询块编号。

一般规则：

- `id` 相同，通常从上到下执行。
- `id` 越大，优先级通常越高。
- 派生表、子查询、联合查询会出现多个 `id`。

示例：

```sql
EXPLAIN
SELECT *
FROM t_order o
WHERE o.user_id IN (
    SELECT id
    FROM t_user
    WHERE city_id = 10
);
```

现代 MySQL 优化器可能把子查询改写成半连接，所以不要只按老规则死记，要结合 `select_type` 和 JSON 计划。

## select_type

常见值：

| 值 | 含义 |
| --- | --- |
| `SIMPLE` | 简单查询，不含子查询或 union |
| `PRIMARY` | 外层主查询 |
| `SUBQUERY` | 子查询 |
| `DERIVED` | 派生表 |
| `UNION` | union 后续查询 |
| `UNION RESULT` | union 结果 |
| `DEPENDENT SUBQUERY` | 依赖外层的子查询，通常要警惕 |

`DEPENDENT SUBQUERY` 常表示子查询会随外层行重复执行，可能导致性能问题。

## type

`type` 是访问类型，反映 MySQL 如何访问表。

常见从好到坏大致为：

```text
system
const
eq_ref
ref
range
index
ALL
```

### system

表只有一行或系统表，极少见。

### const

通过主键或唯一索引等值查询，最多匹配一行。

```sql
EXPLAIN SELECT *
FROM t_user
WHERE id = 1;
```

### eq_ref

连接中使用主键或唯一非空索引，每次外表一行，内表最多匹配一行。

```sql
EXPLAIN SELECT *
FROM t_order o
JOIN t_user u ON o.user_id = u.id;
```

### ref

使用非唯一索引等值匹配，可能匹配多行。

```sql
EXPLAIN SELECT *
FROM t_order
WHERE user_id = 1001;
```

### range

范围扫描。

```sql
EXPLAIN SELECT *
FROM t_order
WHERE created_at >= '2026-01-01'
  AND created_at < '2026-02-01';
```

### index

扫描整个索引树。比全表扫描可能稍好，因为索引比整行小，但仍然是全索引扫描。

### ALL

全表扫描。大表上通常要重点关注，但小表、低选择性场景不一定是问题。

## possible_keys 与 key

`possible_keys` 是优化器认为可能用的索引。

`key` 是最终选择的索引。

情况：

- `possible_keys` 有值，`key` 为 `NULL`：优化器评估后认为不用索引更便宜。
- `possible_keys` 为 `NULL`：没有可用索引，或条件无法使用索引。
- `key` 不是你期望的索引：可能统计信息、选择性、回表成本、排序成本导致选择不同。

## key_len

`key_len` 表示优化器使用索引的字节长度，可用于推断联合索引用到哪些列。

影响因素：

- 列类型。
- 字符集。
- 是否允许 `NULL`。
- 变长字段长度字节。
- 联合索引使用到的列数。

示例：

```sql
CREATE TABLE t_demo (
    a INT NOT NULL,
    b BIGINT NOT NULL,
    c VARCHAR(20) NOT NULL,
    KEY idx_abc (a, b, c)
) ENGINE = InnoDB;
```

查询：

```sql
EXPLAIN SELECT *
FROM t_demo
WHERE a = 1
  AND b = 2;
```

`key_len` 大致能体现用到了 `a` 和 `b`。

注意：`key_len` 不是越长越好，要看是否符合查询目标。

## rows 与 filtered

`rows` 是优化器预估要扫描的行数，不是真实值。

`filtered` 是预估条件过滤后保留百分比。

估算输出行数大致是：

```text
rows * filtered / 100
```

如果 `rows` 远大，通常表示扫描范围大。

如果实际执行和预估差异大，可能是：

- 统计信息过期。
- 数据分布倾斜。
- 低基数字段估算不准。
- 多列相关性优化器难以准确估计。

可尝试：

```sql
ANALYZE TABLE t_order;
SHOW INDEX FROM t_order;
```

MySQL 8.0 还可以考虑直方图：

```sql
ANALYZE TABLE t_order UPDATE HISTOGRAM ON status, city_id;
```

## Extra 常见值

### Using where

表示 Server 层或存储引擎还需要应用 `WHERE` 条件过滤。

它不一定坏，很多正常索引查询也会出现。

### Using index

表示使用覆盖索引。查询列都能从索引中取到，通常不需要回表。

注意：是否使用索引看 `key`，不是看 `Using index`。

### Using index condition

表示使用索引下推 ICP。在回表前利用索引中的列先过滤。

### Using filesort

表示需要额外排序，不一定真的写磁盘文件。排序可能在内存中完成，也可能溢出到磁盘。

需要关注：

- 排序行数。
- 是否有 `LIMIT`。
- 是否有合适的联合索引。
- 排序字段和索引顺序是否一致。

### Using temporary

表示需要临时表，常见于 `GROUP BY`、`DISTINCT`、复杂排序、派生表。

大数据临时表可能导致磁盘临时表，性能风险较高。

### Using join buffer

表示连接时使用 join buffer，常见于被驱动表没有合适索引。

要重点检查 join 条件索引。

### Impossible WHERE

优化器判断 `WHERE` 条件不可能成立。

## 示例分析

表结构：

```sql
CREATE TABLE t_order (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status TINYINT NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    created_at DATETIME NOT NULL,
    KEY idx_user_status_time (user_id, status, created_at),
    KEY idx_created_at (created_at)
) ENGINE = InnoDB;
```

### 好计划

```sql
EXPLAIN SELECT id, user_id, status, created_at
FROM t_order
WHERE user_id = 1001
  AND status = 1
ORDER BY created_at DESC
LIMIT 20;
```

期望：

- `key = idx_user_status_time`
- `type = ref`
- `rows` 较小。
- `Extra` 可能有 `Using where; Using index`。
- 没有 `Using filesort`。

### 可能排序

```sql
EXPLAIN SELECT *
FROM t_order
WHERE user_id = 1001
ORDER BY amount DESC
LIMIT 20;
```

如果没有 `(user_id, amount)` 相关索引，可能出现 `Using filesort`。

### 可能全表扫描

```sql
EXPLAIN SELECT *
FROM t_order
WHERE status = 1;
```

没有以 `status` 开头的索引，或者 `status = 1` 命中大量数据时，可能全表扫描。

## EXPLAIN ANALYZE

`EXPLAIN ANALYZE` 会真实执行 SQL，给出实际耗时和行数。

适合：

- 验证优化器估算偏差。
- 对比改写前后效果。
- 分析 join 每一步耗时。

风险：

- 会真的执行查询。
- 对大查询、写语句或线上高峰要慎用。
- 可能带来明显资源开销。

示例：

```sql
EXPLAIN ANALYZE
SELECT id
FROM t_order
WHERE user_id = 1001
ORDER BY created_at DESC
LIMIT 20;
```

## 优化思路

看到执行计划后按这个顺序问：

1. 是否访问了太多行。
2. 是否用了正确索引。
3. 联合索引是否匹配过滤、排序、分组。
4. 是否大量回表。
5. 是否有 `Using filesort` 或 `Using temporary`。
6. join 被驱动表是否有索引。
7. 统计信息是否过期。
8. 返回数据是否过多。

## 常见追问

### `type = ALL` 一定要优化吗

不一定。小表全表扫描可能比走索引更快。大表、频繁查询、返回少量行却全表扫，才是重点问题。

### `Using filesort` 一定走磁盘吗

不一定。它表示额外排序算法，不等于一定使用磁盘文件。是否落盘取决于排序数据量、内存、字段大小等。

### `rows` 为什么和真实差很多

因为它来自统计信息和成本估算。数据倾斜、多列相关、统计过期都会导致偏差。

### `possible_keys` 有索引但 `key` 为 NULL 怎么办

先判断是否命中行太多、是否需要大量回表、是否条件写法导致不能有效利用索引。必要时更新统计信息、改 SQL、调整联合索引，而不是直接强制索引。

## 易错点

- 只看 `key`，不看 `rows`、`Extra` 和回表。
- 把 `Using index` 理解成“用了索引”，实际它通常表示覆盖索引。
- 把 `rows` 当真实扫描行数。
- 看到 `Using filesort` 就一定认为灾难。
- 看到 `type = index` 就以为很好，实际可能是全索引扫描。
- 不区分 `EXPLAIN` 预估和 `EXPLAIN ANALYZE` 实测。

## 自检清单

- 能否按顺序解释 `type` 常见值。
- 能否说明 `possible_keys` 和 `key` 的区别。
- 能否通过 `key_len` 粗略判断联合索引用到几列。
- 能否解释 `rows`、`filtered` 的估算性质。
- 能否区分 `Using index` 和 `Using index condition`。
- 能否根据 `Using filesort`、`Using temporary` 提出索引或 SQL 改写方向。
- 能否说明什么时候使用 `EXPLAIN ANALYZE`。
