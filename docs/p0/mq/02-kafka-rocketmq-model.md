# Kafka / RocketMQ 基本模型

## 面试定位

Kafka 和 RocketMQ 是 Java 后端面试中最常被比较的两类消息系统。面试重点不是背产品宣传，而是理解它们的基本模型：

- Topic、Partition、Queue、Consumer Group 怎么工作。
- 消息如何存储、复制和消费。
- Kafka 为什么强调日志和分区。
- RocketMQ 为什么有 CommitLog、ConsumeQueue、NameServer。
- 两者在顺序、重试、延迟、事务、广播、可靠性上的差异。

回答时可以先讲共同抽象，再讲实现差异：Kafka 更像分布式追加日志系统，核心是 Partition 日志和 Consumer Group offset；RocketMQ 更像面向业务消息场景的消息中间件，内建了重试、死信、延迟、事务消息等能力。

## 共同抽象

Kafka 和 RocketMQ 都有这些概念：

- Producer：发送消息。
- Broker：存储和转发消息。
- Topic：按业务语义划分消息。
- 分片：Kafka 叫 Partition，RocketMQ 叫 MessageQueue 或 Queue。
- Consumer：消费消息。
- Consumer Group：消费组，同组负载均衡，不同组独立消费。
- Offset：消费进度。

通用数据流：

```text
Producer
  -> Topic
     -> Partition / Queue
        -> Broker 持久化
           -> Consumer Group 拉取并提交 offset
```

共同特征：

- Topic 内部会拆成多个分片，用于并行读写。
- 单个分片内部天然有序。
- 同一消费组内，一个分片通常同一时刻只分配给一个消费者实例。
- 不同消费组维护独立消费进度。
- 消费语义通常是至少一次，因此消费者要幂等。

## Kafka 基本模型

Kafka 的核心是分布式日志。

```text
Kafka Cluster
  -> Broker 1
  -> Broker 2
  -> Broker 3

Topic: order-event
  -> Partition 0: append-only log
  -> Partition 1: append-only log
  -> Partition 2: append-only log
```

每个 Partition 是一个有序、追加写的日志。消息进入 Partition 后会获得递增 offset。

### Kafka 关键角色

| 角色 | 说明 |
| --- | --- |
| Broker | Kafka 服务节点，负责存储 Partition 数据 |
| Topic | 消息主题 |
| Partition | Topic 的物理分片和并行单位 |
| Producer | 生产者，按 key 或分区策略写入 Partition |
| Consumer | 消费者，拉取消息 |
| Consumer Group | 消费组，组内负载均衡 |
| Offset | Partition 内消息位置 |
| Replica | Partition 副本 |
| Leader | Partition 主副本，处理读写 |
| Follower | 从副本，从 Leader 复制 |
| ISR | 与 Leader 保持同步的副本集合 |
| Controller | 集群元数据和 Leader 选举协调者 |
| Coordinator | 管理消费组成员和 offset 提交 |

### Kafka 存储结构

Kafka 中一个 Partition 是多个日志段组成的追加日志：

```text
partition-0/
  00000000000000000000.log
  00000000000000000000.index
  00000000000000000000.timeindex
  00000000000000102400.log
  00000000000000102400.index
```

核心点：

- 写入是顺序追加，吞吐高。
- 读取按 offset 顺序读取。
- 日志按 segment 滚动。
- 通过稀疏索引定位 offset。
- 旧消息按保留时间或保留大小清理。

### Kafka 消费模型

同一 Consumer Group 内多个 Consumer 共同消费 Topic：

```text
Topic order-event: P0 P1 P2 P3

Group order-service:
  C1 -> P0 P1
  C2 -> P2 P3
```

如果消费者数量超过 Partition 数，多出来的消费者会空闲。

不同组之间独立消费：

```text
Group coupon-service   -> 全量消费 order-event
Group search-service   -> 全量消费 order-event
Group data-warehouse   -> 全量消费 order-event
```

## RocketMQ 基本模型

RocketMQ 的核心组件包括 NameServer、Broker、Producer、Consumer。

```text
NameServer
  -> 保存 Topic 路由信息

Broker
  -> Master / Slave
  -> CommitLog
  -> ConsumeQueue
  -> IndexFile

Producer
  -> 从 NameServer 获取路由
  -> 发送到 Broker

Consumer
  -> 从 NameServer 获取路由
  -> 从 Broker 拉取消息
```

### RocketMQ 关键角色

