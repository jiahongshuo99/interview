# 幂等消费

## 面试定位

幂等消费是 MQ 面试中最重要的落地点。面试官通常会问：

- 为什么消费者必须幂等。
- 用 messageId 做幂等够不够。
- 幂等表怎么设计。
- Redis `SETNX` 能不能保证幂等。
- 消费者调用外部接口时如何避免重复副作用。
- 失败、重试、并发消费下幂等记录状态怎么处理。

核心回答：MQ 通常提供至少一次投递，重复消费不可避免。业务不能依赖“只消费一次”，而要通过业务唯一键、数据库唯一约束、状态机、幂等表等方式，让重复消息不会产生重复副作用。

## 什么是幂等

幂等指同一个操作执行一次和执行多次，最终业务结果一致。

例如发券消息：

```text
OrderPaidEvent(orderId=O1001, userId=U1, couponId=C1)
```

正确结果：

```text
用户 U1 因订单 O1001 获得 1 张 C1 优惠券
```

即使消息被消费 3 次，也不能发 3 张券。

## 为什么会重复消费

重复消费来源很多：

- 消费成功后提交 offset 前宕机。
- 提交 offset 超时。
- 消费者处理太慢触发 rebalance。
- Broker 没收到消费成功确认。
- 生产者发送超时后重试，产生业务重复消息。
- 死信或历史消息被人工重放。

所以幂等不是可选项，而是核心业务消费者的基本要求。

## 幂等键选择

幂等键决定“什么算同一次业务操作”。

常见选择：

| 幂等键 | 优点 | 风险 |
| --- | --- | --- |
| MQ messageId | 获取方便 | 生产者重试生成多条消息时 messageId 可能不同 |
| eventId | 业务事件唯一，推荐 | 需要生产者生成并透传 |
| orderId + eventType | 贴近业务语义 | 同一事件类型是否允许多次要定义清楚 |
| requestId | 能串起上游请求 | 异步补偿或批处理时可能缺失 |
| 业务唯一约束 | 最可靠 | 需要业务表支持唯一索引 |

推荐优先级：

```text
业务唯一约束 > eventId / businessKey > requestId > MQ messageId
```

不要只依赖 MQ messageId 做业务幂等。

## 方案一：业务唯一约束

如果业务天然有唯一关系，优先用数据库唯一索引兜底。

发券表：

```sql
CREATE TABLE user_coupon (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id VARCHAR(64) NOT NULL,
    order_id VARCHAR(64) NOT NULL,
    coupon_id VARCHAR(64) NOT NULL,
    created_at DATETIME NOT NULL,
    UNIQUE KEY uk_order_coupon (order_id, coupon_id)
);
```

消费逻辑：

```java
@Transactional
public void handle(OrderPaidEvent event) {
    try {
        userCouponRepository.insert(event.getUserId(), event.getOrderId(), event.getCouponId());
    } catch (DuplicateKeyException duplicate) {
        log.info("coupon already sent, orderId={}, couponId={}",
                event.getOrderId(), event.getCouponId());
    }
}
```

优点：

- 简单可靠。
- 数据库最终兜底。
- 不需要额外幂等表。

缺点：

- 只适合业务表能表达唯一约束的场景。
- 如果操作涉及多个副作用，单一唯一索引可能不够。

## 方案二：幂等消费表

通用做法是为每个消费组维护消费记录。

```sql
CREATE TABLE mq_consume_record (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    consumer_group VARCHAR(128) NOT NULL,
    biz_key VARCHAR(128) NOT NULL,
    message_id VARCHAR(128) DEFAULT NULL,
    status VARCHAR(32) NOT NULL,
    retry_count INT NOT NULL DEFAULT 0,
    error_msg VARCHAR(512) DEFAULT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_group_biz_key (consumer_group, biz_key)
);
```

状态可设计为：

```text
PROCESSING -> SUCCESS
PROCESSING -> FAILED
FAILED -> PROCESSING -> SUCCESS
```

