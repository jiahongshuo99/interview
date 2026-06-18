# 主从复制、读写分离

## 面试定位

主从复制和读写分离考察的是数据库架构和一致性权衡。面试常问：

- MySQL 主从复制流程是什么。
- binlog、relay log、IO thread、SQL thread 分别做什么。
- 主从延迟有哪些原因。
- 读写分离会带来什么一致性问题。
- Java 后端如何保证读己之写。
- 主库故障切换要注意什么。

回答主线：MySQL 复制基于主库 binlog，从库拉取 binlog 写 relay log，再重放到本地。读写分离把写流量路由到主库，读流量路由到从库，提升读扩展能力，但会引入主从延迟和一致性问题。

## 主从复制基本流程

传统异步复制流程：

```text
主库事务提交
  -> 写 binlog
从库 IO thread
  -> 连接主库
  -> 拉取 binlog
  -> 写入 relay log
从库 SQL thread / worker
  -> 读取 relay log
  -> 重放事件
  -> 数据追上主库
```

核心组件：

- binlog：主库记录数据变更。
- dump thread：主库给从库发送 binlog。
- IO thread：从库拉取 binlog。
- relay log：从库本地中继日志。
- SQL thread / worker：从库重放 relay log。

## 复制格式

binlog 常见格式：

- STATEMENT：记录 SQL。
- ROW：记录行变更。
- MIXED：混合。

主从复制通常推荐 ROW：

- 确定性更强。
- 对复杂 SQL、函数、自增等更可靠。
- 便于 CDC 解析。

查看：

```sql
SHOW VARIABLES LIKE 'binlog_format';
```

## 复制模式

### 异步复制

主库提交事务后不等待从库确认。

优点：

- 主库提交延迟低。

缺点：

- 主库宕机时，未同步到从库的 binlog 可能丢失。

### 半同步复制

主库提交时至少等待一个从库确认收到日志后再返回。

优点：

- 降低主库宕机导致数据丢失的概率。

缺点：

- 写入延迟增加。
- 从库或网络异常时可能退化或阻塞，取决于配置。

### 组复制

MySQL Group Replication 提供更强的一致性和故障切换能力，但复杂度更高。普通面试可简要提及，不必展开到协议细节。

## GTID

GTID，全称 Global Transaction Identifier，全局事务 ID。

作用：

- 标识每个事务。
- 简化主从切换和复制位点管理。
- 避免手动找 binlog 文件和 position 的复杂性。

查看：

```sql
SHOW VARIABLES LIKE 'gtid_mode';
SHOW MASTER STATUS;
SHOW REPLICA STATUS\G
```

老版本命令可能是：

```sql
SHOW SLAVE STATUS\G
```

## 主从延迟

主从延迟指从库重放速度落后于主库写入速度。

常见原因：

- 主库写入 QPS 高，从库单线程或并行度不足。
- 大事务导致从库长时间重放。
- 从库硬件或 IO 慢。
- 从库同时承担大量查询，影响重放。
- 网络延迟。
- 表缺主键，ROW 格式回放定位慢。
- 从库上执行了耗时备份或统计任务。
- DDL 重放耗时。

查看：

```sql
SHOW REPLICA STATUS\G
```

关注：

- `Replica_IO_Running`
- `Replica_SQL_Running`
- `Seconds_Behind_Source`
- `Read_Source_Log_Pos`
- `Exec_Source_Log_Pos`
- `Relay_Log_File`
- `Last_SQL_Error`

老版本字段名是 `Slave_IO_Running`、`Slave_SQL_Running`、`Seconds_Behind_Master`。

注意：`Seconds_Behind_Source` 不是绝对可靠，可能为 0 但仍有其他延迟表现，需要结合位点、GTID、业务探针。

## 并行复制

为了减少延迟，从库可以并行回放。

相关配置：

```sql
SHOW VARIABLES LIKE 'replica_parallel_workers';
SHOW VARIABLES LIKE 'slave_parallel_workers';
```

MySQL 8.0 更倾向使用 `replica_*` 命名，旧版本使用 `slave_*`。

并行复制不是万能：

- 单个大事务仍然只能整体处理。
- 热点表、热点行冲突会限制并行度。
- 从库查询压力仍会影响回放。

## 读写分离

读写分离的基本路由：

```text
INSERT/UPDATE/DELETE -> 主库
SELECT -> 从库
```

实现方式：

- 应用代码手动选择数据源。
- Spring 动态数据源。
- ShardingSphere、MyCat 等中间件。
- 数据库代理。
- 云数据库读写分离地址。

目标：

- 分担读流量。
- 提升读扩展能力。
- 降低主库压力。

## 一致性问题

读写分离最大问题是主从延迟。

典型场景：

```text
用户提交订单 -> 写主库成功
立刻查询订单列表 -> 读从库
从库还没同步 -> 查不到刚写入的订单
```

这叫读己之写问题。

## 读己之写解决方案

### 写后读主

同一请求或短时间窗口内，写操作后的读走主库。

```text
request:
  create order -> master
  query order -> master
```

适合强一致链路。

### 基于用户或会话的短期主库路由

用户发生写操作后，在几秒内将该用户读请求路由到主库。

