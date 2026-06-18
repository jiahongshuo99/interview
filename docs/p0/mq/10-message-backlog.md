# 消息堆积排查

## 面试定位

消息堆积是 MQ 线上排查题的高频场景。面试常问：

- 什么是消息堆积，如何判断。
- Kafka lag 怎么看。
- RocketMQ diffTotal 怎么看。
- 堆积时能不能直接加消费者。
- 消费慢的原因有哪些。
- 如何快速止血，如何长期优化。

核心回答：消息堆积表示生产速度长期大于消费速度。排查要先定位瓶颈在生产突增、Broker、消费者、下游依赖、分区热点还是消费组配置，再决定扩容、限流、优化、跳过、重放或补偿。

## 什么是消息堆积

Kafka 中常用 lag 表示堆积：

```text
lag = log end offset - committed offset
```

示例：

```text
Partition 0:
  LOG-END-OFFSET = 10000
  CURRENT-OFFSET = 8000
  LAG = 2000
```

RocketMQ 中常见指标是 `diffTotal`，表示消费进度和最大进度之间的差值。

堆积不一定是故障，关键看：

- lag 是否持续增长。
- 预计追平时间是否超过业务 SLA。
- 消息是否会超过保留时间。
- 是否影响下游业务时效。

## 排查总流程

```text
1. 确认堆积范围
   -> 哪个 topic
   -> 哪个 consumer group
   -> 哪些 partition / queue

2. 判断增长趋势
   -> 瞬时峰值还是持续增长
   -> 生产 TPS 和消费 TPS 差多少

3. 定位瓶颈
   -> 生产突增
   -> 消费异常
   -> 下游慢
   -> 分区热点
   -> 重试风暴
   -> Broker 异常

4. 止血
   -> 限流生产
   -> 扩容消费者
   -> 降级慢逻辑
   -> 暂停非核心消费
   -> 隔离毒丸消息

5. 恢复和复盘
   -> 追平 lag
   -> 补偿失败消息
   -> 调整容量和告警
```

## Kafka 堆积查看

常用命令：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server kafka-1:9092 \
  --describe \
  --group coupon-service
```

输出关注：

```text
TOPIC        PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID
order-paid   0          8000            10000           2000 consumer-1
order-paid   1          9950            10000           50   consumer-2
```

判断：

- 所有 Partition lag 都高：整体消费能力不足或下游慢。
- 单个 Partition lag 高：热点 key、毒丸消息或分区级异常。
- CONSUMER-ID 为空：没有消费者实例消费该分区。
- lag 不降反升：消费 TPS 小于生产 TPS。

## RocketMQ 堆积查看

常用命令：

```bash
mqadmin consumerProgress \
  -n nameserver:9876 \
  -g coupon-service
```

关注：

- topic。
- queue 维度差值。
- diffTotal。
- consumer 在线情况。

也可以查看消费组连接：

```bash
mqadmin consumerConnection \
  -n nameserver:9876 \
  -g coupon-service
