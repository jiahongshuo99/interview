# 延迟消息

## 面试定位

延迟消息是业务面试和系统设计题的常见考点。常见问题：

- 什么是延迟消息，和定时任务有什么区别。
- 订单超时未支付如何关闭。
- Kafka 支不支持延迟消息。
- RocketMQ 延迟消息怎么实现。
- 延迟消息精度、容量、取消、重复触发怎么处理。
- 延迟消息和重试消息有什么区别。

回答时要强调：延迟消息本质是“消息先存储，达到指定时间后再投递”。它适合触发未来某个时间点的业务检查，但真正执行时仍要查询业务当前状态，不能只相信消息内容。

## 使用场景

常见场景：

- 订单创建后 30 分钟未支付自动关闭。
- 付款后 10 分钟检查是否发券成功。
- 外卖下单后预计时间提醒。
- 用户注册后 24 小时未激活发送提醒。
- 失败任务延迟重试。
- 活动开始前提醒。

以订单超时关闭为例：

```text
用户创建订单
  -> 写入订单状态 INIT
  -> 发送 30 分钟延迟消息 order-timeout-check
  -> 30 分钟后消费者收到消息
  -> 查询订单当前状态
  -> 如果仍是 INIT，则关闭订单
  -> 如果已支付，则忽略
```

注意：延迟消息触发的是“检查”，不是无条件执行。

## 延迟消息与定时任务

| 维度 | 延迟消息 | 定时任务扫描 |
| --- | --- | --- |
| 触发粒度 | 每个事件单独触发 | 按时间窗口批量扫描 |
| 实时性 | 通常更及时 | 取决于扫描周期 |
| 容量 | 依赖 MQ 延迟能力 | 依赖数据库扫描能力 |
| 可取消 | 需要业务状态判断或删除能力 | 修改数据库状态即可 |
| 排查 | 看消息轨迹和消费状态 | 看任务日志和扫描条件 |

常见组合：

- 延迟消息负责及时触发。
- 定时任务负责兜底补偿。

不要只依赖延迟消息处理关键业务，因为消息可能延迟、重复、丢失或消费失败。

## RocketMQ 延迟消息

RocketMQ 常见版本支持延迟等级。延迟等级不是任意时间，而是预设等级，例如：

```text
1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

具体等级以集群配置为准。

发送示例：

```java
Message<String> message = MessageBuilder.withPayload(json)
        .setHeader(MessageConst.PROPERTY_KEYS, orderId)
        .build();

// delayLevel=16 通常表示 30m，具体以 broker 配置为准
rocketMQTemplate.syncSend("order-timeout-check", message, 3000, 16);
```

原生 API：

```java
Message message = new Message(
        "order-timeout-check",
        "created",
        orderId,
        body
);
message.setDelayTimeLevel(16);
producer.send(message);
```

RocketMQ 5.x 等新版本支持更灵活的定时 / 延时消息能力，但实际能否使用要看部署环境的集群版本和配置。面试回答时要说明“依赖版本和部署能力”。

## RocketMQ 延迟消息原理

以延迟等级为例，简化理解：

```text
Producer 发送延迟消息
  -> Broker 不立即投递到真实 Topic
  -> 按延迟等级写入延迟队列
  -> 定时任务扫描到期消息
  -> 还原真实 Topic
  -> Consumer 正常消费
```

关键点：

- 延迟消息不是消费者睡眠。
- Broker 负责保存未到期消息。
- 到期后重新投递给真实 Topic。
- 到期时间不是强实时，可能有秒级或更高延迟。

## Kafka 延迟消息方案

Kafka 本身更偏追加日志，不像 RocketMQ 那样普遍内建业务延迟等级。常见实现方式有：

### 方案一：延迟 Topic 分层

建立多个延迟 Topic：

```text
order-delay-1m
order-delay-5m
order-delay-30m
```

消费者从延迟 Topic 拉取，到期后投递到真实 Topic。如果未到期，可以暂停或重新投递到更接近的延迟 Topic。

优点：

- 基于 Kafka 实现。
- 可控。

缺点：

- 实现复杂。
- 精度有限。
- 容易引入重复投递。

### 方案二：时间轮服务

独立 Delay Service 管理延迟任务：

```text
业务服务 -> Delay Service -> 存储延迟任务
Delay Service 到期 -> Kafka real topic
```

底层可用时间轮、数据库、Redis ZSet、RocksDB 等。

优点：

- 能支持任意时间。
- 可做取消、查询、重试。

缺点：

- 需要自研或维护服务。
- Delay Service 自身要保证可靠性。

### 方案三：数据库扫描

把待触发任务写入数据库：

```sql
CREATE TABLE delay_task (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    biz_type VARCHAR(64) NOT NULL,
    biz_key VARCHAR(128) NOT NULL,
    execute_at DATETIME NOT NULL,
    status VARCHAR(32) NOT NULL,
    retry_count INT NOT NULL DEFAULT 0,
    UNIQUE KEY uk_biz (biz_type, biz_key),
    KEY idx_status_execute_at (status, execute_at)
);
```

定时任务扫描：

```sql
SELECT *
FROM delay_task
WHERE status = 'NEW'
  AND execute_at <= NOW()
