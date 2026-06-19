# 表结构设计

## 面试定位

表结构设计考察的是业务建模能力。Java 后端面试里，面试官通常希望你能从需求抽象实体、关系、状态和流水，并能解释主键、唯一键、索引、事务、分库分表、冷热归档和数据一致性。

好的回答不是直接画几张表，而是先讲清：

```text
业务对象 -> 核心状态 -> 读写场景 -> 一致性要求 -> 表结构 -> 索引与扩展
```

## 需求澄清

设计表之前先问：

- 核心实体是什么，例如订单、商品、用户、优惠券、支付单。
- 实体之间是一对一、一对多，还是多对多。
- 主要写入场景是什么，创建、更新状态、追加流水还是批量导入。
- 主要查询场景是什么，按 ID 查、按用户查、按状态查、按时间分页还是聚合统计。
- 数据量级、增长速度、保留周期是多少。
- 哪些字段强一致，哪些可以异步补齐。
- 是否需要审计、对账、追踪、补偿。
- 是否有状态机，状态是否允许回退。
- 是否需要分库分表、归档、脱敏。

## 核心模型

常见表类型：

| 表类型 | 作用 | 示例 |
| --- | --- | --- |
| 主体表 | 保存核心对象当前状态 | orders、coupon、file_meta |
| 明细表 | 保存一对多明细 | order_item、coupon_rule |
| 关系表 | 保存多对多关系 | user_role、user_coupon |
| 流水表 | 保存不可变业务流水 | account_flow、stock_flow |
| 状态流转表 | 记录状态变化历史 | order_status_log |
| 幂等表 | 防重复请求 | idempotent_record |
| 消息表 | 本地消息或 outbox | event_outbox |
| 任务表 | 异步任务和补偿 | retry_task |

面试里要强调：主体表保存当前态，流水表保存事实记录，状态日志用于审计和排障，不要用一张大表混合所有职责。

## 设计原则

### 主键

主键常见选择：

- 自增 ID：简单，单库友好，但分库分表不方便。
- 雪花 ID：全局唯一，适合分布式写入，但趋势递增依赖时钟。
- 业务号：例如订单号、支付单号，适合对外暴露，但不要把复杂业务编码强绑定到主键。

建议：

- 内部主键用 `BIGINT`。
- 对外业务号单独建唯一索引。
- 不要用手机号、邮箱等可变敏感信息做主键。

### 唯一键

唯一键用于表达业务不变量，例如：

- 一个订单号只能有一条订单。
- 一个支付回调流水只能处理一次。
- 一个用户对同一活动只能领取一次优惠券。

示例：

```sql
UNIQUE KEY uk_order_no (order_no),
UNIQUE KEY uk_user_activity (user_id, activity_id)
```

唯一键是幂等和并发安全的重要兜底。

### 状态字段

状态字段要有明确状态机，不要随意用布尔值堆叠。

示例：

```text
订单：CREATED -> PAYING -> PAID -> FULFILLED -> CLOSED
支付：INIT -> PROCESSING -> SUCCESS / FAILED
优惠券：UNUSED -> USED -> EXPIRED
```

状态更新要使用条件更新：

```sql
UPDATE orders
SET status = 'PAID', paid_at = NOW()
WHERE order_no = ?
  AND status = 'CREATED';
```

### 时间字段

常见字段：

- `created_at`：创建时间。
- `updated_at`：更新时间。
- `deleted_at`：软删除时间。
- `expire_at`：过期时间。
- `paid_at`、`used_at`：业务事件时间。

分页和归档通常依赖时间字段，必须建好索引。

### 扩展字段

可以使用 JSON 扩展字段，但不能滥用：

- 适合低频读取、非查询条件、兼容历史字段。
- 不适合高频筛选、排序、聚合、强约束字段。

高频字段应该显式列化，并建立索引。

## 示例：订单表设计

