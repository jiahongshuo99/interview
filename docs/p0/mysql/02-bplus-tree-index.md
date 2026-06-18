# B+Tree 索引

## 面试定位

B+Tree 索引是 MySQL 面试最高频主题之一。面试官通常会从“为什么 MySQL 用 B+Tree”开始，继续追问：

- B+Tree 和 BTree、Hash、红黑树有什么区别。
- InnoDB B+Tree 节点和页是什么关系。
- 一次索引查询大概会访问几层。
- 范围查询为什么适合 B+Tree。
- 索引为什么会页分裂、页合并。
- 什么情况下索引反而不快。

回答主线：MySQL 面向磁盘和 Buffer Pool 访问，核心目标是减少随机 IO。B+Tree 分叉多、高度低、叶子节点有序并通过链表连接，非常适合等值查询、范围查询、排序和分组。

## B+Tree 是什么

B+Tree 是多路平衡搜索树。InnoDB 中每个 B+Tree 节点基本对应一个索引页，默认 16KB。

特点：

- 所有叶子节点在同一层，树高稳定。
- 非叶子节点只存索引键和子页指针，不存完整行。
- 叶子节点存实际索引条目。
- 叶子节点之间按键值顺序形成双向链表。
- 支持从根到叶子的等值查找，也支持叶子链表范围扫描。

简化结构：

```text
root page
  -> internal page
     -> leaf page <-> leaf page <-> leaf page
```

## 为什么不用二叉树或红黑树

二叉树、红黑树每个节点分叉少。数据量大时树高很高。

如果一个节点只存一个 key，千万级数据可能需要二十多层。磁盘场景下每层可能对应一次页访问，随机 IO 成本很高。

B+Tree 一个页可以放很多 key，分叉非常大，树高通常只有 3 到 4 层。

```text
二叉树：分叉少，高度高
B+Tree：分叉多，高度低
```

## 为什么不用 Hash 索引

Hash 查询等值很快，但不适合 MySQL 通用索引：

- 不支持范围查询。
- 不支持排序。
- 不支持最左前缀匹配。
- 不适合 `LIKE 'abc%'` 这种有序前缀查找。
- Hash 冲突需要额外处理。

InnoDB 有自适应哈希索引，但它是 Buffer Pool 内部的自适应优化，不是用户建表时设计的通用索引结构。

## 为什么不用普通 BTree

这里的 BTree 通常指 B-Tree。B-Tree 的非叶子节点也可能存数据记录。B+Tree 把数据都放在叶子节点，非叶子节点可以存更多 key。

B+Tree 更适合数据库：

- 非叶子节点更小，树更矮。
- 所有数据在叶子层，范围扫描可以沿叶子链表顺序走。
- 查询性能更稳定。

## InnoDB B+Tree 节点内容

### 聚簇索引

聚簇索引叶子节点存完整行：

```text
primary key -> full row
```

例如：

```sql
CREATE TABLE t_order (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status TINYINT NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    created_at DATETIME NOT NULL,
    KEY idx_user_time (user_id, created_at)
) ENGINE = InnoDB;
```

聚簇索引 B+Tree：

```text
id -> id, user_id, status, amount, created_at, hidden columns
```

### 二级索引

二级索引叶子节点存二级索引键和主键值：

```text
(user_id, created_at) -> id
```

如果通过二级索引查到 `id` 后还要读取其他列，需要再回到聚簇索引查完整行，这就是回表。

## 树高和 IO

假设一个索引页能容纳几百到上千个键，B+Tree 高度通常很低：

```text
root -> internal -> leaf
```

线上查询不是每层都一定读磁盘，因为根页、热点内部页、热点叶子页很可能在 Buffer Pool 中。

所以索引查询成本大致来自：

- 树高。
- 是否命中 Buffer Pool。
- 是否需要回表。
- 范围扫描行数。
- 是否要排序、临时表。

## 等值查询

示例：

```sql
SELECT *
FROM t_order
WHERE id = 1001;
```

走聚簇索引：

```text
root page
  -> internal page
     -> leaf page
        -> row
```

查看执行计划：

```sql
EXPLAIN SELECT *
FROM t_order
WHERE id = 1001;
```

常见表现：

- `type = const`
- `key = PRIMARY`
- `rows = 1`

## 范围查询

示例：

```sql
SELECT id, user_id, created_at
FROM t_order
WHERE user_id = 100
  AND created_at >= '2026-01-01'
  AND created_at < '2026-02-01'
ORDER BY created_at;
```

如果有联合索引：

```sql
KEY idx_user_time (user_id, created_at)
```

B+Tree 可以先定位 `user_id = 100` 且 `created_at` 的起始位置，然后沿叶子链表向后扫描，直到范围结束。

这就是 B+Tree 对范围查询友好的原因。

## 排序与分组

B+Tree 天然按索引键有序。

如果查询顺序和索引顺序一致，MySQL 可能避免额外排序：

