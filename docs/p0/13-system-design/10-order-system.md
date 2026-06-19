# 订单系统

## 面试定位

订单系统是 Java 后端系统设计的核心题。它综合考察领域建模、状态机、表结构、库存、优惠、支付、幂等、MQ、分布式一致性、超时关闭、对账和高并发。

回答主线：

```text
订单是交易主状态机，创建订单要保证幂等和库存/价格快照，支付后通过事件驱动履约，异常靠补偿和对账闭环。
```

## 需求澄清

先问：

- 是实物订单、虚拟订单、外卖订单还是权益订单。
- 是否支持购物车、多商品、多商家。
- 是否有优惠券、积分、运费、税费。
- 是否需要库存预占。
- 下单后多久未支付关闭。
- 支付渠道有哪些。
- 是否支持退款、取消、售后。
- 订单量级、峰值 TPS、查询 QPS。
- 强一致点有哪些：订单创建、库存、支付、权益。

## 核心模型

核心实体：

- 订单主单：用户、金额、状态、支付信息。
- 订单明细：SKU、数量、单价、商品快照。
- 价格明细：商品金额、优惠、积分、运费。
- 状态日志：订单状态流转。
- 支付单：支付渠道、支付状态、回调流水。
- 库存流水：预占、确认、释放。
- 订单事件：订单创建、支付成功、关闭、退款。

订单状态：

```text
CREATED -> WAIT_PAY -> PAID -> FULFILLING -> FINISHED
WAIT_PAY -> CLOSED
PAID -> REFUNDING -> REFUNDED
```

状态只能按规则流转，不能随意 update。

## 架构方案

```text
下单请求
  -> 参数校验和幂等
  -> 价格计算
  -> 优惠锁定
  -> 库存预占
  -> 创建订单
  -> 发送订单创建事件
  -> 用户支付
  -> 支付回调更新订单 PAID
  -> 发送支付成功事件
  -> 履约/发货/权益发放
```

服务划分：

- 订单服务：订单状态和主数据。
- 商品服务：商品信息和价格快照来源。
- 库存服务：库存预占、确认、释放。
- 营销服务：优惠券、积分、权益。
- 支付服务：支付单和渠道回调。
- 履约服务：发货或权益发放。

订单服务不应同步强依赖所有下游完成，支付后履约可以事件驱动。

## 数据模型

### 订单主表

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    order_no VARCHAR(64) NOT NULL,
    user_id BIGINT NOT NULL,
    status VARCHAR(32) NOT NULL,
    total_amount BIGINT NOT NULL,
    discount_amount BIGINT NOT NULL,
    pay_amount BIGINT NOT NULL,
    pay_order_no VARCHAR(64) DEFAULT NULL,
    expire_at DATETIME NOT NULL,
    paid_at DATETIME DEFAULT NULL,
    closed_at DATETIME DEFAULT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_order_no (order_no),
    KEY idx_user_created (user_id, created_at),
    KEY idx_status_expire (status, expire_at)
);
```

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
    KEY idx_order_no (order_no)
);
```

商品名和价格要保存快照，不能依赖商品后续变化。

### 状态日志

```sql
CREATE TABLE order_status_log (
    id BIGINT PRIMARY KEY,
    order_no VARCHAR(64) NOT NULL,
    from_status VARCHAR(32) DEFAULT NULL,
    to_status VARCHAR(32) NOT NULL,
    reason VARCHAR(256) DEFAULT NULL,
    created_at DATETIME NOT NULL,
    KEY idx_order_no (order_no)
);
```

### 订单幂等表

```sql
CREATE TABLE order_idempotent (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    request_id VARCHAR(128) NOT NULL,
    order_no VARCHAR(64) DEFAULT NULL,
    request_hash VARCHAR(128) NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_user_request (user_id, request_id)
);
```

## 创建订单链路

```text
收到下单请求
  -> 校验 requestId 幂等
  -> 查询商品和价格
  -> 计算优惠和应付金额
  -> 锁定优惠券/积分
  -> 预占库存
  -> 插入订单主表和明细表
  -> 插入状态日志
  -> 插入 outbox 订单创建事件
  -> 返回订单号和支付参数
```

注意：

