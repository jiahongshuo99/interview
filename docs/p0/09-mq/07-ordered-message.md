# 顺序消息

## 面试定位

顺序消息常被用来考察 MQ 模型是否真正理解。常见问题：

- MQ 能不能保证消息顺序。
- Kafka 如何保证顺序。
- RocketMQ 如何保证顺序。
- 全局顺序和局部顺序有什么区别。
- 顺序消息为什么会影响吞吐。
- 顺序消费失败后怎么办。

核心回答：大多数 MQ 只能低成本保证分片内顺序，不能天然保证 Topic 全局顺序。业务通常追求某个维度的局部顺序，例如同一订单、同一用户、同一库存 SKU 的消息顺序。

## 顺序类型

### 全局顺序

整个 Topic 内所有消息严格按发送顺序消费。

实现方式：

```text
Topic 只有 1 个 Partition / Queue
消费者单线程消费
```

优点：

- 顺序语义简单。

缺点：

- 吞吐很低。
- 无法横向扩展。
- 一条消息失败会阻塞所有消息。

全局顺序只适合极少量、强顺序场景。

### 局部顺序

按业务 key 保证顺序，例如同一个订单有序：

```text
orderId=O1: created -> paid -> shipped
orderId=O2: created -> cancelled
```

不同订单之间不要求顺序。

实现方式：

```text
同一个 orderId 的消息发送到同一个 Partition / Queue
同一个 Partition / Queue 内按顺序消费
```

这是生产中最常见的顺序消息。

## Kafka 顺序机制

Kafka 保证单 Partition 内消息有序。

生产端：

```java
kafkaTemplate.send("order-event", orderId, event);
```

同一个 `orderId` 作为 key，会 hash 到同一个 Partition。

消费端：

```text
Partition 0:
  offset 100: O1 created
  offset 101: O1 paid
  offset 102: O1 shipped
```

Consumer 按 offset 顺序 poll，就能保证这个 Partition 内有序。

关键前提：

- 同一业务 key 始终路由到同一 Partition。
- 同一 Partition 的消息按顺序处理。
- 不要对同一 Partition 内消息并发乱序处理。
- 失败消息不能跳过后继续提交 offset。

## Kafka 可能破坏顺序的点

### 生产端重试乱序

如果同一连接上允许多个未确认请求，早发送的批次失败重试，晚发送的批次先成功，可能产生乱序风险。

现代 Kafka 开启幂等生产者后会缓解该问题。

推荐：

```properties
enable.idempotence=true
acks=all
```

对极端顺序要求，可以进一步限制：

```properties
max.in.flight.requests.per.connection=1
```

但吞吐会下降。

### 消费端并发处理

错误示例：

```java
@KafkaListener(topics = "order-event")
public void onMessage(OrderEvent event) {
    executor.submit(() -> orderService.handle(event));
}
```

如果同一 Partition 的消息被提交到线程池并发执行，就可能后面的消息先处理完成。

正确思路：

- 同一 Partition 单线程处理。
- 或按业务 key 做串行队列。
- 处理成功后再提交 offset。

### 增加分区

如果 Topic 分区数从 4 增加到 8，key 到 Partition 的 hash 结果可能变化。后续同一 orderId 的消息可能进入新 Partition，与历史消息顺序断开。

需要顺序保证的 Topic，扩分区前要评估影响。

## RocketMQ 顺序机制

RocketMQ 保证单 MessageQueue 内有序。顺序消息需要生产端顺序发送到同一个 Queue，消费端顺序消费该 Queue。

发送示例：

```java
rocketMQTemplate.syncSendOrderly(
        "order-event",
        event,
        event.getOrderId()
);
```

原生 API 示例：

```java
producer.send(message, (mqs, msg, arg) -> {
    String orderId = (String) arg;
    int index = Math.abs(orderId.hashCode()) % mqs.size();
    return mqs.get(index);
}, event.getOrderId());
```

消费示例：

```java
@RocketMQMessageListener(
        topic = "order-event",
        consumerGroup = "order-state-service",
        consumeMode = ConsumeMode.ORDERLY
)
public class OrderEventListener implements RocketMQListener<OrderEvent> {
    @Override
    public void onMessage(OrderEvent event) {
        orderStateService.handle(event);
    }
}
```

`ORDERLY` 模式下，同一 MessageQueue 会加锁顺序消费。

## 顺序消费失败

顺序消费最大的麻烦是失败处理。

假设：

```text
O1 created
O1 paid
O1 shipped
```

如果 `paid` 处理失败，不能跳过它直接处理 `shipped`，否则业务状态会乱。

处理方式：

- 当前消息稍后重试。
- 阻塞同一分片后续消息。
- 达到最大重试次数后进入人工处理或死信。
- 业务上允许时设计可重入状态机，让后续消息也能暂存等待。

