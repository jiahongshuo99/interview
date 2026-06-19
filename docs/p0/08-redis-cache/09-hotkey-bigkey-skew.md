# 热点 key、大 key、key 倾斜

## 面试定位

热点 key、大 key、key 倾斜是 Redis 线上稳定性高频题。面试官常问：

1. 什么是热点 key、大 key、key 倾斜。
2. 它们分别会造成什么问题。
3. 如何发现热点 key 和大 key。
4. big key 为什么会影响删除、迁移、持久化。
5. Redis Cluster key 倾斜怎么处理。

回答要把三者区分清楚：

- 热点 key：访问频率异常高。
- 大 key：单个 key 的 value 很大或元素很多。
- key 倾斜：数据或访问在节点、slot、分片上分布不均。

## 总览

| 问题 | 衡量维度 | 典型危害 | 常见治理 |
| --- | --- | --- | --- |
| 热点 key | QPS、带宽、CPU | 单点打爆、延迟升高 | 本地缓存、读副本、拆 key、请求合并 |
| 大 key | 内存、元素数量、序列化大小 | 阻塞、迁移慢、删除慢 | 拆分、压缩、分页、UNLINK |
| key 倾斜 | slot/node 数据和流量分布 | 节点负载不均、扩容无效 | 重设 key、hash tag 治理、slot 迁移 |

## 热点 key

### 核心概念

热点 key 是被高频访问的 key。它不一定大，可能只是一个很小的 String，但 QPS 极高。

典型场景：

- 首页配置。
- 爆款商品详情。
- 秒杀库存。
- 热门排行榜。
- 登录态或全局开关。
- 明星用户信息。

### 线上现象

- Redis 单节点 CPU 飙高。
- 网络出入流量集中。
- 某些命令延迟升高。
- Cluster 中单个节点负载远高于其他节点。
- 应用侧大量请求卡在 Redis 读取。

### 发现方式

#### 客户端埋点

应用侧统计 key 访问 topN，是最可靠方式。

建议维度：

- key 模板，例如 `product:{id}`。
- 真实 key topN。
- 操作类型。
- QPS。
- 平均耗时、P99。
- value 大小。

#### Redis 侧工具

```bash
redis-cli --hotkeys
```

注意：`--hotkeys` 依赖 LFU 相关统计，且有采样性质。

监控指标：

```bash
INFO commandstats
INFO stats
```

#### 代理层或网关统计

如果使用 Codis、Twemproxy、Redis Proxy 或服务网格，可以在代理层统计热点。

### 治理方案

#### 本地缓存

适合读多写少、允许短暂不一致的热点数据。

```java
LoadingCache<String, ProductDTO> localCache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofSeconds(5))
        .build(this::loadFromRedisOrDb);
```

注意：

- TTL 要短。
- 更新要能主动失效或接受短暂旧值。
- 防止本地缓存把内存打满。

#### 热点 key 拆分

把一个热点 key 拆成多个副本，读随机分散。

```text
config:home:0
config:home:1
config:home:2
...
```

读取：

```java
int idx = ThreadLocalRandom.current().nextInt(16);
redis.get("config:home:" + idx);
```

写入时更新所有副本或通过异步任务刷新。

适合读热点，不适合频繁写热点。

#### 请求合并

同一 JVM 内多个线程同时加载同一个 key，只让一个线程回源，其余等待结果。

适合缓存 miss 或重建场景，常配合 Caffeine、singleflight 模式。

#### 读写分离和副本扩展

读热点可以让多个 replica 分担读流量。但要接受主从延迟，不适合强一致读。

## 大 key

### 核心概念

大 key 可以指：

- String value 很大。
- Hash field 很多。
- List 长度很长。
- Set/ZSet 元素很多。

没有绝对标准，常见经验阈值：

- String 超过 10KB 或 100KB 要关注。
- 集合元素超过 5000、10000 或更多要治理，具体看业务和实例规格。
- 单 key 内存达到 MB 级通常需要重点治理。

### 大 key 危害

- 网络传输大，应用反序列化慢。
- O(N) 命令阻塞主线程。
- `DEL` 同步释放内存阻塞。
- RDB/AOF rewrite 和主从同步成本增加。
- Cluster slot 迁移慢。
- 内存碎片和 RSS 异常。
- 客户端超时后重试，进一步放大压力。

### 发现方式

```bash
redis-cli --bigkeys
redis-cli --memkeys
```

抽样查看：

```bash
MEMORY USAGE key
STRLEN string:key
HLEN hash:key
LLEN list:key
SCARD set:key
ZCARD zset:key
```

生产注意：

- `--bigkeys` 会扫描 keyspace，低峰执行。
- 不要在线上大实例直接 `KEYS *`。
- 对集合不要用 `HGETALL`、`SMEMBERS` 拉全量确认。

### 不同类型治理

#### String 大 value

问题：

- 大 JSON 或序列化对象。
- 字段冗余。
- 压缩前后不可控。

治理：

- 拆成多个字段 key。
- 只缓存必要字段。
- 压缩 value，但要评估 CPU。
- 大对象走对象存储或 DB，不放 Redis。

#### Hash 大 key

问题：

- 一个 hash 放所有用户或所有商品。
- field 数量持续增长。

治理：

```text
user:profile:{uid}
user:stats:{uid}
```

或按桶拆：

```text
user:bucket:{hash(uid)%1024}
```

