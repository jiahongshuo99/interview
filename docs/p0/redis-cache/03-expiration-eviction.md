# 过期策略、淘汰策略

## 面试定位

过期策略和淘汰策略是 Redis 缓存稳定性的核心。面试官常问：

1. key 设置了 TTL，到期后会立刻删除吗。
2. Redis 怎么删除过期 key。
3. 内存满了 Redis 怎么淘汰。
4. `volatile-lru` 和 `allkeys-lru` 有什么区别。
5. 线上出现内存打满、key 不过期、缓存命中下降怎么排查。

回答要先区分两个概念：

- 过期策略：key 到期后怎么清理。
- 淘汰策略：Redis 内存达到 `maxmemory` 后怎么腾空间。

## 过期时间基础

常用命令：

```bash
SET user:1 value EX 300
EXPIRE user:1 300
PEXPIRE user:1 300000
EXPIREAT user:1 1790000000
TTL user:1
PTTL user:1
PERSIST user:1
```

返回值含义：

```text
TTL = -1：key 存在但没有过期时间
TTL = -2：key 不存在
TTL >= 0：剩余秒数
```

Redis 会把过期时间存储在 expires 字典中，key 指向过期时间戳。

## 过期策略

Redis 过期删除主要是两种策略组合：

1. 惰性删除。
2. 定期删除。

### 惰性删除

访问 key 时检查是否过期。如果过期，则删除并返回 key 不存在。

优点：

- 只在访问时检查，不浪费额外 CPU。

缺点：

- 如果过期 key 长期不被访问，会继续占用内存。

示例：

```bash
SET temp:1 value EX 1
# 过一段时间后 GET，Redis 发现已过期，会删除并返回 nil
GET temp:1
```

### 定期删除

Redis 后台周期性抽样检查带 TTL 的 key，删除其中过期的 key。

特点：

- 不是全量扫描所有 key。
- 是抽样删除，避免阻塞主线程太久。
- 如果过期 key 比例很高，会继续多轮扫描。

这也是为什么 key 到期后不保证毫秒级立刻消失。

### 为什么不定时删除

定时删除是给每个 key 设置一个定时器，到期立即删除。

Redis 不采用这种方式作为主策略，因为：

- key 数量巨大时定时器成本高。
- 大量 key 同时到期会造成 CPU 抖动。
- Redis 更关注整体吞吐和可控延迟。

## 过期删除对业务的影响

### 到期不等于立刻释放内存

TTL 到 0 后，key 可能还在内存中，直到被访问或定期删除扫到。

业务不能依赖“到点物理删除”做强语义，例如优惠券到期不能只靠 Redis key 消失来保证不可用，核心状态必须在 DB 或业务规则中校验。

### 过期删除可能导致延迟毛刺

大量 key 同时过期，会造成：

- 定期删除 CPU 升高。
- 内存释放压力变大。
- 主线程处理正常请求延迟上升。
- 如果 value 很大，删除成本更明显。

优化：

- TTL 加随机抖动。
- 大 key 使用 `UNLINK` 异步删除。
- 避免活动结束时间作为大量 key 的同一 TTL。

```java
int baseTtl = 300;
int jitter = ThreadLocalRandom.current().nextInt(0, 60);
redis.setex(key, baseTtl + jitter, value);
```

## 淘汰策略

当 Redis 配置了 `maxmemory`，并且内存使用达到上限时，写入新数据会触发淘汰策略。

查看配置：

```bash
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
INFO memory
```

注意：没有配置 `maxmemory` 时，Redis 可能继续向系统申请内存，最终被 OS 或容器杀掉。

## 常见淘汰策略

| 策略 | 候选 key | 规则 |
| --- | --- | --- |
| `noeviction` | 无 | 不淘汰，新写入报错 |
| `allkeys-lru` | 所有 key | 近似淘汰最近最少使用 |
| `volatile-lru` | 设置 TTL 的 key | 近似淘汰最近最少使用 |
| `allkeys-lfu` | 所有 key | 淘汰低频访问 |
| `volatile-lfu` | 设置 TTL 的 key | 淘汰低频访问 |
| `allkeys-random` | 所有 key | 随机淘汰 |
| `volatile-random` | 设置 TTL 的 key | 随机淘汰 |
| `volatile-ttl` | 设置 TTL 的 key | 优先淘汰 TTL 更短的 |

### noeviction

默认或常见保守策略。内存满后写命令报错：

```text
OOM command not allowed when used memory > 'maxmemory'
```

适合 Redis 被当作存储或不允许自动丢数据的场景。

### allkeys-lru

所有 key 都可能被淘汰，按近似 LRU 选择。适合纯缓存场景。

如果 Redis 中所有数据都能从 DB 重建，通常比 `volatile-lru` 更符合预期。

### volatile-lru

只淘汰设置了 TTL 的 key。没有 TTL 的 key 不参与淘汰。

风险：如果大量 key 没有 TTL，Redis 内存满时可淘汰候选不足，仍可能写入失败。

### LFU

LFU 按访问频率淘汰。适合热点比较稳定的场景。

Redis 的 LFU 是近似统计，有衰减机制，不是精确访问次数。

相关配置：

```bash
CONFIG GET lfu-log-factor
CONFIG GET lfu-decay-time
```

## LRU 为什么是近似的

Redis 不会维护全局精确 LRU 链表，因为每次访问都调整链表成本较高。

Redis 使用采样方式：

