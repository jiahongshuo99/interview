# 主从复制、哨兵、Cluster

## 面试定位

Redis 高可用和分片是 Java 后端面试重点。常见问法：

1. Redis 主从复制流程是什么。
2. 全量同步和增量同步有什么区别。
3. 哨兵如何判断主库下线并完成故障转移。
4. Cluster 如何分片，为什么是 16384 个槽。
5. Cluster 的 MOVED、ASK 是什么。
6. 主从、哨兵、Cluster 分别解决什么问题，不能解决什么问题。

回答要先分层：

- 主从复制解决读扩展和副本冗余。
- 哨兵解决主从架构下自动故障转移。
- Cluster 解决数据分片和集群级扩展。

## 主从复制

### 核心概念

主从复制是一个 master 对多个 replica。写请求通常进入 master，master 把数据同步给 replica。replica 可以承担读请求，也可以作为故障切换候选。

配置示例：

```conf
replicaof 203.0.113.10 6379
masterauth your-password
```

命令：

```bash
REPLICAOF 203.0.113.10 6379
INFO replication
```

旧版本命令是 `SLAVEOF`，新版本推荐 `REPLICAOF`。

### 复制流程

首次同步简化流程：

```text
replica 连接 master
replica 发送 PSYNC
master 判断无法增量同步
master 执行 BGSAVE 生成 RDB
master 把 RDB 发给 replica
replica 清空旧数据并加载 RDB
master 继续发送 RDB 生成期间的增量命令
之后进入命令传播阶段
```

### 全量同步

全量同步会传输完整 RDB。

触发场景：

- replica 第一次连接。
- 复制偏移量差太多，积压缓冲区找不到增量。
- runid 不匹配且不能部分重同步。
- 复制配置变化。

风险：

- master fork 生成 RDB。
- 网络传输大文件。
- replica 加载 RDB 会清空旧数据。
- 大实例全量同步对 CPU、内存、磁盘、网络都有冲击。

### 增量同步

增量同步依赖：

- master replication id。
- replica offset。
- master replication backlog buffer。

如果 replica 短暂断开后重连，master backlog 中还保留断开期间的命令，就可以只补增量。

配置：

```conf
repl-backlog-size 256mb
repl-backlog-ttl 3600
```

### 复制延迟

主从复制是异步的，replica 可能落后 master。

排查：

```bash
INFO replication
```

关注：

```text
master_repl_offset
slave_repl_offset
master_link_status
master_last_io_seconds_ago
```

业务影响：

- 读从库可能读到旧数据。
- 主库宕机时，未复制到从库的数据可能丢失。
- 分布式锁、库存扣减等强一致场景不应读从库判断。

### 读写分离风险

- 主从延迟导致读己之写不一致。
- 从库故障或重连触发全量同步，读流量异常。
- 热 key 读从库也可能打爆单个 replica。
- 客户端要能识别主从角色变化。

## 哨兵 Sentinel

### 核心概念

Sentinel 用于监控 Redis master 和 replica，并在 master 故障时自动完成故障转移。

Sentinel 本身通常部署奇数个节点，例如 3 个或 5 个。

配置示例：

