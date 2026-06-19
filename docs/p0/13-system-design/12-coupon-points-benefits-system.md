# 优惠券 / 积分 / 权益系统

## 面试定位

优惠券、积分、权益系统是营销和交易系统的典型设计题。面试官会关注库存、发放、领取、使用、核销、过期、退款返还、幂等、防重复领取、批量发放、异步削峰和账务一致性。

核心回答：

```text
优惠券和权益要用状态机管理生命周期，积分要用账户和流水保证可追溯；发放和核销必须幂等，库存和使用结果要可对账。
```

## 需求澄清

先问：

- 是优惠券、积分、会员权益，还是组合系统。
- 优惠券是用户领取还是系统发放。
- 是否有库存和每人限领。
- 使用规则是什么：门槛、品类、时间、叠加、互斥。
- 积分是可消费资产还是成长值。
- 权益是否有有效期和次数。
- 是否支持退款返还。
- 发放量级、核销量级、查询 QPS。
- 是否需要营销活动和风控。

## 核心模型

### 优惠券

- 券模板：规则、有效期、库存。
- 用户券：用户持有的一张券。
- 券库存：总量、已发、已用。
- 券使用流水：锁定、核销、释放。

状态：

```text
UNUSED -> LOCKED -> USED
UNUSED -> EXPIRED
LOCKED -> UNUSED
```

### 积分

- 积分账户：用户当前可用积分。
- 积分流水：获得、消费、冻结、解冻、过期。
- 积分规则：来源、有效期、兑换比例。

积分账户是当前态，积分流水是事实源。

### 权益

- 权益定义：权益类型、规则、有效期。
- 用户权益：用户持有权益。
- 权益核销记录：使用次数和结果。

权益可以是次数型、周期型、身份型。

## 架构方案

```text
营销活动
  -> 发放任务 / 用户领取
  -> 库存校验和幂等
  -> 写用户券/积分/权益
  -> 发送发放事件

交易下单
  -> 查询可用优惠
  -> 试算价格
  -> 锁定优惠券/积分/权益
  -> 支付成功核销
  -> 订单取消/超时释放
```

服务边界：

- 营销规则服务：规则和试算。
- 资产服务：用户券、积分、权益当前态。
- 流水服务：发放、核销、返还记录。
- 任务服务：批量发放、过期、对账。

## 数据模型

### 券模板

```sql
CREATE TABLE coupon_template (
    id BIGINT PRIMARY KEY,
    template_code VARCHAR(64) NOT NULL,
    coupon_name VARCHAR(128) NOT NULL,
    coupon_type VARCHAR(32) NOT NULL,
    threshold_amount BIGINT NOT NULL,
    discount_amount BIGINT NOT NULL,
    total_stock INT NOT NULL,
    issued_stock INT NOT NULL,
    valid_from DATETIME NOT NULL,
    valid_to DATETIME NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_template_code (template_code)
);
```

### 用户券

```sql
CREATE TABLE user_coupon (
    id BIGINT PRIMARY KEY,
    user_coupon_no VARCHAR(64) NOT NULL,
    user_id BIGINT NOT NULL,
    template_code VARCHAR(64) NOT NULL,
    status VARCHAR(32) NOT NULL,
    source_biz_no VARCHAR(128) NOT NULL,
    valid_from DATETIME NOT NULL,
    valid_to DATETIME NOT NULL,
    locked_order_no VARCHAR(64) DEFAULT NULL,
    used_order_no VARCHAR(64) DEFAULT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_user_coupon_no (user_coupon_no),
    UNIQUE KEY uk_source_biz (source_biz_no),
    KEY idx_user_status_valid (user_id, status, valid_to)
);
```

`source_biz_no` 用于发放幂等，例如活动发放流水。

### 积分账户与流水

```sql
CREATE TABLE points_account (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    available_points BIGINT NOT NULL,
    frozen_points BIGINT NOT NULL,
    version BIGINT NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_user_id (user_id)
);
```

```sql
CREATE TABLE points_flow (
    id BIGINT PRIMARY KEY,
    flow_no VARCHAR(64) NOT NULL,
    user_id BIGINT NOT NULL,
    change_type VARCHAR(32) NOT NULL,
    points BIGINT NOT NULL,
    biz_no VARCHAR(128) NOT NULL,
    expire_at DATETIME DEFAULT NULL,
    created_at DATETIME NOT NULL,
    UNIQUE KEY uk_flow_no (flow_no),
    UNIQUE KEY uk_biz_no (biz_no),
    KEY idx_user_created (user_id, created_at)
);
```

### 权益发放记录

