# 短链系统

## 面试定位

短链系统是经典系统设计题，重点考察编码生成、重定向低延迟、读多写少、高并发缓存、唯一性、过期、封禁、防刷、统计和容量估算。

核心回答：

```text
短链写入生成唯一 code 并保存映射，读取链路走缓存快速 302 跳转；DB 是事实源，缓存承接高 QPS，风控和封禁保证安全。
```

## 需求澄清

先问：

- 短链是否永久有效，还是支持过期。
- 是否允许自定义短码。
- 日创建量、日跳转量、峰值 QPS。
- 是否需要访问统计、地域、设备、来源分析。
- 是否需要鉴权、私密短链、一次性短链。
- 是否要防钓鱼、防涉政涉黄、黑名单。
- 是否支持修改长链接或封禁短链。
- 可用性和延迟目标是多少。

## 核心模型

核心实体：

- 短链映射：shortCode -> longUrl。
- 域名：短链域名和业务方。
- 访问日志：跳转事件。
- 风控记录：黑名单、封禁原因。

状态：

```text
ACTIVE -> DISABLED
ACTIVE -> EXPIRED
```

## 短码生成方案

### 自增 ID + Base62

```text
id = 125
base62(id) = cb
```

优点：

- 不碰撞。
- 短码长度可控。
- 生成速度快。

缺点：

- 容易被枚举。
- 需要全局 ID 生成器。

可以加扰动：

```text
code = base62(shuffle(id))
```

### 随机码

随机生成 6 到 8 位 Base62，插入 DB 唯一索引，冲突重试。

优点：

- 不易枚举。
- 实现简单。

缺点：

- 随着容量增长冲突概率上升。
- 需要处理重试。

### Hash 长链接

对长链接 hash 后截断。

优点：

- 相同长链可复用。

缺点：

- 有碰撞。
- 同一长链不同过期时间、用户、渠道时未必能复用。

面试推荐：普通系统用全局 ID + Base62；需要防枚举时加随机扰动或随机码唯一索引。

## 架构方案

```text
创建短链
  -> 参数校验和风控
  -> 生成 shortCode
  -> 写 DB 映射
  -> 写缓存
  -> 返回短链

访问短链
  -> 解析 shortCode
  -> 查本地缓存/Redis
  -> miss 查 DB
  -> 校验状态和过期时间
  -> 记录访问事件
  -> 302 跳转长链接
```

访问链路必须极短，统计日志异步处理，不能阻塞跳转。

## 数据模型

### 短链映射表

```sql
CREATE TABLE short_url_mapping (
    id BIGINT PRIMARY KEY,
    short_code VARCHAR(32) NOT NULL,
    long_url TEXT NOT NULL,
    url_hash VARCHAR(64) NOT NULL,
    domain VARCHAR(128) NOT NULL,
    creator_id BIGINT DEFAULT NULL,
    status VARCHAR(32) NOT NULL,
    expire_at DATETIME DEFAULT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    UNIQUE KEY uk_domain_code (domain, short_code),
    KEY idx_creator_created (creator_id, created_at),
    KEY idx_url_hash (url_hash)
);
```

### 访问事件

```sql
CREATE TABLE short_url_visit_event (
    id BIGINT PRIMARY KEY,
    short_code VARCHAR(32) NOT NULL,
    user_agent VARCHAR(512) DEFAULT NULL,
    referer VARCHAR(512) DEFAULT NULL,
    ip VARCHAR(64) DEFAULT NULL,
    created_at DATETIME NOT NULL,
    KEY idx_code_created (short_code, created_at)
);
```

高流量下访问事件不直接写 MySQL，可以先写 MQ/日志，再进 OLAP。

### 封禁表

```sql
CREATE TABLE short_url_blocklist (
    id BIGINT PRIMARY KEY,
    short_code VARCHAR(32) DEFAULT NULL,
    domain VARCHAR(128) DEFAULT NULL,
    url_hash VARCHAR(64) DEFAULT NULL,
    reason VARCHAR(256) NOT NULL,
    created_at DATETIME NOT NULL,
    KEY idx_short_code (short_code),
    KEY idx_url_hash (url_hash)
);
```

