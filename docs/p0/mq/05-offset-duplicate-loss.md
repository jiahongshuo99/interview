# 消费 offset、重复消费、消息丢失

## 面试定位

消费 offset 是 MQ 可靠消费的核心。面试常问：

- offset 是什么，保存在哪里。
- 自动提交和手动提交有什么区别。
- 为什么 MQ 通常只能保证至少一次。
- 重复消费有哪些来源。
- 消息丢失有哪些来源。
- 如何在消费端做到不丢、不乱、不重复产生副作用。

回答时要把消费拆成两个动作：

```text
执行业务逻辑
提交消费进度
```

只要这两个动作不是一个不可分割的原子事务，就一定存在重复或丢失窗口。

## Offset 是什么

Offset 表示消费者在某个分片上的消费进度。

Kafka 中 offset 是 Partition 内的递增位置：

```text
topic=order-paid, partition=0

offset 100: message A
offset 101: message B
offset 102: message C
```

消费者组提交 offset 后，表示这个组已经消费到某个位置。Kafka 通常提交的是“下一条要消费的 offset”。

```text
committed offset = 103
表示 100、101、102 已经处理完成，下次从 103 开始
```

RocketMQ 中也有按消费组和 MessageQueue 维护的消费进度。

## Offset 维度

Offset 不是 Topic 全局一个数字，而是按消费组和分片维护。

Kafka：

```text
group + topic + partition -> committed offset
```

RocketMQ：

```text
consumer group + topic + queue -> consume offset
```

因此：

- 同一 Topic，不同消费组互不影响。
- 同一消费组，不同 Partition / Queue 进度独立。
- 某个分片慢会导致该分片 lag 增长，不代表所有分片都慢。

## 自动提交

Kafka 可以自动提交 offset：

```properties
enable.auto.commit=true
auto.commit.interval.ms=5000
```

自动提交的问题：

```text
poll 拉到消息
  -> 自动提交 offset
  -> 业务处理
  -> 处理失败
```

如果 offset 已经提交，消息不会再正常投递给这个消费组，造成业务意义上的消息丢失。

自动提交适合：

- 日志类可丢数据。
- 处理逻辑非常简单且失败影响低。

核心业务不建议依赖自动提交。

## 手动提交

推荐核心业务使用手动提交：

```properties
enable.auto.commit=false
```

处理成功后再提交 offset：

```java
@KafkaListener(topics = "order-paid", groupId = "coupon-service")
public void onMessage(OrderPaidEvent event, Acknowledgment ack) {
    couponService.handle(event);
    ack.acknowledge();
}
```

这能降低丢消息风险，但会增加重复消费可能：

```text
poll 拉到消息
  -> 业务处理成功
  -> 提交 offset 前进程宕机
  -> 重启后从旧 offset 再消费
```

所以手动提交通常对应“至少一次”语义，必须配合幂等。

## 消费语义

### 最多一次

先提交 offset，再处理业务：

```text
提交 offset
  -> 处理业务
```

可能丢消息，但不容易重复。

适合：

- 可丢日志。
- 监控采样。

### 至少一次

先处理业务，成功后提交 offset：

```text
处理业务
  -> 提交 offset
```

不容易丢消息，但可能重复。

适合：

- 大多数业务消息。
- 前提是消费者幂等。

### 恰好一次

严格的端到端 exactly once 很难，需要生产、Broker、消费、业务存储一起配合。Kafka 的事务和幂等生产者可以在 Kafka 内部读写链路提供更强语义，但如果消费者还要写 MySQL、调用 RPC，仍然需要业务幂等和本地事务设计。

面试中要谨慎：不要轻易说 MQ 能全局保证“只消费一次”。

## 重复消费来源

### 消费成功但 offset 提交失败

```text
消费者处理业务成功
  -> 提交 offset 超时
  -> 消费者重启
  -> 旧消息重新投递
```

### 消费者处理超时触发重平衡

Kafka 中，如果消费者长时间不 poll，可能超过 `max.poll.interval.ms`：

```text
消费者拉取一批消息
  -> 单批处理很久
  -> 被协调器认为不可用
  -> partition 分配给其他消费者
  -> 原消费者处理完后提交失败
  -> 新消费者重复处理
```

### Broker 没收到消费成功确认

RocketMQ 消费者处理成功但响应 Broker 失败，Broker 会认为消费失败，后续重试。

### 生产者重试导致重复消息

生产者发送超时后重试，而第一次发送其实已经成功。Broker 中可能存在两条业务相同的消息。

### 业务主动重放

排查问题时重置 offset 或重新投递死信，也会导致历史消息再次消费。

## 消息丢失来源

### 发送端丢失

- 异步发送不处理 callback。
- 单向发送。
- `acks=0`。
- 发送异常只打日志。
- 数据库事务成功但消息未发送且无 outbox 记录。

### Broker 丢失

- 异步刷盘时机器掉电。
- 副本未同步时 Leader 宕机。
- Topic 被误删。
- 消息超过保留时间被清理。
- 磁盘故障。

### 消费端丢失

- 先提交 offset 后处理业务。
- 自动提交 offset，但业务处理失败。
- 批量消费时部分失败却提交整批 offset。
- 异常被 catch 后吞掉，框架认为消费成功。
- RocketMQ listener 未抛异常或未返回重试状态。

## Kafka 手动提交示例

### 单条消费

