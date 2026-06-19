# String、Hash、List、Set、ZSet

## 面试定位

Redis 基础数据结构是缓存面试的入口题。面试官通常不会只问命令，而是追问：

1. 每种结构适合什么业务场景。
2. 底层编码为什么会影响内存和性能。
3. 时间复杂度是多少，哪些命令有阻塞风险。
4. Java 后端使用时如何避免 big key、热 key 和误用。
5. 如果线上 Redis CPU、内存或慢查询异常，如何从数据结构使用方式定位。

回答要从“抽象结构、底层编码、典型命令、业务场景、线上风险”五层展开。

## 总览

| 类型 | 逻辑模型 | 常见场景 | 典型风险 |
| --- | --- | --- | --- |
| String | 字符串、整数、二进制安全字节数组 | 缓存对象、计数器、分布式锁、限流 | 大 value、序列化膨胀、误用 `KEYS` |
| Hash | field-value 映射 | 用户资料、商品属性、局部字段更新 | field 过多、整 hash 成 big key |
| List | 双端有序链表语义 | 简单队列、时间线、最新列表 | 阻塞命令堆积、长列表慢操作 |
| Set | 无序去重集合 | 标签、关注集合、抽奖去重、共同好友 | 大集合交并差阻塞 |
| ZSet | score 排序集合 | 排行榜、延迟队列、范围查询 | 热榜集中写、跳表内存开销 |

## Redis 对象模型

Redis 不是直接把命令映射到唯一底层结构。每个 key 对应一个 RedisObject，里面有：

- `type`：对外暴露的数据类型，例如 string、hash、list、set、zset。
- `encoding`：内部编码，例如 int、embstr、raw、listpack、hashtable、quicklist、skiplist。
- `ptr`：指向真实数据。
- LRU/LFU 信息：淘汰策略会使用。
- 引用计数等元信息。

常用查看命令：

```bash
TYPE user:1
OBJECT ENCODING user:1
MEMORY USAGE user:1
```

面试重点：同一个逻辑类型可能因为元素数量、元素大小变化而升级编码。编码升级通常不可逆，可能导致内存和性能突变。

## String

### 核心概念

String 是 Redis 最基础类型，value 是二进制安全的字节数组，最大可到 512MB，但线上绝不应该接近这个上限。

它既可以存文本，也可以存 JSON、序列化对象、整数、bitmap 原始字节等。

常见命令：

```bash
SET user:1 '{"name":"tom","age":18}' EX 300
GET user:1
MGET user:1 user:2 user:3
INCR counter:order
INCRBY counter:order 10
SETNX lock:order:1 token
SET lock:order:1 token NX EX 30
GETSET config:version v2
```

### 底层编码

String 常见编码：

| 编码 | 说明 |
| --- | --- |
| `int` | value 可以表示为整数，直接存整数 |
| `embstr` | 短字符串，一次内存分配存 RedisObject 和 SDS |
| `raw` | 长字符串，RedisObject 和 SDS 分开分配 |

SDS 是 Redis 自己实现的简单动态字符串。相比 C 字符串，它保存长度，支持二进制安全，追加时可预分配空间。

### 典型场景

对象缓存：

```java
String key = "user:" + userId;
String json = redis.get(key);
if (json != null) {
    return objectMapper.readValue(json, UserDTO.class);
}

UserDTO user = userDao.queryById(userId);
redis.setex(key, 300, objectMapper.writeValueAsString(user));
return user;
```

计数器：

```bash
INCR article:1001:read_count
INCRBY sku:2001:stock -1
```

限流：

```bash
INCR rate:login:uid:1001:202606190901
EXPIRE rate:login:uid:1001:202606190901 60
```

实际限流要用 Lua 保证 `INCR` 和 `EXPIRE` 的原子性，否则第一次 `INCR` 成功后服务宕机可能留下无过期 key。

### 线上风险

