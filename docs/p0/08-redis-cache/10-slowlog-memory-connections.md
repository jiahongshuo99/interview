# Redis 慢查询、内存分析、连接数问题

## 面试定位

Redis 排障题通常围绕慢查询、内存和连接数展开。面试官会问：

1. Redis 慢查询怎么查，慢查询日志包含网络耗时吗。
2. Redis CPU 高怎么定位。
3. 内存 used_memory 和 used_memory_rss 有什么区别。
4. 大 key、内存碎片、淘汰怎么排查。
5. 连接数打满、blocked_clients 增长怎么处理。
6. Java 客户端连接池如何配置。

回答要体现线上排查顺序：先看现象和指标，再定位命令、key、客户端、网络和资源瓶颈。

## 慢查询 Slowlog

### 核心概念

Redis slowlog 记录执行时间超过阈值的命令。它记录的是命令在 Redis 线程中执行的耗时，不包含客户端网络传输、排队等待和应用处理时间。

配置：

```bash
CONFIG GET slowlog-log-slower-than
CONFIG GET slowlog-max-len
```

含义：

- `slowlog-log-slower-than`：慢查询阈值，单位微秒。
- `slowlog-max-len`：最多保留多少条慢查询。

查看：

```bash
SLOWLOG LEN
SLOWLOG GET 20
SLOWLOG RESET
```

### 慢查询记录字段

一条 slowlog 通常包含：

- 唯一 ID。
- 执行时间戳。
- 执行耗时，单位微秒。
- 命令和参数。
- 客户端地址。
- 客户端名称。

### 慢查询常见原因

- 大 key 上执行 O(N) 命令。
- `KEYS`、`FLUSHALL`、`FLUSHDB`。
- `HGETALL`、`SMEMBERS`、`LRANGE 0 -1`、`ZRANGE 0 -1`。
- 大集合交并差。
- Lua 脚本执行时间长。
- 删除大 key。
- AOF rewrite、RDB、fork、磁盘 IO 间接造成延迟。

### 慢查询不包含什么

Slowlog 不包含：

- 客户端到 Redis 的网络耗时。
- 请求在客户端连接池排队时间。
- Redis 单线程排队等待前面命令的时间。
- 响应包从 Redis 发回客户端的传输时间。

所以应用感知慢但 slowlog 没记录时，要继续查：

- 网络。
- 客户端连接池。
- Redis 是否有大响应。
- CPU 是否被其他命令占用。
- 客户端超时配置。

## Redis CPU 高排查

### 常用命令

```bash
INFO stats
INFO commandstats
INFO cpu
SLOWLOG GET 20
CLIENT LIST
```

重点字段：

```text
instantaneous_ops_per_sec
used_cpu_sys
used_cpu_user
cmdstat_xxx
rejected_connections
expired_keys
evicted_keys
```

### 排查步骤

1. 看 QPS 是否上涨。
2. 看 commandstats 中哪类命令调用多、耗时高。
3. 看 slowlog 是否有大命令。
4. 查 hot key 和 big key。
5. 看是否大量 key 同时过期。
6. 看是否有 Lua 脚本。
7. 看是否后台持久化或复制导致资源竞争。

### commandstats 示例

```bash
INFO commandstats
```

返回类似：

```text
cmdstat_get:calls=100000,usec=50000,usec_per_call=0.50
cmdstat_hgetall:calls=200,usec=800000,usec_per_call=4000.00
```

`usec_per_call` 高的命令要重点关注。

## 内存分析

### INFO memory

```bash
INFO memory
```

常见字段：

| 字段 | 含义 |
| --- | --- |
| `used_memory` | Redis 分配器统计的已用内存 |
| `used_memory_human` | 可读格式 |
| `used_memory_rss` | OS 看到的 Redis 常驻内存 |
| `used_memory_peak` | 历史峰值 |
| `mem_fragmentation_ratio` | RSS / used_memory |
| `maxmemory` | 最大内存限制 |
| `maxmemory_policy` | 淘汰策略 |

### used_memory 与 RSS

`used_memory` 是 Redis 自己通过内存分配器统计的逻辑使用量。

`used_memory_rss` 是操作系统看到的物理内存占用。

如果 RSS 明显高于 used_memory，可能是：

