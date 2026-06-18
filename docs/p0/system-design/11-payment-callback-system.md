# 支付回调系统

## 面试定位

支付回调系统是订单和支付链路的核心一致性题。面试官通常关注：第三方回调重复、乱序、延迟、伪造时如何处理；支付成功后如何更新订单；支付回调和订单关闭并发怎么办；如何对账补偿。

核心回答：

```text
支付回调以渠道支付结果为事实输入，先验签和幂等落库，再用订单状态机条件更新，最后通过事件驱动履约，并用对账补偿兜底。
```

## 需求澄清

先问：

- 支付渠道有哪些：微信、支付宝、银行卡、内部余额。
- 回调是同步通知、异步通知，还是两者都有。
- 渠道是否保证重试，重试频率多久。
- 回调字段有哪些：商户订单号、渠道流水号、金额、状态、签名。
- 是否允许部分支付、退款、撤销。
- 支付成功后要触发哪些下游：订单、库存、权益、通知。
- 订单超时关闭和支付成功并发如何处理。
- 是否要求对账和差错处理。

## 核心模型

核心实体：

- 支付单：本系统发起的支付请求。
- 渠道交易：第三方支付流水。
- 回调日志：每次渠道通知原文和处理结果。
- 订单：支付成功后更新业务状态。
- 支付事件：支付成功、失败、退款等。

状态：

```text
支付单：INIT -> PROCESSING -> SUCCESS / FAILED / CLOSED
订单：WAIT_PAY -> PAID
订单：WAIT_PAY -> CLOSED
```

回调必须幂等，不能因为渠道重复通知而重复履约。

## 架构方案

```text
第三方渠道回调
  -> 网关接入
  -> 参数校验和验签
  -> 查询/落库回调日志
  -> 幂等判断渠道流水
  -> 校验金额、商户号、订单号
  -> 更新支付单状态
  -> 条件更新订单状态
  -> 写支付成功 outbox 事件
  -> 返回渠道成功响应
  -> 下游异步履约
```

重要原则：

- 验签失败直接拒绝。
- 回调原文必须落库，便于审计。
- 支付单和订单更新尽量在本地事务内。
- 下游履约不要阻塞渠道回调响应。
- 给渠道返回成功前要确保本地关键状态已持久化。

## 数据模型

### 支付单

```sql
CREATE TABLE payment_order (
    id BIGINT PRIMARY KEY,
    pay_order_no VARCHAR(64) NOT NULL,
    biz_order_no VARCHAR(64) NOT NULL,
    user_id BIGINT NOT NULL,
    channel VARCHAR(32) NOT NULL,
    amount BIGINT NOT NULL,
    status VARCHAR(32) NOT NULL,
    channel_trade_no VARCHAR(128) DEFAULT NULL,
    paid_at DATETIME DEFAULT NULL,
    closed_at DATETIME DEFAULT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_pay_order_no (pay_order_no),
    UNIQUE KEY uk_channel_trade_no (channel, channel_trade_no),
    KEY idx_biz_order_no (biz_order_no)
);
```

### 回调日志

```sql
CREATE TABLE payment_callback_log (
    id BIGINT PRIMARY KEY,
    channel VARCHAR(32) NOT NULL,
    pay_order_no VARCHAR(64) NOT NULL,
    channel_trade_no VARCHAR(128) NOT NULL,
    callback_status VARCHAR(32) NOT NULL,
    request_body TEXT NOT NULL,
    signature VARCHAR(512) DEFAULT NULL,
    handle_status VARCHAR(32) NOT NULL,
    error_msg VARCHAR(512) DEFAULT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_channel_trade (channel, channel_trade_no),
    KEY idx_pay_order_no (pay_order_no)
);
```

### 支付事件表

```sql
CREATE TABLE payment_event_outbox (
    id BIGINT PRIMARY KEY,
    event_id VARCHAR(64) NOT NULL,
    pay_order_no VARCHAR(64) NOT NULL,
    biz_order_no VARCHAR(64) NOT NULL,
    event_type VARCHAR(64) NOT NULL,
    payload TEXT NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_event_id (event_id),
    KEY idx_status_created (status, created_at)
);
```

## 回调处理链路

### 验签和校验

必须校验：

