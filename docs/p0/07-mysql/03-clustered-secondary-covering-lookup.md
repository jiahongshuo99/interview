# 聚簇索引、二级索引、回表

## 面试定位

这部分考察你是否真正理解 InnoDB 表数据如何组织。常见问题：

- 聚簇索引是什么，为什么一张表只能有一个。
- 没有主键时 InnoDB 怎么组织数据。
- 二级索引叶子节点存什么。
- 回表是什么，为什么会慢。
- 覆盖索引为什么能减少回表。
- 主键设计为什么影响所有二级索引。

回答主线：InnoDB 的表数据存放在聚簇索引叶子节点上；二级索引叶子节点不存完整行，而是存二级索引列和主键值。通过二级索引查询非索引列时，需要先查二级索引拿主键，再查聚簇索引拿整行，这个过程叫回表。

## 聚簇索引

聚簇索引是按照索引键顺序组织整行数据的 B+Tree。

在 InnoDB 中：

```text
clustered index leaf
  -> primary key
  -> full row columns
  -> hidden trx_id
  -> hidden roll_pointer
```

选择规则：

1. 如果有主键，使用主键作为聚簇索引。
2. 如果没有主键，选择第一个唯一且非空的索引。
3. 如果仍没有，InnoDB 生成隐藏 `row_id` 作为聚簇索引键。

工程建议：每张 InnoDB 表都显式定义主键。

## 为什么一张表只能有一个聚簇索引

因为数据行本身只能按一种物理顺序组织。

如果聚簇索引叶子节点存完整行，那么一张表不可能同时按 `id`、`user_id`、`created_at` 三种顺序各存一份完整数据，否则写入、空间和一致性成本都不可接受。

所以：

- 聚簇索引只有一个。
- 二级索引可以有多个。
- 二级索引通过主键值指向聚簇索引中的完整行。

## 二级索引

二级索引也叫辅助索引、非聚簇索引。

二级索引叶子节点存：

```text
secondary index columns + primary key
```

示例：

```sql
CREATE TABLE t_order (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status TINYINT NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    created_at DATETIME NOT NULL,
    KEY idx_user_status (user_id, status),
    KEY idx_created_at (created_at)
) ENGINE = InnoDB;
```

`idx_user_status` 的叶子节点可以理解为：

```text
(user_id, status, id)
```

`id` 是主键，用于回表。

## 回表

回表流程：

```text
通过二级索引定位记录
  -> 读到主键 id
  -> 再通过聚簇索引按 id 查完整行
```

示例：

```sql
SELECT amount
FROM t_order
WHERE user_id = 1001
  AND status = 1;
```

如果走 `idx_user_status`，二级索引叶子节点只有 `user_id`、`status`、`id`，没有 `amount`，所以需要回表。

执行计划可能类似：

```sql
EXPLAIN SELECT amount
FROM t_order
WHERE user_id = 1001
  AND status = 1;
```

关注：

- `key = idx_user_status`
- `Extra` 通常不会出现 `Using index`

## 覆盖索引

如果查询需要的列都能从某个索引中直接拿到，就不需要回表，这叫覆盖索引。

示例：

```sql
SELECT id, user_id, status
FROM t_order
WHERE user_id = 1001
  AND status = 1;
```

`idx_user_status(user_id, status)` 的叶子节点实际包含 `id`，所以这个查询可以由二级索引覆盖。

执行计划中常见：

```text
Extra: Using index
```

注意：`Using index` 表示覆盖索引，不等于“用了索引”。用了索引看 `key` 字段。

## 回表为什么可能慢

回表慢主要来自随机访问和次数。

如果二级索引命中 10 行，回表 10 次通常没问题。如果命中 100 万行，需要 100 万次聚簇索引查找，成本就很高。

成本包括：

- 二级索引扫描成本。
- 聚簇索引随机查找成本。
- Buffer Pool 未命中导致的随机 IO。
- 返回行太多导致网络和应用处理成本。

优化器会评估：走二级索引再大量回表，是否比全表扫描更贵。

所以低选择性索引有时不被选择。

## 主键大小对二级索引的影响

二级索引叶子节点存主键值。主键越大，每个二级索引项越大。

影响：

