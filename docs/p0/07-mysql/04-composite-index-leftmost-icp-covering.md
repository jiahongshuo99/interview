# 联合索引、最左前缀、索引下推、覆盖索引

## 面试定位

联合索引是 Java 后端最常见的 SQL 优化考点。面试官通常会问：

- 联合索引 `(a, b, c)` 哪些查询能用。
- 最左前缀到底是什么意思。
- 范围条件后面的列还能不能用。
- `WHERE`、`ORDER BY`、`GROUP BY` 如何影响索引列顺序。
- 索引下推 ICP 和覆盖索引区别是什么。
- 为什么有索引还会 `Using filesort`。

回答主线：联合索引按多个列拼接后的有序键组织，能否充分利用索引，取决于查询条件是否匹配索引的有序前缀。等值条件可以继续向后利用，范围条件通常会截断后续列的有序利用，但后续列仍可能通过索引下推做过滤。

## 联合索引是什么

示例：

```sql
CREATE TABLE t_order (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status TINYINT NOT NULL,
    created_at DATETIME NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    KEY idx_user_status_time (user_id, status, created_at)
) ENGINE = InnoDB;
```

联合索引键可以理解为：

```text
(user_id, status, created_at, id)
```

其中 `id` 是 InnoDB 二级索引叶子节点附带的主键。

索引排序规则：

```text
先按 user_id 排序
user_id 相同再按 status 排序
status 相同再按 created_at 排序
```

## 最左前缀原则

联合索引可以高效支持从最左列开始的连续前缀。

索引：

```text
(user_id, status, created_at)
```

可以使用的前缀：

- `(user_id)`
- `(user_id, status)`
- `(user_id, status, created_at)`

不能直接跳过 `user_id` 只用 `status` 做有序定位。

示例：

```sql
-- 能用 user_id
SELECT *
FROM t_order
WHERE user_id = 1001;

-- 能用 user_id + status
SELECT *
FROM t_order
WHERE user_id = 1001
  AND status = 1;

-- 能用完整前缀
SELECT *
FROM t_order
WHERE user_id = 1001
  AND status = 1
  AND created_at >= '2026-01-01';

-- 通常不能靠该索引高效定位 status
SELECT *
FROM t_order
WHERE status = 1;
```

注意：MySQL 优化器可能做索引跳跃扫描等优化，但面试和工程设计不能依赖它替代正确索引设计。

## 等值、范围和后续列

联合索引中，等值条件有利于继续向后利用索引。

```sql
WHERE user_id = 1001
  AND status = 1
  AND created_at >= '2026-01-01'
```

可以定位到 `(1001, 1, '2026-01-01')` 的起始位置。

范围条件后面的列通常不能继续用于有序定位。

索引：

```text
(user_id, created_at, status)
```

SQL：

```sql
SELECT *
FROM t_order
WHERE user_id = 1001
  AND created_at >= '2026-01-01'
  AND status = 1;
```

`status` 在范围列 `created_at` 后面。B+Tree 在 `created_at >= ...` 的范围内，`status` 不再是全局连续有序的定位条件。

但这不代表 `status` 完全没用。MySQL 可能通过索引下推在回表前过滤。

## 索引下推 ICP

ICP，全称 Index Condition Pushdown。

没有 ICP 时，存储引擎通过索引找到候选记录后，可能先回表，再由 Server 层判断剩余条件。

有 ICP 时，能在索引层判断的条件会下推给存储引擎，先过滤，再回表。

示例：

```sql
CREATE TABLE t_user (
    id BIGINT PRIMARY KEY,
    name VARCHAR(64) NOT NULL,
    age INT NOT NULL,
    city_id BIGINT NOT NULL,
    KEY idx_name_age (name, age)
) ENGINE = InnoDB;
```

查询：

```sql
EXPLAIN SELECT *
FROM t_user
WHERE name LIKE '张%'
  AND age = 30;
```

`name LIKE '张%'` 是范围条件，`age` 位于后续列。`age = 30` 可能不能用于进一步缩小索引定位范围，但因为 `age` 在索引里，可以通过 ICP 在回表前过滤。

执行计划可能出现：

```text
Extra: Using index condition
```

## 覆盖索引

覆盖索引指查询需要的列都在同一个索引中，无需回表。

示例：

```sql
SELECT user_id, status, created_at
FROM t_order
WHERE user_id = 1001
  AND status = 1
ORDER BY created_at DESC
LIMIT 20;
```

如果使用索引：

```sql
KEY idx_user_status_time (user_id, status, created_at)
```

查询列都在索引中，并且二级索引叶子节点还包含主键 `id`。执行计划可能出现：

```text
Extra: Using where; Using index
```

`Using index` 表示覆盖索引。

## ICP 和覆盖索引区别

| 对比项 | 索引下推 ICP | 覆盖索引 |
| --- | --- | --- |
| 目的 | 回表前先过滤 | 避免回表 |
| 前提 | 剩余条件涉及索引列 | 查询列都在索引中 |
| 执行计划 | `Using index condition` | `Using index` |
| 是否一定不回表 | 不一定 | 通常不需要回表 |