实现：

- 本地上下文标记。
- Redis 标记用户最近写时间。
- 网关或数据源路由标记。

### 等待复制位点

写入后记录主库 GTID 或 binlog 位点，读从库前等待从库追到该位点。

优点：

- 能在从库上获得较强一致性。

缺点：

- 实现复杂。
- 可能增加读延迟。

### 关键查询强制读主

例如支付结果、订单详情、库存扣减结果、权限变更后查询等强一致场景直接读主。

### 接受最终一致

对首页推荐、非关键列表、统计数据，可以接受短暂延迟，并在产品上做刷新或提示。

## 事务中的读写分离

事务内读通常应走主库，避免：

- 同一事务内读不到刚写的数据。
- 从库没有事务上下文。
- 读到旧版本导致业务判断错误。

Spring 动态数据源要注意：

- `@Transactional` 方法内固定数据源。
- 写方法不要被误路由到从库。
- 只读事务是否读从库要结合一致性要求。

## 主库故障切换

主库故障后，需要选新主。

关注：

- 哪个从库数据最新。
- 是否存在未同步事务。
- GTID 是否连续。
- 应用写入口如何切换。
- 旧主恢复后如何处理，避免双主写入。
- DNS、VIP、代理缓存是否及时刷新。

风险：

- 数据丢失。
- 脑裂。
- 主从拓扑混乱。
- 应用连接未刷新。

工程上通常依赖 MHA、Orchestrator、云数据库高可用或自研管控平台，不建议业务服务自己做复杂切主。

## 复制过滤和只读保护

从库通常开启只读：

```sql
SHOW VARIABLES LIKE 'read_only';
SHOW VARIABLES LIKE 'super_read_only';
```

避免业务误写从库。

复制过滤要谨慎：

- `replicate_do_db`
- `replicate_ignore_db`
- 表级过滤

过滤配置容易造成主从数据不一致，生产要严格管理。

## SQL 示例

查看主库 binlog 状态：

```sql
SHOW BINARY LOG STATUS;
```

旧版本：

```sql
SHOW MASTER STATUS;
```

查看从库状态：

```sql
SHOW REPLICA STATUS\G
```

旧版本：

```sql
SHOW SLAVE STATUS\G
```

查看复制线程：

```sql
SHOW PROCESSLIST;
```

查看复制相关配置：

```sql
SHOW VARIABLES LIKE 'server_id';
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'binlog_format';
```

## 排查主从延迟

步骤：

1. 看 IO 线程和 SQL 线程是否正常。
2. 看 `Seconds_Behind_Source` 和位点差距。
3. 看 `Last_IO_Error`、`Last_SQL_Error`。
4. 看是否有大事务或 DDL。
5. 看从库 CPU、IO、磁盘、查询压力。
6. 看并行复制配置。
7. 看表是否缺主键。

常见处理：

- 暂时摘掉延迟从库读流量。
- 杀掉从库慢查询。
- 提升从库规格。
- 调整并行复制。
- 拆分大事务。
- 为表补主键。
- 业务强一致读走主库。

## Java 后端实践

### 数据源路由

常见做法：

```text
@ReadOnly -> replica
default write -> primary
transaction with write -> primary
```

注意：

- 默认走主库更安全。
- 明确只读且可接受延迟的查询才走从库。
- 写后读要有强制主库策略。

### 幂等和重试

主从切换时连接可能断开，业务要处理：

- 写请求重试幂等。
- 唯一键防重复。
- 消息消费幂等。
- 分布式事务或最终一致补偿。

### 缓存一致性

读写分离常和缓存一起出现。写主库后如果从库延迟，同时缓存回源读从库，可能把旧数据写回缓存。

处理：

- 写后删除缓存。
- 强一致回源读主。
- 延迟双删要结合业务评估。
- 缓存值带版本。

## 常见追问

### 主从复制是强一致吗

异步复制不是强一致。半同步降低数据丢失风险，但也不等于所有从库实时可读一致。

### `Seconds_Behind_Master` 为 0 就没有延迟吗

不一定。该指标有局限，复制线程异常、时钟、并行复制、网络等都可能影响判断。要结合位点、GTID 和业务探针。

### 读写分离能提升写性能吗

不能直接提升主库写能力。写仍在主库。它主要扩展读能力，降低主库读压力。

### 为什么表没有主键会影响复制

ROW 格式下从库重放更新或删除需要定位行。没有主键时可能扫描更多数据，复制延迟变大。

## 易错点

- 误以为读写分离后所有查询都可以读从库。
- 误以为半同步就是强一致。
- 误以为从库延迟只看 `Seconds_Behind_Master`。
- 误以为主从复制可以替代备份。
- 误以为主库故障切换只是改连接地址，不考虑数据差异和脑裂。
- 忽略缓存回源读从库导致旧数据污染缓存。

## 自检清单

- 能否画出主库 binlog 到从库 relay log 再回放的流程。
- 能否说明异步、半同步、组复制的区别。
- 能否解释主从延迟原因和排查步骤。
- 能否设计写后读主、会话粘主、等待位点等一致性方案。
- 能否说明事务内为什么通常读主。
- 能否说明故障切换的风险点。