```conf
sentinel monitor mymaster 203.0.113.10 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

`2` 表示至少 2 个 Sentinel 同意主观下线，才能进入客观下线判断。

### 主观下线和客观下线

主观下线 SDOWN：

- 单个 Sentinel 认为 master 超过时间未响应。

客观下线 ODOWN：

- 多个 Sentinel 达成 quorum，认为 master 确实下线。

面试重点：单个 Sentinel 不能直接发起故障转移，避免误判。

### 故障转移流程

简化流程：

```text
Sentinel 发现 master 主观下线
多个 Sentinel 投票确认客观下线
Sentinel 之间选举 leader
leader 从 replica 中选择新 master
对新 master 执行 REPLICAOF NO ONE
让其他 replica 复制新 master
通知客户端主库地址变化
旧 master 恢复后变成 replica
```

### 新 master 选择规则

会综合考虑：

- replica 是否在线。
- 优先级 `replica-priority`。
- 复制 offset，越新越好。
- runid 等稳定排序。

不是随机选。

### Sentinel 客户端

Java 客户端通常连接 Sentinel，查询当前 master 地址。

伪代码：

```java
Set<String> sentinels = Set.of("203.0.113.20:26379", "203.0.113.30:26379", "203.0.113.40:26379");
JedisSentinelPool pool = new JedisSentinelPool("mymaster", sentinels);
try (Jedis jedis = pool.getResource()) {
    jedis.set("k", "v");
}
```

客户端必须支持主从切换后的连接刷新。

### Sentinel 不能解决的问题

- 不能自动分片扩容。
- 不能保证零数据丢失。
- 不能避免网络分区下的复杂一致性问题。
- 不能让单 master 写能力无限扩展。

## Redis Cluster

### 核心概念

Redis Cluster 是 Redis 原生分布式方案。它把 key 空间划分为 16384 个 hash slot，每个 master 负责一部分 slot。

key 到 slot：

```text
slot = CRC16(key) % 16384
```

查看：

```bash
CLUSTER INFO
CLUSTER NODES
CLUSTER SLOTS
```

### 为什么是槽

槽让数据迁移和扩容更容易：

- 不需要按 key 一个个配置路由。
- 节点扩缩容时迁移部分 slot。
- 客户端缓存 slot 到节点的映射。

### hash tag

Cluster 中多 key 命令要求 key 在同一个 slot。可以用 `{}` 指定 hash tag。

```bash
SET user:{1001}:profile v1
SET user:{1001}:stats v2
MGET user:{1001}:profile user:{1001}:stats
```

只有 `{1001}` 参与 slot 计算。

风险：滥用同一个 hash tag 会造成 key 倾斜。

### MOVED 和 ASK

`MOVED`：

- 表示 slot 已经稳定迁移到另一个节点。
- 客户端应更新 slot 缓存。

`ASK`：

- 表示 slot 正在迁移中，这次请求临时去目标节点。
- 客户端不应永久更新 slot 映射。
- 请求前通常要发送 `ASKING`。

### Cluster 故障转移

Cluster 中每个 master 可以有 replica。master 故障后，其 replica 会被提升为新的 master，继续负责原来的 slot。

前提：

- 大多数 master 节点可用。
- 故障 master 至少有可提升 replica。

如果负责某些 slot 的 master 和 replica 都不可用，集群可能无法提供完整服务。

### Cluster 不支持的典型操作

跨 slot 多 key 操作不支持或受限：

```bash
MGET key1 key2
SUNION set1 set2
```

如果 key 不在同一个 slot，会报：

```text
CROSSSLOT Keys in request don't hash to the same slot
```

使用 hash tag 可以让相关 key 落同槽，但要避免单槽过热。

## 三种方案对比

| 方案 | 解决问题 | 不解决问题 |
| --- | --- | --- |
| 主从复制 | 副本、读扩展、故障候选 | 自动切换、分片扩容、强一致 |
| Sentinel | 主从自动故障转移 | 分片扩容、零丢失 |
| Cluster | 分片、扩容、集群故障转移 | 跨 slot 事务、多 key 任意操作、强一致 |

## 线上风险

### 主从延迟

现象：

- 写后读从库读不到。
- failover 后丢少量数据。
- `master_link_down_since_seconds` 增长。

优化：

- 关键读走主库。
- 使用本地版本号或 DB 校验。
- 监控 replication offset 差值。
- 网络和实例负载治理。

### 频繁全量同步

原因：

- backlog 太小。
- 网络不稳定。
- replica 重启频繁。
- 大 key 导致同步慢。

优化：

- 增大 `repl-backlog-size`。
- 控制单实例内存。
- 修复网络。
- 避免高峰期批量重启 replica。

### 脑裂

网络分区时，旧 master 可能仍接受写入，Sentinel 或 Cluster 另一侧提升新 master。网络恢复后旧 master 降级，旧 master 上未同步数据丢失。

缓解：

```conf
min-replicas-to-write 1
min-replicas-max-lag 10
```

含义：master 至少有 1 个延迟不超过 10 秒的 replica，才接受写入。

### Cluster key 倾斜

原因：

- hash tag 使用不当。
- 某业务天然热点集中。
- 大 key 集中在某个 slot。

排查：

```bash
CLUSTER KEYSLOT user:{1001}:profile
CLUSTER COUNTKEYSINSLOT 1234
```

优化：

- 调整 key 设计。
- 热 key 拆分。
- 对大 key 拆桶。
- 重新分配 slot。

## 排查命令

主从：

```bash
INFO replication
ROLE
```

Sentinel：

```bash
redis-cli -p 26379 SENTINEL masters
redis-cli -p 26379 SENTINEL replicas mymaster
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
```

Cluster：

```bash
CLUSTER INFO
CLUSTER NODES
CLUSTER SLOTS
CLUSTER KEYSLOT key
CLUSTER COUNTKEYSINSLOT slot
```

## 常见追问

### Redis 主从是强一致吗

不是。Redis 主从复制默认异步，master 写成功后不等待 replica 全部确认。因此主库宕机时可能丢失未同步数据。

### WAIT 命令能保证强一致吗

`WAIT numreplicas timeout` 可以等待写入传播到指定数量 replica，但不能保证绝对强一致。它提高数据同步概率，会增加延迟，也不能覆盖所有故障场景。

### Sentinel 为什么要奇数个

为了投票和 quorum 更可靠。奇数个节点在多数派判断时资源利用更高，例如 3 个能容忍 1 个故障，5 个能容忍 2 个故障。

### Cluster 为什么多 key 操作要求同 slot

Redis Cluster 没有为任意跨节点多 key 命令提供分布式事务。要求同 slot 可以让命令在单个节点执行，保持 Redis 单节点命令语义。

### Cluster 扩容会影响业务吗

迁移 slot 时可能出现 ASK 重定向，增加延迟。客户端必须支持 Cluster 协议。大 key 迁移会更慢，可能造成阻塞或超时。

## 易错点

- 主从复制不等于高可用，自动切换需要 Sentinel 或 Cluster。
- Sentinel 不负责数据分片。
- Cluster 也不能保证零数据丢失。
- 读从库要接受复制延迟。
- hash tag 可以解决多 key 同槽，但也可能制造倾斜。
- `MOVED` 更新路由，`ASK` 是迁移中的临时重定向。

## 自检清单

- 能画出主从全量同步和增量同步流程。
- 能解释 replication backlog、offset、runid。
- 能说清 Sentinel 的 SDOWN、ODOWN、leader 选举和 failover。
- 能解释 Cluster slot、hash tag、MOVED、ASK。
- 能比较主从、Sentinel、Cluster 的适用边界。
- 能排查主从延迟、全量同步、脑裂和 key 倾斜。