二者可以同时出现，也可以分别出现。

## ORDER BY 和索引

索引能否支持排序，取决于排序列是否匹配索引有序前缀。

索引：

```text
(user_id, status, created_at)
```

可以支持：

```sql
SELECT *
FROM t_order
WHERE user_id = 1001
  AND status = 1
ORDER BY created_at;
```

因为 `user_id`、`status` 是等值，剩余范围内按 `created_at` 有序。

可能不能支持：

```sql
SELECT *
FROM t_order
WHERE user_id = 1001
ORDER BY created_at;
```

因为同一 `user_id` 下先按 `status` 排，再按 `created_at` 排。跨不同 `status` 时，`created_at` 不是全局有序。

## GROUP BY 和索引

`GROUP BY` 也可以利用索引顺序。

索引：

```text
(user_id, status, created_at)
```

适合：

```sql
SELECT user_id, status, COUNT(*)
FROM t_order
GROUP BY user_id, status;
```

不适合跳列分组：

```sql
SELECT status, COUNT(*)
FROM t_order
GROUP BY status;
```

除非另有以 `status` 开头的索引。

## SQL 示例

创建测试表：

```sql
CREATE TABLE t_trade (
    id BIGINT PRIMARY KEY,
    buyer_id BIGINT NOT NULL,
    seller_id BIGINT NOT NULL,
    status TINYINT NOT NULL,
    pay_time DATETIME NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    KEY idx_buyer_status_pay (buyer_id, status, pay_time),
    KEY idx_seller_pay (seller_id, pay_time)
) ENGINE = InnoDB;
```

高效查询：

```sql
EXPLAIN SELECT id, buyer_id, status, pay_time
FROM t_trade
WHERE buyer_id = 1001
  AND status = 2
  AND pay_time >= '2026-01-01'
ORDER BY pay_time
LIMIT 20;
```

可能出现文件排序：

```sql
EXPLAIN SELECT *
FROM t_trade
WHERE buyer_id = 1001
ORDER BY pay_time
LIMIT 20;
```

因为 `status` 位于 `buyer_id` 和 `pay_time` 中间，`pay_time` 在多个 status 下不是整体有序。

覆盖索引：

```sql
EXPLAIN SELECT id, buyer_id, status, pay_time
FROM t_trade
WHERE buyer_id = 1001
  AND status = 2
ORDER BY pay_time
LIMIT 20;
```

`id` 是二级索引附带主键，查询列可被 `idx_buyer_status_pay` 覆盖。

## 排查与优化

### 看 key_len

`key_len` 可以帮助判断联合索引用到了多少列，但它是字节长度估算，不是最终结论。

```sql
EXPLAIN SELECT ...
```

重点看：

- `key`
- `key_len`
- `rows`
- `Extra`

### 调整联合索引顺序

常用经验：

- 等值过滤列放前面。
- 高选择性列优先，但要结合排序、分组和范围。
- 范围列通常放在等值列后面。
- 排序列要尽量跟在等值列后面。
- 不要只按单列选择性设计，要按核心 SQL 模式设计。

### 避免索引失效写法

常见问题：

```sql
-- 在索引列上做函数
WHERE DATE(pay_time) = '2026-01-01'

-- 隐式类型转换
WHERE buyer_id = '1001'

-- 前置模糊匹配
WHERE name LIKE '%abc'

-- 不连续使用联合索引
WHERE status = 1
```

改写示例：

```sql
WHERE pay_time >= '2026-01-01'
  AND pay_time < '2026-01-02'
```

## 常见追问

### `WHERE a = ? AND c = ?` 能用 `(a, b, c)` 吗

可以用 `a` 做定位，`c` 不能跳过 `b` 作为连续前缀定位条件。但如果 `c` 在索引里，可能通过 ICP 做过滤。

### 范围条件后面的列一定没用吗

不是。它通常不能继续用于缩小索引扫描范围或保证有序输出，但可能用于索引下推过滤，也可能用于覆盖索引避免回表。

### `Using filesort` 一定很慢吗

不一定。少量数据内存排序可能很快。但大数据排序、磁盘临时文件、深分页排序会明显变慢。面试回答要结合行数、内存、LIMIT 和索引设计。

### `WHERE` 条件顺序会影响联合索引吗

SQL 文本中条件顺序通常不影响，优化器会重排。真正影响的是联合索引列顺序和条件类型。

## 易错点

- 误以为联合索引中只要出现列就都能高效定位。
- 误以为范围条件后的列完全无用。
- 误把 `Using index condition` 当成覆盖索引。
- 误以为 `ORDER BY` 中列出现在索引里就一定不排序。
- 误以为索引列顺序只看区分度，不看查询模式。

## 自检清单

- 能否判断 `(a,b,c)` 对不同 SQL 用到哪些前缀。
- 能否解释等值条件和范围条件对后续列的影响。
- 能否区分 ICP 与覆盖索引。
- 能否通过 `Extra` 判断 `Using index` 和 `Using index condition`。
- 能否设计同时兼顾过滤、排序、分页的联合索引。
- 能否解释为什么有索引仍然 `Using filesort`。
