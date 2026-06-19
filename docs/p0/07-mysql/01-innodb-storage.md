# InnoDB 存储结构

## 面试定位

InnoDB 存储结构是 MySQL 索引、事务、锁、MVCC、崩溃恢复的底座。面试通常不会只问“页是什么”，而是继续追问：

- InnoDB 一行数据到底存在哪里。
- 聚簇索引为什么决定数据组织方式。
- 页、区、段、表空间分别是什么。
- 一页 16KB 里有什么，为什么能二分查找。
- 主键设计为什么会影响插入性能和页分裂。
- 大字段、删除、碎片、Buffer Pool 和磁盘文件有什么关系。

回答主线：InnoDB 按 B+Tree 组织表数据，B+Tree 的基本读写单位是页，页属于区，区组成段，段分布在表空间里。逻辑上的一张表，底层不是“数组行”，而是一棵以页为节点的聚簇索引树。

## 总体层次

从逻辑到物理可以按下面理解：

```text
database
  -> table
     -> clustered index / secondary indexes
        -> segment
           -> extent
              -> page
                 -> record
```

核心概念：

| 层级 | 作用 | 常见大小或特点 |
| --- | --- | --- |
| 表空间 tablespace | 存放表、索引、undo、临时对象等数据 | 系统表空间、独立表空间、undo 表空间、临时表空间 |
| 段 segment | 管理某类对象的空间，例如索引叶子段、非叶子段 | 按需申请区和碎片页 |
| 区 extent | 一组连续页 | 默认 1MB，包含 64 个 16KB 页 |
| 页 page | InnoDB 磁盘和 Buffer Pool 的基本单位 | 默认 16KB |
| 行 record | 存储用户数据和隐藏列 | 受行格式影响 |

## 表空间

表空间是 InnoDB 存储文件的逻辑容器。

常见类型：

- 系统表空间：历史上存放数据字典、undo、change buffer 等，文件通常是 `ibdata1`。
- 独立表空间：开启 `innodb_file_per_table` 后，每张表有自己的 `.ibd` 文件，存放该表的数据和索引。
- undo 表空间：存放 undo log，用于回滚和 MVCC。
- 临时表空间：存放临时表、排序等临时数据。
- 通用表空间：可以让多张表共享一个表空间。

查看配置：

```sql
SHOW VARIABLES LIKE 'innodb_file_per_table';
SHOW VARIABLES LIKE 'innodb_page_size';
SHOW VARIABLES LIKE 'innodb_data_file_path';
```

独立表空间的好处：

- 表删除后更容易回收空间。
- 单表迁移、备份、传输更方便。
- 不同表的数据文件隔离更清晰。

注意：`DELETE` 删除大量数据通常只把页内记录标记为删除，空间可能在表内部复用，不一定立刻归还给操作系统。

## 段、区、页

### 段

段是为了管理某类对象空间而存在的逻辑结构。一个索引通常至少有两个段：

- 叶子节点段：存放 B+Tree 叶子页。
- 非叶子节点段：存放 B+Tree 内部节点页。

这样可以让叶子页和非叶子页分别扩展，减少管理混乱。

### 区

区是连续页的集合。默认页大小是 16KB，一个区是 1MB，所以一个区包含 64 个页。

InnoDB 不会永远一次申请一个完整区。小表或刚开始增长时会先使用碎片页，表变大后再按区扩展。

### 页

页是 InnoDB 读写磁盘、缓存到 Buffer Pool、组织 B+Tree 的基本单位。默认 16KB。

常见页类型：

- 索引页：存放 B+Tree 节点，也是最常见的数据页。
- undo log 页：存放 undo 记录。
- inode 页：管理段信息。
- change buffer 页：和变更缓冲相关。
- 系统页：存放表空间元信息。

## 索引页内部结构

一页 16KB 不是简单地顺序塞行。索引页大致包含：

```text
File Header
Page Header
Infimum + Supremum
User Records
Free Space
Page Directory
File Trailer
```

