# 缓存穿透、缓存击穿、缓存雪崩

## 面试定位

缓存穿透、击穿、雪崩是 Java 后端缓存设计必问题。面试官通常关注：

1. 三者定义能不能区分清楚。
2. 每种问题的触发条件和线上现象。
3. 解决方案是否会引入新的风险。
4. 是否能结合代码讲出互斥、空值、随机 TTL、预热、限流降级。
5. 面对真实事故如何排查。

回答时先用一句话区分：

- 穿透：查不存在的数据，请求绕过缓存打到 DB。
- 击穿：热点 key 过期，瞬时大量请求打到 DB。
- 雪崩：大量 key 同时失效或 Redis 整体不可用，流量大面积打到 DB。

## 总览

| 问题 | 核心原因 | 典型现象 | 常见方案 |
| --- | --- | --- | --- |
| 缓存穿透 | 查询不存在 key | 缓存 miss 高，DB 查空高 | 缓存空值、布隆过滤器、参数校验 |
| 缓存击穿 | 热点 key 失效 | 单个 DB 查询被打爆 | 互斥重建、逻辑过期、热点不过期 |
| 缓存雪崩 | 大量 key 同时失效或 Redis 故障 | DB 整体 QPS 飙升 | TTL 抖动、预热、限流降级、多级缓存 |

## 缓存穿透

### 核心概念

请求查询的数据不存在，缓存中没有，DB 中也没有。每次请求都会 miss 缓存并访问 DB。

常见来源：

- 恶意请求随机 ID。
- 业务传入不存在的商品、用户、订单 ID。
- 删除数据后缓存未处理空结果。
- 参数校验缺失。

### 线上现象

- Redis 命中率下降。
- DB QPS 上升，但查到记录数很少。
- 慢 SQL 增多。
- 请求参数呈随机分布。
- 某些接口 404 或空结果比例异常升高。

### 方案一：缓存空值

DB 查询为空时，把空结果也写入缓存，设置较短 TTL。

```java
public ProductDTO getProduct(long productId) {
    String key = "product:" + productId;
    String cached = redis.get(key);
    if (cached != null) {
        if ("__NULL__".equals(cached)) {
            return null;
        }
        return deserialize(cached);
    }

    ProductDTO product = productDao.queryById(productId);
    if (product == null) {
        redis.setex(key, 60, "__NULL__");
        return null;
    }

    redis.setex(key, 600 + randomJitter(), serialize(product));
    return product;
}
```

优点：

- 实现简单。
- 对普通误查有效。

缺点：

- 恶意随机 key 仍可能写入大量空值。
- 短时间内 DB 新增数据后，空值缓存可能造成短暂不一致。

优化：

- 空值 TTL 要短。
- key 必须做前缀和参数校验。
- 对随机攻击配合限流和布隆过滤器。

### 方案二：布隆过滤器

布隆过滤器用于判断“某个元素一定不存在或可能存在”。

查询流程：

```text
请求 ID
  |
  v
布隆过滤器判断
  |
  | 一定不存在 -> 直接返回空
  |
  | 可能存在 -> 查缓存
  |
  | 缓存 miss -> 查 DB 并回填
```

伪代码：

```java
public ProductDTO getProduct(long productId) {
    if (!bloomFilter.mightContain(productId)) {
        return null;
    }
    return queryWithCache(productId);
}
```

优点：

- 拦截大量不存在 ID。
- 内存比 Set 小。

缺点：

- 有误判：可能把不存在判断为可能存在。
- 无法直接删除，删除需要计数布隆或重建。
- 数据新增要同步加入过滤器。
- 过滤器失效或未预热会造成误拦或放行。

面试重点：布隆过滤器没有假阴性，正常情况下存在的数据不会被判断为一定不存在；但有假阳性，不存在的数据可能被判断为可能存在。

### 方案三：参数校验和风控

在进入缓存前先拦截明显非法请求：

