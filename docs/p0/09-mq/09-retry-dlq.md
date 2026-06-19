# 重试、死信队列

## 面试定位

重试和死信队列是 MQ 线上稳定性的关键。常见问题：

- 消费失败后 MQ 会怎么处理。
- 重试是立即重试还是延迟重试。
- Kafka 和 RocketMQ 的重试机制有什么区别。
- 什么消息应该重试，什么消息不应该重试。
- 死信队列有什么用，如何处理死信。
- 如何避免重试风暴。

核心回答：重试用于处理临时失败，死信用于隔离多次失败仍无法处理的消息。重试不能代替业务幂等，死信不能只堆着不处理，必须配套告警、排查、修复和重放机制。

## 失败类型

消费失败先要分类。

### 临时失败

适合重试：

- 下游 RPC 超时。
- 数据库连接池临时满。
- Redis 短暂抖动。
- 网络异常。
- 限流导致稍后可恢复。

### 永久失败

不适合无限重试：

- 消息格式错误。
- 必填字段缺失。
- 业务状态非法且无法恢复。
- 配置错误。
- 代码 bug。
- 下游明确返回不可重试错误。

正确做法：

- 临时失败延迟重试。
- 永久失败尽快进入死信或失败表。
- 所有失败都要有告警和可追踪上下文。

## 重试层次

### 生产者重试

发送消息失败时重试。

风险：

- 上一次发送可能已经成功，重试可能产生重复消息。

要求：

- 消息带业务唯一键。
- 消费者幂等。
- 发送状态未知要可补偿。

### 消费者重试

消费业务失败后重试。

风险：

- 重复调用下游。
- 阻塞顺序消费。
- 短时间大量重试压垮下游。

要求：

- 区分可重试和不可重试异常。
- 设置最大重试次数。
- 使用退避策略。
- 超过阈值进入死信。

### 业务补偿重试

通过失败表、任务表、对账任务补偿。

适合：

- 需要人工确认。
- MQ 重试窗口不够。
- 跨系统状态未知。
- 需要查询外部状态后再处理。

## RocketMQ 重试

RocketMQ 在集群消费模式下，消费者返回失败或抛异常，消息会进入重试流程。

原生 API：

```java
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
    try {
        for (MessageExt msg : msgs) {
            handle(msg);
        }
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    } catch (RetryableException e) {
        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
    } catch (Exception e) {
        log.error("non-retryable consume failed", e);
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
```

注意：不可重试错误如果直接返回成功，必须先落失败表或告警，否则就是静默丢处理。

Spring RocketMQ 监听器中，抛出异常通常会触发重试：

```java
@RocketMQMessageListener(
        topic = "order-paid",
        consumerGroup = "coupon-service"
)
public class CouponListener implements RocketMQListener<OrderPaidEvent> {
    @Override
    public void onMessage(OrderPaidEvent event) {
        couponService.handle(event);
    }
}
```

如果 `handle` 抛异常，框架会按 RocketMQ 机制重试。

## RocketMQ 死信队列

当消息超过最大重试次数后，会进入死信队列。

RocketMQ 死信 Topic 通常形如：

```text
%DLQ%consumerGroup
```

例如：

```text
%DLQ%coupon-service
```

死信特点：

- 表示该消费组多次消费失败。
- 不同消费组有各自死信队列。
- 需要人工或任务订阅死信并处理。

处理流程：

```text
发现 DLQ 告警
  -> 查询死信消息内容
  -> 定位失败原因
  -> 修复代码 / 数据 / 配置
  -> 按业务幂等规则重放
  -> 验证消费成功
```

广播消费模式要特别注意：RocketMQ 广播消费失败通常不会像集群消费那样由 Broker 统一重试和进入 DLQ，因此核心业务不要随意使用广播消费承载可靠处理。

## Kafka 重试

Kafka 原生消费模型没有像 RocketMQ 那样统一的 Broker 端消费重试队列。常见方案由客户端或框架实现。

### 方案一：不提交 offset，阻塞重试

消费失败就抛异常，不提交 offset。下次仍从失败消息开始。

优点：

- 简单。
- 不会跳过失败消息。
- 适合顺序要求强的场景。

缺点：

- 一条坏消息阻塞整个 Partition。
- 临时失败可能导致 lag 快速增长。

### 方案二：重试 Topic

失败后把消息投递到重试 Topic：

```text
order-paid
  -> consume failed
  -> order-paid-retry-1m
  -> order-paid-retry-5m
  -> order-paid-dlt
```

优点：

- 不阻塞主 Topic。
- 可以做延迟和退避。
- 可按次数进入 DLT。

缺点：

- 实现复杂。
- 可能破坏顺序。
- 要维护原始 topic、partition、offset、重试次数等上下文。

### Spring Kafka 示例

```java
@Bean
DefaultErrorHandler kafkaErrorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
    DeadLetterPublishingRecoverer recoverer =
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                    (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));

    FixedBackOff backOff = new FixedBackOff(1000L, 3L);
    return new DefaultErrorHandler(recoverer, backOff);
}
```

含义：

- 失败后间隔 1 秒重试。
- 最多重试 3 次。
- 仍失败则投递到 DLT。

实际生产中更常用指数退避，并区分可重试异常和不可重试异常。

