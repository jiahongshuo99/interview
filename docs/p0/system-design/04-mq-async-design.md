# MQ 异步化

## 面试定位

MQ 异步化是系统设计里解决削峰、解耦、最终一致和异步扩展的核心手段。面试官通常会追问：为什么要用 MQ、消息会不会丢、会不会重复、顺序如何保证、堆积怎么办、如何和数据库事务一致。

一句话回答：

```text
MQ 不是为了让系统更简单，而是用异步和缓冲换吞吐、可扩展性和隔离性，同时必须处理丢失、重复、乱序和堆积。
```

## 需求澄清

先问：

- 业务是否允许异步，用户是否必须立即看到结果。
- 消息是否允许丢失。
- 是否要求顺序消费。
- 是否需要延迟消息、定时消息。
- 峰值写入 TPS 和平均消费 TPS 是多少。
- 消费失败后重试多久，是否需要死信队列。
- 生产者和消费者是否跨服务、跨语言、跨机房。
- 是否有事务一致性要求，例如订单创建成功必须发消息。

## 核心模型

基础角色：

```text
Producer -> Topic/Queue -> Broker -> Consumer Group -> Consumer
```

关键概念：

- Topic：消息主题，按业务事件划分。
- Partition/Queue：并行和顺序的基本单位。
- Consumer Group：同一组内负载均衡消费。
- Offset：消费进度。
- Ack：消费确认。
- Retry：失败重试。
- DLQ：死信队列。

## 适用场景

### 削峰填谷

秒杀下单、批量导入、推送发送等场景，入口流量瞬时很高，下游处理能力有限。MQ 把突发请求变成可控消费。

### 系统解耦

订单支付成功后，积分、优惠券、通知、推荐、数据分析都可以订阅订单事件，而不是订单服务同步调用所有下游。

### 最终一致

主流程写本地事务，异步消息驱动下游完成后续动作。适合可补偿、可重试的业务。

### 异步耗时任务

文件转码、报表生成、消息推送、风控分析等不适合阻塞用户请求。

## 不适用场景

- 用户必须立即拿到强一致结果。
- 操作不可重试且不可补偿。
- 下游失败不能接受最终一致。
- 业务量很小，用 MQ 只会增加复杂度。

## 架构方案

通用异步架构：

```text
业务请求
  -> 参数校验和幂等
  -> 写业务 DB
  -> 写 outbox 消息表
  -> 提交事务
  -> 消息投递任务发送 MQ
  -> 消费者幂等消费
  -> 更新下游状态
```

如果使用事务消息：

```text
发送半消息
  -> 执行本地事务
  -> 提交/回滚消息
  -> Broker 异常时回查本地事务状态
```

面试中要说清：本地事务和 MQ 发送不是天然原子操作，需要 outbox、事务消息或可靠消息最终一致。

## 数据模型

### 业务事件

```json
{
  "eventId": "evt_202606190001",
  "eventType": "ORDER_PAID",
  "aggregateId": "order_1001",
  "occurredAt": "2026-06-19T10:00:00",
  "payload": {
    "orderNo": "order_1001",
    "userId": 9527,
    "amount": 9900
  }
}
```

事件必须有：

- `eventId`：全局唯一，用于幂等。
- `eventType`：事件类型。
- `aggregateId`：业务聚合 ID，例如订单号。
- `occurredAt`：事件发生时间。
- `payload`：业务数据，避免消费者反查过多。

### Outbox 表

```sql
CREATE TABLE event_outbox (
    id BIGINT PRIMARY KEY,
    event_id VARCHAR(64) NOT NULL,
    topic VARCHAR(128) NOT NULL,
    event_type VARCHAR(64) NOT NULL,
    aggregate_id VARCHAR(128) NOT NULL,
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

状态：

```text
INIT -> SENT
INIT -> FAILED -> INIT
SENT -> CONFIRMED
```

### 消费记录表

```sql
CREATE TABLE mq_consume_record (
    id BIGINT PRIMARY KEY,
    consumer_group VARCHAR(128) NOT NULL,
    biz_key VARCHAR(128) NOT NULL,
    event_id VARCHAR(128) NOT NULL,
    status VARCHAR(32) NOT NULL,
    error_msg VARCHAR(512) DEFAULT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_group_biz_key (consumer_group, biz_key)
);
```

用于消费幂等和排障。

## 生产链路

可靠生产流程：

```text
业务事务中写 outbox
  -> 后台扫描 INIT 消息
  -> 发送 MQ
  -> 发送成功更新 SENT
  -> 发送失败增加 retry_count
  -> 超过阈值报警人工处理