1. 从候选 key 中随机采样若干个。
2. 根据 LRU/LFU/TTL 信息选出最应该淘汰的。
3. 删除被选 key。

采样数量由 `maxmemory-samples` 控制：

```bash
CONFIG GET maxmemory-samples
CONFIG SET maxmemory-samples 10
```

采样越大，淘汰越接近真实 LRU，但 CPU 成本更高。

## 过期策略与淘汰策略区别

| 对比项 | 过期策略 | 淘汰策略 |
| --- | --- | --- |
| 触发条件 | key 到 TTL | 内存达到 `maxmemory` |
| 目标 | 删除过期 key | 腾出内存 |
| 候选范围 | 设置 TTL 且过期的 key | 取决于 maxmemory-policy |
| 是否表示业务过期 | 是 | 否，只是内存压力下被动丢弃 |

面试重点：被淘汰不等于业务过期，业务必须能承受缓存 miss。

## 命令示例

设置缓存 TTL：

```bash
SET product:1001 '{"name":"phone"}' EX 600
```

查看 TTL：

```bash
TTL product:1001
PTTL product:1001
```

查看内存和淘汰：

```bash
INFO memory
INFO stats | grep evicted_keys
INFO stats | grep expired_keys
```

查看 key 是否设置过期：

```bash
TTL some:key
```

## Java 使用示例

```java
Duration ttl = Duration.ofMinutes(5)
        .plusSeconds(ThreadLocalRandom.current().nextInt(60));

redisTemplate.opsForValue().set(
        "product:" + productId,
        json,
        ttl
);
```

缓存空值也要设置较短 TTL：

```java
if (product == null) {
    redisTemplate.opsForValue().set(key, "NULL", Duration.ofSeconds(30));
    return null;
}
```

## 线上风险

### 大量 key 同时过期

现象：

- Redis CPU 升高。
- P99 延迟升高。
- 命中率波动。
- 后端 DB 请求突然升高。

处理：

- TTL 加随机抖动。
- 活动数据分批过期。
- 提前预热。
- 热点 key 使用逻辑过期。

### maxmemory-policy 配错

典型问题：

- 缓存场景用 `noeviction`，内存满后写入失败。
- 纯缓存用 `volatile-lru`，但很多 key 没 TTL，导致淘汰效果差。
- 混合存储和缓存放同一实例，淘汰误删重要数据。

处理：

- 缓存和持久状态拆实例。
- 纯缓存优先考虑 `allkeys-lru` 或 `allkeys-lfu`。
- 重要数据不要依赖 Redis 自动淘汰保护。

### key 没有 TTL

长期累积导致内存上涨。

排查：

```bash
SCAN 0 MATCH prefix:* COUNT 100
TTL key
```

生产环境不要全量扫描大实例；优先用离线分析、采样脚本或 Redis 自带 bigkeys/memkeys 工具。

## 排查优化

### 内存打满排查步骤

1. 看 `INFO memory`，确认 used_memory、used_memory_rss、mem_fragmentation_ratio。
2. 看 `maxmemory` 和 `maxmemory-policy`。
3. 看 `evicted_keys` 是否增长。
4. 看 `expired_keys` 是否异常增长。
5. 找 top key、big key、无 TTL key。
6. 结合业务发布、活动、流量变化判断来源。

命令：

```bash
INFO memory
INFO stats
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
redis-cli --bigkeys
redis-cli --memkeys
```

### 淘汰导致命中率下降

现象：

- `evicted_keys` 持续增长。
- 缓存命中率下降。
- DB QPS 上升。

优化：

- 增大内存或拆分实例。
- 调整淘汰策略。
- 压缩 value。
- 清理无用 key。
- 热点数据单独实例或本地缓存。

## 常见追问

### key 过期后会不会立刻删除

不会保证立刻删除。Redis 使用惰性删除和定期删除结合，过期 key 可能短时间留在内存中，但对访问方表现为不存在。

### 为什么不用全量扫描删除过期 key

全量扫描会阻塞主线程，影响正常请求。Redis 用抽样方式在 CPU 和内存释放之间折中。

### 纯缓存场景选什么淘汰策略

常见选择是 `allkeys-lru` 或 `allkeys-lfu`。如果热点稳定，LFU 更适合；如果访问具有时间局部性，LRU 更常见。前提是所有数据都能从后端重建。

### volatile-lru 有什么坑

它只淘汰设置 TTL 的 key。没有 TTL 的 key 不会被淘汰，可能导致内存满后依然写失败。

### Redis 删除大 key 为什么会阻塞

`DEL` 会同步释放对象内存。大 Hash、大 Set、大 ZSet 删除时释放成本高，可能阻塞主线程。可用 `UNLINK` 异步释放。

## 易错点

- TTL 过期是业务过期；淘汰是内存压力下丢缓存，两者语义不同。
- `TTL=-1` 表示存在但永不过期，不是不存在。
- `allkeys-lru` 也不是精确 LRU。
- `volatile-*` 策略不会淘汰无 TTL key。
- 大量相同 TTL 会制造缓存雪崩风险。
- 不要把重要状态和可淘汰缓存混在一个 Redis 实例。

## 自检清单

- 能解释惰性删除和定期删除。
- 能说清 key 到期不保证立刻物理删除。
- 能列出 Redis 主要淘汰策略。
- 能区分 `allkeys-lru` 和 `volatile-lru`。
- 能排查 `evicted_keys` 增长和 OOM 写入失败。
- 能给出 TTL 抖动、拆实例、`UNLINK`、big key 治理等优化手段。