```

关注消费者实例是否在线、订阅关系是否一致。

## 常见原因一：生产流量突增

表现：

- 生产 TPS 突然升高。
- 消费 TPS 没有下降，但 lag 增长。
- 活动、批量任务、补偿任务同时发消息。

处理：

- 确认是否符合预期活动流量。
- 对生产端限流或分批发送。
- 临时扩容消费者。
- 确认 Broker 磁盘和网络容量。
- 核算追平时间。

追平时间估算：

```text
堆积量 = 1,000,000
生产速度 = 2,000 msg/s
消费速度 = 5,000 msg/s
净追赶速度 = 3,000 msg/s
预计追平 = 1,000,000 / 3,000 = 333 秒
```

如果消费速度小于等于生产速度，永远追不平。

## 常见原因二：消费者异常

表现：

- 消费 TPS 降为 0 或大幅下降。
- 错误日志增多。
- 消息不断重试。
- 消费者实例频繁重启。

可能原因：

- 新版本代码 bug。
- 反序列化失败。
- NPE 或业务异常。
- 配置错误。
- 依赖不可用。

处理：

- 回滚有问题版本。
- 修复异常后重启消费。
- 对毒丸消息进入死信或失败表。
- 不要 catch 异常后无限快速重试。

## 常见原因三：下游依赖慢

表现：

- 消费线程都在等待 RPC、DB、Redis。
- 消费 RT 上升。
- 消费者 CPU 不高，但吞吐下降。
- 下游连接池满或慢查询增多。

处理：

- 给下游限流，避免打垮。
- 增加超时和熔断。
- 批量写入或批量查询。
- 优化慢 SQL。
- 对非核心下游降级。
- 临时扩大下游容量。

注意：如果瓶颈是数据库，加消费者可能会让数据库更慢。

## 常见原因四：分区热点

表现：

- 只有少数 Partition / Queue lag 高。
- 对应消费者实例负载高。
- 某个 key 的消息量远大于其他 key。

原因：

- 分区 key 选择不均匀。
- 大商家、大用户、大活动集中。
- 顺序消息按热点 key 路由。

处理：

- 分析消息 key 分布。
- 热点 key 拆桶。
- 热点业务单独 Topic。
- 顺序要求不强时调整路由策略。
- 增加分区不一定立即解决已有热点消息。

## 常见原因五：消费者并行度不足

表现：

- 所有消费者都健康。
- 单实例消费能力正常。
- lag 增长。
- 消费者实例数小于分区数。

处理：

- 增加消费者实例。
- 提高 listener concurrency。
- 增加消费线程数。
- 使用批量消费。

限制：

- 同一消费组最大有效实例数不超过 Partition / Queue 数。
- 顺序消费不能随意提高并发。
- 下游能力必须同步评估。

Kafka 配置示例：

```properties
spring.kafka.listener.concurrency=8
spring.kafka.consumer.max-poll-records=500
```

RocketMQ 配置示例：

```properties
rocketmq.consumer.consume-thread-min=20
rocketmq.consumer.consume-thread-max=64
```

具体配置名会随框架封装不同而变化，要以项目使用的客户端为准。

## 常见原因六：重试风暴或死信堆积

表现：

- 同一批消息反复失败。
- 重试 Topic 或 retry 队列增长。
- 正常消息被失败消息挤占。
- 日志中相同异常刷屏。

处理：

- 识别异常类型。
- 临时暂停或限速消费。
- 不可恢复消息进入死信。
- 修复数据或代码后重放。
- 对下游故障使用退避重试。

## 常见原因七：Broker 或集群问题

表现：

- 多个消费组同时堆积。
- 生产和消费请求延迟都高。
- Broker 磁盘水位高。
- 网络抖动。
- 副本同步异常。

处理：

- 查看 Broker CPU、磁盘、网络。
- 查看 Kafka ISR、Controller、Under Replicated Partitions。
- 查看 RocketMQ Broker 主从状态、刷盘耗时。
- 扩容 Broker 或迁移分区。
- 清理无用 Topic 或调整保留策略。

## 止血策略

### 扩容消费者

适合：

- 消费端 CPU 或线程成为瓶颈。
- 下游仍有容量。
- 分区数足够。

不适合：

- 下游数据库已经慢。
- 单分区热点。
- 顺序消费被单条消息阻塞。

### 限流生产者

适合：

- 生产突增超过系统容量。
- 消息不是必须立刻进入 MQ。

方式：

- 活动入口限流。
- 批量任务分批。
- 降低非核心消息发送。

### 跳过或隔离坏消息

适合：

- 单条毒丸消息阻塞。
- 消息格式错误且无法自动修复。

方式：

- 投递到 DLQ。
- 保存失败表。
- 人工修复后重放。

注意：跳过消息必须评估业务影响。

### 临时降级消费逻辑

例如消费者同时做三件事：

```text
发券
更新画像
同步搜索
```

如果堆积影响核心发券，可以临时关闭非核心画像和搜索逻辑，先追平核心链路。

## 长期优化

- Topic 分区数按容量规划。
- 消费端批量处理。
- 下游调用设置超时、限流、熔断。
- 消费逻辑拆分核心和非核心。
- 热点 key 治理。
- 消息体瘦身。
- 建立 lag、消费 TPS、消费 RT、失败率、重试量、死信量告警。
- 压测生产峰值和消费追平能力。

## 代码优化示例

### 批量查询替代逐条查询

低效：

```java
for (OrderPaidEvent event : events) {
    User user = userClient.getUser(event.getUserId());
    couponService.send(user, event);
}
```

优化：

```java
List<String> userIds = events.stream()
        .map(OrderPaidEvent::getUserId)
        .distinct()
        .toList();

Map<String, User> userMap = userClient.batchGet(userIds);
for (OrderPaidEvent event : events) {
    couponService.send(userMap.get(event.getUserId()), event);
}
```

### 状态条件更新避免重复慢处理

```sql
UPDATE coupon_task
SET status = 'SUCCESS',
    updated_at = NOW()
WHERE order_id = ?
  AND status = 'INIT';
```

重复消息快速返回，减少堆积期间的无效下游调用。

## 常见追问

### 堆积了直接加消费者可以吗？

不一定。只有当消费端本身是瓶颈、分区数足够、下游也能承受时，加消费者才有效。如果瓶颈是单分区热点、顺序阻塞或数据库慢，加消费者可能没效果甚至加重故障。

### lag 很高但业务没报警，要不要处理？

要看业务 SLA 和消息保留时间。如果是离线数仓类消息，允许慢慢追；如果是订单发券，延迟会影响用户体验。还要确认堆积是否可能超过 retention 导致消息被清理。

### 单个 Partition lag 很高怎么办？

优先怀疑热点 key 或毒丸消息。查看该 Partition 消息 key、消费者日志和处理耗时。增加消费者实例通常无效，因为一个 Partition 同时只能由一个消费者实例消费。

### 如何估算恢复时间？

用堆积量除以净消费速度：

```text
预计追平时间 = 当前 lag / (消费 TPS - 生产 TPS)
```

如果消费 TPS 小于生产 TPS，必须扩容、优化或限流，否则无法追平。

## 易错点

- 看到堆积就盲目扩消费者。
- 忽略 Partition / Queue 数上限。
- 下游已经慢还继续加消费并发。
- 只看总 lag，不看分区维度。
- 毒丸消息无限重试，阻塞整个分区。
- 没有监控消息保留时间，堆积到过期才发现。
- 临时 reset offset 跳过消息但没有业务补偿。

## 自检清单

- 是否能解释 Kafka lag 和 RocketMQ diffTotal。
- 是否能按 topic、group、partition / queue 定位堆积范围。
- 是否能区分生产突增、消费慢、下游慢、热点、重试风暴。
- 是否知道加消费者的前提和上限。
- 是否能估算追平时间。
- 是否能给出止血和长期优化方案。
- 是否能说明跳过消息或 reset offset 的业务风险。