```sql
SELECT *
FROM t_order
WHERE user_id = 100
ORDER BY created_at
LIMIT 20;
```

索引：

```sql
KEY idx_user_time (user_id, created_at)
```

当 `user_id` 是等值条件时，`created_at` 在同一用户范围内有序，可以支持 `ORDER BY created_at`。

## 页分裂与页合并

### 页分裂

向 B+Tree 中插入记录时，如果目标叶子页已满，需要分裂：

```text
old leaf page
  -> split into old page + new page
  -> move part of records
  -> update parent page
```

随机主键、随机索引值更容易造成中间页分裂。

影响：

- 写入放大。
- 页空间利用率下降。
- B+Tree 维护成本增加。
- Buffer Pool 热点变差。

### 页合并

删除大量记录后，如果页内数据很少，InnoDB 可能尝试页合并。

删除不一定马上释放磁盘空间，但页内空间可复用。

## 索引的代价

索引不是越多越好。

代价包括：

- 占用磁盘空间。
- 插入、更新、删除时要维护每个索引。
- 二级索引越多，写入越慢。
- 低选择性索引可能被优化器放弃。
- 索引列过宽会降低页内记录密度。
- 索引可能让优化器误判，需要更新统计信息。

## SQL 示例

建表和索引：

```sql
CREATE TABLE t_product (
    id BIGINT PRIMARY KEY,
    category_id BIGINT NOT NULL,
    brand_id BIGINT NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    status TINYINT NOT NULL,
    created_at DATETIME NOT NULL,
    KEY idx_category_price (category_id, price),
    KEY idx_status_created (status, created_at)
) ENGINE = InnoDB;
```

适合索引的查询：

```sql
EXPLAIN SELECT id, price
FROM t_product
WHERE category_id = 10
  AND price BETWEEN 100 AND 200
ORDER BY price
LIMIT 20;
```

可能不适合单列索引的查询：

```sql
EXPLAIN SELECT *
FROM t_product
WHERE status = 1;
```

如果 `status = 1` 命中大部分数据，优化器可能认为全表扫描比走索引再大量回表更便宜。

## 排查与优化

### 看执行计划

```sql
EXPLAIN SELECT ...
EXPLAIN FORMAT=JSON SELECT ...
EXPLAIN ANALYZE SELECT ...
```

重点看：

- `type`
- `possible_keys`
- `key`
- `key_len`
- `rows`
- `filtered`
- `Extra`

### 看索引基数

```sql
SHOW INDEX FROM t_product;
```

关注：

- `Cardinality`：估算的唯一值数量。
- 索引列顺序。
- 是否存在重复、冗余索引。

统计信息不准时可以评估：

```sql
ANALYZE TABLE t_product;
```

### 设计索引原则

- 高频查询优先。
- 区分度高的列更适合单列索引。
- 联合索引把等值过滤列放前面，范围、排序、分组列结合查询模式设计。
- 覆盖索引可以减少回表，但不要为了覆盖把索引做得过宽。
- 避免在索引列上做函数、隐式转换和前置模糊匹配。
- 同一组列前缀重复的索引要评估是否冗余。

## 常见追问

### B+Tree 查询一定是 O(logN) 吗

理论上定位是 O(logN)，但数据库实际成本不只看复杂度，还看页是否在 Buffer Pool、扫描多少叶子记录、是否回表、是否排序、是否锁等待。

### B+Tree 索引能解决所有慢查询吗

不能。慢查询可能来自锁等待、数据量过大、返回行太多、排序临时表、网络传输、连接池耗尽、SQL 写法、事务长时间占用资源等。

### 低基数字段要不要建索引

单独建索引通常收益有限，尤其命中大量数据并需要回表时可能不走。但低基数字段可以作为联合索引的一部分，例如 `(status, created_at)` 支持状态下按时间范围查询。

### 为什么叶子节点要链起来

为了范围扫描。定位到起始叶子页后，可以沿叶子页链表顺序读取，不需要反复从根节点查找下一条记录。

## 易错点

- 误以为 B+Tree 每个节点只存一个 key。InnoDB 一个页可以存很多 key。
- 误以为走索引就一定快。扫描行数大、回表多、随机 IO 多时可能更慢。
- 误以为 `rows` 是真实扫描行数。`EXPLAIN rows` 是估算值。
- 误以为 Hash 一定比 B+Tree 好。数据库通用查询需要范围、排序和前缀能力。
- 误以为索引只影响查询。写入也要维护索引。

## 自检清单

- 能否解释 B+Tree 为什么适合磁盘数据库。
- 能否画出根页、内部页、叶子页、叶子链表。
- 能否说清楚聚簇索引和二级索引叶子节点分别存什么。
- 能否解释范围查询如何沿叶子链表扫描。
- 能否说明页分裂发生的条件和影响。
- 能否说明索引带来的写入和空间成本。
- 能否结合 `EXPLAIN` 判断索引是否被使用。