消费逻辑：

```java
@Transactional
public void consume(OrderPaidEvent event) {
    String group = "coupon-service";
    String bizKey = event.getOrderId() + ":ORDER_PAID";

    boolean first = consumeRecordRepository.tryInsertProcessing(group, bizKey, event.getEventId());
    if (!first) {
        ConsumeRecord record = consumeRecordRepository.selectForUpdate(group, bizKey);
        if ("SUCCESS".equals(record.getStatus())) {
            return;
        }
        if ("PROCESSING".equals(record.getStatus()) && !record.isExpired()) {
            throw new IllegalStateException("message is processing");
        }
        consumeRecordRepository.markProcessing(group, bizKey);
    }

    couponService.sendCoupon(event.getUserId(), event.getOrderId());
    consumeRecordRepository.markSuccess(group, bizKey);
}
```

关键点：

- `consumer_group + biz_key` 建唯一索引。
- 幂等记录和业务处理尽量在同一个本地事务内。
- `PROCESSING` 状态要考虑超时恢复。
- `SUCCESS` 后重复消息直接返回成功。

## 方案三：状态机幂等

如果业务有明确状态流转，可以利用状态机防止重复。

订单状态：

```text
INIT -> PAID -> SHIPPED -> FINISHED
```

消费支付成功消息：

```sql
UPDATE orders
SET status = 'PAID',
    paid_at = NOW()
WHERE order_id = ?
  AND status = 'INIT';
```

如果重复消费：

- 第一次更新行数为 1。
- 第二次因为状态已经是 `PAID`，更新行数为 0。
- 查询当前状态发现已经处理过，返回成功。

Java 示例：

```java
@Transactional
public void handle(OrderPaidEvent event) {
    int updated = orderRepository.markPaid(event.getOrderId());
    if (updated == 0) {
        Order order = orderRepository.findByOrderId(event.getOrderId());
        if ("PAID".equals(order.getStatus())) {
            return;
        }
        throw new IllegalStateException("invalid order status: " + order.getStatus());
    }
}
```

优点：

- 不需要额外消费表。
- 符合业务状态约束。

缺点：

- 只适合状态型业务。
- 状态流转必须严谨，不能随意跳转。

## 方案四：Redis 去重

Redis 常用 `SET NX EX` 做短期去重：

```java
Boolean ok = stringRedisTemplate.opsForValue()
        .setIfAbsent("mq:coupon:" + bizKey, "1", Duration.ofHours(24));

if (Boolean.FALSE.equals(ok)) {
    return;
}

couponService.sendCoupon(event);
```

优点：

- 性能高。
- 适合短窗口重复过滤。

风险：

- Redis 成功后业务失败，消息重试会被误判已处理。
- TTL 到期后重复消息可能再次处理。
- Redis 数据丢失会失去幂等保护。
- 无法替代数据库最终一致约束。

更安全的用法：

- Redis 只做前置防抖。
- 最终幂等仍依赖数据库唯一索引或状态机。
- 业务失败时删除 Redis key 或记录失败状态。

## 外部接口幂等

消费者可能调用外部服务，例如发券、退款、发货。

风险：

```text
消费者调用外部接口成功
  -> 本地记录成功前宕机
  -> 消息重试
  -> 再次调用外部接口
```

治理方式：

- 调用外部接口时传幂等号。
- 外部服务基于幂等号做唯一约束。
- 本地保存请求流水和外部返回结果。
- 状态未知时先查询外部结果，再决定是否重试。

示例：

```java
String requestNo = "refund:" + event.getOrderId();
RefundResult result = paymentClient.refund(new RefundRequest(
        requestNo,
        event.getOrderId(),
        event.getAmount()
));
```

外部服务必须保证同一个 `requestNo` 多次请求只退款一次。

## 事务边界

幂等记录和业务操作最好放在同一个数据库事务中：

