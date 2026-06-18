# Bitmap、HyperLogLog、Geo、Stream

## 面试定位

Bitmap、HyperLogLog、Geo、Stream 属于 Redis 的高级数据结构或特殊能力。面试中常见问法是：

1. Bitmap 怎么做签到、活跃用户统计。
2. HyperLogLog 为什么能做 UV，误差和限制是什么。
3. Geo 底层是不是一种新结构，如何查附近的人。
4. Stream 和 List 做队列有什么区别。
5. 这些结构的线上风险是什么。

回答关键不是背命令，而是说清“解决什么问题、牺牲什么、不能做什么”。

## 总览

| 能力 | 适合场景 | 核心优势 | 主要限制 |
| --- | --- | --- | --- |
| Bitmap | 签到、布尔状态、用户活跃 | 极省内存、位运算快 | offset 过大浪费空间，只能表达有限状态 |
| HyperLogLog | UV、去重计数 | 固定小内存估算基数 | 有误差，不能取明细 |
| Geo | 附近门店、附近的人 | 经纬度索引、距离计算 | 地球模型近似，底层是 ZSet |
| Stream | 消息流、轻量 MQ | 消费者组、ack、pending | 不是专业 MQ，堆积和治理能力有限 |

## Bitmap

### 核心概念

Bitmap 本质上不是独立类型，而是 String 的位操作能力。一个 bit 表示一个布尔状态。

常见命令：

```bash
SETBIT sign:1001:202606 18 1
GETBIT sign:1001:202606 18
BITCOUNT sign:1001:202606
BITOP AND active:both active:20260618 active:20260619
BITPOS sign:1001:202606 1
```

offset 从 0 开始。`SETBIT key 18 1` 表示把第 19 位设置为 1。

### 典型场景

用户签到：

```bash
# 用户 1001 在 2026 年 6 月 19 日签到，按月存储，第 19 天 offset 为 18
SETBIT sign:1001:202606 18 1

# 查询当天是否签到
GETBIT sign:1001:202606 18

# 查询本月签到天数
BITCOUNT sign:1001:202606
```

日活用户：

```bash
# uid 作为 offset，某天登录则置 1
SETBIT active:20260619 1001 1

# 当天活跃用户数
BITCOUNT active:20260619

# 连续两天都活跃
BITOP AND active:20260618_19 active:20260618 active:20260619
BITCOUNT active:20260618_19
```

### 底层机制

Bitmap 存在 String 的字节数组里。offset 越大，Redis 需要扩展 SDS 到对应长度。

内存估算：

```text
内存字节数约等于 max_offset / 8
```

如果 uid 是 10 亿，哪怕只有一个用户，直接用 uid 做 offset 也可能分配约 125MB。

### Java 使用示例

```java
String key = "sign:" + userId + ":" + YearMonth.now().format(DateTimeFormatter.ofPattern("yyyyMM"));
int offset = LocalDate.now().getDayOfMonth() - 1;

Boolean signed = redisTemplate.opsForValue().getBit(key, offset);
redisTemplate.opsForValue().setBit(key, offset, true);
Long count = redisTemplate.execute((RedisCallback<Long>) connection ->
        connection.bitCount(key.getBytes(StandardCharsets.UTF_8)));
```

### 线上风险

- offset 直接使用稀疏大 ID，造成内存浪费。
- `BITOP` 对超大 bitmap 计算，阻塞主线程。
- 多天 bitmap 做交并运算时临时结果 key 很大。
- 用 Bitmap 表达复杂状态，后期扩展困难。

### 优化建议

- uid 稀疏时先做映射，或按 uid 分片。
- 按日期、业务线拆 key。
- 大规模 `BITOP` 放低峰或离线。
- 临时结果设置 TTL。

## HyperLogLog

### 核心概念

HyperLogLog 用于基数估算，适合统计 UV。它能以很小固定内存估算一个集合中不同元素数量。

常见命令：

```bash
PFADD uv:20260619 uid1 uid2 uid3
PFCOUNT uv:20260619
PFMERGE uv:202606 uv:20260601 uv:20260602 uv:20260603
```

### 底层机制

HyperLogLog 基于概率统计。它通过 hash 后观察二进制中前导零等特征估算基数。

特点：