```

关键点：

- 业务数据和 outbox 同事务。
- 发送 MQ 不放在业务事务里等待太久。
- 投递任务要幂等，重复发送由消费者兜底。
- 消息体包含足够上下文，减少消费者反查。

## 消费链路

可靠消费流程：

```text
拉取消息
  -> 反序列化和参数校验
  -> 根据 eventId / bizKey 做幂等
  -> 执行业务处理
  -> 本地事务提交
  -> ack 消息
```

如果消费成功但 ack 失败，消息会重投，所以消费者必须幂等。

## 顺序消息

局部顺序常见于：

- 同一订单状态流转。
- 同一用户账户流水。
- 同一商品库存变更。

方案：

```text
同一个 orderNo / userId / skuId 路由到同一个 partition
同一个 consumer 单线程顺序处理该 partition
```

代价：

- 并行度受 partition 数限制。
- 热点 key 会拖慢整个分区。
- 消费失败会阻塞后续消息。

如果只需要状态机正确，可以用状态条件更新抵御乱序，不一定强制全局顺序。

## 重试和死信

消费失败要区分：

- 可重试异常：网络超时、下游临时不可用、DB 死锁。
- 不可重试异常：参数非法、业务对象不存在、状态非法。

策略：

- 可重试：指数退避，限制最大次数。
- 不可重试：直接记录失败，进入死信或人工处理。
- 重试消息必须保留原始 eventId，避免变成新业务。
- 死信队列要有监控和重放工具。

## 容量估算

估算：

```text
生产 TPS = 业务 TPS * 每笔业务消息数
消息体大小 = header + payload
存储 = 生产 TPS * 消息大小 * 保留时间 * 副本数
消费能力 = 消费者实例数 * 单实例 TPS
```

示例：

```text
订单峰值 TPS = 2000
每单产生 5 条消息
消息 TPS = 10000
单条消息 2KB
保留 3 天
副本 3
存储 ≈ 10000 * 2KB * 259200 * 3 ≈ 15TB
```

消费端还要看：

- 单条消息处理 RT。
- 下游接口容量。
- 消费者线程数。
- 分区数量。
- 堆积追平时间。

## 风险与降级

- 消息丢失：outbox、事务消息、发送确认、补偿扫描。
- 重复消费：幂等表、唯一索引、状态机。
- 消息乱序：按 key 分区、状态机校验、版本号。
- 消息堆积：扩容消费者、限流生产者、降级非核心事件。
- 死信堆积：告警、排查、人工重放。
- 消息体过大：只传必要字段，大对象放存储，消息传引用。
- 消费者雪崩：下游熔断、退避重试、隔离线程池。

降级示例：

```text
订单支付成功：
发货/权益发放优先
短信、推荐、数据分析可以延迟
消息堆积时暂停非核心消费组
```

## 常见追问

### MQ 如何保证不丢消息？

生产端确认发送成功，本地 outbox 补偿未发送消息；Broker 开启持久化和副本；消费端处理成功后再 ack。严格说不是绝对不丢，而是用多层机制把丢失概率降到可接受，并可补偿。

### 如何保证消费幂等？

优先用业务唯一键，其次用 `eventId`、消费记录表、状态机条件更新。不要只依赖 MQ 的 messageId。

### MQ 堆积怎么办？

先定位是生产突增、消费失败还是下游慢。短期扩容消费者、提高并行度、暂停非核心消费、限流生产者；长期优化消费逻辑、拆 topic、增加分区。

### 顺序消息会影响吞吐吗？

会。同一个 key 必须进入同一分区串行处理，并行度下降。能用状态机解决的，不要过度追求全局顺序。

### 事务消息和 outbox 怎么选？

如果 MQ 支持成熟事务消息，可以用事务消息；如果希望方案通用、可审计、可重放，outbox 更稳。两者目标都是解决本地事务和消息发送的一致性。

## 自检清单

- 是否说明为什么需要异步。
- 是否区分可异步和必须同步的业务。
- 是否设计 eventId、eventType、aggregateId。
- 是否解决本地事务和消息发送一致性。
- 是否说明消费者幂等。
- 是否覆盖失败重试、死信、重放。
- 是否说明顺序消息的代价。
- 是否估算消息 TPS、存储、消费能力。
- 是否有堆积、降级和监控方案。
