# 缓存一致性：Cache Aside、延迟双删、binlog 订阅

## 面试定位

缓存一致性是 Java 后端面试中最容易追问细节的题。面试官通常会问：

1. 更新 DB 和更新缓存的顺序怎么选。
2. 为什么常用删除缓存而不是更新缓存。
3. Cache Aside 是什么。
4. 先删缓存再更新 DB 有什么问题。
5. 延迟双删解决什么，为什么不是完美方案。
6. binlog 订阅如何保证最终一致。

回答要先承认现实：缓存和数据库很难做到强一致，常见目标是最终一致，并通过过期时间、消息补偿、幂等和监控降低不一致窗口。

## 基础模型

缓存一致性讨论通常默认：

- DB 是主存储。
- Redis 是旁路缓存。
- 读请求先查缓存，miss 后查 DB 并回填。
- 写请求以 DB 成功为准，再处理缓存。

读流程：

```text
读请求 -> 查 Redis
  -> 命中：返回
  -> 未命中：查 DB -> 写 Redis -> 返回
```

写流程有多种选择，最常用是更新 DB 后删除缓存。

## Cache Aside

### 核心概念

Cache Aside 也叫旁路缓存。应用代码自己负责读写缓存，不是由 Redis 或 DB 自动同步。

读：

```java
public ProductDTO getProduct(long id) {
    String key = "product:" + id;
    String value = redis.get(key);
    if (value != null) {
        return deserialize(value);
    }

    ProductDTO product = productDao.queryById(id);
    if (product == null) {
        redis.setex(key, 60, "__NULL__");
        return null;
    }
    redis.setex(key, 600 + jitter(), serialize(product));
    return product;
}
```

写：

```java
@Transactional
public void updateProduct(ProductUpdateCommand command) {
    productDao.update(command);
    redis.del("product:" + command.getId());
}
```

### 为什么删除缓存，而不是更新缓存

删除缓存更常见，原因：

- 缓存 value 可能由多个表聚合，写请求不一定有完整新值。
- 更新缓存容易和并发读写乱序，产生旧值覆盖新值。
- 删除后下次读从 DB 回填，以 DB 为准。
- 写多读少的数据没必要每次写都重建缓存。

但删除缓存也不是绝对。如果缓存是简单计数、强热点且重建成本高，可以考虑更新缓存或写 DB 后主动刷新。

## 常见写缓存方案对比

### 先更新 DB，再删除缓存

流程：

```text
更新 DB 成功
删除缓存
```

这是最常见方案。

仍可能不一致：

```text
线程 A 读缓存 miss
线程 A 查 DB 得到旧值
线程 B 更新 DB 新值
线程 B 删除缓存
线程 A 把旧值写回缓存
```

这个窗口相对小，但真实存在。

缓解：

- 缓存设置较短 TTL。
- 延迟双删。
- 回填时带版本号。
- 订阅 binlog 再次删除。

### 先删除缓存，再更新 DB

流程：

```text
删除缓存
更新 DB
```

风险更明显：

```text
线程 A 删除缓存
线程 B 读缓存 miss
线程 B 查 DB 旧值
线程 B 写旧值到缓存
线程 A 更新 DB 新值
```

结果：缓存长期保存旧值，直到 TTL 过期。

因此一般不推荐单纯“先删缓存再更新 DB”。

### 更新 DB，再更新缓存

流程：

```text
更新 DB
更新缓存为新值
```

风险：

```text
线程 A 更新 DB 为 v1
线程 B 更新 DB 为 v2
线程 B 更新缓存为 v2
线程 A 因网络慢，最后更新缓存为 v1
```

结果：旧值覆盖新值。

可以通过版本号 CAS、消息顺序、单线程消费等降低风险，但复杂度更高。

## 延迟双删

### 核心概念

延迟双删通常用于缓解并发读回填旧值问题。

流程：

