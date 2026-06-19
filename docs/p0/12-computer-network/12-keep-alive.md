# keep-alive

## 面试定位

keep-alive 是一个容易混淆的词。面试中可能指 HTTP keep-alive，也可能指 TCP keepalive，还可能指应用层心跳。回答时第一步要澄清语境。

核心结论：

- HTTP keep-alive：复用 TCP 连接承载多个 HTTP 请求。
- TCP keepalive：内核探测空闲 TCP 连接是否仍然存活。
- 应用心跳：业务协议自己定义的保活和健康检测。

## 三种 keep-alive 对比

| 类型 | 所在层 | 目的 | 典型配置 | 常见场景 |
| --- | --- | --- | --- | --- |
| HTTP keep-alive | 应用层 | 连接复用 | 连接池空闲时间、最大连接数 | HTTP/1.1 调用 |
| TCP keepalive | 传输层 | 探测死连接 | keep idle、interval、count | 长连接、网关、RPC |
| 应用心跳 | 应用层 | 协议级保活、业务健康 | ping/pong 周期、超时 | WebSocket、RPC、注册中心 |

## HTTP keep-alive

HTTP/1.1 默认持久连接。客户端和服务端通过连接复用减少握手成本。

常见 Header：

```http
Connection: keep-alive
Keep-Alive: timeout=60, max=1000
```

注意：

- Header 只是协议层表达，不等于连接一定保留到指定时间。
- 代理、LB、服务端、客户端任一方都可能关闭空闲连接。
- HTTP/2 多路复用与 HTTP/1.1 keep-alive 不是同一概念。

## TCP keepalive

TCP keepalive 是内核机制，用于探测长时间空闲连接是否还活着。

典型过程：

```text
连接空闲超过 keepalive time
  -> 发送探测包
  -> 对端响应 ACK，连接继续保持
  -> 多次无响应，内核认为连接失效并关闭
```

Linux 常见参数：

```bash
sysctl net.ipv4.tcp_keepalive_time
sysctl net.ipv4.tcp_keepalive_intvl
sysctl net.ipv4.tcp_keepalive_probes
```

含义：

- `tcp_keepalive_time`：空闲多久开始探测。
- `tcp_keepalive_intvl`：探测间隔。
- `tcp_keepalive_probes`：探测次数。

默认值往往偏长，不适合快速发现业务故障。

## 应用层心跳

应用层心跳由协议或业务自己实现，例如：

```text
client -> PING
server -> PONG
```

优势：

- 可携带业务状态。
- 可由应用控制超时和重连。
- 能穿透某些代理空闲关闭策略。
- 对 WebSocket、长连接 RPC 更直接。

风险：

- 心跳过频会增加流量和 CPU。
- 心跳线程阻塞会误判。
- 只证明心跳链路正常，不代表业务处理正常。

## keep-alive 和 idle timeout

线上最常见问题是空闲连接超时不一致。

例子：

```text
客户端连接池保留空闲连接 5 分钟
LB 空闲连接 60 秒关闭
客户端 2 分钟后复用该连接
可能收到 RST 或读写失败
```

治理：

- 客户端连接池 idle timeout 小于 LB 和服务端 idle timeout。
- 对失效连接做检测。
- 对幂等请求做一次安全重试。
- 监控连接复用失败和 reset。

## Java 配置关注点

HTTP Client：

- 连接池最大连接数。
- 空闲连接存活时间。
- 连接最大生命周期。
- 是否启用 TCP keepalive。
- 连接可用性检测。

Netty：

```java
bootstrap.option(ChannelOption.SO_KEEPALIVE, true);
```

但 `SO_KEEPALIVE` 只打开 TCP keepalive，默认探测时间仍取系统参数。业务上需要快速感知断连时，通常还要应用层心跳。

## 命令示例

查看系统 TCP keepalive 参数：

```bash
sysctl net.ipv4.tcp_keepalive_time
sysctl net.ipv4.tcp_keepalive_intvl
sysctl net.ipv4.tcp_keepalive_probes
```

查看 Socket 是否开启 keepalive，通常需要结合进程和工具：

```bash
ss -antoi
```

HTTP 查看连接复用：

```bash
curl -v --http1.1 https://example.com/a https://example.com/b
```

抓空闲连接被关闭：

```bash
sudo tcpdump -i any host 203.0.113.20 and tcp port 443 -nn -vv
```

## 线上排查

### 长连接过一段时间后首次请求失败

高概率是空闲连接被中间设备关闭，客户端连接池未及时感知。

排查：

- 看失败间隔是否接近某个 idle timeout。
- 抓包看是否收到 RST。
- 查 LB、Nginx、网关、服务端 keepalive timeout。
- 查客户端连接池 idle 配置。

### TCP keepalive 开了但仍然发现故障慢

原因可能是系统默认探测时间很长。例如默认 7200 秒才开始探测。

解决：

- 调整系统参数。
- 对特定 Socket 配置 keepalive 参数，取决于语言和库支持。
- 使用应用层心跳。

### 心跳正常但业务请求失败

可能原因：

- 心跳路径轻量，业务路径依赖数据库或下游。
- 网关只放行心跳接口。
- 线程池隔离导致心跳线程正常、业务线程池满。

不要把心跳成功等同于业务健康。

## 常见追问

### HTTP keep-alive 和 TCP keepalive 有什么区别

HTTP keep-alive 是复用连接，减少建连开销；TCP keepalive 是探测空闲连接是否存活。

### 开了 TCP keepalive 是否就不需要应用心跳

不一定。TCP keepalive 默认慢，且只能证明连接层可达。应用心跳可表达协议级状态和更短故障发现时间。

### keep-alive 会不会导致负载不均

会。长连接复用可能让流量固定在少数后端连接上。HTTP/2 单连接多路复用时更明显，需要结合 LB 策略和连接生命周期治理。

## 易错点

- 不区分 HTTP keep-alive 和 TCP keepalive。
- 认为 keep-alive 可以防止任何断连。
- 忽略 LB idle timeout。
- 心跳过频导致系统额外负载。
- 只开 `SO_KEEPALIVE`，却不知道系统默认探测时间很长。

## 自检清单

- 能否解释三种 keep-alive。
- 能否说明 HTTP keep-alive 与连接池的关系。
- 能否说明 TCP keepalive 的触发条件。
- 能否排查空闲连接复用后 reset。
- 能否设计合理的心跳周期和超时。