- 把大 JSON、大 HTML、大图片、压缩包塞进 String，导致网络和内存抖动。
- `MGET` 一次取太多 key，响应包过大。
- 缓存对象字段局部更新困难，频繁整体反序列化和回写。
- 计数器没有设置边界，异常流量把 key 打爆。
- 使用 Java 原生序列化，value 体积大且不可读。

### 排查优化

```bash
MEMORY USAGE user:1
STRLEN user:1
OBJECT ENCODING user:1
```

优化方向：

- 大对象拆字段或拆 key。
- value 使用 JSON、MessagePack、Kryo 等要结合可观测性选择。
- 对热点 String 使用本地缓存、二级缓存或读写分片。
- 控制 `MGET` 批量大小。
- 所有缓存 key 尽量设置 TTL，确实永久的 key 要有白名单。

## Hash

### 核心概念

Hash 是 field-value 映射，适合存对象的多个属性。

常见命令：

```bash
HSET user:1 name tom age 18 city beijing
HGET user:1 name
HMGET user:1 name age
HGETALL user:1
HINCRBY user:1 score 10
HDEL user:1 city
HLEN user:1
HSCAN user:1 0 COUNT 100
```

### 底层编码

Hash 常见编码：

| 编码 | 说明 |
| --- | --- |
| `listpack` | 元素少且 field/value 小时使用，内存紧凑 |
| `hashtable` | 元素多或 value 大时使用，查询更稳定 |

旧版本常见 `ziplist`，新版本逐步使用 `listpack`。面试时可以说“压缩列表或 listpack 类紧凑编码”，再补一句具体版本不同。

### 适用场景

适合：

- 用户资料字段独立读写。
- 商品基础属性。
- 小对象聚合。
- 计数字段，例如 `HINCRBY post:1:stats like_count 1`。

不适合：

- 一个 Hash 放几十万 field。
- 把所有用户都塞进一个 `users` hash。
- value 本身还是超大 JSON。

### Hash 与 String 存对象的取舍

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| String 存 JSON | 简单，一次读写，适合整体缓存 | 局部更新困难，反序列化成本高 |
| Hash 存字段 | 可局部读写，可字段计数 | field 多会形成 big key，跨字段一致性要自己保证 |

面试回答：小对象、字段级更新明显时可用 Hash；读多且总是整体读取时 String 更简单。不能因为 Hash 看起来像对象就把所有对象都改成 Hash。

### 线上风险

- `HGETALL` 读取大 hash 阻塞 Redis 主线程并产生大响应。
- 单 key field 过多导致迁移、删除、过期成本高。
- Hash 只有 key 级 TTL，没有 field 级 TTL。
- field 命名无规范，后期难清理。

### 排查优化

```bash
HLEN user:1
HSCAN user:1 0 COUNT 100
MEMORY USAGE user:1
OBJECT ENCODING user:1
```

优化方向：

- 对大 Hash 按业务维度拆 key，例如 `user:{uid}:profile`、`user:{uid}:stats`。
- 避免线上使用 `HGETALL` 拉全量。
- 删除大 Hash 使用 `UNLINK`，不要在高峰期 `DEL`。
- 如果需要 field 级过期，拆成多个 String 或引入额外索引。

## List

### 核心概念

List 是按插入顺序排列的双端列表，支持头尾 push/pop，也支持阻塞 pop。

常见命令：

```bash
LPUSH queue:email msg1
RPOP queue:email
RPUSH timeline:1001 item1 item2
LRANGE timeline:1001 0 19
LTRIM timeline:1001 0 999
BLPOP queue:email 5
BRPOP queue:email 5
```

### 底层编码

Redis List 现在主要使用 `quicklist`。quicklist 可以理解为“链表 + listpack”的组合：

- 外层是双向链表，便于两端操作。
- 每个节点内部是紧凑连续结构，减少纯链表指针开销。
- 配置会影响每个节点压缩和大小。

### 典型场景

最新列表：

```bash
LPUSH article:latest 1005
LTRIM article:latest 0 99
LRANGE article:latest 0 19
```

