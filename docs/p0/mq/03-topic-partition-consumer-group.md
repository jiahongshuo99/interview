# Topic、Partition、Consumer Group

## 面试定位

Topic、Partition、Consumer Group 是 MQ 消费模型的核心。面试经常围绕这些问题展开：

- Topic 怎么设计，粒度粗细如何取舍。
- Partition / Queue 的作用是什么。
- 消费组如何实现负载均衡和广播。
- 为什么消费者实例数超过分区数后不会继续提升吞吐。
- 如何保证局部顺序。
- offset 为什么要按消费组维护。

回答时要抓住一句话：Topic 是业务分类，Partition / Queue 是并行和顺序的物理单位，Consumer Group 是消费进度和负载均衡的隔离单位。

## Topic

Topic 是消息的逻辑主题，通常按业务事件或数据流分类。

示例：

```text
order-created
order-paid
order-cancelled
user-register-event
inventory-change-event
```

Topic 设计要稳定。Topic 一旦被多个系统订阅，随意改名、拆分、合并都会带来兼容成本。

## Topic 粒度设计

### 过粗的问题

例如所有订单事件都放进一个 `order-event` Topic：

```json
{
  "eventType": "ORDER_PAID",
  "orderId": "O1001",
  "payload": {}
}
```

优点：

- Topic 数量少。
- 订阅订单全量事件方便。
- 运维治理简单。

缺点：

- 消费者要过滤无关事件。
- 消息结构容易膨胀。
- 某类事件流量突增会影响同 Topic 下其他事件。
- 权限、保留时间、重试策略难以细分。

### 过细的问题

例如拆成非常多 Topic：

```text
order-paid-coupon
order-paid-score
order-paid-search-index
order-paid-data-warehouse
```

问题：

- Topic 爆炸，治理困难。
- 生产者开始感知消费者，重新耦合。
- 新增消费者可能要新增 Topic。
- 监控、权限、容量评估复杂。

### 推荐思路

按“业务事实事件”设计，而不是按“消费者动作”设计。

推荐：

```text
order-paid
```

不推荐：

```text
send-coupon-after-order-paid
```

原因：

- `order-paid` 是业务事实。
- `send-coupon-after-order-paid` 是某个消费者的处理动作，会让生产者依赖消费者。

Topic 粒度判断：

- 消息结构是否稳定相近。
- 消费者是否大多关心同类事件。
- 流量和保留策略是否相近。
- 权限隔离要求是否一致。
- 是否有独立扩容和监控需求。

## Partition / Queue

Kafka 中叫 Partition，RocketMQ 中叫 MessageQueue。它们都是 Topic 下的分片。

```text
Topic: order-paid
  -> Partition 0
  -> Partition 1
  -> Partition 2
  -> Partition 3
```

作用：

- 提升并行写入能力。
- 提升并行消费能力。
- 提供分片内顺序。
- 作为消费者负载均衡单位。

关键结论：

- 单个 Partition / Queue 内消息有序。
- Topic 全局通常不保证有序。
- 消费并行度受 Partition / Queue 数限制。
- 同一消费组内，一个 Partition / Queue 同一时刻通常只由一个消费者实例消费。

## 分区路由

生产者发送消息时要决定写入哪个分区。

常见方式：

- 指定分区：强控制，但业务侵入高。
- 按 key hash：同一 key 进入同一分区，适合局部顺序。
- 轮询或黏性分区：提升吞吐和负载均衡。

Kafka 示例：

```java
kafkaTemplate.send("order-paid", orderId, event);
```

这里 `orderId` 是 key，同一个 `orderId` 的消息会进入同一个 Partition，从而保证单订单维度有序。

RocketMQ 顺序发送示例：

```java
rocketMQTemplate.syncSendOrderly(
        "order-event",
        event,
        event.getOrderId()
);
```

底层会根据 `orderId` 选择固定 MessageQueue。

## Consumer Group

