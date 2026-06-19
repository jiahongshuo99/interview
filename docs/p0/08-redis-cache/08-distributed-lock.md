# 分布式锁：SET NX EX、Lua 解锁、续期、误删

## 面试定位

Redis 分布式锁是 Java 后端高频题。面试官通常会追问：

1. 为什么 `SETNX` 后再 `EXPIRE` 有问题。
2. 正确加锁命令怎么写。
3. 为什么解锁要用 Lua。
4. 锁过期但业务没执行完怎么办。
5. 自动续期有什么风险。
6. RedLock 是否一定可靠。
7. Redis 分布式锁适合哪些场景，不适合哪些场景。

回答要明确：Redis 锁是工程上常用的互斥工具，但不是强一致分布式事务锁。关键场景必须评估故障模型。

## 基础目标

一个可用的 Redis 分布式锁至少要满足：

- 互斥：同一时刻尽量只有一个客户端持有锁。
- 防死锁：客户端宕机后锁能自动释放。
- 防误删：不能删除别人的锁。
- 可超时：获取锁不能无限等待。
- 可观测：加锁失败、持锁时间、续期失败要有监控。

## 错误写法：SETNX + EXPIRE

```bash
SETNX lock:order:1001 token
EXPIRE lock:order:1001 30
```

问题：两条命令不是原子操作。如果 `SETNX` 成功后应用宕机，`EXPIRE` 没执行，锁就没有 TTL，形成死锁。

因此不要在面试中把这个当最终方案。

## 正确加锁：SET NX EX

Redis 推荐用一条 `SET` 命令完成加锁和设置过期时间：

```bash
SET lock:order:1001 2f6c3a7e NX EX 30
```

含义：

- `NX`：key 不存在时才设置。
- `EX 30`：30 秒过期。
- value 是唯一 token，用来标识锁持有者。

毫秒级：

```bash
SET lock:order:1001 2f6c3a7e NX PX 30000
```

返回 `OK` 表示加锁成功，返回 nil 表示锁已存在。

## Java 加锁示例

```java
public Optional<RedisLock> tryLock(String bizKey, Duration ttl) {
    String key = "lock:" + bizKey;
    String token = UUID.randomUUID().toString();
    Boolean success = redisTemplate.opsForValue()
            .setIfAbsent(key, token, ttl);
    if (Boolean.TRUE.equals(success)) {
        return Optional.of(new RedisLock(key, token));
    }
    return Optional.empty();
}
```

注意：

- token 必须每次加锁唯一。
- TTL 要大于业务正常执行时间。
- 获取失败要有明确策略：失败返回、短暂重试、排队或降级。

## 为什么解锁要用 Lua

错误解锁：

```java
if (token.equals(redis.get(lockKey))) {
    redis.del(lockKey);
}
```

问题：`GET` 和 `DEL` 不是原子操作。

典型误删：

```text
线程 A 持有锁 token=A
线程 A 执行业务超时，锁自动过期
线程 B 获取同一把锁 token=B
线程 A 执行 GET，看到 token=B 之前或之后发生竞态
线程 A DEL，可能删除 B 的锁
```

正确做法：用 Lua 保证“判断 token + 删除”原子执行。

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

Java 示例：

```java
private static final String UNLOCK_SCRIPT = """
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
""";

public boolean unlock(String key, String token) {
    Long result = redisTemplate.execute(
            new DefaultRedisScript<>(UNLOCK_SCRIPT, Long.class),
            Collections.singletonList(key),
            token
    );
    return Long.valueOf(1L).equals(result);
}
```

## 锁过期但业务没执行完

这是 Redis 锁最核心风险。

场景：

```text
A 获取锁，TTL=10s
A 执行业务发生 Full GC 或下游慢调用，超过 10s
锁过期
B 获取锁并执行业务
A 恢复后继续执行业务
```

结果：A 和 B 的业务临界区发生重叠。

解决方向：

- TTL 设置覆盖 P99.9 执行时间。
- 业务尽量短小，不把慢 IO 放锁内。
- 自动续期。
- 使用 fencing token 防旧持有者写入。
- 对强一致场景改用 DB 乐观锁、唯一约束或 ZooKeeper/etcd。

## 自动续期

### 基本思路

客户端加锁成功后，后台 watchdog 定期检查锁是否仍由自己持有，如果是则延长 TTL。

伪流程：

```text
加锁成功 TTL=30s
每 10s 检查 token
如果 token 匹配，PEXPIRE 到 30s
业务完成后 Lua 解锁
```

续期 Lua：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("PEXPIRE", KEYS[1], ARGV[2])
else
    return 0