- 内存固定，Redis 中一个 HLL 大约 12KB。
- 标准误差约 0.81%。
- 只能估算数量，不能取回元素明细。

### 典型场景

页面 UV：

```bash
PFADD uv:page:1001:20260619 user:1 user:2 user:3
PFCOUNT uv:page:1001:20260619
```

月 UV：

```bash
PFMERGE uv:page:1001:202606 uv:page:1001:20260601 uv:page:1001:20260602
PFCOUNT uv:page:1001:202606
```

### HyperLogLog 与 Set 的区别

| 维度 | HyperLogLog | Set |
| --- | --- | --- |
| 内存 | 固定小内存 | 随元素数量增长 |
| 结果 | 估算值，有误差 | 精确值 |
| 明细 | 不能取元素 | 可以取元素 |
| 场景 | 大规模 UV | 精确去重、成员判断 |

面试回答：如果业务能接受小误差且只要数量，用 HLL；如果需要精确判断某用户是否访问过，必须用 Set 或明细表。

### 线上风险

- 把 HLL 当精确计数器使用。
- 需要用户明细时才发现 HLL 无法反查。
- 小数据量场景使用 HLL 反而没有必要。
- `PFMERGE` 合并大量 key 时也会消耗 CPU。

## Geo

### 核心概念

Geo 用于存储经纬度并做距离和附近查询。它底层不是全新类型，而是基于 ZSet 实现。

常见命令：

```bash
GEOADD shop:geo 116.397128 39.916527 shop1
GEODIST shop:geo shop1 shop2 km
GEOPOS shop:geo shop1
GEOHASH shop:geo shop1
GEOSEARCH shop:geo FROMLONLAT 116.40 39.90 BYRADIUS 5 km WITHDIST ASC COUNT 20
```

旧命令 `GEORADIUS`、`GEORADIUSBYMEMBER` 仍常见，但新版本推荐 `GEOSEARCH`、`GEOSEARCHSTORE`。

### 底层机制

Redis Geo 会把经纬度编码成 geohash，再作为 ZSet 的 score 存储。

因此：

- Geo member 唯一。
- 可以用 ZSet 相关命令观察或删除成员。
- 范围查询本质是按 geohash 区间粗筛，再计算距离精排。

### 典型场景

附近门店：

```bash
GEOADD takeout:shops 116.397128 39.916527 shop:1
GEOADD takeout:shops 116.407128 39.926527 shop:2

GEOSEARCH takeout:shops FROMLONLAT 116.400000 39.920000 BYRADIUS 3 km WITHDIST ASC COUNT 10
```

Java 侧通常还要结合门店状态、营业时间、配送范围、库存等过滤，Redis Geo 只负责空间候选集。

### 线上风险

- 经纬度写反。Redis 命令顺序是 longitude、latitude。
- 以为 Geo 可以直接处理复杂多边形配送范围。
- 半径过大导致候选结果过多。
- 一个城市所有门店放单 key，形成热点。
- 附近的人频繁更新位置，写热点明显。

### 优化建议

- 按城市、业务线或 geohash 前缀拆 key。
- 限制半径和返回数量。
- Redis 只做粗筛，复杂规则放业务或搜索引擎。
- 对频繁移动对象做更新频率限制。

## Stream

### 核心概念

Stream 是 Redis 提供的消息流结构，支持消息 ID、消费者组、ack、pending list。它比 List 更适合做轻量消息队列。

常见命令：

```bash
XADD order:stream * orderId 1001 status created
XLEN order:stream
XRANGE order:stream - +
XREAD COUNT 10 BLOCK 5000 STREAMS order:stream $

XGROUP CREATE order:stream group-a 0 MKSTREAM
XREADGROUP GROUP group-a consumer-1 COUNT 10 BLOCK 5000 STREAMS order:stream >
XACK order:stream group-a 1718760000000-0
XPENDING order:stream group-a
XCLAIM order:stream group-a consumer-2 60000 1718760000000-0
XTRIM order:stream MAXLEN ~ 100000
```

### 消息 ID

Stream 消息 ID 通常形如：

```text
1718760000000-0
```

前半部分是毫秒时间戳，后半部分是同毫秒内序号。`XADD key *` 表示由 Redis 自动生成 ID。

### 消费者组机制

消费者组让多个消费者协同消费同一条 Stream：

