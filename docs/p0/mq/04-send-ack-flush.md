# 消息发送确认、刷盘策略

## 面试定位

消息可靠性通常从发送端开始问。常见问题：

- Producer 发送成功到底表示什么。
- Kafka 的 `acks=0/1/all` 有什么区别。
- RocketMQ 的 `SEND_OK`、`FLUSH_DISK_TIMEOUT` 等状态代表什么。
- 同步刷盘和异步刷盘怎么选。
- 主从同步和刷盘策略如何影响消息丢失。
- 业务数据库提交成功后，消息发送失败怎么办。

回答时要分清三层确认：

```text
生产者收到响应
  -> Broker 是否收到
  -> Broker 是否写入内存 / page cache
  -> Broker 是否刷盘
  -> 副本是否复制
```

“发送成功”不是绝对概念，不同 MQ、不同配置下语义不同。

## 发送方式

常见发送方式：

- 同步发送：调用线程等待结果，可靠性和可感知性最好，RT 较高。
- 异步发送：发送后通过 callback 感知结果，吞吐更好，但必须正确处理回调失败。
- 单向发送：只管发，不关心结果，适合日志、监控等可丢场景。

对核心业务消息，通常不建议使用单向发送。

## Kafka 发送确认

Kafka Producer 发送链路：

```text
send()
  -> 序列化
  -> 选择 partition
  -> 写入本地 RecordAccumulator
  -> Sender 线程批量发送到 partition leader
  -> leader 写入日志
  -> 按 acks 策略返回结果
```

### acks=0

生产者发送后不等待 Broker 确认。

特点：

- 吞吐高，延迟低。
- 生产者无法知道 Broker 是否收到。
- Broker 不可用、网络失败都可能静默丢失。

适合：

- 可丢日志。
- 监控埋点。
- 对可靠性要求低的数据。

不适合订单、支付、库存等核心业务。

### acks=1

Leader 副本写入成功后返回。

特点：

- 性能和可靠性折中。
- Leader 写入成功但 Follower 尚未复制时，如果 Leader 宕机，消息可能丢失。

适合：

- 一般业务事件。
- 可以通过补偿修复的消息。

### acks=all

Leader 等 ISR 中满足条件的副本确认后再返回。

特点：

- 可靠性最高。
- 延迟更高。
- 需要配合 `min.insync.replicas`。

推荐关键配置：

```properties
acks=all
enable.idempotence=true
retries=3
min.insync.replicas=2
replication.factor=3
```

注意：`acks=all` 不代表所有副本都确认，而是 ISR 中满足 `min.insync.replicas` 约束。

## Kafka 幂等生产者

Kafka Producer 开启幂等后，可以避免生产者重试导致的单分区重复写入。

```properties
enable.idempotence=true
acks=all
retries=2147483647
max.in.flight.requests.per.connection=5
```

幂等生产者的作用范围：

- 解决生产者因网络超时重试导致的重复写入。
- 依赖 Producer ID、序列号和 Broker 去重。
- 主要保证单 Producer 会话内、单分区的幂等写入。

它不能解决：

- 业务方法被调用两次导致发送两条不同消息。
- 消费端重复消费。
- 跨系统业务幂等。

## Kafka 刷盘与复制

Kafka 写入通常先进入操作系统 page cache，再由操作系统刷盘。Kafka 依赖顺序写和 page cache 获取高吞吐。

简化流程：

```text
Producer -> Broker Leader
  -> append 到 log 文件
  -> 进入 page cache
  -> 按策略 flush 到磁盘
  -> Follower 拉取复制
  -> 满足 ack 条件后返回
```

关键点：

- Kafka 不一定每条消息都 fsync。
- Broker 宕机但机器不掉电时，page cache 通常仍可落盘；机器断电则可能丢 page cache 中数据。
- 副本复制比每条同步刷盘更常用来提升可靠性。

常见配置：

```properties
log.flush.interval.messages=9223372036854775807
log.flush.interval.ms=null
unclean.leader.election.enable=false
min.insync.replicas=2
```

生产上更常通过多副本和 ISR 保证可靠性，而不是强制每条消息刷盘。

## RocketMQ 发送确认

RocketMQ 同步发送返回 `SendResult`，常见状态：