```java
@Transactional
public void handle(OrderPaidEvent event) {
    consumeRecordRepository.insertIfAbsent(group, bizKey);
    couponRepository.insertIfAbsent(event.getOrderId(), event.getCouponId());
    consumeRecordRepository.markSuccess(group, bizKey);
}
```

如果业务操作跨多个系统，无法放进同一个事务，就要引入：

- 本地状态表。
- 外部幂等号。
- 查询补偿。
- 最终一致任务。

不要只用内存 Set、JVM 锁或本地缓存做幂等，因为消费者实例可能多机部署，重启后状态也会丢失。

## 线上风险

### 幂等键设计错误

例如只用 `userId` 做发券幂等，结果用户多笔订单只能发一次券。

正确做法是结合业务语义：

```text
orderId + couponActivityId
```

### PROCESSING 状态卡死

消费者插入幂等记录后宕机，状态停在 `PROCESSING`。

处理方式：

- 记录更新时间。
- 超过阈值允许重试抢占。
- 后台任务扫描卡住的记录。

### 幂等记录先成功，业务后失败

如果先标记 `SUCCESS` 再做业务，业务失败后消息不会再重试，造成丢处理。

正确顺序：

```text
插入 PROCESSING
  -> 执行业务
  -> 业务成功后标记 SUCCESS
```

### 幂等表无限增长

幂等记录会持续增长。

处理方式：

- 按时间归档。
- 只保留超过 MQ 最大重放窗口的必要数据。
- 对核心业务保留更长审计周期。
- 分库分表或按月份分区。

## 排查优化

排查重复处理：

- 查消息是否重复投递。
- 查业务唯一键是否相同。
- 查幂等表是否有唯一冲突。
- 查消费组是否发生 rebalance。
- 查外部接口是否支持幂等号。

排查消息一直重试：

- 幂等表是否卡在 `PROCESSING`。
- 业务状态是否不允许当前事件。
- 是否唯一键冲突被当异常抛出。
- 是否外部接口返回已处理但本地未识别。

性能优化：

- 幂等表唯一索引字段尽量短。
- 使用业务 hash 分表。
- 幂等查询和业务更新合并到一次 SQL。
- 对重复消息快速返回，减少下游调用。

## 常见追问

### 用 MQ 的 messageId 做幂等可以吗？

只能作为辅助。生产者发送同一个业务事件时，如果第一次超时但实际成功，第二次重试可能产生新的 messageId。业务上这两条消息仍然代表同一次操作，所以应该用 eventId 或 businessKey。

### Redis `SETNX` 做幂等安全吗？

单独使用不够安全。Redis key 设置成功后业务可能失败，此时消息重试会被挡掉。Redis TTL 到期后历史消息又可能重复处理。核心业务要用数据库唯一约束或状态机兜底。

### 幂等表插入后业务失败怎么办？

不能直接留下永久成功记录。通常先插入 `PROCESSING`，业务成功后改 `SUCCESS`；业务失败改 `FAILED` 或抛异常让 MQ 重试。卡住的 `PROCESSING` 通过超时恢复。

### 消费者幂等会不会影响性能？

会增加一次或多次存储访问，但这是核心业务可靠性的必要成本。可以通过唯一索引、批量处理、分表、状态机更新等方式优化。

## 易错点

- 认为 MQ 不会重复投递。
- 只用 messageId 做幂等。
- 先标记消费成功，再执行业务。
- 用 Redis 幂等替代数据库最终约束。
- 幂等键粒度过粗或过细。
- 幂等表没有处理 `PROCESSING` 超时。
- 外部接口不传幂等号。

## 自检清单

- 是否能解释为什么消费者必须幂等。
- 是否能选择正确业务幂等键。
- 是否能设计幂等消费表和唯一索引。
- 是否能说明 Redis 去重的风险。
- 是否能处理外部接口调用的重复副作用。
- 是否能解释 `PROCESSING`、`SUCCESS`、`FAILED` 状态。
- 是否能从幂等角度排查重复发券、重复扣款、重复退款。