| 角色 | 说明 |
| --- | --- |
| NameServer | 轻量级路由注册中心，Broker 注册 Topic 路由 |
| Broker | 消息存储节点 |
| Topic | 消息主题 |
| MessageQueue | Topic 下的队列，是负载均衡和顺序消费单位 |
| Producer Group | 生产者组，事务消息回查等场景会用到 |
| Consumer Group | 消费组 |
| CommitLog | 消息主体存储文件，所有 Topic 消息混合顺序追加 |
| ConsumeQueue | 按 Topic + Queue 建立的逻辑消费队列 |
| IndexFile | 按 key 查询消息的索引文件 |
| Tag | Topic 内的轻量过滤标签 |

### RocketMQ 存储结构

RocketMQ 消息主体写入 CommitLog：

```text
CommitLog
  -> 所有 Topic 的消息混合追加写

ConsumeQueue
  -> topicA queue0: commitLogOffset, size, tagHash
  -> topicA queue1: commitLogOffset, size, tagHash
  -> topicB queue0: commitLogOffset, size, tagHash
```

消费者不是直接扫描整个 CommitLog，而是先读 ConsumeQueue，再按 commitLogOffset 读取消息内容。

这种设计的好处：

- CommitLog 顺序写，写入吞吐高。
- ConsumeQueue 按 Topic 和 Queue 建立逻辑视图，便于消费。
- IndexFile 支持按 message key 查询。

### RocketMQ 消费模型

RocketMQ Topic 下有多个 MessageQueue：

```text
Topic order-event
  -> queue0
  -> queue1
  -> queue2
  -> queue3

Consumer Group coupon-service
  -> C1: queue0 queue1
  -> C2: queue2 queue3
```

同一消费组内，一个 MessageQueue 通常分配给一个消费者实例。不同消费组独立消费。

## Kafka 与 RocketMQ 对比

| 维度 | Kafka | RocketMQ |
| --- | --- | --- |
| 核心定位 | 分布式日志、流处理生态强 | 业务消息能力完整 |
| 路由组件 | Broker 元数据，现代版本逐步去 ZooKeeper | NameServer 提供路由 |
| 分片单位 | Partition | MessageQueue |
| 存储模型 | 每个 Partition 独立日志 | CommitLog + ConsumeQueue |
| 顺序性 | 单 Partition 有序 | 单 Queue 有序 |
| 消费模式 | Pull 模型 | 客户端本质也是拉取，PushConsumer 是封装后的长轮询 |
| 重试 / 死信 | 通常由客户端框架或重试 Topic 实现 | 内建消费重试和 DLQ |
| 延迟消息 | 通常需要额外方案或延迟 Topic | 内建延迟消息能力，具体能力依赖版本 |
| 事务消息 | 支持事务性生产和消费位点协作 | 支持半消息、本地事务、事务回查 |
| 生态 | 流处理、日志采集、大数据生态强 | 电商交易、业务消息场景常见 |

## 生产模型

### Kafka Producer

Kafka Producer 发送时会先序列化，再根据分区策略选择 Partition：

```text
send(record)
  -> serialize key/value
  -> partitioner choose partition
  -> append to batch
  -> sender thread send to broker leader
  -> wait ack
```

常见分区策略：

- 指定 partition，直接写入。
- 指定 key，按 key hash 到固定 partition。
- 不指定 key，按黏性分区或轮询策略提高批量效果。

配置示例：

```properties
spring.kafka.producer.acks=all
spring.kafka.producer.retries=3
spring.kafka.producer.properties.enable.idempotence=true
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
```

### RocketMQ Producer

RocketMQ Producer 从 NameServer 获取 Topic 路由，选择 MessageQueue 后发送到 Broker。

```text
send(message)
  -> 获取 topic route
  -> 选择 message queue
  -> 发送到 broker
  -> broker 写入 commitlog
  -> 返回 SendResult
```

示例：

```java
Message<String> message = MessageBuilder.withPayload(json)
        .setHeader(MessageConst.PROPERTY_KEYS, orderId)
        .build();

rocketMQTemplate.syncSend("order-event:paid", message);
```

`order-event:paid` 中 Topic 是 `order-event`，Tag 是 `paid`。

## 消费模型

### Kafka Consumer

Kafka Consumer 按 poll 拉取消息：

```java
@KafkaListener(topics = "order-event", groupId = "coupon-service")
public void onMessage(ConsumerRecord<String, OrderEvent> record, Acknowledgment ack) {
    orderEventService.handle(record.value());
    ack.acknowledge();
}
```