Consumer Group 是一组消费者实例的逻辑集合。

同一组内负载均衡：

```text
Topic order-paid: P0 P1 P2 P3

Group coupon-service:
  consumer-1 -> P0 P1
  consumer-2 -> P2 P3
```

不同组之间广播：

```text
Group coupon-service -> 消费全量 order-paid
Group score-service  -> 消费全量 order-paid
Group search-service -> 消费全量 order-paid
```

每个 Group 有自己的 offset，因此不同业务系统互不影响。

## 消费者数量与分区数关系

如果 Topic 有 4 个 Partition，同一消费组内：

| 消费者实例数 | 实际并行情况 |
| --- | --- |
| 1 | 1 个实例消费 4 个 Partition |
| 2 | 每个实例约 2 个 Partition |
| 4 | 每个实例约 1 个 Partition |
| 8 | 最多 4 个实例消费，另外 4 个空闲 |

所以堆积时不能只说“加机器”。如果消费者数已经大于等于分区数，继续加实例不会提升同组消费并行度。

可选优化：

- 增加 Partition / Queue 数。
- 提升单实例消费线程和批处理能力。
- 优化下游 RT。
- 拆分热点 key。
- 拆分 Topic 或消费组职责。

## Offset 与消费组

Offset 是消费进度。

Kafka 中 offset 维度是：

```text
consumer group + topic + partition
```

RocketMQ 中消费进度也是按消费组和队列维护。

同一 Topic，不同消费组可以消费到不同位置：

```text
order-paid P0:
  offset 1000

coupon-service committed offset: 900
search-service committed offset: 1000
warehouse-service committed offset: 500
```

这意味着：

- 搜索系统消费快，不影响数仓系统慢慢追。
- 数仓系统堆积，不影响优惠券系统。
- 删除或重置某个组 offset，只影响该组。

## Rebalance

Rebalance 是消费组成员或分区变化后，重新分配分区的过程。

触发原因：

- 消费者实例上线或下线。
- 消费者心跳超时。
- Topic 分区数变化。
- 订阅关系变化。
- 消费者处理过慢，被认为失联。

Rebalance 影响：

- 分区短暂停止消费。
- 可能导致重复消费。
- 本地缓存状态可能丢失。
- 频繁 rebalance 会明显影响吞吐。

Kafka 中常见相关配置：

```properties
max.poll.interval.ms=300000
session.timeout.ms=45000
heartbeat.interval.ms=15000
max.poll.records=500
```

如果单批处理时间超过 `max.poll.interval.ms`，消费者可能被踢出组，触发 rebalance。

## 代码 / 配置示例

### Kafka 监听指定消费组

```java
@KafkaListener(
        topics = "order-paid",
        groupId = "coupon-service",
        concurrency = "4"
)
public void onMessage(OrderPaidEvent event, Acknowledgment ack) {
    couponService.handle(event);
    ack.acknowledge();
}
```

注意：`concurrency = 4` 不代表一定有 4 个线程有效消费。如果 Topic 只有 2 个 Partition，最多只有 2 个并行分区。

### Kafka Topic 创建示例

```bash
kafka-topics.sh \
  --bootstrap-server kafka-1:9092 \
  --create \
  --topic order-paid \
  --partitions 12 \
  --replication-factor 3
```

### RocketMQ 消费组示例

```java
@RocketMQMessageListener(
        topic = "order-paid",
        consumerGroup = "coupon-service",
        consumeMode = ConsumeMode.CONCURRENTLY
)
public class CouponOrderPaidListener implements RocketMQListener<OrderPaidEvent> {
    @Override
    public void onMessage(OrderPaidEvent event) {
        couponService.handle(event);
    }
}
```

## 线上风险

### Topic 设计不合理

表现：

- 消费者过滤大量无关消息。
- 单 Topic 流量过大，影响所有订阅方。
- 权限和保留策略无法细分。
- Topic 数过多，治理成本高。