```java
@KafkaListener(topics = "order-paid", groupId = "coupon-service")
public void onMessage(OrderPaidEvent event, Acknowledgment ack) {
    try {
        couponService.handle(event);
        ack.acknowledge();
    } catch (Exception e) {
        log.error("consume order paid failed, orderId={}", event.getOrderId(), e);
        throw e;
    }
}
```

关键点：

- 业务成功后提交。
- 异常不要吞掉。
- 消费逻辑要幂等。

### 批量消费

```java
@KafkaListener(topics = "order-paid", groupId = "coupon-service")
public void onMessage(List<ConsumerRecord<String, OrderPaidEvent>> records,
                      Acknowledgment ack) {
    for (ConsumerRecord<String, OrderPaidEvent> record : records) {
        couponService.handle(record.value());
    }
    ack.acknowledge();
}
```

批量消费风险：

- 第 1 条成功，第 2 条失败，不能提交整批。
- 失败后整批重试，第 1 条会重复。
- 要求单条处理幂等。

## RocketMQ 消费确认示例

原生 RocketMQ 并发消费：

```java
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
    try {
        for (MessageExt msg : msgs) {
            OrderPaidEvent event = parse(msg.getBody());
            couponService.handle(event);
        }
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    } catch (Exception e) {
        log.error("consume failed", e);
        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
    }
});
```

要点：

- 成功才返回 `CONSUME_SUCCESS`。
- 失败返回 `RECONSUME_LATER`。
- 不要 catch 异常后仍返回成功。

## 防丢设计

### 生产端

- 核心消息使用同步发送或可靠异步回调。
- 发送失败要落库、告警或抛异常。
- 数据库事务和消息发送用 outbox 或事务消息衔接。
- 发送结果未知时按未知状态处理。

### Broker

- Kafka 设置合理副本数和 `min.insync.replicas`。
- RocketMQ 核心 Topic 使用同步刷盘或同步复制。
- 禁止未授权删除 Topic。
- 设置合理消息保留时间。

### 消费端

- 业务成功后再提交 offset。
- 异常不要吞掉。
- 批量消费要处理部分失败。
- 消费者必须幂等。
- 失败进入重试和死信。

## 防重复设计

常见幂等键：

- messageId：MQ 生成的消息 ID，但生产者重试生成两条消息时可能不同。
- eventId：业务事件 ID，推荐生产者生成。
- businessKey：例如 `orderId + eventType`。
- requestId：上游请求 ID。

推荐用业务唯一键做幂等，而不是只依赖 MQ messageId。

示例表：

```sql
CREATE TABLE mq_consume_record (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    consumer_group VARCHAR(128) NOT NULL,
    biz_key VARCHAR(128) NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_group_biz_key (consumer_group, biz_key)
);
```

消费时先插入幂等记录，唯一键冲突说明已处理或正在处理。

## 线上排查

### 怀疑重复消费

检查：

- 是否有消费者重启或 rebalance。
- offset 提交是否失败。
- 是否有消费超时。
- 生产端是否重试发送。
- 业务幂等表是否命中唯一冲突。

### 怀疑消息丢失

按链路查：

```text
生产者日志
  -> 是否发送成功
  -> 是否有 outbox 记录

Broker
  -> topic 中是否存在消息
  -> offset 范围是否覆盖
  -> 消息是否过期

消费者
  -> group offset 是否越过该消息
  -> 是否自动提交
  -> 是否异常被吞
  -> 是否进入 retry / DLQ
```

### 怀疑 offset 异常

检查：

- 是否有人 reset offset。
- 是否切换了 consumer group。
- 是否测试消费者用了生产 group。
- 是否变更了 Topic 分区。
- 是否部署了新版本导致 groupId 改变。

## 常见追问

### 手动提交 offset 就不会丢消息吗？

不能绝对保证。手动提交可以避免“业务未处理就提交 offset”的丢失窗口，但如果业务处理里先调用外部系统再失败，或者异常被吞掉，仍可能产生业务不一致。手动提交必须配合事务、幂等和重试。

### 重复消费和消息重复是一回事吗？

不是。消息重复是 Broker 中可能存在多条业务相同的消息；重复消费是同一条消息被同一消费组处理多次。两者都可能发生，最终都要求业务幂等。

### offset 提交失败怎么办？

一般让消息后续重试，依靠幂等处理重复。不要为了避免重复而先提交 offset，因为那会增加丢消息风险。

### 如何做到消息不丢也不重复？

工程上通常是“不丢 + 可重复 + 幂等”。生产端可靠投递，Broker 高可靠配置，消费端成功后提交 offset，业务侧用唯一键或状态机消除重复副作用。

## 易错点

- 自动提交 offset 用在核心业务。
- catch 了消费异常但没有抛出或返回失败。
- 批量消费部分成功后直接提交整批 offset。
- 使用 MQ messageId 做唯一幂等，忽略业务重复发送。
- 认为 offset 提交成功等于业务一定成功。
- 排查时重置 offset 却没有评估重复消费影响。

## 自检清单

- 是否能解释 offset 的维度和含义。
- 是否能说明自动提交和手动提交的风险。
- 是否能列举重复消费来源。
- 是否能列举消息丢失来源。
- 是否能解释至少一次和最多一次。
- 是否能设计消费端幂等表。
- 是否能按生产端、Broker、消费端排查丢消息。