- `>` 表示读取从未投递给该消费者组的新消息。
- 消费后要 `XACK`。
- 未 ack 的消息进入 PEL，即 Pending Entries List。
- 消费者宕机后，可通过 `XPENDING` 查看，再用 `XCLAIM` 或 `XAUTOCLAIM` 转移。

### Stream 与 List 对比

| 维度 | List | Stream |
| --- | --- | --- |
| 消息 ID | 无内置 ID | 有 ID |
| 消费者组 | 不支持 | 支持 |
| ack | 不支持 | 支持 |
| pending | 不支持 | 支持 |
| 多播 | 困难 | 多消费者组可各自消费 |
| 使用复杂度 | 低 | 中 |

### Stream 与专业 MQ 对比

Stream 适合：

- Redis 已经是系统组件。
- 消息量中等。
- 需要轻量异步解耦。
- 可以接受 Redis 级别的可靠性。

不适合：

- 超大堆积。
- 复杂重试和死信治理。
- 跨机房强可靠。
- 严格顺序和事务消息。
- 完整消息审计。

### Java 消费伪代码

```java
while (running) {
    List<MapRecord<String, Object, Object>> records = streamOps.read(
            Consumer.from("group-a", "consumer-1"),
            StreamReadOptions.empty().count(10).block(Duration.ofSeconds(5)),
            StreamOffset.create("order:stream", ReadOffset.lastConsumed())
    );

    for (MapRecord<String, Object, Object> record : records) {
        try {
            handle(record.getValue());
            streamOps.acknowledge("order:stream", "group-a", record.getId());
        } catch (Exception e) {
            // 记录失败，等待重试或转移 pending
        }
    }
}
```

### 线上风险

- 忘记 `XACK`，pending 持续增长。
- 没有 `XTRIM`，Stream 无限增长。
- 消费者组创建 ID 选择错误，导致历史消息全量涌入或漏消费。
- 消费逻辑非幂等，pending 重投后重复处理。
- 单 Stream key 成为写热点。

### 排查优化

```bash
XLEN order:stream
XINFO STREAM order:stream
XINFO GROUPS order:stream
XINFO CONSUMERS order:stream group-a
XPENDING order:stream group-a
XRANGE order:stream - + COUNT 10
```

优化方向：

- 设置最大长度：`XTRIM MAXLEN ~ N` 或 `XADD ... MAXLEN ~ N`。
- 消费必须幂等。
- 定期处理 pending。
- 按业务分片 Stream。
- 监控 lag、pending 数、最老 pending 时间。

## 常见追问

### Bitmap 和 Set 都能做活跃用户，怎么选

如果只需要统计数量和集合运算，且用户 ID 可以映射到紧凑 offset，Bitmap 更省内存。若要保存用户明细、用户 ID 稀疏、需要随机取成员或判断成员集合，Set 更直接。

### HyperLogLog 为什么不能统计精确 UV

它是概率估算结构，只保留统计摘要，不保留完整成员集合。它牺牲精确性和明细能力换固定小内存。

### Geo 能不能替代 Elasticsearch 地理查询

不能完全替代。Redis Geo 适合简单半径查询和附近排序；复杂多边形、组合过滤、全文检索、多条件排序更适合搜索引擎。

### Stream 消费者宕机怎么办

消息会留在消费者组的 pending 中。其他消费者可以通过 `XPENDING` 查看，再用 `XCLAIM` 或 `XAUTOCLAIM` 把超时未 ack 的消息转移过来处理。业务处理必须幂等。

## 易错点

- Bitmap offset 不是元素值本身，offset 过大直接浪费内存。
- HLL 有误差，不能用于强一致精确计费。
- Redis Geo 命令参数顺序是经度在前、纬度在后。
- Stream 不 ack 不代表消息没消费，而是留在 pending。
- `XREADGROUP ... >` 只读新消息，不读 pending 旧消息。
- `XTRIM MAXLEN ~` 是近似裁剪，不保证精确长度。

## 自检清单

- 能用 Bitmap 设计签到和连续活跃统计。
- 能说清 HyperLogLog 的误差、内存和不能反查明细。
- 能解释 Geo 底层基于 ZSet 和 geohash。
- 能说清 Stream 的 ID、消费者组、ack、pending。
- 能比较 List、Stream、专业 MQ 的边界。
- 能列出每种结构的线上风险和排查命令。