简单队列：

```bash
LPUSH queue:task task-1
BRPOP queue:task 10
```

### List 做队列的局限

List 可以做简单队列，但不是完整消息队列：

- 没有消费者组。
- 没有 ack 机制。
- 消费者 pop 后宕机，消息可能丢失。
- 重试、死信、顺序、幂等都要业务自己做。

更复杂的队列建议用 Redis Stream、Kafka、RocketMQ 等。

### 线上风险

- `LRANGE key 0 -1` 拉全量长列表。
- 阻塞命令连接数堆积。
- 列表只进不出，形成 big key。
- 使用 `LINDEX`、`LINSERT` 等中间位置操作，长列表性能差。

### 排查优化

```bash
LLEN timeline:1001
LRANGE timeline:1001 0 20
OBJECT ENCODING timeline:1001
```

优化方向：

- 最新列表必须配合 `LTRIM` 控制长度。
- 队列消费要有积压监控。
- 长列表按时间或用户分片。
- 高可靠消息不要只靠 List。

## Set

### 核心概念

Set 是无序去重集合，适合表达“某个元素是否属于集合”。

常见命令：

```bash
SADD tag:java uid1 uid2 uid3
SISMEMBER tag:java uid1
SREM tag:java uid3
SCARD tag:java
SRANDMEMBER lottery:pool 10
SPOP lottery:pool 10
SINTER user:1:follows user:2:follows
SUNION set:a set:b
SDIFF set:a set:b
SSCAN tag:java 0 COUNT 100
```

### 底层编码

Set 常见编码：

| 编码 | 说明 |
| --- | --- |
| `intset` | 元素全是整数且数量少，内存紧凑 |
| `hashtable` | 通用编码，支持任意字符串元素 |

当元素不再全是整数，或数量超过阈值，会升级为 hashtable。

### 典型场景

- 用户标签。
- 活动参与用户去重。
- 点赞去重。
- 黑白名单。
- 共同关注、共同好友。

点赞去重示例：

```bash
SADD post:1001:liked_users 2001
SISMEMBER post:1001:liked_users 2001
SCARD post:1001:liked_users
```

### 线上风险

- 对大集合直接 `SMEMBERS`。
- 对多个大集合执行 `SINTER`、`SUNION`、`SDIFF` 阻塞主线程。
- 抽奖集合过大且频繁 `SPOP`，造成写热点。
- 只用 Set 存点赞用户，不同步维护计数，导致 `SCARD` 热点。

### 排查优化

```bash
SCARD post:1001:liked_users
SSCAN post:1001:liked_users 0 COUNT 100
MEMORY USAGE post:1001:liked_users
```

优化方向：

- 大集合交并差放到从库、离线任务或业务侧分批做。
- 计数单独维护 String，Set 只做去重明细。
- 按时间、分桶或 hash tag 拆分大集合。

## ZSet

### 核心概念

ZSet 是带 score 的有序集合，member 唯一，按 score 排序。适合排行榜、延迟任务、按权重范围检索。

常见命令：

```bash
ZADD rank:score 100 uid1 95 uid2 120 uid3
ZSCORE rank:score uid1
ZINCRBY rank:score 10 uid1
ZREVRANGE rank:score 0 9 WITHSCORES
ZRANGE rank:score 0 9 WITHSCORES
ZRANK rank:score uid1
ZREVRANK rank:score uid1
ZRANGEBYSCORE delay:queue -inf 1718760000000 LIMIT 0 100
ZREM rank:score uid2
ZCARD rank:score
```

### 底层编码

ZSet 常见编码：

| 编码 | 说明 |
| --- | --- |
| `listpack` | 元素少且 member 小时使用 |
| `skiplist` | 元素多时使用，通常配合 dict |

skiplist 编码下通常有两部分：

- dict：member 到 score 的映射，便于 O(1) 查 member。
- skiplist：按 score 排序，便于范围查询和排名。

### 跳表机制