### 订单主表

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    order_no VARCHAR(64) NOT NULL,
    user_id BIGINT NOT NULL,
    status VARCHAR(32) NOT NULL,
    total_amount BIGINT NOT NULL,
    pay_amount BIGINT NOT NULL,
    currency VARCHAR(16) NOT NULL DEFAULT 'CNY',
    pay_order_no VARCHAR(64) DEFAULT NULL,
    remark VARCHAR(512) DEFAULT NULL,
    created_at DATETIME NOT NULL,
    paid_at DATETIME DEFAULT NULL,
    closed_at DATETIME DEFAULT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_order_no (order_no),
    KEY idx_user_created (user_id, created_at),
    KEY idx_status_created (status, created_at)
);
```

说明：

- `id` 是内部主键。
- `order_no` 是业务唯一号。
- 金额用分为单位的整数，避免浮点误差。
- `idx_user_created` 支持用户订单列表。
- `idx_status_created` 支持后台扫描待处理订单。

### 订单明细表

```sql
CREATE TABLE order_item (
    id BIGINT PRIMARY KEY,
    order_no VARCHAR(64) NOT NULL,
    sku_id BIGINT NOT NULL,
    sku_name VARCHAR(256) NOT NULL,
    unit_price BIGINT NOT NULL,
    quantity INT NOT NULL,
    total_amount BIGINT NOT NULL,
    created_at DATETIME NOT NULL,
    KEY idx_order_no (order_no),
    KEY idx_sku_created (sku_id, created_at)
);
```

商品名、单价要快照保存，避免商品后续改名影响历史订单。

### 状态日志表

```sql
CREATE TABLE order_status_log (
    id BIGINT PRIMARY KEY,
    order_no VARCHAR(64) NOT NULL,
    from_status VARCHAR(32) DEFAULT NULL,
    to_status VARCHAR(32) NOT NULL,
    operator_type VARCHAR(32) NOT NULL,
    operator_id VARCHAR(64) DEFAULT NULL,
    reason VARCHAR(256) DEFAULT NULL,
    created_at DATETIME NOT NULL,
    KEY idx_order_no (order_no)
);
```

日志表只追加，不更新，便于审计和问题追踪。

### 本地消息表

```sql
CREATE TABLE event_outbox (
    id BIGINT PRIMARY KEY,
    event_id VARCHAR(64) NOT NULL,
    aggregate_type VARCHAR(64) NOT NULL,
    aggregate_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(64) NOT NULL,
    payload TEXT NOT NULL,
    status VARCHAR(32) NOT NULL,
    retry_count INT NOT NULL DEFAULT 0,
    next_retry_at DATETIME DEFAULT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_event_id (event_id),
    KEY idx_status_retry (status, next_retry_at)
);
```

用于解决本地事务成功但 MQ 发送失败的问题。

## 读写链路

### 写链路

```text
参数校验
  -> 生成业务号
  -> 插入主体表
  -> 插入明细/流水/状态日志
  -> 插入 outbox 消息
  -> 提交本地事务
  -> 异步投递 MQ
```

关键点：

- 核心表和流水表放在同一个本地事务内。
- 外部 RPC 不要放在数据库事务里长时间执行。
- 幂等依赖唯一键和状态机。
- MQ 发送用 outbox 或事务消息兜底。

### 读链路

```text
按业务 ID 查询 -> 主表唯一索引
用户列表分页 -> user_id + created_at 组合索引
后台状态扫描 -> status + created_at 组合索引
统计分析 -> 离线数仓或 OLAP，不直接扫业务库
```

分页建议：

- 小数据量可以 `limit offset`。
- 大数据量用游标分页，例如 `created_at < lastCreatedAt and id < lastId`。
- 排序字段必须和索引匹配。

## 缓存、MQ 与一致性

表设计要为缓存和 MQ 留接口：

- 主表有 `updated_at` 或 `version`，支持缓存版本比较。
- 状态流转日志能补偿丢失事件。
- outbox 表保证业务写入和消息事件同事务。
- 幂等表保证重复请求和重复消息不产生副作用。
- 软删除数据要同步删除或失效缓存。

缓存一致性常用策略：

```text
先更新 DB -> 再删除缓存 -> 失败时重试/订阅 binlog 补偿
```

不要在表结构里只考虑 DB，还要考虑后续事件、缓存和排障。

## 容量估算

表结构容量要看：

- 单行大小。
- 日新增行数。
- 保留周期。
- 索引数量和索引字段大小。
- 分库分表键是否均匀。
- 历史数据是否归档。

示例：

```text
日订单 1000 万
订单主表单行 1KB
保留 3 年约 1000 天
原始数据约 10TB
考虑索引和副本，实际可能超过 40TB
```

此时不能只建单表，需要按 `user_id` 或 `order_no` 分库分表，并把历史订单归档到冷存储。

## 风险与降级

- 唯一键缺失：并发下重复创建。
- 索引过多：写入变慢、存储膨胀。
- 索引不匹配：线上慢 SQL。
- 状态机不严谨：订单重复支付、重复发货。
- 大字段混在主表：列表查询拖慢。
- JSON 字段滥用：后续无法高效查询。
- 单表无限增长：DDL、备份、查询都变慢。
- 分表键选择错误：热点分片或跨分片查询严重。

降级思路：

- 非核心字段异步补齐。
- 查询走缓存或只返回摘要。
- 后台统计走离线，不压业务库。
- 历史数据查归档库。

## 常见追问

### 为什么金额不用 double？

浮点数有精度问题。金额应该用整数分或 `BigDecimal`，数据库建议用 `BIGINT` 分或 `DECIMAL`。

### 为什么要有状态日志表？

主表只保存当前状态，排查问题时需要知道状态如何变化。状态日志也能支持审计、补偿和对账。

### 唯一索引和幂等表怎么选？

如果业务表天然能表达唯一约束，优先用唯一索引。如果需要记录请求状态、响应结果、参数哈希，使用独立幂等表。

### 分库分表键怎么选？

优先选择高频查询维度和高基数字段，例如 `user_id`、`order_no`。要避免低基数字段如 `status`，也要考虑跨分片查询和热点用户。

### 软删除还是硬删除？

业务主数据一般软删除，便于恢复和审计。流水、订单、支付等强审计数据不要物理删除，走归档或脱敏。

## 自检清单

- 是否识别了主体表、明细表、流水表、状态日志表。
- 是否定义了主键、业务唯一键和核心索引。
- 是否有明确状态机和条件更新。
- 是否考虑了幂等、事务和 MQ 事件。
- 是否区分了当前态和历史流水。
- 是否避免大字段拖慢高频查询。
- 是否估算了单表规模和归档策略。
- 是否考虑分库分表键、热点和跨分片查询。
- 是否说明了缓存失效和数据一致性。