关键部分：

- `File Header`：页号、上一页、下一页、校验信息等。
- `Page Header`：页内记录数、空闲空间、目录槽数量等。
- `Infimum`：页内最小伪记录。
- `Supremum`：页内最大伪记录。
- `User Records`：真实用户记录。
- `Free Space`：页内空闲空间。
- `Page Directory`：页目录，支持页内快速查找。
- `File Trailer`：校验页是否完整写入。

### 页内记录如何查找

页内记录按索引键有序，并通过单向链表连接。单向链表本身不适合二分查找，所以 InnoDB 在页尾维护 Page Directory。

查找过程可以简化为：

```text
在 B+Tree 中定位到目标页
  -> 在 Page Directory 中二分定位槽
  -> 在槽对应的一小组记录里顺序查找
```

所以 B+Tree 解决页之间定位，页目录解决页内定位。

## 行记录格式

InnoDB 常见行格式：

- `COMPACT`
- `DYNAMIC`
- `COMPRESSED`
- `REDUNDANT`

MySQL 5.7/8.0 常见默认是 `DYNAMIC`。

查看行格式：

```sql
SHOW TABLE STATUS LIKE 't_order'\G
SELECT table_name, row_format
FROM information_schema.tables
WHERE table_schema = DATABASE()
  AND table_name = 't_order';
```

一条 InnoDB 记录除了用户列，还可能包含隐藏列：

- `DB_TRX_ID`：最近一次修改该行的事务 ID。
- `DB_ROLL_PTR`：指向 undo log，用于 MVCC 和回滚。
- `DB_ROW_ID`：如果表没有显式主键，也没有合适的唯一非空索引，InnoDB 自动生成隐藏 row id。

## 大字段如何存储

`VARCHAR`、`TEXT`、`BLOB` 等大字段不一定完整放在聚簇索引页内。

对于 `DYNAMIC` 行格式：

- 小字段尽量放在页内。
- 大字段可能只在页内保留前缀和指针。
- 完整内容放到溢出页。

工程影响：

- 大字段会降低页内记录密度。
- 覆盖索引如果包含大字段，索引会膨胀。
- 查询列表不要无脑 `SELECT *`。
- 频繁访问的大字段和冷热字段可以考虑拆表。

## 聚簇索引组织表数据

InnoDB 表数据本身就是聚簇索引的叶子节点。

```text
clustered index leaf page
  -> primary key
  -> all columns
  -> hidden trx id
  -> roll pointer
```

如果有主键，按主键组织；没有主键，选择第一个唯一非空索引；再没有则使用隐藏 row id。

这解释了几个常见结论：

- InnoDB 表必须有聚簇索引。
- 一张表只能有一个聚簇索引。
- 主键越大，所有二级索引也会变大，因为二级索引叶子节点存主键值。
- 随机主键容易导致页分裂和缓存局部性差。

## Buffer Pool 与磁盘页

InnoDB 不会每次查询都直接访问磁盘文件。数据页会缓存在 Buffer Pool 中。

读流程简化：

```text
访问某条记录
  -> 判断目标页是否在 Buffer Pool
  -> 命中则直接读内存页
  -> 未命中则从磁盘加载页到 Buffer Pool
  -> 在页内查找记录
```

写流程简化：

```text
修改记录所在页
  -> 修改 Buffer Pool 中的数据页
  -> 页变为 dirty page
  -> 先写 redo log 保证崩溃恢复
  -> 后台线程择机刷脏页到磁盘
```

这也是为什么 MySQL 写入不是每改一行就立刻同步完整数据页。

## SQL 示例

建表：

```sql
CREATE TABLE t_user (
    id BIGINT PRIMARY KEY,
    name VARCHAR(64) NOT NULL,
    profile TEXT,
    created_at DATETIME NOT NULL,
    KEY idx_created_at (created_at)
) ENGINE = InnoDB ROW_FORMAT = DYNAMIC;
```

查看表和索引：