| 状态 | 含义 |
| --- | --- |
| SEND_OK | 发送成功 |
| FLUSH_DISK_TIMEOUT | Broker 写入成功，但同步刷盘超时 |
| FLUSH_SLAVE_TIMEOUT | Broker 写入成功，但同步复制到 Slave 超时 |
| SLAVE_NOT_AVAILABLE | Slave 不可用 |

注意：某些非 `SEND_OK` 状态下，消息可能已经写入 Master，只是刷盘或复制确认超时。生产者不能简单地无脑重发而不考虑业务幂等，否则可能产生重复消息。

## RocketMQ 刷盘策略

### 异步刷盘

Broker 写入内存映射文件后返回，由后台线程刷盘。

优点：

- 吞吐高。
- 延迟低。

风险：

- Broker 机器宕机或掉电时，未刷盘消息可能丢失。

适合：

- 大多数普通业务消息。
- 可通过补偿恢复的场景。

### 同步刷盘

Broker 写入后等待刷盘完成再返回。

优点：

- 单机宕机丢失风险更低。

缺点：

- 延迟高。
- 吞吐下降。

适合：

- 金融、交易等强可靠核心消息。
- 消息量可控且对延迟不极端敏感的场景。

Broker 配置示例：

```properties
flushDiskType=SYNC_FLUSH
```

或：

```properties
flushDiskType=ASYNC_FLUSH
```

## RocketMQ 主从复制策略

常见复制策略：

- 异步复制：Master 写成功后返回，Slave 异步追。
- 同步双写：Master 等 Slave 写入确认后返回。

配置示例：

```properties
brokerRole=SYNC_MASTER
```

或：

```properties
brokerRole=ASYNC_MASTER
```

同步双写可靠性更好，但延迟更高。如果 Slave 不可用，可能返回 `SLAVE_NOT_AVAILABLE`。

## 代码示例

### Kafka 同步等待发送结果

```java
public void send(OrderPaidEvent event) {
    try {
        SendResult<String, OrderPaidEvent> result = kafkaTemplate
                .send("order-paid", event.getOrderId(), event)
                .get(3, TimeUnit.SECONDS);

        RecordMetadata metadata = result.getRecordMetadata();
        log.info("send success topic={}, partition={}, offset={}",
                metadata.topic(), metadata.partition(), metadata.offset());
    } catch (Exception e) {
        // 核心消息不要只打日志，应落库补偿或抛出让上层事务失败
        throw new IllegalStateException("send order paid event failed", e);
    }
}
```

### Kafka 异步发送处理回调

```java
public void sendAsync(OrderPaidEvent event) {
    kafkaTemplate.send("order-paid", event.getOrderId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("send order paid event failed, orderId={}", event.getOrderId(), ex);
                    messageOutboxService.markSendFailed(event.getEventId());
                    return;
                }
                messageOutboxService.markSendSuccess(event.getEventId());
            });
}
```

异步发送不能忽略回调，否则发送失败会被静默吞掉。

### RocketMQ 同步发送检查状态

```java
public void send(OrderPaidEvent event) {
    SendResult result = rocketMQTemplate.syncSend("order-paid", event);
    if (result.getSendStatus() != SendStatus.SEND_OK) {
        log.warn("rocketmq send not ok, orderId={}, status={}, msgId={}",
                event.getOrderId(), result.getSendStatus(), result.getMsgId());
        messageOutboxService.markUnknown(event.getEventId(), result.getSendStatus().name());
    }
}
```

对于超时状态，建议标记为未知，由补偿任务根据业务唯一键重试或核对，不要简单假设一定失败。

## 本地事务与消息发送

典型问题：订单数据库提交成功，但 MQ 发送失败。

错误写法：

```java
@Transactional
public void pay(Order order) {
    orderRepository.markPaid(order.getId());
    kafkaTemplate.send("order-paid", order.getId(), new OrderPaidEvent(order.getId()));
}
```

风险：

- 数据库提交了，消息发送失败。
- 消息发送成功了，数据库事务后来回滚。
- 发送在事务内耗时，拖长数据库锁持有时间。

常见方案：

### 本地消息表

业务数据和消息表写入同一个事务：