优化：

- 按业务事件重构 Topic。
- 用 Tag 或事件类型做轻量过滤。
- 对高流量事件独立 Topic。

### 分区数不足

表现：

- 消费者实例空闲少，但 lag 持续增长。
- 单分区消费速度达到上限。
- 无法继续横向扩容。

优化：

- 增加分区数。
- 优化消费耗时。
- 批量消费。
- 拆分热点业务。

注意：增加分区可能影响 key 到分区的映射，从而影响顺序性和热点分布。

### 分区热点

表现：

- 总体分区数不少，但某个 Partition lag 极高。
- 某个消费者实例负载特别高。
- 某些业务 key 消息量巨大。

原因：

- 使用用户 ID、商家 ID、订单 ID 等 key 时分布不均。
- 大客户、大商家、热门活动造成热点。

优化：

- 热点 key 拆桶，例如 `merchantId + bucketNo`。
- 对超大客户独立 Topic。
- 业务上允许时打散顺序约束。

### 消费组误用

常见错误：

- 多个不同业务共用同一个 group，导致互相抢消息。
- 同一个业务每次发布都换 group，导致从头消费或 offset 混乱。
- 测试环境和生产环境 group 混用。
- 临时排查消费者使用正式 group，推进了正式消费进度。

## 排查优化

排查消费异常时先确认四件事：

```text
1. topic 是否正确
2. group 是否正确
3. partition / queue 数是否足够
4. offset / lag 是否正常
```

Kafka 常用命令：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server kafka-1:9092 \
  --describe \
  --group coupon-service
```

重点看：

- CURRENT-OFFSET：当前提交 offset。
- LOG-END-OFFSET：分区最新 offset。
- LAG：堆积量。
- CONSUMER-ID：当前分区分配给哪个消费者。

RocketMQ 常用命令：

```bash
mqadmin consumerProgress \
  -n nameserver:9876 \
  -g coupon-service
```

重点看：

- diffTotal：总堆积。
- topic / queue 维度进度。
- 消费组在线情况。

## 常见追问

### 一个 Topic 分区越多越好吗？

不是。分区越多，并行能力越强，但也会增加文件句柄、元数据、Leader 选举、网络复制和客户端调度成本。分区数应该基于吞吐、消费者并行度、未来增长和运维能力预估。

### 消费者数为什么不能超过分区数？

同一消费组内，一个分区同一时刻只能分配给一个消费者实例，否则同一个 offset 区间会被并发消费，破坏进度管理和分区内顺序。因此消费者数超过分区数后，多余实例无法分配到分区。

### 不同消费组会不会互相影响？

消费进度互不影响，但共享 Broker、网络、磁盘和 Topic 存储资源。一个消费组堆积不会推进另一个组 offset，但如果堆积导致消息保留时间内未消费完，慢组可能丢历史消息。

### 如何保证同一个订单的消息顺序？

生产端用订单 ID 作为分区 key，让同一个订单的消息进入同一个 Partition / Queue；消费端保证该 Partition / Queue 单线程或顺序消费；同时处理失败不能跳过前一条消息。

## 易错点

- 认为 Topic 内所有消息天然全局有序。
- 认为加消费者实例一定能提升吞吐。
- 临时消费者使用正式消费组，导致正式 offset 被推进。
- 分区 key 使用低基数字段，例如状态、类型，导致严重热点。
- 增加分区后忘记评估顺序性变化。
- 多业务共用同一个 Consumer Group。

## 自检清单

- 是否能说明 Topic、Partition / Queue、Consumer Group 的职责。
- 是否能解释同组负载均衡、不同组广播。
- 是否知道 offset 为什么按消费组维护。
- 是否能计算消费者实例和分区数的并行关系。
- 是否能解释 rebalance 的触发原因和影响。
- 是否能给出 Topic 粒度设计原则。
- 是否能定位堆积是全局慢还是分区热点。
