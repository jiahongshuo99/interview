# RDB、AOF、混合持久化

## 面试定位

Redis 持久化是高频面试题。常见问法：

1. RDB 和 AOF 有什么区别。
2. Redis 宕机最多丢多少数据。
3. AOF 为什么要 rewrite。
4. Redis 重启加载 RDB 还是 AOF。
5. 混合持久化解决了什么问题。
6. 持久化会不会影响线上性能。

回答要围绕“数据安全、恢复速度、写入性能、文件大小、运维风险”做权衡。

## 总览

| 机制 | 内容 | 优点 | 缺点 |
| --- | --- | --- | --- |
| RDB | 某一时刻内存快照 | 文件紧凑、恢复快、适合备份 | 两次快照之间可能丢数据 |
| AOF | 写命令追加日志 | 数据更安全，可读性较好 | 文件大，恢复慢，需要 rewrite |
| 混合持久化 | RDB 快照 + AOF 增量 | 恢复快，丢数据少 | 文件可读性下降，仍需 rewrite |

## RDB

### 核心概念

RDB 是 Redis 在某个时间点生成的内存快照文件，默认文件名通常是 `dump.rdb`。

触发方式：

- `SAVE`：同步生成 RDB，会阻塞主线程。
- `BGSAVE`：fork 子进程生成 RDB，主进程继续处理请求。
- 配置规则自动触发。
- 主从全量同步时生成。
- Redis 正常关闭时可能触发。

配置示例：

```conf
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb
dir /var/lib/redis
```

含义：900 秒内至少 1 次写、300 秒内至少 10 次写、60 秒内至少 10000 次写，会触发快照。

### BGSAVE 机制

`BGSAVE` 流程：

```text
主进程 fork 子进程
子进程遍历内存生成临时 RDB 文件
写完后用临时文件替换旧 RDB
主进程继续处理请求
```

fork 后依赖操作系统 Copy On Write：

- 父子进程共享物理内存页。
- 如果主进程修改某个页，OS 会复制该页。
- 写入越频繁，COW 额外内存越多。

### RDB 优点

- 文件紧凑，适合备份和跨环境迁移。
- 恢复速度快，加载二进制快照通常比重放 AOF 快。
- 子进程持久化，主进程影响相对较小。

### RDB 缺点

- 两次快照之间的数据可能丢失。
- fork 大实例时可能造成延迟毛刺。
- COW 可能导致内存瞬时上涨。
- 生成 RDB 期间磁盘 IO 压力上升。

### 命令

```bash
BGSAVE
LASTSAVE
INFO persistence
```

## AOF

### 核心概念

AOF 是 Append Only File。Redis 把写命令追加到 AOF 文件，重启时重放命令恢复数据。

配置示例：

```conf
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
```

### appendfsync 策略

| 策略 | 含义 | 数据安全 | 性能 |
| --- | --- | --- | --- |
| `always` | 每个写命令都 fsync | 最安全 | 最慢 |
| `everysec` | 每秒 fsync 一次 | 最多丢约 1 秒 | 常用折中 |
| `no` | 由 OS 决定刷盘 | 可能丢更多 | 最快 |

生产常见选择是 `everysec`。

### AOF 写入流程

简化流程：

```text
执行写命令
追加到 AOF 缓冲区
根据 appendfsync 策略刷盘
后台 rewrite 压缩 AOF
```

注意：命令执行成功不等于已经落盘，取决于 fsync 策略。

### AOF rewrite

AOF 文件会不断追加，必须 rewrite 压缩。

rewrite 不是简单删除旧命令，而是根据当前内存状态生成最小命令集合。例如：

```bash
INCR counter
INCR counter
INCR counter
```

rewrite 后可能变成：

```bash
SET counter 3
```

触发命令：

```bash
BGREWRITEAOF
```

配置：