## 死信处理

死信队列不是垃圾桶。它是异常消息隔离区。

死信处理要包含：

- 告警：有死信立即通知。
- 查询：能按 topic、group、key、时间查死信。
- 归因：区分代码 bug、脏数据、下游故障、配置问题。
- 修复：修代码、修数据或补配置。
- 重放：修复后重新投递。
- 审计：记录谁在什么时候重放了什么。

死信表设计示例：

```sql
CREATE TABLE mq_dead_message (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    topic VARCHAR(128) NOT NULL,
    consumer_group VARCHAR(128) NOT NULL,
    biz_key VARCHAR(128) DEFAULT NULL,
    message_id VARCHAR(128) DEFAULT NULL,
    payload MEDIUMTEXT NOT NULL,
    error_msg VARCHAR(1024) DEFAULT NULL,
    status VARCHAR(32) NOT NULL,
    retry_count INT NOT NULL DEFAULT 0,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    KEY idx_group_status (consumer_group, status),
    KEY idx_biz_key (biz_key)
);
```

## 重试退避

不要对失败消息做无间隔无限重试。

退避策略：

- 固定间隔：1 秒、5 秒、1 分钟。
- 指数退避：1 秒、2 秒、4 秒、8 秒。
- 分级重试 Topic：1m、5m、30m。
- 按错误类型设置不同间隔。

原因：

- 下游故障时立即重试会放大流量。
- 大量失败消息会挤占正常消息。
- 日志和告警会被刷爆。

## 可重试异常分类

示例：

```java
public void handle(OrderPaidEvent event) {
    try {
        couponService.sendCoupon(event);
    } catch (TimeoutException | ConnectException e) {
        throw new RetryableConsumeException(e);
    } catch (InvalidMessageException e) {
        deadMessageService.save(event, e);
        return;
    }
}
```

原则：

- 临时资源问题抛出，让 MQ 重试。
- 消息本身错误落失败表或死信，不要无限重试。
- 业务状态不满足要判断是暂时未就绪还是永久非法。

## 线上风险

### 重试风暴

下游服务故障，消费者不断重试，导致下游更难恢复。

治理：

- 指数退避。
- 限制最大并发。
- 熔断降级。
- 暂停消费或降低消费速率。
- 失败消息进入延迟重试 Topic。

### 毒丸消息

某条消息永远处理失败，阻塞分区或队列。

治理：

- 最大重试次数。
- 死信队列。
- 非法消息快速隔离。
- 顺序消费场景要有人工处理通道。

### 死信无人处理

DLQ 不告警、不消费、不重放，最终业务永久不一致。

治理：

- DLQ 数量监控。
- 每日巡检。
- 死信处理 SOP。
- 重放工具和权限审计。

### 重放导致重复副作用

死信重放会再次触发业务逻辑。

要求：

- 消费者幂等。
- 重放前评估业务状态。
- 重放工具支持按 bizKey 精确选择。

## 排查优化

排查消费失败：

```text
看消费者异常日志
  -> 判断可重试 / 不可重试
  -> 看重试次数
  -> 看是否进入 DLQ
  -> 查消息 payload 和 bizKey
  -> 查业务状态和下游依赖
```

排查重试风暴：

- 失败率是否突然升高。
- 下游 RT 是否升高。
- 是否某类异常占比最高。
- retry topic 或 DLQ 是否快速增长。
- 消费者是否在无间隔重试。

优化：

- 按异常分类处理。
- 失败日志限频，但指标不能丢。
- 消息体带 traceId、eventId、bizKey。
- 重试次数、最后异常、首次失败时间可观测。

## 常见追问

### 消费失败应该一直重试吗？

不应该。临时失败可以重试，永久失败应进入死信或失败表。无限重试会造成堆积、阻塞和重试风暴。

### Kafka 没有 Broker 端重试怎么办？

可以选择不提交 offset 阻塞重试，也可以用 retry topic / DLT，或者使用 Spring Kafka 的错误处理器。具体取舍取决于是否要求顺序、是否能接受主分区阻塞。

### 死信队列里的消息怎么处理？

先告警和归因，修复代码或数据后再重放。重放必须依赖消费者幂等，不能盲目全量重放。

### 重试会不会破坏顺序？

可能。把失败消息投递到重试 Topic 后，主 Topic 后续消息可能先被处理，顺序会被破坏。强顺序场景通常选择阻塞当前分区或队列，直到失败消息处理成功或人工介入。

## 易错点

- 所有异常都无限重试。
- catch 异常后返回消费成功，且没有失败记录。
- 死信队列没有告警。
- Kafka 重试 Topic 方案破坏顺序却没有评估。
- 重试没有退避，故障时打爆下游。
- 死信重放前不检查幂等。
- 把广播消费当作可靠重试消费。

## 自检清单

- 是否能区分生产者重试、消费者重试、业务补偿。
- 是否能说明 RocketMQ retry 和 DLQ 的基本机制。
- 是否能说明 Kafka 常见重试方案。
- 是否能设计可重试和不可重试异常分类。
- 是否能解释死信队列的处理 SOP。
- 是否能说明重试风暴和毒丸消息如何治理。
- 是否知道重放死信必须依赖幂等。