- 价格计算结果要保存快照。
- 库存预占和订单创建要有一致性策略。
- 优惠券锁定失败要回滚库存。
- 请求超时后客户端重试必须返回同一订单。

## 支付链路

```text
用户发起支付
  -> 创建支付单
  -> 跳转第三方支付
  -> 接收支付回调
  -> 验签
  -> 回调幂等
  -> 更新支付单 SUCCESS
  -> 更新订单 WAIT_PAY -> PAID
  -> 发送支付成功事件
```

支付回调可能重复，必须用渠道流水号和订单状态机幂等。

## 关闭订单链路

```text
定时任务扫描 WAIT_PAY 且 expire_at 过期订单
  -> 条件更新 WAIT_PAY -> CLOSED
  -> 释放库存
  -> 释放优惠券/积分
  -> 发送订单关闭事件
```

如果支付成功和超时关闭并发：

- 订单状态条件更新决定谁成功。
- 如果关闭成功后又收到支付成功，要走退款或人工处理。
- 支付回调处理前主动查询渠道状态能降低异常。

## 缓存、MQ 与一致性

### 缓存

订单详情可以缓存，但一致性要求较高：

- 用户刚下单或支付后优先读 DB。
- 历史订单列表可以缓存短时间。
- 订单状态变更后删除缓存。

### MQ

订单事件：

- `OrderCreatedEvent`
- `OrderPaidEvent`
- `OrderClosedEvent`
- `OrderRefundedEvent`

下游消费必须幂等，例如发券、积分、通知、履约。

### 一致性

关键一致性：

- 订单和明细：本地事务。
- 订单和库存：预占/确认/释放 + 库存流水 + 补偿。
- 订单和支付：支付单 + 回调幂等 + 对账。
- 订单和优惠：锁定/使用/释放状态机。

## 容量估算

示例：

```text
日订单 1000 万
下单峰值 TPS 3000
订单查询峰值 QPS 30000
订单保留 3 年
```

设计：

- 写链路按 3000 TPS 压测订单库。
- 查询走 `user_id + created_at` 索引和缓存。
- 订单表按用户或订单号分库分表。
- 历史订单归档，近 3 到 6 个月保留在线库。
- 订单事件按每单 3 到 5 条估算 MQ 容量。

## 风险与降级

- 重复下单：requestId 幂等 + 订单唯一键。
- 库存超卖：预占、条件扣减、库存流水。
- 支付回调重复：回调流水唯一键 + 状态机。
- 订单关闭和支付成功并发：状态条件更新。
- MQ 消息丢失：outbox 或事务消息。
- 下游履约失败：重试、死信、补偿、人工处理。
- 订单库压力大：分库分表、读写分离、冷热归档。

降级：

- 订单列表只返回摘要。
- 非核心推荐和营销展示关闭。
- 支付成功后的通知、积分延迟发放。
- 下游异常时订单进入处理中，不返回假成功。

## 常见追问

### 订单号怎么生成？

可以用雪花 ID、号段、数据库序列等生成全局唯一订单号。订单号不要包含过多业务含义，避免规则变更困难。

### 下单时库存怎么扣？

常见是预占库存，支付成功确认，超时未支付释放。秒杀可用 Redis 预扣，普通订单可用库存服务 DB 条件更新或库存流水。

### 支付成功但订单更新失败怎么办？

支付回调要可重试，订单服务根据支付单或渠道对账补偿更新。支付成功是事实源之一，不能因为订单更新失败就丢掉。

### 订单为什么要状态日志？

排查用户投诉、支付异常、履约异常时，需要知道订单从哪个状态、因为什么原因变成当前状态。

### 订单系统如何分库分表？

常按 `user_id` 或 `order_no` 分片。用户订单列表按 userId 友好，按 orderNo 查询需要路由规则或映射表。历史订单要归档。

## 自检清单

- 是否定义了订单主表、明细表、状态日志。
- 是否有清晰订单状态机。
- 是否处理创建订单幂等。
- 是否保存价格和商品快照。
- 是否处理库存预占、确认、释放。
- 是否处理支付回调幂等和对账。
- 是否有订单超时关闭。
- 是否通过 MQ 事件驱动下游。
- 是否有分库分表、归档和查询方案。