```conf
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

含义：AOF 当前大小比上次 rewrite 后增长 100%，且超过 64MB，触发 rewrite。

### rewrite 期间增量写怎么处理

rewrite 由子进程完成。rewrite 期间主进程还会继续接收写命令，这些写命令会写入 AOF 缓冲，同时也会记录到 rewrite 增量缓冲。子进程完成后，主进程把增量追加到新 AOF 文件，再原子替换旧文件。

### AOF 优点

- 数据安全性通常比 RDB 好。
- 文件追加写，逻辑简单。
- AOF 文本协议相对可读，可人工修复部分尾部损坏。

### AOF 缺点

- 文件通常比 RDB 大。
- 重启重放命令可能更慢。
- fsync 带来磁盘 IO 压力。
- rewrite 也需要 fork 和 COW。

## 混合持久化

### 核心概念

混合持久化开启后，AOF rewrite 生成的新 AOF 文件前半部分是 RDB 格式快照，后半部分是 rewrite 期间的 AOF 增量命令。

配置：

```conf
aof-use-rdb-preamble yes
```

### 优点

- 重启先加载 RDB 部分，速度快。
- 再重放少量 AOF 增量，数据更完整。
- 兼顾 RDB 恢复速度和 AOF 数据安全。

### 缺点

- AOF 文件不再是纯文本命令，可读性下降。
- 仍然有 fork、COW、磁盘 IO 压力。
- 旧版本 Redis 可能不兼容。

## Redis 重启加载顺序

如果开启 AOF，Redis 重启通常优先加载 AOF，因为 AOF 数据一般比 RDB 更完整。

如果开启混合持久化，本质上仍是加载 AOF 文件，只是 AOF 文件开头包含 RDB preamble。

简化规则：

```text
appendonly yes -> 优先加载 AOF
appendonly no  -> 加载 RDB
```

## 数据丢失边界

| 配置 | 宕机可能丢失 |
| --- | --- |
| 只 RDB | 上次快照后的数据 |
| AOF always | 极少，取决于磁盘和 OS |
| AOF everysec | 通常最多约 1 秒 |
| AOF no | 取决于 OS 刷盘 |
| 无持久化 | 重启后数据丢失 |

注意：主从复制不是持久化。主库误删或写坏数据，会同步到从库。

## 线上风险

### fork 延迟

RDB、AOF rewrite、主从全量同步都可能 fork。大内存实例 fork 成本高，会造成请求延迟毛刺。

排查：

```bash
INFO stats | grep latest_fork_usec
INFO persistence
```

优化：

- 控制单实例内存。
- 避免高峰期 rewrite。
- 保证物理内存充足，降低 COW 风险。

### COW 内存放大

持久化子进程运行期间，主进程写入越多，复制的内存页越多，可能导致 RSS 暴涨。

风险：

- 容器内存超过限制被 OOMKilled。
- 系统开始 swap，Redis 延迟暴涨。

优化：

- 留足内存冗余。
- 控制大 key 和高写入。
- 避免多个 Redis 实例同时 rewrite。

### 磁盘 IO 抖动

AOF fsync、RDB 写文件、rewrite 都会打磁盘。

排查：

```bash
iostat -x 1
INFO persistence
```

Redis 指标：

```text
aof_delayed_fsync
rdb_bgsave_in_progress
aof_rewrite_in_progress
```

### AOF 文件损坏

Redis 提供检查修复工具：

```bash
redis-check-aof --fix appendonly.aof
redis-check-rdb dump.rdb
```

修复可能截断损坏尾部，意味着会丢部分数据。

## 排查命令

```bash
INFO persistence
CONFIG GET save
CONFIG GET appendonly
CONFIG GET appendfsync
CONFIG GET auto-aof-rewrite-percentage
CONFIG GET aof-use-rdb-preamble
LASTSAVE
```

重点字段：

| 字段 | 含义 |
| --- | --- |
| `rdb_last_bgsave_status` | 最近一次 RDB 是否成功 |
| `rdb_last_bgsave_time_sec` | 最近一次 RDB 耗时 |
| `aof_enabled` | 是否开启 AOF |
| `aof_last_bgrewrite_status` | 最近一次 AOF rewrite 状态 |
| `aof_delayed_fsync` | fsync 延迟次数 |
| `current_cow_size` | 当前 COW 相关内存 |

## Java 后端视角

应用侧不要把 Redis 持久化当成数据库事务保证。

合理做法：

- 关键数据落 DB，Redis 做缓存。
- Redis 做分布式锁时，持久化不保证锁语义，锁要靠 TTL、token、Lua。
- Redis 做队列时，必须考虑宕机丢失、重复消费、堆积恢复。
- 缓存预热和降级方案要独立于 Redis 持久化。

## 常见追问

### RDB 和 AOF 能同时开启吗

可以。重启时一般优先加载 AOF，因为 AOF 通常数据更完整。RDB 仍可用于备份和全量快照。

### AOF everysec 一定只丢 1 秒吗

不是绝对保证。正常情况下最多约 1 秒，但极端情况下 OS、磁盘、Redis 进程状态都可能影响实际落盘。

### AOF rewrite 会阻塞吗

rewrite 主体由子进程做，不会像 `SAVE` 那样长时间阻塞主线程。但 fork、rewrite 完成后的增量合并、磁盘 IO 都可能造成延迟毛刺。

### RDB 适合做灾备吗

适合做周期性备份，因为文件紧凑、恢复快。但如果要求尽量少丢数据，需要结合 AOF 或上游数据库。

### 主从复制能替代持久化吗

不能。复制解决副本可用性，持久化解决重启恢复。误删除、程序 bug 写坏数据会同步到从库，不能当备份。

## 易错点

- `SAVE` 会阻塞主线程，线上慎用。
- AOF 文件大不代表数据多，可能是历史命令未 rewrite。
- 开启 AOF 不等于零丢失。
- RDB 子进程写快照期间，主进程写入会带来 COW 内存。
- 混合持久化仍属于 AOF 加载路径。
- Redis 做缓存时，持久化不是核心一致性手段。

## 自检清单

- 能比较 RDB、AOF、混合持久化。
- 能解释 BGSAVE、fork、COW。
- 能说明 AOF rewrite 如何处理增量写。
- 能说清 Redis 重启加载 RDB/AOF 的顺序。
- 能评估不同 fsync 策略的数据丢失边界。
- 能排查 fork 延迟、COW 内存、AOF fsync 延迟和文件损坏。
