# 幂等设计

## 面试定位

幂等设计是订单、支付、发券、消息消费、回调、补偿任务的基础能力。Java 后端面试里，面试官会重点看你能不能识别重复来源，并用业务唯一键、数据库唯一索引、幂等表、状态机、Redis 防重、MQ 消费幂等组合出可靠方案。

核心观点：

```text
分布式系统无法避免重复请求，只能让重复请求产生相同业务结果。
```

## 需求澄清

先问：

- 什么操作需要幂等：创建、支付、发券、扣库存、回调、消费消息。
- 重复请求来自哪里：前端重复提交、RPC 重试、MQ 重投、第三方回调、补偿任务。
- 重复时要返回同一结果，还是只保证不重复副作用。
- 幂等窗口多长：几分钟、几小时、永久。
- 请求参数是否需要校验一致。
- 业务是否有天然唯一键。
- 是否涉及多个资源和外部系统。
- 幂等失败时是返回处理中、成功、失败还是重试。

## 核心模型

### 幂等键

幂等键定义“什么算同一次业务操作”。

常见选择：

| 幂等键 | 适用 | 风险 |
| --- | --- | --- |
| requestId | 前端/API 请求 | 客户端必须稳定生成和透传 |
| orderNo | 订单创建、支付 | 业务号必须唯一 |
| eventId | MQ 事件 | 生产者必须生成全局唯一事件 |
| callbackNo | 第三方回调 | 依赖第三方字段可靠 |
| userId + activityId | 领取类业务 | 要确认每用户是否只允许一次 |

推荐优先级：

```text
业务唯一键 > eventId/requestId > MQ messageId
```

不要只用时间戳、用户 ID 或随机参数做幂等键。

### 幂等状态

通用状态机：

```text
INIT / PROCESSING -> SUCCESS
INIT / PROCESSING -> FAILED
FAILED -> PROCESSING -> SUCCESS
```

重复请求处理：

- `SUCCESS`：返回历史结果。
- `PROCESSING`：返回处理中或稍后重试。
- `FAILED`：根据业务决定返回失败或允许重试。

## 方案一：数据库唯一索引

适合创建类操作。

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    order_no VARCHAR(64) NOT NULL,
    user_id BIGINT NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at DATETIME NOT NULL,
    UNIQUE KEY uk_order_no (order_no)
);
```

处理逻辑：

```java
public OrderDTO createOrder(CreateOrderCommand command) {
    try {
        orderRepository.insert(command.toOrder());
        return orderRepository.findByOrderNo(command.getOrderNo());
    } catch (DuplicateKeyException e) {
        Order existing = orderRepository.findByOrderNo(command.getOrderNo());
        return OrderDTO.from(existing);
    }
}
```

优点：

- 简单可靠。
- 数据库兜底并发。
- 不依赖应用锁。

缺点：

- 只能防同一唯一键重复。
- 不能天然保存历史响应。
- 唯一键设计错会误判。

## 方案二：幂等表

适合需要记录请求状态和响应结果的接口。

```sql
CREATE TABLE idempotent_record (
    id BIGINT PRIMARY KEY,
    biz_type VARCHAR(64) NOT NULL,
    idem_key VARCHAR(128) NOT NULL,
    request_hash VARCHAR(128) NOT NULL,
    status VARCHAR(32) NOT NULL,
    result TEXT DEFAULT NULL,
    expire_at DATETIME NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_biz_key (biz_type, idem_key),
    KEY idx_expire_at (expire_at)
);
```

流程：

```text
收到请求
  -> 计算 request_hash
  -> 插入幂等记录 PROCESSING
  -> 插入成功，执行业务
  -> 业务成功，保存 result 并标记 SUCCESS
  -> 重复请求查询记录
     -> hash 不一致，拒绝
     -> SUCCESS，返回历史结果
     -> PROCESSING，返回处理中
     -> FAILED，按策略重试或返回失败
```

关键点：

- `biz_type + idem_key` 唯一。
- `request_hash` 防止同一个 key 对应不同请求体。
- 业务处理和幂等状态更新要尽量在同一事务。
- `PROCESSING` 要有超时恢复。
- 结果过大时只保存结果摘要或业务 ID。

## 方案三：状态机幂等

适合订单、支付、券状态等有明确状态流转的业务。

```sql
UPDATE orders
SET status = 'PAID',
    paid_at = NOW()
WHERE order_no = ?
  AND status = 'CREATED';