- ID 必须为正数。
- ID 长度、范围符合业务规则。
- 枚举值白名单。
- 用户登录态、权限校验。
- 对随机探测进行 IP、账号、设备限流。

缓存不是安全边界，穿透攻击要在网关、应用、缓存多层处理。

## 缓存击穿

### 核心概念

某个热点 key 在过期瞬间，大量并发请求同时发现缓存 miss，然后一起打到 DB。

典型场景：

- 首页配置。
- 秒杀商品。
- 热门商品详情。
- 明星用户主页。
- 活动库存或活动状态。

### 线上现象

- 某一个 DB 查询 QPS 突然飙升。
- Redis miss 集中在同一个 key。
- 应用线程池被 DB 请求占满。
- P99 延迟上升。
- DB CPU 或连接数上涨。

### 方案一：互斥锁重建缓存

只有一个线程查 DB 并回填，其他线程等待或短暂重试。

```java
public ProductDTO getHotProduct(long productId) {
    String key = "product:" + productId;
    String value = redis.get(key);
    if (value != null) {
        return deserialize(value);
    }

    String lockKey = "lock:rebuild:" + key;
    String token = UUID.randomUUID().toString();
    boolean locked = redis.set(lockKey, token, "NX", "EX", 10);
    if (locked) {
        try {
            String doubleCheck = redis.get(key);
            if (doubleCheck != null) {
                return deserialize(doubleCheck);
            }
            ProductDTO product = productDao.queryById(productId);
            redis.setex(key, 600 + randomJitter(), serialize(product));
            return product;
        } finally {
            unlockWithLua(lockKey, token);
        }
    }

    sleepBriefly();
    String retry = redis.get(key);
    if (retry != null) {
        return deserialize(retry);
    }
    return productDao.queryById(productId);
}
```

注意点：

- 加锁后要 double check。
- 锁要有过期时间。
- 解锁要校验 token。
- 等待方要有超时和降级，不能无限阻塞。

### 方案二：逻辑过期

缓存 value 中保存业务数据和逻辑过期时间。物理 TTL 可以很长，逻辑过期后只允许一个线程异步重建，其他线程继续返回旧值。

```java
class CacheEnvelope<T> {
    private T data;
    private long expireAtMillis;
}
```

流程：

```text
读缓存
  |
  | 不存在 -> 查 DB 回填
  |
  | 未逻辑过期 -> 返回
  |
  | 已逻辑过期 -> 尝试加锁异步重建
  |
  v
返回旧值
```

优点：

- 热点 key 不会在过期瞬间打爆 DB。
- 请求延迟稳定。

缺点：

- 可能返回旧数据。
- 需要异步线程池和重建监控。
- 不适合强一致读。

### 方案三：热点 key 不过期

对极少数强热点 key 不设置物理 TTL，由后台任务或变更事件主动刷新。

适用：

- 首页配置。
- 活动配置。
- 榜单摘要。

风险：

- 更新链路失败会长期脏读。
- 必须有兜底刷新和监控。

## 缓存雪崩

### 核心概念

大量缓存同时失效，或 Redis 整体不可用，导致请求大面积打到 DB 或下游服务。

触发场景：

- 大量 key 使用相同 TTL。
- 活动开始前统一预热，统一过期。
- Redis 主从切换或集群故障。
- Redis 网络故障。
- 缓存实例被淘汰策略大量驱逐。

### 线上现象

- Redis 命中率整体下降。
- DB QPS、连接数、CPU 全面上升。
- 应用线程池堆积。
- 接口大面积超时。
- Redis 可能伴随连接失败或超时。

### 方案一：TTL 随机抖动

```java
int ttl = 300 + ThreadLocalRandom.current().nextInt(0, 120);
redis.setex(key, ttl, value);
```

让 key 分散过期，避免同一时刻失效。

### 方案二：缓存预热

在大促、活动、发布后，提前把热点数据加载到 Redis。

预热注意：

- 控制并发，避免预热本身打爆 DB。
- 预热要校验成功率。
- 预热 key 加随机 TTL。
- 支持灰度和失败重试。