```java
@Transactional
public void pay(String orderId) {
    orderRepository.markPaid(orderId);
    outboxRepository.insert(new OutboxMessage(
            UUID.randomUUID().toString(),
            "order-paid",
            orderId,
            toJson(new OrderPaidEvent(orderId)),
            "NEW"
    ));
}
```

后台任务扫描 `NEW` 消息发送 MQ，成功后标记 `SENT`，失败继续重试。

优点：

- 数据库事务内只写本地数据。
- 不会出现业务成功但消息完全无记录。
- 发送失败可补偿。

### 事务消息

RocketMQ 支持事务消息：

```text
发送 half message
  -> Broker 暂不投递
  -> 执行业务本地事务
  -> 提交 commit / rollback
  -> Broker 根据状态投递或丢弃
  -> 状态未知时回查本地事务
```

事务消息也需要可靠的本地事务状态表，回查逻辑必须能根据业务数据判断提交或回滚。

## 线上风险

### 生产者认为发送成功但实际丢失

常见原因：

- `acks=0`。
- 单向发送。
- 异步发送忽略 callback。
- RocketMQ 异步刷盘时 Broker 机器掉电。
- Kafka 副本不足仍允许写入。

### 生产者认为发送失败但实际成功

常见原因：

- 网络超时。
- Broker 写入成功，响应丢失。
- RocketMQ 刷盘或从库复制超时。

风险：

- 生产者重试后产生重复消息。

治理：

- 消息带业务唯一键。
- 消费端幂等。
- 发送记录标记 UNKNOWN，由补偿任务核对。

### 过度追求强可靠导致吞吐下降

同步刷盘、同步复制、`acks=all` 都会增加延迟。核心链路应按消息重要性分级：

- 支付、订单状态：高可靠配置 + 补偿。
- 普通业务事件：副本可靠 + 幂等 + 重试。
- 日志埋点：可适当牺牲可靠性换吞吐。

## 排查优化

生产端排查：

- 是否捕获并处理发送异常。
- 异步 callback 是否执行。
- 超时配置是否过短。
- 是否存在批量发送积压。
- 消息体是否过大。

Broker 排查：

- Kafka ISR 是否正常。
- RocketMQ Broker 主从状态是否正常。
- 磁盘水位、刷盘耗时是否异常。
- 网络抖动和请求超时是否增多。

业务排查：

- 业务表和消息表是否在同一事务。
- 是否存在 UNKNOWN 状态消息未补偿。
- 消费端是否具备幂等能力。

## 常见追问

### `acks=all` 会不会绝对不丢消息？

不会。它只表示满足 ISR 和 `min.insync.replicas` 条件后返回。仍要考虑生产者配置、Broker 宕机、磁盘损坏、错误删除 Topic、消息过期、消费端错误提交 offset 等风险。

### 同步刷盘是不是一定比异步刷盘好？

不是。同步刷盘可靠性更强，但吞吐和延迟更差。是否使用取决于业务消息等级、吞吐要求和补偿能力。很多场景用异步刷盘 + 多副本 + 补偿更划算。

### 发送超时后能不能直接重试？

可以重试，但必须假设上一次可能已经成功。因此消息要有业务唯一键，消费者要幂等，生产端最好记录发送状态，避免重试造成副作用。

### 本地消息表和事务消息怎么选？

本地消息表通用、可控、易排查，但有扫描延迟；事务消息减少自建投递逻辑，但依赖 MQ 能力和正确回查。无论哪种，都不能省掉消费者幂等。

## 易错点

- 认为 producer `send()` 返回就代表 Broker 持久化完成。
- 异步发送不处理 callback。
- RocketMQ 非 `SEND_OK` 状态一律当失败重发，导致重复。
- Kafka 只配置 `acks=all`，但副本数和 `min.insync.replicas` 不合理。
- 在数据库事务中做长时间 MQ 同步发送。
- 只保证生产端可靠，忽略消费端 offset 和幂等。

## 自检清单

- 是否能解释 Kafka `acks=0/1/all`。
- 是否能解释 RocketMQ `SEND_OK` 和刷盘超时的含义。
- 是否能说明同步刷盘、异步刷盘的取舍。
- 是否能说明主从同步对可靠性的影响。
- 是否知道发送超时时消息状态可能是未知。
- 是否能设计本地事务和 MQ 发送的一致性方案。
- 是否能按消息等级选择可靠性配置。