```

重复支付回调：

- 第一次从 `CREATED` 改成 `PAID`，更新行数为 1。
- 第二次更新行数为 0，查询订单已是 `PAID`，返回成功。

状态机幂等比单纯去重更贴近业务，因为它能识别非法状态。

## 方案四：Redis 防重

适合短时间重复提交拦截。

```java
String lockKey = "idem:createOrder:" + requestId;
boolean first = redis.set(lockKey, "1", "NX", "EX", 60);
if (!first) {
    throw new DuplicateSubmitException();
}
```

注意：

- Redis 防重不等于完整幂等。
- Redis 成功后业务失败，需要考虑是否释放 key。
- Redis 丢数据或过期后仍可能重复。
- 核心业务必须有 DB 唯一键或状态机兜底。

适合前置削峰，不适合作为支付、发券、扣款的唯一保护。

## 方案五：MQ 消费幂等

消费者必须假设消息会重复。

```text
收到 OrderPaidEvent
  -> bizKey = orderNo + eventType
  -> 插入消费记录 PROCESSING
  -> 发放权益
  -> 更新消费记录 SUCCESS
```

消费记录和业务写入最好在同一个本地事务里。如果不能同事务，要确保业务表有唯一键或状态机兜底。

## 架构方案

通用幂等架构：

```text
入口层：requestId、防重、限流
业务层：幂等表、状态机、参数 hash
数据层：唯一索引、条件更新
异步层：eventId、消费记录、重试/死信
补偿层：按业务状态重放，重复执行安全
```

不同场景组合：

| 场景 | 推荐方案 |
| --- | --- |
| 创建订单 | orderNo 唯一索引 + 返回已有订单 |
| 支付回调 | callbackNo 去重 + 订单状态机 |
| 发优惠券 | userId + activityId 唯一索引 + MQ 消费幂等 |
| 扣库存 | 库存流水唯一键 + 条件扣减 |
| 文件上传 | fileHash / uploadId 幂等 + 状态机 |
| 消息通知 | bizKey 去重 + 渠道发送记录 |

## 容量估算

幂等记录容量：

```text
记录数 = 峰值请求量 * 幂等保留时间
存储 = 记录数 * 单行大小 * 索引膨胀
```

示例：

```text
支付回调日 1000 万
幂等记录保留 30 天
总记录 3 亿
单行 500B，原始约 150GB
加索引和副本后需要按数百 GB 规划
```

过期策略：

- 支付、订单等审计类记录长期保留或归档。
- 普通 API requestId 可保留几小时到几天。
- MQ 消费记录按业务重放窗口保留。

## 风险与降级

- 幂等 key 不稳定：重复请求无法识别。
- 幂等 key 粒度过粗：误拦正常请求。
- 幂等 key 粒度过细：拦不住业务重复。
- `PROCESSING` 永久卡住：需要超时恢复。
- Redis 防重丢失：需要 DB 兜底。
- 响应结果保存过大：只保存业务 ID 或摘要。
- 幂等表成为热点：按 biz_type 或 hash 分表。

降级思路：

- 幂等服务异常时，核心写接口宁可失败也不要裸奔。
- 非核心接口可以只做唯一索引兜底。
- 重复请求过多时入口限流。

## 常见追问

### 幂等和防重有什么区别？

防重偏入口，防止短时间重复提交。幂等偏结果，即使重复执行也不会产生重复副作用。防重是幂等体系的一部分，但不能替代幂等。

### 同一个 requestId 参数不一样怎么办？

保存请求参数 hash。重复请求如果 key 相同但 hash 不同，说明客户端复用了幂等 key，要拒绝并报警。

### PROCESSING 状态卡住怎么办？

记录更新时间和超时时间。超过阈值后允许抢占重试，或由补偿任务根据业务状态修复为 SUCCESS/FAILED。

### 幂等表和业务表如何保持一致？

最好放在同一个本地事务。如果跨库或外部系统，业务表必须有唯一索引/状态机兜底，幂等表只做辅助记录。

### 第三方接口不幂等怎么办？

本系统先做本地幂等和状态控制，调用前生成外部幂等号。如果第三方不支持幂等，只能减少重试、人工对账补偿，不能盲目重复调用。

## 自检清单

- 是否识别重复来源。
- 是否定义了稳定的幂等键。
- 是否说明重复请求要返回什么。
- 是否有唯一索引或状态机兜底。
- 是否处理 PROCESSING、SUCCESS、FAILED。
- 是否校验请求参数 hash。
- 是否说明 Redis 防重的局限。
- 是否覆盖 MQ 重复消费。
- 是否估算幂等记录容量和过期策略。