- 签名是否合法。
- 商户号是否匹配。
- 回调订单号是否存在。
- 金额是否和支付单一致。
- 渠道交易号是否重复。
- 支付状态是否为成功。

金额不一致不能更新成功，要进入异常处理。

### 幂等处理

```text
以 channel + channel_trade_no 建唯一键
重复回调：
  如果支付单已 SUCCESS，直接返回渠道成功
  如果处理中，返回稍后重试或内部重试
  如果状态冲突，记录异常
```

也可以用 `pay_order_no + callback_status` 做辅助幂等，但渠道流水号更关键。

### 状态更新

支付单：

```sql
UPDATE payment_order
SET status = 'SUCCESS',
    channel_trade_no = ?,
    paid_at = ?,
    updated_at = NOW()
WHERE pay_order_no = ?
  AND status IN ('INIT', 'PROCESSING');
```

订单：

```sql
UPDATE orders
SET status = 'PAID',
    paid_at = NOW(),
    updated_at = NOW()
WHERE order_no = ?
  AND status = 'WAIT_PAY';
```

如果订单已经 `PAID`，重复回调返回成功。如果订单已经 `CLOSED`，要根据渠道支付事实触发退款或人工差错处理。

## 缓存、MQ 与一致性

支付回调不依赖缓存判断最终状态，必须读写 DB。

MQ 用于后续异步履约：

- 支付成功发放权益。
- 确认库存。
- 发送通知。
- 数据分析。

一致性：

```text
支付单 SUCCESS + 订单 PAID + 支付成功事件
```

三者建议通过本地事务 + outbox 保证。消费者重复收到支付成功事件，必须按订单号或 eventId 幂等。

## 对账与补偿

支付系统必须有对账：

```text
下载渠道账单
  -> 匹配 payment_order
  -> 比较金额、状态、时间
  -> 发现本地未成功但渠道成功
  -> 补偿更新支付单和订单
  -> 发现本地成功但渠道失败
  -> 标记差错人工处理
```

补偿任务：

- 扫描 `PROCESSING` 超时支付单，主动查询渠道。
- 扫描订单 `WAIT_PAY` 但支付单 `SUCCESS` 的异常记录。
- 扫描回调处理失败日志重试。

## 容量估算

支付回调量通常接近支付请求量，但要考虑渠道重复通知。

```text
日支付 1000 万
平均每笔回调 1.2 次
日回调 1200 万
峰值系数 5
峰值回调 QPS ≈ 1200 万 / 86400 * 5 ≈ 694
```

支付回调链路不一定 QPS 极高，但一致性和可靠性要求很高。表设计要按长期审计保留，回调日志可能比支付单更大。

## 风险与降级

- 验签缺失：伪造支付成功。
- 金额不校验：小额支付冒充大额订单。
- 回调重复：重复发货、重复发券。
- 回调乱序：失败覆盖成功。
- 订单关闭并发：支付成功后订单已关闭。
- MQ 发送失败：支付成功但下游不履约。
- 对账缺失：长期账实不符。

降级原则：

- 支付回调不能随便降级为成功。
- 下游通知、积分、推荐可以延迟。
- 支付状态不确定时展示处理中，后台查询渠道确认。

## 常见追问

### 为什么要保存回调原文？

用于验签复查、对账、审计、排查渠道问题和人工差错处理。

### 支付成功但订单已关闭怎么办？

以渠道支付成功为事实，不能忽略。通常触发退款流程或人工差错处理，同时记录异常日志。

### 渠道重复回调怎么处理？

用 `channel + channel_trade_no` 唯一键和支付单状态机。已成功的重复回调直接返回成功。

### 回调处理中下游履约失败怎么办？

回调只负责更新支付和订单关键状态，并写 outbox 事件。下游履约失败由 MQ 重试、死信和补偿处理。

### 支付状态以谁为准？

渠道支付结果是外部事实源，本地支付单是系统内事实源。两者不一致时通过主动查询和对账修复。

## 自检清单

- 是否验签、校验商户、金额、订单号。
- 是否保存回调原文。
- 是否用渠道流水做幂等。
- 是否有支付单和订单状态机。
- 是否处理订单关闭并发。
- 是否用 outbox 发支付成功事件。
- 是否有对账和补偿。
- 是否说明下游履约异步且幂等。
