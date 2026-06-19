# 消息通知系统

## 面试定位

消息通知系统是常见 P0 系统设计题，覆盖站内信、Push、短信、邮件、企微/钉钉等多渠道触达。面试官关注：模板、用户偏好、异步发送、渠道限流、去重、重试、失败补偿、优先级、延迟、已读状态和发送结果追踪。

核心回答：

```text
通知系统用统一消息模型接收业务事件，通过模板渲染和用户偏好决策生成发送任务，再按渠道异步投递，发送记录保证幂等、可追踪、可重试。
```

## 需求澄清

先问：

- 支持哪些渠道：站内信、Push、短信、邮件、IM。
- 通知类型：交易通知、营销通知、系统通知、告警通知。
- 是否要求实时，延迟容忍多久。
- 是否需要用户订阅偏好和免打扰。
- 是否需要模板、多语言、变量渲染。
- 是否需要批量群发。
- 是否有优先级和限流。
- 是否要保证必达，失败如何补偿。
- 是否需要已读、撤回、搜索。

## 核心模型

核心实体：

- 通知请求：业务方提交的原始请求。
- 模板：标题、正文、变量、渠道配置。
- 用户偏好：订阅、退订、免打扰、渠道优先级。
- 发送任务：某个用户某个渠道的一次投递。
- 发送记录：渠道返回结果、失败原因、重试次数。
- 站内信消息：可查询、可已读。

状态：

```text
任务：INIT -> SENDING -> SUCCESS / FAILED / CANCELED
站内信：UNREAD -> READ
批次：CREATED -> RUNNING -> FINISHED / PAUSED
```

## 架构方案

```text
业务服务
  -> 发送通知请求 / 业务事件
  -> 通知服务幂等接收
  -> 模板渲染
  -> 用户偏好和风控过滤
  -> 生成渠道发送任务
  -> MQ 分发
  -> 渠道 Worker 发送
  -> 记录发送结果
  -> 失败重试/死信/人工处理
```

服务划分：

- 接入层：统一 API、鉴权、幂等。
- 编排层：模板、偏好、渠道选择。
- 任务层：发送任务、优先级、延迟。
- 渠道层：短信、邮件、Push、站内信适配器。
- 查询层：站内信列表、已读状态、发送结果。

## 数据模型

### 通知请求

```sql
CREATE TABLE notification_request (
    id BIGINT PRIMARY KEY,
    request_id VARCHAR(128) NOT NULL,
    biz_type VARCHAR(64) NOT NULL,
    biz_no VARCHAR(128) NOT NULL,
    template_code VARCHAR(64) NOT NULL,
    receiver_type VARCHAR(32) NOT NULL,
    receiver_id VARCHAR(128) NOT NULL,
    variables TEXT NOT NULL,
    priority INT NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_request_id (request_id),
    KEY idx_biz_no (biz_type, biz_no)
);
```

### 模板表

```sql
CREATE TABLE notification_template (
    id BIGINT PRIMARY KEY,
    template_code VARCHAR(64) NOT NULL,
    channel VARCHAR(32) NOT NULL,
    title_template VARCHAR(512) DEFAULT NULL,
    body_template TEXT NOT NULL,
    status VARCHAR(32) NOT NULL,
    version INT NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_template_channel_version (template_code, channel, version)
);
```

### 发送任务

```sql
CREATE TABLE notification_send_task (
    id BIGINT PRIMARY KEY,
    task_no VARCHAR(64) NOT NULL,
    request_id VARCHAR(128) NOT NULL,
    user_id BIGINT NOT NULL,
    channel VARCHAR(32) NOT NULL,
    receiver_address VARCHAR(256) NOT NULL,
    title VARCHAR(512) DEFAULT NULL,
    content TEXT NOT NULL,
    priority INT NOT NULL,
    status VARCHAR(32) NOT NULL,
    retry_count INT NOT NULL DEFAULT 0,
    next_retry_at DATETIME DEFAULT NULL,
    provider_msg_id VARCHAR(128) DEFAULT NULL,
    error_msg VARCHAR(512) DEFAULT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_task_no (task_no),
    UNIQUE KEY uk_request_channel_user (request_id, channel, user_id),
    KEY idx_status_retry (status, next_retry_at)
);
```

### 站内信

```sql
CREATE TABLE inbox_message (
    id BIGINT PRIMARY KEY,
    message_no VARCHAR(64) NOT NULL,
    user_id BIGINT NOT NULL,
    title VARCHAR(512) NOT NULL,
    content TEXT NOT NULL,
    biz_type VARCHAR(64) NOT NULL,
    biz_no VARCHAR(128) NOT NULL,
    read_status VARCHAR(32) NOT NULL,
    created_at DATETIME NOT NULL,
    read_at DATETIME DEFAULT NULL,
    UNIQUE KEY uk_message_no (message_no),
    KEY idx_user_created (user_id, created_at),
    KEY idx_user_read (user_id, read_status, created_at)
);
```

