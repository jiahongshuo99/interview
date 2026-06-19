# HTTP/1.1 长连接

## 面试定位

HTTP/1.1 长连接是后端面试中连接池、TIME_WAIT、性能优化的基础。面试常问：HTTP/1.1 为什么默认长连接、长连接和连接池关系是什么、长连接有什么风险、服务端和客户端 idle timeout 不一致会怎样。

核心回答：

HTTP/1.1 默认启用持久连接，多个请求响应可以复用同一个 TCP 连接，减少 TCP 握手、TLS 握手和慢启动成本。

## 核心概念

HTTP/1.0 默认短连接，通常每个请求完成后关闭 TCP 连接。

HTTP/1.1 默认长连接：

```http
Connection: keep-alive
```

在 HTTP/1.1 中即使不显式写该 Header，通常也默认保持连接。若要关闭：

```http
Connection: close
```

长连接的本质：

- TCP 连接不在一次 HTTP 响应后立即关闭。
- 后续请求可以继续使用同一连接。
- 客户端连接池负责管理这些可复用连接。

## 为什么需要长连接

短连接成本：

- 每次请求都要 TCP 三次握手。
- HTTPS 还要 TLS 握手。
- TCP 慢启动导致初始吞吐较低。
- 高 QPS 下产生大量 TIME_WAIT。

长连接收益：

- 降低握手开销。
- 降低延迟。
- 减少临时端口消耗。
- 减少 TIME_WAIT 数量。
- 提高连接池复用率。

## HTTP/1.1 长连接限制

HTTP/1.1 同一个连接上的响应必须按请求顺序返回。

虽然 HTTP/1.1 有 pipelining 概念，可以连续发送多个请求，但实际使用很少，因为存在应用层队头阻塞：

```text
请求 A、B、C 共用一个连接
服务端必须按 A、B、C 顺序返回
A 慢会阻塞 B、C 的响应
```

因此实际工程里 HTTP/1.1 客户端通常通过连接池维护多个连接并发请求。

## 长连接与连接池

长连接是协议能力，连接池是客户端工程实现。

连接池负责：

- 创建连接。
- 复用空闲连接。
- 控制最大连接数。
- 控制每个目标地址的连接数。
- 清理过期或失效连接。
- 管理连接获取超时。

如果每次请求都 new 一个 HTTP Client，连接池往往无法复用，会退化成短连接。

## 长连接生命周期

```text
创建连接
  -> TCP 握手
  -> TLS 握手，HTTPS 场景
  -> 发送请求 1
  -> 读取响应 1
  -> 连接回到空闲池
  -> 发送请求 2
  -> 读取响应 2
  -> 空闲超时或服务关闭
```

连接能否复用取决于：

- 请求和响应是否完整读写。
- 服务端是否返回 `Connection: close`。
- 客户端连接池是否认为连接可用。
- 是否超过最大空闲时间或最大生命周期。
- 中间代理是否关闭连接。

## 常见 Header

HTTP/1.1：

```http
Connection: keep-alive
Keep-Alive: timeout=60, max=1000
```

注意：

- `Keep-Alive` Header 不是所有服务都严格遵守。
- 真实空闲关闭策略还受服务端、代理、LB、客户端连接池影响。
- HTTP/2 禁止使用部分连接级 Header。

## Java 后端实践

Apache HttpClient 常见配置：

- `MaxTotal`：总连接数。
- `DefaultMaxPerRoute`：每个路由最大连接数。
- `ConnectionRequestTimeout`：从连接池取连接的超时。
- `ConnectTimeout`：建连超时。
- `SocketTimeout`：读超时。
- idle connection eviction：清理空闲连接。

OkHttp 连接池：

```java
ConnectionPool pool = new ConnectionPool(100, 5, TimeUnit.MINUTES);
OkHttpClient client = new OkHttpClient.Builder()
        .connectionPool(pool)
        .connectTimeout(Duration.ofSeconds(1))
        .readTimeout(Duration.ofSeconds(3))
        .build();
```

原则：

- HTTP Client 通常应复用单例。
- 响应体必须关闭，否则连接不能归还连接池。
- 客户端 idle timeout 要小于下游或 LB 的 idle timeout。

## 命令示例

查看连接复用：

```bash
curl -v --http1.1 https://example.com/a https://example.com/b
```

强制关闭：

```bash
curl -v -H 'Connection: close' https://example.com/
```

查看本机连接：

```bash
ss -antp | grep ':443'
```

抓包看是否反复握手：

```bash
sudo tcpdump -i any host example.com and tcp port 443 -nn
```

## 线上排查

### TIME_WAIT 很多

可能是短连接或连接复用差：

- 每次请求创建新客户端。
- 服务端返回 `Connection: close`。
- 连接池最大空闲时间太短。
- 响应体未关闭导致连接不可复用。
- 代理关闭连接。

### 偶发 connection reset

常见于复用已经被服务端或 LB 关闭的空闲连接。

治理：

- 缩短客户端空闲连接时间。
- 开启连接可用性检测。
- 对幂等请求做有限重试。
- 对齐客户端、服务端、LB idle timeout。

### 连接池耗尽

可能原因：

- 下游慢，连接长时间被占用。
- 响应体未关闭。
- 每个路由连接数太小。
- 请求并发超过连接池容量。

排查：

- 查看连接池指标。
- 查看线程是否等待连接。
- 查看 CLOSE_WAIT。
- 看下游耗时和错误率。

## 常见追问

### HTTP 长连接是不是一个连接并发很多请求

HTTP/1.1 通常不是。HTTP/1.1 一个连接同一时刻一般处理一个请求响应，靠多个连接并发。HTTP/2 才支持同一连接上的多路复用。

### 长连接是否一定更好

不一定。低频请求保持大量空闲连接会浪费资源；长连接也需要处理失效连接、负载均衡不均、连接池耗尽等问题。

### 服务端重启时长连接怎么办

老连接会被关闭或重置。客户端需要处理失败，对幂等请求可重试，连接池要能剔除失效连接。

## 易错点

- 认为 HTTP/1.1 长连接等于 HTTP/2 多路复用。
- 只配置连接池最大连接数，不配置连接获取超时。
- 不关闭响应体，导致连接泄漏。
- 客户端 idle timeout 大于 LB idle timeout，引发复用失效连接。
- 用 `Connection: keep-alive` 判断连接一定会长期保持。

## 自检清单

- 能否说明 HTTP/1.1 默认长连接。
- 能否解释长连接降低哪些成本。
- 能否区分长连接、连接池、HTTP/2 多路复用。
- 能否说出连接复用失败的常见原因。
- 能否给出 TIME_WAIT 和 connection reset 的排查方向。