```text
删除缓存
更新 DB
等待一段时间
再次删除缓存
```

也有人采用：

```text
更新 DB
删除缓存
等待一段时间
再次删除缓存
```

第二种更符合常见 Cache Aside 写法。

示例：

```java
@Transactional
public void updateProduct(ProductUpdateCommand command) {
    productDao.update(command);
    String key = "product:" + command.getId();
    redis.del(key);
    delayDeleteExecutor.schedule(() -> redis.del(key), 500, TimeUnit.MILLISECONDS);
}
```

### 延迟时间怎么定

延迟时间应覆盖：

- 读请求查 DB 的耗时。
- 缓存回填耗时。
- 主从读延迟。
- 业务接口 P99 延迟。

不能拍脑袋。通常根据监控和压测数据设定，并接受它只是降低概率，不是强保证。

### 延迟双删的问题

- 第二次删除可能失败。
- 应用宕机可能丢失延迟任务。
- 延迟过短无效，过长增加不一致窗口。
- 高写入场景会产生大量延迟任务。

改进：

- 延迟删除任务放可靠消息队列。
- 删除失败重试。
- 缓存 TTL 兜底。
- 结合 binlog 订阅。

## binlog 订阅

### 核心概念

通过订阅 MySQL binlog 感知 DB 变更，再异步删除或更新缓存。常见组件包括 Canal、Debezium、Flink CDC 等。

流程：

```text
业务写 DB
MySQL 产生 binlog
CDC 组件订阅 binlog
解析出表和主键
投递消息
缓存同步服务删除 Redis key
```

### 优点

- 业务代码侵入少。
- 能覆盖多入口写 DB 的场景。
- 可统一做重试、监控和补偿。
- 对复杂聚合缓存可以集中维护依赖关系。

### 缺点

- 最终一致，有延迟。
- binlog 消费链路本身会故障。
- 表到缓存 key 的映射复杂。
- 消息重复和乱序需要处理。
- DDL、分库分表、多源数据会增加复杂度。

### 关键设计点

#### key 映射

必须知道某张表某行变化影响哪些缓存 key。

示例：

```text
product.id=1001 -> product:1001
sku.product_id=1001 -> product:1001, sku:list:1001
category.id=10 -> category:10, category:tree
```

聚合缓存要维护依赖关系，否则只删主表 key 不够。

#### 幂等

删除缓存天然幂等：

```bash
DEL product:1001
```

重复删除不会破坏正确性，因此 binlog 订阅更推荐删除缓存，而不是直接更新缓存。

#### 顺序

同一主键的变更最好按 binlog 顺序处理。若直接删除缓存，乱序影响较小；若更新缓存，乱序可能旧值覆盖新值。

#### 补偿

需要处理：

- 消费失败重试。
- 死信队列。
- 定期全量校验。
- 消费 lag 告警。
- Redis 删除失败告警。

## 版本号方案

对一致性要求更高的缓存，可以在 DB 和缓存中加入版本号。

回填时：

```java
ProductDTO product = productDao.queryById(id);
String key = "product:" + id;
CacheValue value = new CacheValue(product.getVersion(), product);
redis.setex(key, ttl, serialize(value));
```

更新缓存时只允许新版本覆盖旧版本：

```lua
-- 伪 Lua：只有 newVersion >= oldVersion 才更新
```

适合：

- 并发写频繁。
- 不能接受旧值覆盖新值。
- 缓存更新而非删除。

代价：

- 实现复杂。
- 需要 DB 有可靠版本字段。
- Lua 和序列化协议要维护。

## 删除缓存失败怎么办

这是面试追问重点。

可选方案：

1. 删除失败后重试。
2. 投递 MQ 异步删除。
3. 记录本地 outbox 表，事务内和 DB 更新一起提交。
4. CDC/binlog 订阅兜底删除。
5. 缓存 TTL 最终兜底。

事务 outbox 思路：