ORDER BY execute_at
LIMIT 100;
```

优点：

- 简单可靠。
- 易查询、易取消、易补偿。

缺点：

- 扫描压力要控制。
- 实时性取决于调度频率。

## 延迟消息消费示例

订单超时关闭：

```java
@RocketMQMessageListener(
        topic = "order-timeout-check",
        consumerGroup = "order-timeout-service"
)
public class OrderTimeoutListener implements RocketMQListener<OrderTimeoutEvent> {
    @Override
    public void onMessage(OrderTimeoutEvent event) {
        orderTimeoutService.closeIfUnpaid(event.getOrderId());
    }
}
```

业务逻辑：

```java
@Transactional
public void closeIfUnpaid(String orderId) {
    Order order = orderRepository.findByOrderIdForUpdate(orderId);
    if (order == null) {
        return;
    }
    if (!"INIT".equals(order.getStatus())) {
        return;
    }
    orderRepository.close(orderId);
    inventoryService.release(orderId);
}
```

关键点：

- 到期后重新查订单状态。
- 已支付则忽略。
- 关闭操作要幂等。

## 延迟消息取消

多数 MQ 延迟消息一旦发出，不容易像本地任务那样删除。常见做法不是取消消息，而是让消息到期后检查状态。

示例：

```text
创建订单 -> 发送 30 分钟延迟检查
用户 10 分钟后支付 -> 订单状态变 PAID
30 分钟后延迟消息到达 -> 查询状态 PAID -> 忽略
```

这种方式要求：

- 消费逻辑幂等。
- 状态判断准确。
- 不把延迟消息当作命令，而当作检查触发器。

如果业务强依赖取消能力，可以使用自建 Delay Service 或数据库 delay_task，通过状态字段取消：

```sql
UPDATE delay_task
SET status = 'CANCELLED'
WHERE biz_type = 'ORDER_TIMEOUT'
  AND biz_key = ?;
```

## 延迟消息精度

延迟消息不是精确定时器。

影响因素：

- Broker 扫描周期。
- 延迟队列堆积。
- Broker 负载。
- 消费者堆积。
- 网络和调度延迟。
- 客户端重试。

如果业务要求毫秒级强准时，普通 MQ 延迟消息通常不合适。

面试回答：

- 延迟消息适合秒级、分钟级最终触发。
- 不能承诺绝对准时。
- 到期后还要看消费者是否有堆积。

## 延迟消息与重试消息

延迟消息：

- 业务主动指定未来时间触发。
- 例如订单 30 分钟后关闭检查。

重试消息：

- 消费失败后系统安排稍后再次投递。
- 触发原因是失败。

二者都可能使用延迟队列实现，但业务语义不同。

## 线上风险

### 延迟消息堆积

表现：

- 到期消息不能及时投递。
- 订单超时关闭延迟。
- 消费 lag 增长。

处理：

- 查看 Broker 延迟队列堆积。
- 查看真实 Topic 消费堆积。
- 扩容消费者。
- 降低单条消费耗时。

### 消息重复到达

延迟消息到期后消费失败会重试，可能重复关闭订单。

处理：

- 状态机幂等。
- 关闭订单 SQL 带状态条件。
- 外部释放库存也要幂等。

### 延迟时间不匹配

RocketMQ 延迟等级可能没有精确的 45 分钟，业务误以为支持任意时间。

处理：

- 核对集群延迟等级。
- 不支持时使用数据库扫描或 Delay Service。

### 大量延迟消息占用 Broker

活动场景一次性写入大量长延迟消息，会增加 Broker 存储和扫描压力。

处理：

- 评估容量。
- 分批投递。
- 对长时间延迟任务考虑数据库任务表。

## 排查优化

排查延迟消息未触发：

```text
生产者是否发送成功
延迟等级或执行时间是否正确
Broker 是否支持该延迟能力
到期后是否投递到真实 Topic
消费者 group 是否正确
是否消费失败进入重试或死信
业务状态是否已经变化导致被忽略
```

优化建议：

- 延迟消息体只放业务 ID，不放完整可变状态。
- 到期后查询数据库当前状态。
- 核心任务加定时扫描兜底。
- 建立延迟触发成功率、触发延迟、消费失败、死信监控。

## 常见追问

### 订单超时关闭用延迟消息还是定时任务？

可以组合使用。延迟消息负责及时触发，定时任务负责兜底补偿。消费者收到延迟消息后必须查询订单当前状态，未支付才关闭，已支付直接忽略。

### Kafka 如何实现延迟消息？

常见做法是延迟 Topic、多级时间轮、Redis ZSet、数据库任务表或独立 Delay Service。Kafka 本身不应被简单当作任意精度延迟队列使用。

### 延迟消息能取消吗？

很多 MQ 中已发送的延迟消息不容易取消。常见做法是到期后检查业务状态，状态已变化则忽略。如果强取消需求明显，应使用任务表或 Delay Service 保存可取消状态。

### 延迟消息到期后一定准时消费吗？

不一定。到期只表示可以投递，还受 Broker 扫描、Topic 堆积、消费者处理能力影响。业务上要接受一定延迟，并通过监控和补偿兜底。

## 易错点

- 把延迟消息当精确定时器。
- 延迟消息到期后不查业务状态，直接关闭订单。
- 认为延迟消息发送后可以随意取消。
- 使用 RocketMQ 延迟等级前不核对集群配置。
- 长延迟、大批量任务全部压到 MQ，不评估 Broker 容量。
- 没有定时任务兜底。

## 自检清单

- 是否能说明延迟消息适合哪些业务场景。
- 是否能解释订单超时关闭的正确流程。
- 是否知道 RocketMQ 延迟等级和版本依赖。
- 是否能说出 Kafka 延迟消息常见实现方案。
- 是否能解释为什么到期后仍要查业务状态。
- 是否能处理延迟消息重复消费。
- 是否能设计延迟消息的监控和兜底补偿。