重点：

- offset 是按 `group + topic + partition` 维护。
- 自动提交可能导致处理失败但 offset 已提交。
- 手动提交更可控。
- rebalance 时 Partition 会在消费者之间重新分配。

### RocketMQ Consumer

RocketMQ 消费者监听消息：

```java
@RocketMQMessageListener(
        topic = "order-event",
        consumerGroup = "coupon-service"
)
public class OrderEventListener implements RocketMQListener<OrderEvent> {
    @Override
    public void onMessage(OrderEvent event) {
        orderEventService.handle(event);
    }
}
```

如果使用原生 API，可以通过返回状态控制成功或稍后重试。

## 线上风险

### Kafka 常见风险

- Partition 数不足导致消费者无法继续横向扩展。
- key 分布不均导致热点 Partition。
- 自动提交 offset 导致消息丢失。
- 消费者处理时间超过 `max.poll.interval.ms` 触发 rebalance。
- 副本数和 `min.insync.replicas` 配置不合理导致可靠性不足。
- retention 设置过短，堆积期间消息被清理。

### RocketMQ 常见风险

- Topic Queue 数不足导致消费并行度不足。
- 顺序消费中单条失败阻塞整个队列。
- 广播消费失败不会像集群消费一样走服务端重试。
- Tag 设计混乱导致消费者过滤复杂。
- 事务消息回查逻辑不可靠导致半消息状态不明确。
- 延迟消息能力依赖版本和配置，不能想当然支持任意精度。

## 排查优化

Kafka 排查常用视角：

```text
Topic 是否存在
Partition 数是否符合并行度
Consumer Group 是否正确
Lag 是否增长
是否频繁 rebalance
生产端 ack / retry 是否异常
Broker 磁盘和副本同步是否正常
```

RocketMQ 排查常用视角：

```text
NameServer 路由是否正常
Topic Queue 是否创建正确
Broker 主从和刷盘是否正常
Consumer Group 是否在线
是否存在 retry / DLQ 消息
consumeTps / consumeRT / diffTotal 是否异常
```

优化建议：

- Topic 和 Group 命名要按业务语义稳定设计。
- 分片数量要预留增长空间，但不要无限增大。
- 消息体保持精简，只放必要业务字段。
- 消费逻辑要幂等，失败要可重试。
- 核心链路补充发送成功率、消费成功率、堆积量、重试量、死信量监控。

## 常见追问

### Kafka 为什么吞吐高？

主要来自顺序写磁盘、批量发送、页缓存、零拷贝、分区并行和顺序读。高吞吐不是因为完全不落盘，而是充分利用顺序 I/O 和操作系统页缓存。

### RocketMQ 为什么要有 CommitLog 和 ConsumeQueue？

CommitLog 负责消息主体顺序追加，提升写入吞吐；ConsumeQueue 是按 Topic 和 Queue 建立的逻辑索引，消费者通过它快速定位属于自己队列的消息，避免扫描全量 CommitLog。

### Kafka 和 RocketMQ 如何选择？

如果偏日志采集、流处理、大数据生态，Kafka 更常见；如果偏交易业务消息、延迟、重试、死信、事务消息等内建能力，RocketMQ 更贴近业务消息场景。实际选型还要看现有基础设施、运维能力、生态和团队经验。

### RocketMQ PushConsumer 是服务端主动推吗？

不是严格意义上的服务端主动推。RocketMQ PushConsumer 是客户端封装的长轮询拉取模型，对业务代码表现为监听器回调。

## 易错点

- 把 Kafka Partition 和 RocketMQ Queue 混为完全相同的实现。
- 认为 Consumer Group 越多，单个业务消费越快。
- 认为同一个 Topic 下所有消息全局有序。
- 认为 RocketMQ 广播消费失败会自动进入 DLQ。
- 认为 Kafka 自动提交 offset 就一定不会丢消息。
- 忽略消息保留时间，堆积过久后再消费可能已经过期。

## 自检清单

- 是否能画出 Kafka Topic、Partition、Broker、Consumer Group 的关系。
- 是否能画出 RocketMQ NameServer、Broker、CommitLog、ConsumeQueue 的关系。
- 是否知道 Kafka 单 Partition 有序、RocketMQ 单 Queue 有序。
- 是否能解释不同 Consumer Group 为什么能独立消费。
- 是否能说出 Kafka 与 RocketMQ 在重试、延迟、事务上的差异。
- 是否能从模型层解释消息堆积和扩容上限。