- 内存碎片。
- fork/COW。
- allocator 未归还内存。
- 大量释放后 RSS 还没下降。

### 内存碎片

`mem_fragmentation_ratio` 经验判断：

- 接近 1：较健康。
- 明显大于 1.5：可能碎片较多。
- 小于 1：可能发生 swap 或统计异常，要警惕。

主动碎片整理配置：

```bash
CONFIG GET activedefrag
CONFIG SET activedefrag yes
```

是否能开启取决于 Redis 构建和版本。开启后会消耗一定 CPU。

### 查 key 内存

```bash
MEMORY USAGE key
OBJECT ENCODING key
```

扫描大 key：

```bash
redis-cli --bigkeys
redis-cli --memkeys
```

不要在线上高峰直接全量扫描大实例。

### 淘汰和过期

```bash
INFO stats | grep expired_keys
INFO stats | grep evicted_keys
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
```

判断：

- `expired_keys` 快速增长：大量过期删除。
- `evicted_keys` 快速增长：内存达到上限后正在淘汰。
- `evicted_keys` 增长伴随命中率下降：缓存容量不足或策略不合适。

## 连接数问题

### 常用指标

```bash
INFO clients
CONFIG GET maxclients
CLIENT LIST
CLIENT COUNT
```

关键字段：

| 字段 | 含义 |
| --- | --- |
| `connected_clients` | 当前客户端连接数 |
| `blocked_clients` | 被阻塞命令阻塞的客户端数 |
| `maxclients` | 最大连接数 |
| `client_recent_max_input_buffer` | 最近客户端输入缓冲峰值 |
| `client_recent_max_output_buffer` | 最近客户端输出缓冲峰值 |

### 连接数打满现象

- 应用报 `ERR max number of clients reached`。
- Redis `rejected_connections` 增长。
- 应用连接池获取连接超时。
- 客户端大量短连接。

排查：

```bash
INFO stats | grep rejected_connections
CLIENT LIST
```

`CLIENT LIST` 中关注：

- addr：来源地址。
- name：客户端名。
- age：连接存活时间。
- idle：空闲时间。
- cmd：最后执行命令。
- omem：输出缓冲区占用。

### blocked_clients 增长

常见原因：

- `BLPOP`、`BRPOP`。
- `XREAD BLOCK`。
- 阻塞式模块命令。

blocked_clients 不一定是坏事，但要确认是否符合预期。如果大量业务请求连接被阻塞，可能耗尽连接池。

### 输出缓冲区过大

大查询或慢客户端会导致输出缓冲区膨胀。

场景：

- `HGETALL` 大 hash。
- `SMEMBERS` 大 set。
- pub/sub 客户端消费慢。
- 网络慢导致响应发不出去。

配置：

```bash
CONFIG GET client-output-buffer-limit
```

风险：达到限制后 Redis 会断开客户端。

## Java 客户端连接池

### 常见问题

- 每次请求新建 Redis 连接。
- 连接池太小，业务线程阻塞等待连接。
- 连接池太大，Redis maxclients 被打满。
- 超时时间过长，故障时线程堆积。
- 没有区分普通命令和阻塞命令连接池。
- Lettuce 共享连接使用不当。

### Jedis 连接池示例

```java
JedisPoolConfig config = new JedisPoolConfig();
config.setMaxTotal(200);
config.setMaxIdle(50);
config.setMinIdle(10);
config.setMaxWait(Duration.ofMillis(200));
config.setTestOnBorrow(false);

JedisPool pool = new JedisPool(config, "127.0.0.1", 6379, 1000);
```

配置原则：

- `maxTotal` 根据应用实例数和 Redis maxclients 反推。
- `maxWait` 不要过长。
- 连接池耗尽要快速失败或降级。
- 阻塞命令使用独立连接池。

### Lettuce 注意点

Lettuce 是基于 Netty 的异步客户端，可以多线程共享连接。但如果使用阻塞命令、事务、pipeline 或连接状态相关命令，要注意连接隔离。

Spring Boot 使用 Lettuce 时也要关注：

- command timeout。
- shutdown timeout。
- pool enabled 与否。
- topology refresh。
- Cluster 重定向支持。

## Pipeline 与慢查询

Pipeline 可以减少网络 RTT，但不会减少 Redis 执行命令本身的 CPU。