跳表通过多层索引加速有序链表查询。查找时从高层索引开始，逐层向下逼近目标。平均时间复杂度接近 O(logN)。

Redis 选择跳表而不是红黑树，主要因为：

- 实现简单。
- 范围查询自然。
- 插入删除局部调整简单。
- 性能足够稳定。

### 典型场景

排行榜：

```bash
ZINCRBY rank:daily:20260619 1 uid1001
ZREVRANGE rank:daily:20260619 0 99 WITHSCORES
ZREVRANK rank:daily:20260619 uid1001
```

延迟队列：

```bash
ZADD delay:order 1718760000000 order:1001
ZRANGEBYSCORE delay:order -inf 1718760000000 LIMIT 0 10
ZREM delay:order order:1001
```

延迟队列消费要用 Lua 把“查到期任务”和“删除任务”合成原子操作，否则多个消费者可能重复取到。

### 线上风险

- 热门排行榜集中写同一个 key。
- `ZREVRANGE key 0 -1` 拉全量。
- member 很大导致内存暴涨。
- score 使用浮点数，精度和排序规则理解错误。
- 延迟队列没有幂等，任务重复执行。

### 排查优化

```bash
ZCARD rank:score
MEMORY USAGE rank:score
ZSCAN rank:score 0 COUNT 100
```

优化方向：

- 排行榜按业务、时间、分区拆 key。
- 只保留 TopN，用 `ZREMRANGEBYRANK` 裁剪。
- member 存短 ID，不存大 JSON。
- 高写热点可以先本地聚合再批量写。

## 常见追问

### Redis 各结构时间复杂度怎么答

常见复杂度：

| 命令 | 复杂度 |
| --- | --- |
| `GET`、`SET` | O(1) |
| `HGET`、`HSET` | 平均 O(1) |
| `LPUSH`、`RPOP` | O(1) |
| `LRANGE` | O(S+N)，S 是偏移，N 是返回数量 |
| `SADD`、`SISMEMBER` | 平均 O(1) |
| `SINTER` | 与参与集合大小相关 |
| `ZADD`、`ZREM` | O(logN) |
| `ZRANGE` | O(logN+M) |

面试不要只背 O(1)。Redis 是单线程执行命令，大 key 的 O(N) 命令会阻塞其他请求。

### Hash 一定比 String 省内存吗

不一定。小 Hash 在紧凑编码下可能省内存，但 field 多、value 大或升级 hashtable 后，未必比一个压缩后的 JSON String 更省。还要考虑访问模式、序列化成本和网络包大小。

### ZSet 为什么能同时查排名和查分数

因为 skiplist 负责有序范围和排名，dict 负责 member 快速定位。两套结构共同维护，换来查询效率，代价是内存更高。

### List、Stream、MQ 怎么选

List 适合简单轻量队列；Stream 有消费者组、消息 ID 和 pending 机制；专业 MQ 适合跨服务高可靠、堆积、削峰、重试、死信和可观测性。

## 易错点

- `HGETALL`、`SMEMBERS`、`LRANGE 0 -1`、`ZRANGE 0 -1` 都可能是线上危险命令。
- Redis 单 key 过大不仅占内存，还会影响删除、迁移、持久化和主从同步。
- `SCAN` 系列是渐进式遍历，不保证一次返回固定数量，也可能返回重复元素，业务要能去重。
- Hash 没有 field 级 TTL。
- ZSet score 是 double，不能无限精确表示所有十进制小数。
- List 做队列不等于可靠消息队列。

## 自检清单

- 能说清五种基础结构的业务场景和反场景。
- 能解释 String 的 SDS、Hash/Set/ZSet 的紧凑编码升级。
- 能指出哪些命令会因为 big key 阻塞。
- 能说清 ZSet 为什么用跳表。
- 能给出 Java 后端对象缓存、计数器、排行榜、简单队列的命令示例。
- 能从 `TYPE`、`OBJECT ENCODING`、`MEMORY USAGE`、`SCAN` 系列命令排查结构误用。