end
```

### 续期风险

- 续期线程挂了，锁仍会过期。
- Redis 网络抖动导致续期失败。
- 业务线程阻塞但续期线程还活着，锁可能被长期占用。
- 如果没有最大持锁时间，可能掩盖业务死循环。

工程建议：

- 设置最大续期次数或最大持锁时间。
- 监控持锁时长。
- 锁内代码必须可超时。
- 解锁失败、续期失败要打日志。

## Redisson 锁

Redisson 提供了 Redis 分布式锁封装，支持 watchdog 自动续期。

示例：

```java
RLock lock = redissonClient.getLock("lock:order:" + orderId);
boolean locked = lock.tryLock(100, 30, TimeUnit.SECONDS);
if (!locked) {
    throw new BizException("系统繁忙");
}
try {
    processOrder(orderId);
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

注意：

- 指定 leaseTime 时，到期后不会无限续期。
- 不指定 leaseTime 时，watchdog 会续期。
- 仍然要评估业务幂等和锁失效后的安全性。

## Fencing Token

### 核心概念

即使锁过期导致两个客户端都执行，也要让旧客户端的写入被拒绝。做法是每次获取锁时生成单调递增 token，下游资源只接受更大的 token。

流程：

```text
A 获取锁，token=100
A 卡顿
B 获取锁，token=101
B 写 DB，DB 记录 last_token=101
A 恢复后写 DB，token=100 < 101，被拒绝
```

Redis 生成 fencing token：

```bash
INCR lock:order:1001:fencing
SET lock:order:1001 token NX EX 30
```

这里需要把获取 token 和加锁语义设计清楚，通常要 Lua 或服务端组件保证流程一致。

适合：

- 对共享资源写入有版本校验能力。
- 下游 DB 表可以保存 version/token。

## RedLock

RedLock 是 Redis 作者提出的多 Redis 实例加锁算法，大致流程是在多个独立 Redis master 上加锁，获得多数成功且耗时小于 TTL 才算成功。

争议点：

- 对时钟、网络分区、暂停时间敏感。
- 实现和运维复杂。
- 并不能让 Redis 锁变成严格 CP 锁。

面试表达建议：

```text
普通业务互斥可以用单 Redis 主从或 Redisson 锁，但核心金融级强一致锁不应只依赖 Redis。RedLock 提高了单点故障容忍，但仍需评估故障模型。更严格场景考虑 DB 约束、ZooKeeper、etcd 或业务幂等。
```

## 适用场景

适合：

- 防止缓存击穿的重建锁。
- 定时任务防重复执行。
- 非核心资源的短临界区互斥。
- 幂等保护的一部分。
- 限制同一业务对象短时间并发处理。

不适合：

- 长事务。
- 强一致资金扣减。
- 无幂等保护的跨系统写入。
- 锁内包含不可控慢调用。
- 需要严格公平锁的场景。

## 线上风险

### 锁粒度过大

例如所有订单共用一把 `lock:order`，会导致吞吐极低。应按业务对象加锁：

```text
lock:order:{orderId}
lock:user:{userId}:coupon
```

### 锁 TTL 过短

业务未完成锁已过期，引发并发执行。要根据 P99/P999 耗时设置，并尽量缩短锁内逻辑。

### 锁 TTL 过长

客户端宕机后恢复慢，其他请求长时间等待。需要平衡。

### 解锁误删

没有 token 或没有 Lua 解锁，会删除别人的锁。

### 重试风暴

大量线程获取锁失败后频繁自旋，会打爆 Redis。

优化：

- 指数退避。
- 最大等待时间。
- 本地限流。
- 请求合并。

## 排查优化

排查命令：

```bash
GET lock:order:1001
PTTL lock:order:1001
CLIENT LIST
SLOWLOG GET 20
```

应用指标：

- 加锁成功率。
- 加锁等待耗时。
- 持锁时间。
- 解锁失败次数。
- 续期次数和失败次数。
- 锁竞争 key topN。

优化方向：

- 缩小锁粒度。
- 锁内只放必须互斥的代码。
- 加超时和降级。
- 业务操作保证幂等。
- 高竞争场景考虑队列化。

## 常见追问

### SET NX EX 是否绝对安全

不是。它解决了加锁和过期的原子性，但不能解决锁过期后业务仍在执行、主从切换丢锁、网络分区等问题。

### 为什么 value 要放 token

为了标识锁持有者。解锁时只有 token 匹配才删除，避免 A 删除 B 的锁。

### Lua 脚本为什么是原子的

Redis 单线程执行命令和 Lua 脚本，脚本执行期间不会被其他命令插入，因此判断和删除可以作为一个原子步骤。

### 主从切换会影响锁吗

会。Redis 主从复制异步，master 加锁成功后还未同步到 replica 就宕机，replica 提升为新 master 后可能没有这把锁，其他客户端可再次加锁。

### 锁内业务失败怎么办

锁只负责互斥，不负责事务回滚。业务要自己处理异常、幂等、补偿和状态机。

## 易错点

- 使用 `SETNX` 后单独 `EXPIRE`。
- 解锁直接 `DEL`。
- token 固定为线程名或用户 ID，不保证唯一。
- 锁内做长时间 RPC。
- 获取锁失败无限自旋。
- 以为 Redisson watchdog 能解决所有锁安全问题。
- 忽略主从异步复制导致的锁丢失。

## 自检清单

- 能写出 `SET key value NX EX seconds` 加锁命令。
- 能解释 `SETNX + EXPIRE` 的死锁问题。
- 能写出 Lua 解锁脚本。
- 能说明锁过期但业务未完成的风险。
- 能解释续期、fencing token、RedLock 的边界。
- 能结合业务说明 Redis 锁适用和不适用场景。