RocketMQ 顺序消费失败通常会暂停当前队列一段时间后重试。Kafka 中通常不提交失败消息之后的 offset，或者把失败消息转储后明确接受乱序风险。

## 吞吐影响

顺序消息牺牲并行度。

普通并发消费：

```text
Partition 0 -> 多条消息可以并发处理
```

顺序消费：

```text
Partition 0 -> 必须按 offset 一条条成功推进
```

影响：

- 单个 key 热点会拖慢所在 Partition。
- 单条慢消息阻塞后续消息。
- 失败重试阻塞同队列。
- 扩容受 Partition / Queue 数限制。

优化：

- 只对真正需要顺序的业务使用顺序消息。
- 用更细粒度 key 分散热点。
- 热点 key 拆桶，但会牺牲该 key 的严格顺序。
- 消费逻辑尽量短，慢操作异步化但要保持状态顺序。

## 业务设计示例

订单状态事件：

```text
OrderCreatedEvent
OrderPaidEvent
OrderShippedEvent
OrderFinishedEvent
```

按 `orderId` 路由，保证单订单顺序。

消费者状态机：

```sql
UPDATE orders
SET status = 'PAID'
WHERE order_id = ?
  AND status = 'CREATED';
```

如果收到重复或乱序消息：

- 当前状态已经是 `PAID`，重复 `PAID` 可以忽略。
- 当前状态还是 `INIT`，却收到 `SHIPPED`，说明乱序或前置消息失败，应暂存或重试。

顺序消息最好配合状态机，而不是完全依赖 MQ 顺序。

## 线上风险

### 热点 key

大商家、大用户、大活动可能产生大量同 key 消息，全部打到一个 Partition / Queue。

表现：

- 单分片 lag 高。
- 其他分片空闲。
- 一个消费者实例负载高。

处理：

- 拆 key，例如 `merchantId + bucketNo`。
- 将热点业务独立 Topic。
- 评估是否真的需要该维度严格顺序。

### 消费失败阻塞

一条消息反复失败，阻塞后续同队列消息。

处理：

- 快速识别不可恢复错误。
- 达到阈值进入死信或人工处理。
- 消费者要区分临时错误和业务永久错误。
- 加强告警。

### 扩分区导致顺序变化

分区数变化会改变 hash 路由。需要顺序的 Topic 不应随意扩分区。

处理：

- 初始分区数预留容量。
- 使用自定义一致性路由。
- 扩容时停写、迁移、验证顺序影响。

## 排查优化

排查乱序：

- 检查生产端是否使用同一个 sharding key。
- 检查 Topic 分区数是否变更。
- 检查消费者是否把消息丢到线程池并发处理。
- 检查失败消息是否被跳过。
- 检查是否有多个生产者时钟或事件生成顺序不一致。

排查顺序消费慢：

- 看单分片 lag。
- 看消费 RT。
- 看是否某条消息一直重试。
- 看是否热点 key 集中。
- 看下游依赖是否慢。

## 常见追问

### Kafka 能保证消息顺序吗？

Kafka 能保证单 Partition 内顺序，不能天然保证 Topic 全局顺序。要保证某个业务维度顺序，需要用该业务 key 路由到同一个 Partition，并保证消费端按顺序处理。

### RocketMQ 顺序消息是不是全局有序？

不是默认全局有序。RocketMQ 顺序消息通常也是按 MessageQueue 保证局部顺序。全局顺序需要 Topic 只有一个 Queue，吞吐会明显受限。

### 顺序消费失败后能不能跳过？

技术上可以，但业务上通常不应该。跳过会破坏后续消息的前置条件。除非业务设计允许乱序，或者失败消息已经转入人工处理并有补偿逻辑。

### 如何兼顾顺序和吞吐？

使用局部顺序，而不是全局顺序。选择合适 sharding key，让需要顺序的消息进入同一分片，不相关 key 分散到不同分片并行处理。

## 易错点

- 认为 MQ Topic 天然全局有序。
- 生产端没有指定 key，却要求同订单顺序。
- 消费端把同分区消息扔进线程池并发处理。
- 顺序消费失败后直接提交 offset。
- 为了顺序把所有消息放一个分区，导致吞吐无法扩展。
- 扩分区前没有评估 key 路由变化。

## 自检清单

- 是否能区分全局顺序和局部顺序。
- 是否知道 Kafka 单 Partition 有序。
- 是否知道 RocketMQ 单 Queue 有序。
- 是否能写出按 orderId 路由的发送示例。
- 是否能说明顺序消费失败为什么会阻塞。
- 是否能分析热点 key 对顺序消息的影响。
- 是否能用状态机增强顺序消息的可靠性。
