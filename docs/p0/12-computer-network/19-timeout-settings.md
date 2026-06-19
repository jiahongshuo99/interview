# 超时设置：连接超时、读超时、写超时、整体超时

## 面试定位

超时设置是后端稳定性面试的重点。面试官常问：connect timeout、read timeout、write timeout、call timeout 分别是什么，为什么不能只配一个超时，超时和重试怎么配，网关超时和客户端超时谁应该更短。

核心回答：

超时是故障隔离和资源保护手段。一次请求要覆盖获取连接、建连、TLS、写请求、读响应和整体耗时，避免线程和连接无限等待。

## 常见超时类型

| 超时 | 含义 | 典型故障 |
| --- | --- | --- |
| DNS 超时 | 域名解析等待时间 | DNS 服务器慢或不可达 |
| 连接池获取超时 | 从连接池拿连接的等待时间 | 连接池满、连接泄漏 |
| 连接超时 | TCP connect 超时 | 网络不通、SYN 丢包、防火墙丢弃 |
| TLS 握手超时 | TLS 协商超时 | 证书、协议、网络慢 |
| 写超时 | 写请求数据超时 | 对端不读、窗口变小、大请求体 |
| 读超时 | 等待响应数据超时 | 服务端慢、网络重传、对端不回 |
| 整体超时 | 单次调用总耗时上限 | 防止各阶段叠加过长 |

不同库命名不同，但思想类似。

## 连接超时

连接超时指 TCP 建连阶段超时。

常见异常：

```text
connect timed out
```

可能原因：

- 目标 IP 不可达。
- 安全组或防火墙丢弃 SYN。
- 服务端半连接队列满。
- 路由异常。
- 跨机房链路故障。

连接超时不包含服务端业务处理时间。

## 读超时

读超时指连接已建立，请求可能已发出，但在指定时间内没有读到响应数据。

常见异常：

```text
read timed out
```

可能原因：

- 服务端业务处理慢。
- 下游依赖慢。
- 服务端线程池满。
- 网络丢包重传。
- 响应体很大。

读超时发生时，服务端可能已经处理成功。因此读超时后的重试必须考虑幂等。

## 写超时

写超时指发送请求数据时阻塞超过阈值。

常见原因：

- 请求体很大。
- 对端应用不读。
- 对端接收窗口变小或零窗口。
- 网络慢。
- TLS 写入阻塞。

写超时在上传文件、批量请求、长连接协议中更常见。

## 整体超时

整体超时限制一次调用从开始到结束的总时间。

如果只配置分阶段超时，可能出现：

```text
获取连接 1s
连接 1s
TLS 1s
写 2s
读 3s
重试一次又重复
总耗时远超上游预算
```

整体超时用于控制调用预算，避免重试和多阶段等待叠加。

## 超时预算

超时配置要从上游 SLA 倒推。

例如接口总预算 500ms：

```text
网关超时：450ms
服务 A 处理预算：350ms
服务 A 调 B：100ms
服务 A 调 C：150ms
预留序列化、排队、日志：50ms
```

原则：

- 上游超时应大于下游单次调用超时加必要处理时间。
- 客户端超时通常应小于网关超时，避免网关先断而客户端继续处理。
- 重试总耗时必须纳入整体预算。
- 慢请求要快速失败或降级，避免占满线程和连接。

## Java HTTP Client 示例

OkHttp：

```java
OkHttpClient client = new OkHttpClient.Builder()
        .connectTimeout(Duration.ofMillis(500))
        .readTimeout(Duration.ofSeconds(2))
        .writeTimeout(Duration.ofSeconds(2))
        .callTimeout(Duration.ofSeconds(3))
        .build();
```

Apache HttpClient 常见：

- connectionRequestTimeout：连接池获取超时。
- connectTimeout：建连超时。
- socketTimeout：读超时。

注意库版本差异，配置前要确认具体语义。

## 网关和服务超时

一次请求可能经过：

```text
客户端
  -> 网关
  -> 服务 A
  -> 服务 B
  -> 数据库
```

每层都有超时。常见问题：

- 网关 1s 超时，服务内部还在跑 5s。
- 客户端 3s 超时，网关 1s 已返回 504。
- 下游 DB 超时 10s，服务线程被拖死。

治理：

- 统一超时预算。
- 超时后取消下游请求，若库支持。
- 服务端感知客户端断开并尽快停止无用工作。
- 长任务改异步。

## 命令示例

curl 分阶段耗时：

```bash
curl -o /dev/null -s -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} start=%{time_starttransfer} total=%{time_total}\n' https://example.com/
```

设置 curl 超时：

```bash
curl --connect-timeout 1 --max-time 3 https://example.com/
```

测试端口：

```bash
nc -vz -w 2 203.0.113.20 8080
```

抓包区分 connect 和 read：

```bash
sudo tcpdump -i any host 203.0.113.20 and tcp port 8080 -nn
```

## 线上排查

### connect timeout

看：

- DNS 结果。
- 路由和安全组。
- 服务端端口监听。
- SYN/SYN+ACK 抓包。
- 半连接队列。

### read timeout

看：

- 服务端是否收到请求。
- 服务端处理耗时。
- 下游调用耗时。
- 网络重传。
- 响应是否过大。

### write timeout

看：

- 请求体大小。
- 对端是否读取。
- TCP 窗口是否为零。
- 上传链路是否限速。

### connection pool timeout

看：

- 连接池 leased、idle、pending。
- 下游慢。
- 连接泄漏。
- per route 限制。

## 常见追问

### 超时设得越短越好吗

不是。太短会误伤正常慢请求，放大重试和错误率。太长会占用线程、连接和队列，拖垮系统。要结合 SLA、下游耗时分布和重试策略。

### read timeout 后服务端会停止处理吗

不一定。客户端超时只是客户端放弃等待，服务端可能仍在处理并提交结果。

### 为什么要整体超时

防止多个阶段和多次重试叠加，突破调用方预算。

### 超时和熔断是什么关系

超时是单次调用等待上限，熔断是根据一段时间内错误率、慢调用比例等保护系统，避免持续打向故障下游。

## 易错点

- 只配 read timeout，不配连接池获取超时和 connect timeout。
- 不区分 connect timeout 和 read timeout。
- 重试没有整体超时，导致放大耗时。
- 上游超时短于下游，出现无意义后台处理。
- 超时后对非幂等操作盲目重试。

## 自检清单

- 能否解释连接池获取、connect、read、write、整体超时。
- 能否根据异常判断卡在哪个阶段。
- 能否设计一条调用链的超时预算。
- 能否说明超时和重试、幂等、熔断的关系。
- 能否用 curl 输出分阶段耗时。