## 接入链路

```text
业务方调用 sendNotification
  -> 鉴权和参数校验
  -> requestId 幂等
  -> 保存 notification_request
  -> 发送编排 MQ
  -> 快速返回 accepted
```

业务方不应同步等待短信或 Push 渠道返回，除非是极少数强同步场景。

## 编排链路

```text
消费通知请求
  -> 查询模板
  -> 渲染变量
  -> 查询用户偏好
  -> 判断免打扰和黑名单
  -> 生成一个或多个发送任务
  -> 按渠道投递 MQ
```

渠道选择示例：

- 交易通知：站内信 + Push，必要时短信。
- 营销通知：尊重退订和频控。
- 告警通知：短信 + IM，优先级高。

## 发送链路

```text
渠道 Worker 消费任务
  -> 检查任务状态
  -> 渠道限流
  -> 调用供应商 API
  -> 成功记录 provider_msg_id
  -> 失败按错误类型重试或终止
```

失败分类：

- 可重试：超时、限流、供应商 5xx。
- 不可重试：手机号非法、模板不存在、用户退订。
- 需人工：供应商异常、签名被封、余额不足。

## 站内信链路

站内信可以作为可靠兜底渠道：

```text
生成 inbox_message
  -> 用户列表按 user_id + created_at 分页
  -> 用户点击后 UNREAD -> READ
```

已读状态更新：

```sql
UPDATE inbox_message
SET read_status = 'READ',
    read_at = NOW()
WHERE message_no = ?
  AND user_id = ?
  AND read_status = 'UNREAD';
```

重复已读天然幂等。

## 缓存、MQ 与一致性

缓存：

- 模板缓存。
- 用户偏好缓存。
- 渠道配置缓存。
- 频控计数用 Redis。

MQ：

- 请求接入后异步编排。
- 编排后按渠道拆 topic。
- 失败重试和死信队列。
- 批量群发任务拆分。

一致性：

- 通知通常最终一致。
- 交易通知可以要求“至少有一个渠道成功”。
- 发送记录是事实源，渠道回执用于修正状态。
- 重复发送用 `request_id + channel + user_id` 唯一键。

## 容量估算

示例：

```text
日通知请求 5000 万
平均每请求 2 个渠道
日发送任务 1 亿
峰值系数 5
峰值任务 TPS ≈ 5787
```

还要估算：

- 站内信存储量。
- 短信供应商 QPS 限额。
- Push 通道推送批量大小。
- 模板渲染 CPU。
- MQ 堆积追平时间。

群发营销要按批次限速，不能和交易通知抢资源。

## 风险与降级

- 重复发送：requestId 和任务唯一键。
- 渠道限流：本地令牌桶和供应商配额。
- 供应商故障：多供应商切换或降级到站内信。
- 模板错误：模板发布审核和灰度。
- 变量缺失：渲染校验，失败不发送。
- 营销打扰用户：退订、免打扰、频控。
- MQ 堆积：交易通知高优先级，营销通知暂停。
- 短信成本失控：预算、频控、黑名单。

降级：

- 营销通知暂停。
- 非核心渠道延迟。
- 交易通知保站内信和 Push。
- 短信供应商异常时切备用或延迟重试。

## 常见追问

### 如何保证不重复发短信？

业务方传 requestId，通知服务用 `request_id + channel + user_id` 唯一键。渠道超时后重试要复用同一个任务，不生成新任务。

### 渠道返回超时怎么办？

不能确定是否发送成功。记录为未知或待重试，优先查询渠道回执；如果渠道不支持查询，按业务重要性决定是否重试，并接受可能重复的风险。短信可用内容和业务幂等降低影响。

### 模板如何管理？

模板按 code、channel、version 管理，发布需要审核和灰度。发送记录保存渲染后的 title/content，避免模板变更影响历史。

### 用户偏好怎么处理？

交易通知通常不可关闭或只允许关闭部分渠道；营销通知必须尊重退订、免打扰和频控。偏好要缓存，但以 DB 为事实源。

### 批量群发怎么做？

创建批次任务，按用户分片拆分子任务，通过 MQ 限速发送。要支持暂停、恢复、失败重试和进度统计。

## 自检清单

- 是否支持多渠道和统一通知模型。
- 是否有模板、变量、用户偏好。
- 是否通过 MQ 异步编排和发送。
- 是否有发送任务和发送记录。
- 是否用 requestId 和唯一键去重。
- 是否支持渠道限流、重试、死信。
- 是否区分交易通知和营销通知优先级。
- 是否有站内信、已读状态和查询索引。
- 是否估算任务量、渠道 QPS、存储和堆积。
- 是否有供应商故障和营销暂停降级方案。