- 二级索引占用更多磁盘。
- 每页容纳的索引项变少。
- B+Tree 高度可能增加。
- Buffer Pool 能缓存的有效索引项减少。
- 写入维护成本更高。

所以常见建议是主键尽量短，例如 `BIGINT` 通常比 `VARCHAR(64)` 更适合作为聚簇主键。

## SQL 示例

建表：

```sql
CREATE TABLE t_member (
    id BIGINT PRIMARY KEY,
    phone VARCHAR(20) NOT NULL,
    name VARCHAR(64) NOT NULL,
    city_id BIGINT NOT NULL,
    level TINYINT NOT NULL,
    created_at DATETIME NOT NULL,
    UNIQUE KEY uk_phone (phone),
    KEY idx_city_level (city_id, level)
) ENGINE = InnoDB;
```

主键查询：

```sql
EXPLAIN SELECT *
FROM t_member
WHERE id = 10001;
```

二级索引查询并回表：

```sql
EXPLAIN SELECT name, created_at
FROM t_member
WHERE city_id = 310000
  AND level = 3;
```

覆盖索引：

```sql
EXPLAIN SELECT id, city_id, level
FROM t_member
WHERE city_id = 310000
  AND level = 3;
```

唯一二级索引查询：

```sql
EXPLAIN SELECT id, phone
FROM t_member
WHERE phone = '13800000000';
```

因为 `uk_phone` 叶子节点包含 `phone` 和主键 `id`，只查这两列可以覆盖。

## 排查与优化

### 判断是否回表

看查询列是否被索引覆盖。

```sql
SHOW INDEX FROM t_member;
EXPLAIN SELECT ...
```

如果 `Extra` 有 `Using index`，通常说明覆盖索引。

如果查询列不在索引中，并且走的是二级索引，就需要回表。

### 减少回表

方式：

- 只查询需要的列，避免 `SELECT *`。
- 为高频查询设计覆盖索引。
- 通过更高选择性的条件减少命中行数。
- 避免低选择性条件导致大量回表。
- 分页场景先用覆盖索引拿主键，再回表取详情。

分页优化示例：

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

这个写法让深分页先在覆盖索引里定位少量 `id`，再回表 20 次。

### 覆盖索引不要过度设计

覆盖索引不是越宽越好。过宽索引会：

- 增加磁盘空间。
- 降低页内记录密度。
- 增加写入成本。
- 增加 Buffer Pool 压力。

需要按核心 SQL 设计，不要为了覆盖低频查询牺牲整体写入性能。

## 常见追问

### 二级索引为什么存主键，不存物理地址

如果存物理地址，页分裂、页移动、表重建都会导致地址变化，维护成本高。存主键更稳定，回表通过聚簇索引定位。

### 使用唯一二级索引还会回表吗

看查询列。唯一二级索引能快速定位一条记录，但如果查询列不在该索引中，仍然需要回表。

### `count(*)` 会回表吗

不一定。优化器会选择成本较低的索引扫描。`count(*)` 不需要取完整行，通常可以扫描较小的二级索引，不需要回表读取所有列。

### 覆盖索引和索引下推有什么区别

覆盖索引是查询列都在索引中，可以不回表。索引下推是把部分过滤条件下推到存储引擎，在回表前先过滤，减少回表次数。两者关注点不同，但可以同时出现。

## 易错点

- 误以为所有索引叶子节点都存完整行。只有聚簇索引叶子节点存完整行。
- 误以为走唯一索引就一定不回表。是否回表取决于查询列。
- 误以为 `Using index` 只是“用了索引”。它通常表示覆盖索引。
- 误以为主键大小只影响主键索引。它会影响所有二级索引。
- 误以为覆盖索引可以无脑加列。过宽索引会拖慢写入。

## 自检清单

- 能否说明聚簇索引和二级索引叶子节点分别存什么。
- 能否画出二级索引回表流程。
- 能否解释一张表为什么只有一个聚簇索引。
- 能否说明没有主键时 InnoDB 的选择规则。
- 能否判断一个 SQL 是否能覆盖索引。
- 能否解释低选择性索引为什么可能被优化器放弃。
- 能否说明主键过宽对二级索引的影响。