## 读链路

```text
GET /{shortCode}
  -> 校验 code 格式
  -> 查本地缓存
  -> 查 Redis
  -> 查 DB
  -> 判断 ACTIVE 和未过期
  -> 异步发送访问事件
  -> 返回 302 Location: longUrl
```

缓存策略：

- 热门短链本地缓存。
- Redis 缓存映射和状态。
- 不存在的 code 缓存空值，防穿透。
- 封禁后删除缓存或提高封禁表优先级。

## 写链路

```text
创建请求
  -> 校验 longUrl 格式
  -> 黑名单和安全扫描
  -> 生成 shortCode
  -> 插入映射表
  -> 缓存 shortCode -> longUrl
  -> 返回短链
```

自定义短码要校验：

- 格式。
- 保留词。
- 是否已占用。
- 是否包含敏感词。

## 缓存、MQ 与一致性

缓存：

- 访问链路读多写少，缓存是核心。
- 更新、过期、封禁时删除缓存。
- 缓存 value 包含状态和过期时间，不只存 longUrl。

MQ：

- 访问日志异步写入。
- 统计聚合异步处理。
- 安全扫描异步回调封禁。

一致性：

- 创建成功后 DB 是事实源。
- 缓存删除失败可能导致被封禁链接短暂可访问，封禁场景可先查本地封禁缓存或强制短 TTL。

## 容量估算

示例：

```text
日创建短链 100 万
日跳转 1 亿
峰值系数 5
平均跳转 QPS ≈ 1157
峰值跳转 QPS ≈ 5785
短链保留 3 年约 10 亿条
```

存储：

```text
映射单行按 500B
10 亿条原始 500GB
加索引、副本、备份按数 TB 规划
```

读写特点：

- 跳转读远大于创建写。
- 访问日志量远大于映射表写入。
- 热门短链可能形成热点 key。

## 风险与降级

- 短码碰撞：唯一索引和重试。
- 被枚举：加扰动、随机码、限流。
- 恶意链接：安全扫描、黑名单、封禁。
- 热点短链：本地缓存、Redis 副本 key。
- 缓存穿透：格式校验、空值缓存、布隆过滤器。
- 访问日志阻塞跳转：异步 MQ，不阻塞主链路。
- DB 故障：已有热门缓存可继续跳转，新建和冷门跳转降级。

降级：

- 统计延迟。
- 安全扫描异步，但高风险域名同步拦截。
- 管理后台慢查询降级，跳转链路优先。

## 常见追问

### 302 还是 301？

通常用 302。301 会被浏览器和搜索引擎长期缓存，不利于统计、封禁和修改跳转。永久稳定短链才考虑 301。

### 短码长度怎么估算？

Base62 长度为 n，可表示 `62^n` 个 code。6 位约 568 亿，足够很多业务，但要考虑保留、冲突和增长。

### 如何防止短链被枚举？

不要直接暴露连续自增 ID，可以打乱 ID、加随机扰动、使用随机码，并对访问频率异常的 IP/设备限流。

### 热门短链怎么优化？

本地缓存、Redis 多副本 key、CDN 边缘规则、访问日志异步化。封禁时要能快速失效。

### 访问统计怎么做？

跳转链路只投递事件，不同步统计。日志进入 MQ/流处理/OLAP，离线或实时聚合 PV、UV、地域、设备。

## 自检清单

- 是否说明短码生成和碰撞处理。
- 是否设计映射表、访问日志、封禁能力。
- 是否把跳转链路做短并异步统计。
- 是否使用缓存和空值防穿透。
- 是否考虑热点短链和本地缓存。
- 是否处理过期、封禁、修改。
- 是否估算短码空间、存储、跳转 QPS。
- 是否说明安全风控和防枚举。