### 方案三：多级缓存

典型层次：

```text
本地缓存 Caffeine
  -> Redis
  -> DB
```

本地缓存适合热点读，能在 Redis 抖动时兜底一部分流量。

风险：

- 多级一致性更复杂。
- 本地缓存容量和过期要严格控制。
- 多实例本地缓存更新延迟。

### 方案四：限流、熔断、降级

当 Redis 或 DB 异常时，要保护核心链路。

可选策略：

- 对非核心接口限流。
- 返回兜底数据。
- 降级部分字段。
- 熔断 DB 查询。
- 请求合并。

示例：

```java
if (!rateLimiter.tryAcquire()) {
    return fallbackProduct(productId);
}
```

### 方案五：高可用架构

- Redis 主从 + Sentinel 或 Cluster。
- 客户端连接池合理配置。
- 跨可用区部署要评估延迟。
- 关键业务拆独立实例。
- 做容量水位和慢查询监控。

## 三者对比

| 维度 | 穿透 | 击穿 | 雪崩 |
| --- | --- | --- | --- |
| key 特征 | 不存在 key | 单个热点 key | 大量 key 或整体 Redis |
| DB 压力 | 大量空查 | 单点热查 | 全面上涨 |
| 常用方案 | 空值、布隆、校验 | 互斥、逻辑过期 | TTL 抖动、预热、降级 |
| 是否恶意流量 | 常见 | 不一定 | 不一定 |

## 事故排查流程

1. 看 Redis 命中率是否下降。
2. 看 DB QPS 是集中单 SQL 还是整体上升。
3. 采样 miss key，判断是随机不存在、单热点、批量过期。
4. 看 Redis 是否有连接超时、主从切换、evicted_keys 增长。
5. 看应用线程池、连接池是否耗尽。
6. 临时限流或降级保护 DB。
7. 回填热点缓存、延长 TTL 或关闭问题入口。
8. 复盘 key 设计、TTL、预热和监控。

常用命令：

```bash
INFO stats
INFO keyspace
INFO memory
SLOWLOG GET 20
CLIENT LIST
```

关键指标：

- keyspace_hits。
- keyspace_misses。
- evicted_keys。
- expired_keys。
- connected_clients。
- blocked_clients。
- instantaneous_ops_per_sec。

## 常见追问

### 缓存空值会不会污染缓存

会，所以空值 TTL 要短，并且要做 key 前缀隔离、容量监控和参数校验。对于恶意随机攻击，空值缓存不够，需要布隆过滤器和限流。

### 布隆过滤器误判会导致什么

不存在的数据可能被判断为可能存在，从而继续查缓存和 DB。但存在的数据不会被判断为一定不存在。误判只会放过部分穿透，不会误杀正常存在数据。

### 互斥锁等待期间怎么办

可以短暂 sleep 重试、返回旧值、排队等待或直接降级。不能让所有请求无限阻塞，否则会把应用线程池拖死。

### 逻辑过期适合什么场景

适合允许短暂旧数据的热点读场景，例如商品详情、配置、榜单。不适合强一致交易状态。

### Redis 挂了怎么防雪崩

靠多级缓存、限流、熔断、降级、隔离和 Redis 高可用。不要让所有请求在 Redis 异常后无保护地打到 DB。

## 易错点

- 把穿透、击穿、雪崩混为一谈。
- 只说“加锁”但不说锁超时、token、double check 和降级。
- 空值缓存 TTL 设置过长，导致新数据短期不可见。
- 布隆过滤器没有处理新增数据同步。
- 所有 key 使用固定 TTL。
- Redis 故障时没有限流，导致 DB 被打垮。

## 自检清单

- 能一句话区分穿透、击穿、雪崩。
- 能写出缓存空值和互斥重建伪代码。
- 能解释布隆过滤器误判特点。
- 能说清逻辑过期的优缺点。
- 能给出 TTL 抖动、预热、多级缓存、限流降级方案。
- 能按指标判断事故属于哪一类缓存问题。