```text
本地事务：
  更新业务表
  插入 cache_delete_event 表

后台任务：
  扫描未处理事件
  删除 Redis key
  标记成功
```

优点是不会因为应用在 DB 提交后、发 MQ 前宕机而丢事件。

## 强一致能不能做到

如果要求缓存和 DB 每次读都强一致，Redis Cache Aside 很难满足。可选做法：

- 不使用缓存，直接读 DB。
- 关键读强制读 DB。
- 使用带事务语义的存储系统。
- 使用读写串行化和版本校验，但吞吐会下降。

面试表达：缓存系统通常追求最终一致，核心交易链路要避免把 Redis 当强一致主存储。

## 线上风险

### 脏读窗口

任何异步删除、延迟删除、binlog 删除都有窗口。业务要评估是否可接受。

### 缓存回填旧值

并发读写下，读线程可能把旧 DB 值回填到缓存。

缓解：

- 写后删缓存。
- 延迟双删。
- 版本号。
- 逻辑过期。

### 主从读延迟

如果读 DB 走从库，写后删除缓存后，读请求 miss 再查从库可能读到旧值并回填。

处理：

- 回填缓存读主库。
- 延迟双删覆盖主从延迟。
- 使用版本号或 binlog 删除兜底。

### 热点 key 删除后的击穿

删除热点缓存会导致大量请求回源 DB。

处理：

- 互斥重建。
- 逻辑过期。
- 写后主动刷新。
- 本地缓存兜底。

## 排查优化

不一致排查步骤：

1. 明确 DB 当前值、Redis 当前值、TTL、缓存生成时间。
2. 查业务写日志，确认 DB 更新时间。
3. 查缓存删除日志或 MQ/binlog 消费日志。
4. 查是否有读请求在写后回填旧值。
5. 查 DB 读源是否为从库。
6. 查缓存 key 映射是否漏删。
7. 查序列化版本和字段兼容。

命令：

```bash
GET product:1001
TTL product:1001
OBJECT IDLETIME product:1001
```

应用侧建议记录：

- cache hit/miss。
- rebuild source。
- cache write time。
- value version。
- delete reason。
- binlog event id。

## 常见追问

### 为什么不是先删缓存再更新 DB

因为删除后到 DB 更新前，如果有读请求 miss，会读到旧 DB 值并写回缓存。DB 更新完成后缓存仍是旧值，直到 TTL 过期。

### 更新 DB 后删除缓存就一定没问题吗

不是。仍可能有并发读在 DB 更新前查到旧值，并在删除缓存后把旧值写回。这个窗口较小，可用延迟双删、版本号、TTL、binlog 兜底降低风险。

### 延迟双删为什么要 sleep

等待可能正在进行的旧读请求完成回填，然后第二次删除把它回填的旧缓存删掉。sleep 时间要参考接口 P99、DB 查询耗时和主从延迟。

### binlog 订阅适合什么场景

适合多服务、多入口写 DB，或者缓存 key 需要统一治理的场景。它提供最终一致和补偿能力，但链路更复杂。

### 删除缓存和更新缓存怎么选

多数业务选删除缓存。只有新值容易计算、写入顺序可控、热点重建成本高时，才考虑更新缓存，并通常需要版本控制。

## 易错点

- 认为 Redis 和 DB 可以天然强一致。
- 只说延迟双删，不说第二次删除失败怎么办。
- 忽略读从库导致旧值回填。
- 聚合缓存漏删依赖 key。
- 写 DB 成功后删除缓存失败没有补偿。
- 缓存没有 TTL，脏数据永久存在。

## 自检清单

- 能解释 Cache Aside 读写流程。
- 能比较先删缓存、后删缓存、更新缓存的并发问题。
- 能说明为什么常用删除而不是更新。
- 能说清延迟双删的原理和局限。
- 能设计 binlog 订阅删除缓存链路。
- 能给出删除失败补偿、版本号、主从延迟处理方案。
