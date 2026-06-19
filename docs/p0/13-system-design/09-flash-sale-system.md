# 秒杀系统

## 面试定位

秒杀系统是系统设计 P0 题，综合考察高并发、限流、缓存、库存一致性、异步削峰、幂等、防超卖、防重复下单、热点治理和降级。面试官最关注：大流量如何挡住、库存如何不超卖、订单如何可靠创建、失败如何补偿。

核心思路：

```text
入口限流挡流量，Redis 原子扣库存，MQ 异步创建订单，DB 唯一键和库存流水兜底，超时释放和对账补偿闭环。
```

## 需求澄清

先问：

- 秒杀商品数量、库存数量、活动时间。
- 预计峰值 QPS、参与人数、成功订单量。
- 是否一人限购一件或多件。
- 是否允许排队，用户是否需要立即知道结果。
- 库存是否必须严格不超卖。
- 下单后多久未支付释放库存。
- 是否有风控、防刷、黑名单。
- 是否有多活动、多商品同时秒杀。

## 核心模型

核心对象：

- 活动：开始时间、结束时间、状态、规则。
- 秒杀商品：活动 ID、SKU ID、秒杀价、库存。
- 库存：Redis 可售库存、DB 库存、库存流水。
- 秒杀资格：用户是否可参与、限购数量。
- 秒杀订单：处理中、待支付、已支付、已关闭。

状态：

```text
活动：INIT -> WARMED -> RUNNING -> ENDED
订单：CREATING -> WAIT_PAY -> PAID / CLOSED
库存流水：PRE_DEDUCT -> CONFIRMED / RELEASED
```

## 架构方案

```text
用户请求
  -> CDN/静态页
  -> 网关限流、防刷
  -> 秒杀服务校验活动和资格
  -> Redis Lua 原子校验并扣库存
  -> 写入排队消息
  -> 返回排队中/抢购成功待确认
  -> 订单消费者创建订单
  -> 支付成功确认库存
  -> 超时未支付释放库存
```

系统分层：

- 活动页静态化，走 CDN。
- 活动配置和商品信息预热到缓存。
- 秒杀入口独立服务和独立资源池。
- Redis 承接高并发库存扣减。
- MQ 削峰创建订单。
- DB 保存最终订单和库存流水。

## 数据模型

### 秒杀活动

```sql
CREATE TABLE flash_sale_activity (
    id BIGINT PRIMARY KEY,
    activity_code VARCHAR(64) NOT NULL,
    status VARCHAR(32) NOT NULL,
    start_at DATETIME NOT NULL,
    end_at DATETIME NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_activity_code (activity_code)
);
```

### 秒杀商品

```sql
CREATE TABLE flash_sale_sku (
    id BIGINT PRIMARY KEY,
    activity_code VARCHAR(64) NOT NULL,
    sku_id BIGINT NOT NULL,
    sale_price BIGINT NOT NULL,
    total_stock INT NOT NULL,
    sold_stock INT NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_activity_sku (activity_code, sku_id)
);
```

### 秒杀订单

```sql
CREATE TABLE flash_sale_order (
    id BIGINT PRIMARY KEY,
    order_no VARCHAR(64) NOT NULL,
    activity_code VARCHAR(64) NOT NULL,
    sku_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    status VARCHAR(32) NOT NULL,
    amount BIGINT NOT NULL,
    expire_at DATETIME NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_order_no (order_no),
    UNIQUE KEY uk_user_activity_sku (user_id, activity_code, sku_id),
    KEY idx_status_expire (status, expire_at)
);
```

`uk_user_activity_sku` 防止同一用户重复下单。

### 库存流水

```sql
CREATE TABLE flash_sale_stock_flow (
    id BIGINT PRIMARY KEY,
    flow_no VARCHAR(64) NOT NULL,
    activity_code VARCHAR(64) NOT NULL,
    sku_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    order_no VARCHAR(64) NOT NULL,
    change_type VARCHAR(32) NOT NULL,
    quantity INT NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_flow_no (flow_no),
    KEY idx_order_no (order_no)
);
```

## 读链路

活动页：

```text
CDN 静态页
  -> 活动配置本地缓存
  -> 商品详情 Redis / 本地缓存
  -> 库存展示使用缓存近似值
```

注意：展示库存可以近似，真正扣库存必须原子校验。

## 写链路

### 抢购请求