#### List 大 key

治理：

- `LTRIM` 控制最新列表长度。
- 按时间分片：`feed:{uid}:20260619`。
- 分页数据不要无限追加。

#### Set 大 key

治理：

- 按时间、业务、hash 桶拆。
- 精确明细转离线存储。
- 只在 Redis 保留近期或 topN。

#### ZSet 大 key

治理：

- 排行榜只保留 topN。
- 按日期、地区、业务拆榜。
- member 存短 ID。

```bash
ZREMRANGEBYRANK rank:daily 0 -10001
```

注意上面命令含义是保留最后 10000 名还是删除低分段，要结合正序/倒序设计确认。

### 删除大 key

不要高峰期直接 `DEL` 大 key。

推荐：

```bash
UNLINK big:key
```

`UNLINK` 把释放内存放到后台线程，主线程只做 key 解绑。

集合渐进删除：

```bash
HSCAN big:hash 0 COUNT 100
HDEL big:hash field1 field2
```

但如果最终能接受异步释放，`UNLINK` 更简单。

## key 倾斜

### 核心概念

key 倾斜指数据量或访问量在 Redis 节点、slot 或分片上分布不均。

在 Redis Cluster 中尤其常见：

- 某些 slot 上 key 特别多。
- 某个 slot 有 big key。
- 某些 hash tag 导致大量 key 落同 slot。
- 某个业务天然热点落单节点。

### hash tag 导致倾斜

hash tag 用 `{}` 指定参与 slot 计算的部分：

```bash
SET user:{1001}:profile v1
SET user:{1001}:stats v2
```

这能保证同一用户相关 key 同 slot，方便多 key 操作。

错误用法：

```bash
SET product:{hot}:1001 v1
SET product:{hot}:1002 v2
SET product:{hot}:1003 v3
```

所有 key 都落到 `{hot}` 对应 slot，形成倾斜。

### 发现方式

```bash
CLUSTER INFO
CLUSTER NODES
CLUSTER SLOTS
CLUSTER KEYSLOT key
CLUSTER COUNTKEYSINSLOT 1234
```

如果使用云 Redis，通常可直接看节点级内存、QPS、带宽、CPU。

### 治理方案

- 修正 hash tag。
- 迁移 slot。
- 拆分热点业务 key。
- 对大 key 做分桶。
- 对读热点加本地缓存或副本。
- 必要时独立 Redis 实例隔离热点业务。

## key 设计规范

推荐格式：

```text
业务:实体:标识:属性
```

示例：

```text
product:1001:detail
user:1001:profile
rank:score:daily:20260619
lock:order:1001
```

原则：

- 前缀清晰，方便扫描和治理。
- 避免过长 key。
- 避免一个 key 混多个业务生命周期。
- Cluster 下谨慎使用 hash tag。
- key 中不要放大段 JSON 或不可控字符串。

## 线上排查流程

### Redis CPU 高

1. 看是否是热点 key 或慢命令。
2. 看 `SLOWLOG GET`。
3. 看 `INFO commandstats` 哪类命令多。
4. 应用侧统计 key topN。
5. 判断是否有大 key O(N) 命令。

### Redis 内存高

1. 看 `INFO memory`。
2. 用 bigkeys/memkeys 低峰扫描。
3. 按 key 前缀统计。
4. 检查无 TTL key。
5. 检查最近发布或活动新增缓存。

### Cluster 单节点高

1. 看节点级 CPU、内存、带宽。
2. 看 slot 分布是否均衡。
3. 查该节点上 top key、big key、hot key。
4. 看是否 hash tag 误用。
5. 迁移 slot 或拆 key。

## 常见追问

### big key 多大算大

没有统一标准，要结合实例规格、命令耗时、网络和业务 SLA。面试可以给经验值：String 超过几十 KB 到百 KB 要关注，集合元素上万要关注，MB 级单 key 通常要治理。

### hot key 和 big key 有什么区别

hot key 是访问频率高；big key 是数据体积大。一个 key 可以很小但很热，也可以很大但访问不频繁。治理手段不同。

### 为什么 big key 会影响主从同步

big key 会让 RDB/AOF、复制缓冲、网络传输、从库加载都变重。全量同步和迁移时尤其明显。

### 如何安全删除大 key

优先 `UNLINK` 异步释放；或者对集合类型用 `SCAN` 系列分批删除元素。不要高峰期 `DEL` 大 key。

### Cluster 中热点 key 怎么办

单个 key 只能属于一个 slot，不能自动分散到多个节点。需要应用层拆副本、拆分业务 key、本地缓存、读副本或独立实例隔离。

## 易错点

- 用 `KEYS *` 在线上查大 key。
- 用 `HGETALL`、`SMEMBERS` 确认大集合内容。
- 认为 Cluster 会自动解决单 key 热点。
- 滥用 hash tag 导致所有 key 落同槽。
- 大 key 只关注内存，不关注删除、迁移、持久化和网络。
- 本地缓存热点 key 不设置容量和 TTL。

## 自检清单

- 能区分热点 key、大 key、key 倾斜。
- 能列出发现 hot key 和 big key 的工具。
- 能解释 big key 对延迟、删除、迁移、复制的影响。
- 能设计 key 拆分、本地缓存、副本读、请求合并方案。
- 能说明 Cluster hash tag 的用途和风险。
- 能给出线上 Redis CPU 高、内存高、单节点高的排查步骤。
