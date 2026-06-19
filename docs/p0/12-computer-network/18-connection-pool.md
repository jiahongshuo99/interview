# 连接池

## 面试定位

连接池是 Java 后端网络性能和稳定性的核心。面试常问：为什么需要连接池、连接池有哪些关键参数、连接泄漏怎么排查、连接池满和下游慢有什么关系、HTTP/RPC/DB 连接池有什么共性。

核心回答：

连接池复用已经建立的连接，减少建连成本，并通过容量、超时、空闲回收和生命周期控制保护调用方和下游。

## 为什么需要连接池

每次请求新建连接的成本：

- DNS 解析。
- TCP 三次握手。
- TLS 握手。
- TCP 慢启动。
- 临时端口占用。
- TIME_WAIT 增多。

连接池收益：

- 降低请求延迟。
- 降低 CPU 和网络开销。
- 控制并发连接数。
- 提供背压。
- 复用 TLS 会话和 TCP 连接。

## 核心参数

常见连接池参数：

| 参数 | 含义 | 风险 |
| --- | --- | --- |
| maxTotal | 总最大连接数 | 太小导致等待，太大压垮下游 |
| maxPerRoute | 单目标最大连接数 | 热点下游可能被限制 |
| acquire timeout | 获取连接超时 | 不配置可能请求长时间阻塞 |
| connect timeout | 建连超时 | 太长拖慢失败恢复 |
| read timeout | 读响应超时 | 太短误杀慢请求，太长占连接 |
| idle timeout | 空闲连接保留时间 | 与 LB 不一致会复用失效连接 |
| max lifetime | 连接最大生命周期 | 太长可能负载不均，太短降低复用 |
| validation | 连接可用性检测 | 检测过频有开销 |

## 连接池状态

连接通常分为：

- leased：正在被业务使用。
- idle：空闲可复用。
- pending：等待获取连接的请求。
- closed：已关闭。

健康状态：

```text
leased 不应长期接近 max
pending 不应持续积压
idle 不应全部不可用
```

## 连接池满的常见原因

### 下游慢

请求占用连接时间变长，连接归还慢，池很快打满。

表现：

- leased 持续高。
- pending 增加。
- 客户端超时。
- 下游 P99 升高。

### 连接泄漏

业务没有关闭响应体或流，连接无法归还。

表现：

- leased 持续增长不下降。
- 下游不一定慢。
- CLOSE_WAIT 可能增多。

### 池太小

请求并发超过池容量。

需要结合 QPS 和平均耗时估算：

```text
并发数 ≈ QPS * 平均耗时秒数
```

例如 QPS 200、平均耗时 100ms，平均并发约 20。还要考虑峰值和长尾。

### 目标路由限制

总连接数够，但单路由连接数小，访问同一下游时仍等待。

## Java HTTP Client 示例

OkHttp：

```java
ConnectionPool pool = new ConnectionPool(100, 5, TimeUnit.MINUTES);

OkHttpClient client = new OkHttpClient.Builder()
        .connectionPool(pool)
        .connectTimeout(Duration.ofSeconds(1))
        .readTimeout(Duration.ofSeconds(3))
        .writeTimeout(Duration.ofSeconds(3))
        .callTimeout(Duration.ofSeconds(5))
        .build();
```

使用原则：

- `OkHttpClient` 复用单例。
- `Response` 使用 try-with-resources 关闭。
- 按下游维度设置合理并发。

错误示例：

```java
for (...) {
    OkHttpClient client = new OkHttpClient();
    client.newCall(request).execute();
}
```

这会破坏连接复用。

## 连接池和负载均衡

长连接会影响负载均衡：

- 客户端对某个实例建立连接后，会持续复用。
- 新实例上线后，不一定马上有流量。
- 旧实例下线时，老连接需要优雅关闭。
- HTTP/2 单连接多路复用可能让负载更集中。

治理：

- 连接最大生命周期。
- 客户端定期刷新连接。
- 服务端优雅下线。
- LB 支持连接排水。

## 命令和指标

查看连接状态：

```bash
ss -antp | grep ':443'
ss -antp state established
ss -antp state close-wait
```

查看进程打开文件：

```bash
lsof -p <pid> | wc -l
lsof -iTCP -p <pid>
```

应用指标建议：

- 最大连接数。
- leased 连接数。
- idle 连接数。
- pending 请求数。
- 获取连接耗时。
- 连接创建数。
- 连接关闭数。
- 连接复用率。
- 连接池拒绝或超时数。

## 线上排查

### 获取连接超时

排查顺序：

1. 看连接池 max 是否过小。
2. 看 leased 是否接近上限。
3. 看 pending 是否积压。
4. 看下游耗时是否升高。
5. 看是否响应体未关闭。
6. 看是否每次请求创建新 client。

### CLOSE_WAIT 多

优先查连接释放：

- HTTP response 是否关闭。
- 异常路径是否 close。
- 流是否 try-with-resources。
- 线程是否卡住无法释放。

### connection reset 增多

可能是复用失效连接：

- 客户端 idle timeout 大于 LB idle timeout。
- 服务端滚动重启。
- 代理主动关闭。

治理：

- 调整 idle timeout。
- 开启失效连接检测。
- 幂等请求有限重试。

## 常见追问

### 连接池越大越好吗

不是。过大连接池会增加下游并发压力、内存占用、上下文切换和排队风险。连接池是隔离和背压手段，不是无限扩容。

### 连接池满应该扩容吗

先判断原因。如果下游慢或连接泄漏，扩容只会放大问题。只有容量确实不足且下游可承载时才扩容。

### 连接池和线程池有什么关系

线程池控制本地任务并发，连接池控制对下游连接并发。线程很多但连接少会等连接；连接很多但线程少也发不出去。二者要协同配置。

## 易错点

- 每次请求创建 HTTP Client。
- 不配置连接获取超时。
- 只看总连接数，不看 per route。
- 连接池满就直接扩大。
- 不关闭 response body。
- 忽略连接池对负载均衡的影响。

## 自检清单

- 能否说明连接池收益和风险。
- 能否解释 leased、idle、pending。
- 能否估算连接池容量。
- 能否排查获取连接超时和连接泄漏。
- 能否说明连接池与 idle timeout、LB、重试的关系。