```text
校验登录和风控
  -> 校验活动时间和状态
  -> 校验用户限购
  -> Redis Lua 原子扣库存和写用户标记
  -> 发送创建订单消息
  -> 返回排队中
```

Lua 逻辑：

```text
检查活动库存 > 0
检查用户未购买
扣减库存
写用户购买标记
返回成功
```

### 创建订单消费

```text
消费创建订单消息
  -> 幂等校验 eventId/orderNo
  -> 插入订单 WAIT_PAY
  -> 插入库存流水 PRE_DEDUCT
  -> 发送订单创建成功事件
```

DB 唯一键兜底重复消息和重复用户下单。

### 支付和释放

```text
支付成功回调
  -> 更新订单 WAIT_PAY -> PAID
  -> 更新库存流水 CONFIRMED

超时任务扫描
  -> 订单 WAIT_PAY 且 expire_at 过期
  -> 更新 CLOSED
  -> Redis 回补库存
  -> 库存流水 RELEASED
```

## 缓存、MQ 与一致性

### 缓存

- 活动配置预热到本地缓存和 Redis。
- 库存保存在 Redis，活动开始前预热。
- 售罄标记本地缓存，减少 Redis 访问。

### MQ

- 秒杀成功请求写 MQ，订单服务按能力消费。
- MQ 消息必须有 eventId、orderNo、userId、activityCode、skuId。
- 消费者幂等，失败重试，超过阈值进死信。

### 一致性

关键目标：

- 不超卖。
- 不重复下单。
- Redis 库存和 DB 订单最终一致。

手段：

- Redis Lua 原子扣库存。
- DB 唯一键防重复用户订单。
- 库存流水记录每次预扣、确认、释放。
- 定时对账 Redis、订单、库存流水。

## 容量估算

假设：

```text
参与用户 1000 万
活动持续 10 分钟
库存 10 万
峰值请求 QPS 20 万
成功订单最多 10 万
```

设计重点：

- 20 万 QPS 不能进 DB。
- Redis 和入口限流承接大部分请求。
- MQ 只承接扣库存成功的 10 万级消息。
- DB 写入按成功订单量估算，而不是按请求量估算。

机器估算：

```text
秒杀入口服务按 20 万 QPS 压测扩容
Redis 按热点 key 压力做分片/副本
订单消费者按可接受排队时间扩容
DB 按订单写 TPS 和索引数量估算
```

## 风险与降级

- 活动开始瞬间流量过高：网关限流、排队、验证码。
- Redis 热点库存 key：分桶库存、本地售罄标记。
- MQ 堆积：扩容消费者，暂停非核心消息。
- 创建订单失败：补偿释放 Redis 库存。
- 用户重复请求：用户购买标记 + DB 唯一键。
- 黄牛刷接口：登录态、设备、IP、行为风控、验证码。
- 活动配置错误：预热校验、灰度、紧急下线开关。

降级：

- 商品详情页只展示核心信息。
- 库存展示改成“有/无”。
- 非核心推荐、评论、埋点延后。
- 售罄后入口直接本地失败。

## 常见追问

### Redis 扣库存后订单创建失败怎么办？

发送补偿消息或定时扫描创建失败记录，回补 Redis 库存，并把库存流水标记为释放。所有补偿必须幂等。

### 如何防止超卖？

入口用 Redis Lua 原子扣减，DB 用库存流水和唯一键兜底，最终对账修复。若直接 DB 扣减，用 `available > 0` 条件更新。

### 如何防止重复下单？

Redis 写用户购买标记快速拦截，DB 用 `user_id + activity_code + sku_id` 唯一键兜底。

### 为什么不直接写 DB？

秒杀请求远大于库存，直接打 DB 会造成行锁和连接池爆满。应把失败请求挡在入口和 Redis 层，DB 只处理成功订单。

### 排队中用户怎么查结果？

提供查询接口，按 orderNo/requestId 查订单创建状态。也可以通过 WebSocket/推送通知，但查询接口仍要有。

## 自检清单

- 是否把活动页静态化和缓存预热说清。
- 是否有入口限流、防刷、风控。
- 是否使用 Redis 原子扣库存。
- 是否有 MQ 异步创建订单。
- 是否有 DB 唯一键防重复下单。
- 是否有库存流水、超时释放、对账补偿。
- 是否按请求量和成功订单量分别估算容量。
- 是否说明热点 key、售罄标记和降级方案。
- 是否说明支付后确认库存和未支付释放库存。