```sql
SHOW CREATE TABLE t_user\G
SHOW INDEX FROM t_user\G
SHOW TABLE STATUS LIKE 't_user'\G
```

查看 InnoDB 运行状态：

```sql
SHOW ENGINE INNODB STATUS\G
```

查看 Buffer Pool 相关指标：

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';
SHOW GLOBAL STATUS LIKE 'Innodb_pages%';
```

## 排查与优化

### 主键选择

推荐主键：

- 短。
- 稳定。
- 单调或大体递增。
- 不携带业务含义或业务含义稳定。

常见选择：

- 自增 `BIGINT`：插入局部性好，但分库分表或多主场景要处理 ID 生成。
- 雪花 ID：全局唯一，大体递增，适合分布式系统。
- UUID：随机性强，索引大，插入容易分散，通常不建议直接作为聚簇主键。

### 页分裂

当向一个已满页中间插入记录时，InnoDB 可能分裂页：

```text
原页记录过多
  -> 分配新页
  -> 移动部分记录
  -> 更新页链表和上层索引
```

随机主键、高频插入、页填充紧张时更容易出现页分裂。

优化方向：

- 使用递增或趋势递增主键。
- 避免过宽主键。
- 控制二级索引数量。
- 批量写入尽量按主键或索引顺序。

### 删除后空间不释放

大量 `DELETE` 后：

- 记录被标记删除。
- 页内空间可复用。
- `.ibd` 文件不一定缩小。

如果确实要回收磁盘空间，可以评估：

```sql
OPTIMIZE TABLE t_user;
```

但该操作可能重建表，对线上有风险，需要结合版本、表大小、业务流量评估。

### 大字段治理

优化思路：

- 查询只取需要字段。
- 大字段拆到扩展表。
- 热字段和冷字段分离。
- 避免在大字段上建立普通索引。
- 对长字符串建立前缀索引时评估选择性。

## 常见追问

### InnoDB 和 MyISAM 存储结构有什么区别

InnoDB 数据按聚簇索引组织，叶子节点存完整行；MyISAM 数据文件和索引文件分离，索引叶子节点存数据文件地址。InnoDB 支持事务、行锁、崩溃恢复和 MVCC，是 Java 后端业务表的主流选择。

### 为什么页默认是 16KB

页太小，B+Tree 高度更高，随机 IO 增多；页太大，读放大和内存浪费更明显。16KB 是通用场景的折中。实际也可以初始化实例时选择不同页大小，但不是日常优化优先项。

### 没有主键会怎样

InnoDB 会尝试选择唯一非空索引作为聚簇索引。如果没有，会生成隐藏 row id。问题是业务不可控，二级索引仍要引用聚簇索引键，排查和复制场景也不友好。工程上建议所有 InnoDB 表显式定义主键。

### InnoDB 行锁为什么说锁索引

因为记录是通过索引定位的，锁对象也是索引记录、索引间隙或 next-key 范围。没有合适索引时，扫描范围会扩大，锁范围也可能扩大。

## 易错点

- 误以为 InnoDB 表就是按插入顺序堆放。实际按聚簇索引组织。
- 误以为二级索引叶子节点存完整行。实际存二级索引列和主键值。
- 误以为删除数据后 `.ibd` 一定变小。通常只是表内空间可复用。
- 误以为 `SELECT *` 只影响网络传输。它还可能破坏覆盖索引，加重大字段读取。
- 误以为没有主键也没关系。InnoDB 会生成隐藏聚簇键，但工程可控性更差。

## 自检清单

- 能否画出表空间、段、区、页、行的层次。
- 能否说清楚一页 16KB 内部有哪些主要区域。
- 能否解释 Page Directory 为什么能帮助页内查找。
- 能否说明聚簇索引叶子节点存什么。
- 能否说明二级索引为什么会受主键大小影响。
- 能否解释随机主键为什么影响插入性能。
- 能否区分 Buffer Pool 中的脏页和磁盘上的数据页。
- 能否说出大字段、删除空间、页分裂的常见优化方式。