```sql
CREATE TABLE benefit_grant (
    id BIGINT PRIMARY KEY,
    grant_no VARCHAR(64) NOT NULL,
    user_id BIGINT NOT NULL,
    benefit_code VARCHAR(64) NOT NULL,
    total_times INT NOT NULL,
    used_times INT NOT NULL,
    status VARCHAR(32) NOT NULL,
    valid_from DATETIME NOT NULL,
    valid_to DATETIME NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_grant_no (grant_no),
    KEY idx_user_benefit (user_id, benefit_code, status)
);
```

## 发放链路

### 用户领取优惠券

```text
校验活动状态
  -> 校验用户资格和限领
  -> 扣减券模板库存
  -> 创建 user_coupon
  -> 记录发放流水
  -> 返回领取成功
```

防重复：

- `user_id + template_code + activity_code` 唯一。
- 发放请求带 `source_biz_no`。
- Redis 可做短期防重，DB 唯一键兜底。

### 批量发放

```text
导入用户包
  -> 创建发放任务
  -> 拆分批次
  -> MQ 异步消费
  -> 幂等发放
  -> 统计成功/失败
```

批量发放必须支持断点续跑和失败重试。

## 使用链路

### 下单锁定

```text
用户选择优惠
  -> 校验券/积分/权益可用
  -> 计算价格
  -> 条件更新 UNUSED -> LOCKED
  -> 冻结积分或权益次数
  -> 返回锁定结果
```

券锁定：

```sql
UPDATE user_coupon
SET status = 'LOCKED',
    locked_order_no = ?,
    updated_at = NOW()
WHERE user_coupon_no = ?
  AND user_id = ?
  AND status = 'UNUSED'
  AND valid_to > NOW();
```

### 支付成功核销

```text
支付成功事件
  -> 幂等消费
  -> LOCKED -> USED
  -> 冻结积分转消费
  -> 权益次数确认使用
  -> 写核销流水
```

### 订单关闭释放

```text
订单关闭事件
  -> LOCKED -> UNUSED
  -> 解冻积分
  -> 释放权益次数
```

### 退款返还

是否返还要看规则：

- 未过有效期可以返还。
- 已过期是否返还由业务决定。
- 部分退款要按分摊规则返还。

返还必须有独立流水，不能直接改历史流水。

## 缓存、MQ 与一致性

缓存：

- 券模板和营销规则适合缓存。
- 用户可用券列表可以短 TTL 缓存。
- 库存热点活动可用 Redis 预扣。

MQ：

- 批量发放。
- 支付成功核销。
- 订单关闭释放。
- 积分过期。
- 通知和数据分析。

一致性：

- 用户资产变更必须有流水。
- 当前态和流水要能对账。
- 支付成功核销失败要重试补偿。
- 订单关闭释放失败要补偿。

## 容量估算

示例：

```text
日发券 5000 万
日核销 500 万
用户查券峰值 QPS 20000
券模板 10 万
```

设计：

- 券模板缓存。
- 用户券表按 user_id 分库分表。
- 发放和核销流水长期保留或归档。
- 批量发券用 MQ 削峰。
- 查询可用券走 user_id + status + valid_to 索引。

## 风险与降级

- 超发券：库存扣减必须原子，DB 条件更新或 Redis + 对账。
- 重复发放：source_biz_no 唯一键。
- 重复核销：订单号 + user_coupon_no 唯一。
- 优惠计算不一致：价格快照和规则版本。
- 积分账户并发：乐观锁或行锁。
- 过期任务漏跑：查询时也校验有效期。
- 批量发放堆积：任务拆批、限速、失败重试。

降级：

- 营销规则服务异常时不展示非核心优惠。
- 用户券列表可返回缓存。
- 支付核销失败时订单进入处理中并补偿，不能直接丢失权益状态。

## 常见追问

### 如何防止优惠券超发？

券模板库存用条件更新或 Redis 原子扣减，用户券发放用唯一键兜底。高并发活动要有发放流水和对账。

### 积分为什么要流水？

账户余额只是当前态，流水用于审计、对账、退款返还和问题排查。余额可以由流水重算校验。

### 下单锁券后不支付怎么办？

订单超时关闭事件释放券、积分和权益。也要有定时任务扫描长时间 LOCKED 的资产做补偿。

### 退款时优惠券返还吗？

看规则。要在下单时保存优惠分摊和规则版本，退款时按规则返还或不返还，并记录返还流水。

### 可用券查询怎么优化？

按 `user_id + status + valid_to` 建索引，券模板缓存，规则计算本地化。复杂规则可以先粗筛再精算。

## 自检清单

- 是否区分券模板、用户券、积分账户、权益记录。
- 是否有状态机：未用、锁定、已用、过期。
- 是否有发放、锁定、核销、释放、退款返还链路。
- 是否用唯一键保证发放和核销幂等。
- 是否有积分流水和权益核销记录。
- 是否说明缓存规则和 MQ 异步发放。
- 是否考虑库存超发、重复领取、过期补偿。
- 是否有分库分表和归档策略。