风险：

- 一次 pipeline 太大，Redis 输出缓冲区暴涨。
- 客户端内存膨胀。
- 某条慢命令仍阻塞后续命令。

建议：

- 控制批大小，例如 100 到 1000 视 value 大小调整。
- 避免 pipeline 中混入大 key 命令。
- 对失败做部分重试和幂等设计。

## Lua 脚本排查

Lua 脚本在 Redis 主线程执行。长脚本会阻塞其他请求。

排查：

```bash
SLOWLOG GET 20
SCRIPT KILL
```

`SCRIPT KILL` 只能杀掉未执行写命令的脚本。已经执行写命令的脚本不能随便杀，否则会破坏原子性。

设计原则：

- Lua 只做短小原子逻辑。
- 不在 Lua 中遍历大集合。
- 参数数量和 key 数量可控。
- 脚本耗时纳入监控。

## 网络和延迟

应用感知 Redis 慢，但 Redis slowlog 正常时，要考虑网络和客户端。

排查：

```bash
redis-cli --latency
redis-cli --latency-history
redis-cli --intrinsic-latency 100
```

方向：

- 客户端和 Redis 是否跨机房。
- 是否有丢包、重传。
- 响应包是否过大。
- 应用连接池是否等待。
- Redis 所在机器是否 CPU steal 或磁盘抖动。

## 典型排查场景

### 场景一：接口 Redis 超时

步骤：

1. 看应用日志是获取连接超时还是命令执行超时。
2. 看 Redis `connected_clients`、`rejected_connections`。
3. 看 slowlog。
4. 看 commandstats。
5. 看网络延迟。
6. 查是否有大 key 或热点 key。

### 场景二：Redis 内存持续上涨

步骤：

1. 看 `used_memory` 和 `used_memory_rss`。
2. 看 key 数量：`INFO keyspace`。
3. 看 `expired_keys` 和 `evicted_keys`。
4. 查大 key 和 key 前缀增长。
5. 查是否有无 TTL key。
6. 查最近上线功能是否新增缓存。

### 场景三：连接数突然升高

步骤：

1. `CLIENT LIST` 找来源 IP。
2. 查应用发布和扩容。
3. 查是否短连接或连接泄漏。
4. 查连接池配置。
5. 查是否阻塞命令占用连接。
6. 临时限流或扩 maxclients，但要评估系统 fd 限制。

## 常见追问

### slowlog 为空但应用超时，说明 Redis 没问题吗

不能这么判断。slowlog 只记录命令执行时间，不包括排队、网络、连接池等待和大响应传输。需要继续查客户端和网络。

### Redis 单线程为什么还能这么快

Redis 主要操作在内存中完成，数据结构高效，单线程避免了锁竞争；网络 IO 使用多路复用。新版本部分网络 IO 和后台释放等能力也有多线程辅助，但命令执行核心仍要按单线程模型理解。

### used_memory_rss 很高怎么办

先判断是否正在 RDB/AOF rewrite 或复制导致 COW，再看内存碎片。可以低峰重启、开启 activedefrag、治理大 key 和频繁分配释放模式。

### maxclients 调大就能解决连接问题吗

不一定。连接过多会增加 Redis 和 OS 资源消耗。要先找来源，是连接池配置、短连接、阻塞命令还是故障重试。调大 maxclients 只是临时手段。

### pipeline 越大越好吗

不是。pipeline 太大会造成输出缓冲区和客户端内存上涨，单次请求延迟变长，也可能放大失败重试成本。

## 易错点

- 认为 slowlog 包含网络耗时。
- 线上使用 `KEYS *` 排查。
- 看到连接数高就直接调大 maxclients。
- 忽略客户端连接池等待时间。
- 用 `DEL` 删除大 key。
- Lua 脚本遍历大集合。
- pipeline 无限制批量提交。

## 自检清单

- 能使用 `SLOWLOG GET`、`INFO commandstats` 定位慢命令。
- 能解释 slowlog 不包含哪些耗时。
- 能区分 `used_memory` 和 `used_memory_rss`。
- 能排查内存碎片、淘汰、过期、大 key。
- 能分析 `connected_clients`、`blocked_clients`、`rejected_connections`。
- 能给出 Java 连接池、pipeline、Lua、网络延迟的优化建议。
