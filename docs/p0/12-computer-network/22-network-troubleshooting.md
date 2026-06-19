# 连接拒绝、连接重置、请求超时、丢包排查

## 面试定位

网络排查题最能体现工程经验。面试官常给现象：连接拒绝、连接重置、请求超时、偶发丢包、P99 抖动，让你说排查路径。回答要避免只说“看日志”，而要按 DNS、路由、TCP、TLS、HTTP、应用和中间件逐层收敛。

核心方法：

先判断失败发生在哪个阶段，再用最小命令验证。不要直接猜代码问题，也不要直接怪网络。

## 快速分类

| 现象 | 常见错误 | 初步含义 |
| --- | --- | --- |
| 连接拒绝 | connection refused | 目标可达，但端口无人监听或主动拒绝 |
| 连接超时 | connect timed out | SYN 无响应，网络或入口路径不通 |
| 连接重置 | connection reset | 已有连接被 RST 异常关闭 |
| 读超时 | read timed out | 已建连，但响应迟迟没读到 |
| 写超时 | write timed out | 请求数据写不出去或对端读慢 |
| 丢包重传 | retransmission | 链路丢包、拥塞或接收端处理慢 |
| 证书错误 | PKIX、hostname mismatch | TLS 证书链、信任、域名问题 |
| 502/503/504 | 网关错误 | 网关到上游连接、可用性或超时问题 |

## 标准排查顺序

```text
确认目标
  -> DNS 解析
  -> 路由和连通性
  -> 端口监听
  -> TCP 建连
  -> TLS 握手
  -> HTTP/RPC 协议
  -> 应用处理
  -> 线程池、连接池、下游依赖
```

每一步都要用证据推进。

## 基础命令

确认域名：

```bash
dig example.com
nslookup example.com
cat /etc/resolv.conf
```

确认端口：

```bash
nc -vz -w 2 example.com 443
telnet example.com 443
```

查看 HTTP 详情：

```bash
curl -v https://example.com/path
curl -o /dev/null -s -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} start=%{time_starttransfer} total=%{time_total}\n' https://example.com/path
```

查看连接：

```bash
ss -antp
ss -ti dst 203.0.113.20
lsof -iTCP -p <pid>
```

抓包：

```bash
sudo tcpdump -i any host 203.0.113.20 and tcp port 443 -nn -vv
sudo tcpdump -i any host 203.0.113.20 and tcp port 443 -w /tmp/net.pcap
```

## 连接拒绝

典型表现：

```text
Connection refused
```

含义：

客户端发 SYN，目标返回 RST。通常表示主机可达，但目标端口没有服务监听，或防火墙主动拒绝。

排查：

```bash
nc -vz 203.0.113.20 8080
ss -lntp | grep 8080
curl -v http://203.0.113.20:8080/health
```

常见原因：

- 服务未启动。
- 监听端口配置错误。
- 服务只监听 `127.0.0.1`。
- 容器端口未映射。
- 安全策略主动 reject。
- 发布重启窗口。

区分：

- refused：对端明确拒绝，通常很快失败。
- timeout：没有响应，等待超时。

## 连接超时

典型表现：

```text
connect timed out
```

含义：

客户端发 SYN 后没有收到 SYN+ACK 或 RST。

排查：

```bash
nc -vz -w 2 203.0.113.20 8080
traceroute 203.0.113.20
sudo tcpdump -i any host 203.0.113.20 and tcp port 8080 -nn
```

抓包判断：

- 只有 SYN 重发：对端无响应或回包路径不通。
- 有 SYN+ACK 但客户端没 ACK：本机或回程路径异常。
- 完成握手后才慢：不是 connect timeout 阶段问题。

常见原因：

- 安全组、防火墙丢弃。
- 路由不通。
- 目标 IP 错误。
- 服务端半连接队列溢出。
- 跨机房链路故障。

## 连接重置

典型表现：

```text
Connection reset
Connection reset by peer
```

含义：

TCP 连接被 RST 异常终止。

常见原因：

- 服务端进程关闭连接。
- 复用已被对端关闭的空闲连接。
- LB 或网关 idle timeout 关闭连接。
- 协议不匹配，例如把 HTTP 发到 HTTPS 端口。
- 应用未读完数据就 close。
- 服务重启。

排查：

```bash
sudo tcpdump -i any host 203.0.113.20 and tcp port 443 -nn -vv
ss -antp | grep 203.0.113.20
```

看 RST 方向：

- 对端发 RST：对端或中间设备主动重置。
- 本端发 RST：本端应用或内核主动重置。

治理：

- 对齐 idle timeout。
- 失效连接检测。
- 响应体正确关闭。
- 协议和端口配置正确。
- 幂等请求有限重试。

## 请求超时

请求超时要先区分：

- 连接超时。
- 连接池获取超时。
- 写超时。
- 读超时。
- 整体超时。

读超时排查：

```bash
curl -v --max-time 3 https://example.com/path
```

服务端侧确认：

- 是否收到请求。
- 入口日志时间。
- 业务处理耗时。
- 下游调用耗时。
- 线程池、连接池、DB 池。
- GC 暂停。

如果客户端超时但服务端最终成功，要考虑幂等和补偿。

## 丢包和重传

表现：

- P99 抖动。
- 偶发 read timeout。
- 大文件传输慢。
- `ss -ti` 看到 retrans。
- 抓包看到 duplicate ACK、retransmission。

命令：

```bash
ss -ti dst 203.0.113.20
ip -s link
ethtool -S eth0
```

抓包：

```bash
sudo tcpdump -i any host 203.0.113.20 and tcp port 8080 -w /tmp/loss.pcap
```

Wireshark 过滤：

```text
tcp.analysis.retransmission
tcp.analysis.fast_retransmission
tcp.analysis.duplicate_ack
```

常见原因：

- 网络拥塞。
- 网卡或宿主机丢包。
- MTU 不一致。
- 容器网络异常。
- 跨机房链路质量差。
- 接收端 CPU 高或应用读慢。

## DNS 问题

表现：

- 请求偶发卡在解析阶段。
- 不同机器访问不同 IP。
- 域名切流后仍访问旧地址。

排查：

```bash
dig example.com
dig @nameserver example.com
curl -v --resolve example.com:443:1.2.3.4 https://example.com/
```

关注：

- `/etc/resolv.conf`。
- JVM DNS 缓存。
- 负缓存。
- CoreDNS。
- 连接池旧连接。

## TLS 问题

表现：

- 证书过期。
- PKIX path building failed。
- hostname mismatch。
- handshake_failure。

排查：

```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
curl -vk https://example.com/
```

关注：

- 证书链。
- SAN。
- SNI。
- TLS 版本。
- 密码套件。
- JDK truststore。

## 网关问题

### 502

网关访问上游失败或上游响应非法。

查：

- upstream 端口。
- RST。
- 协议。
- TLS。
- 上游是否重启。

### 503

服务不可用。

查：

- 健康实例数。
- 限流熔断。
- 发布摘除。
- 网关过载。

### 504

网关等待上游超时。

查：

- upstream 响应时间。
- 服务线程池和连接池。
- 下游依赖。
- 网关 timeout。

## Java 侧排查

线程：

```bash
jstack <pid> > /tmp/jstack.txt
```

GC：

```bash
jstat -gcutil <pid> 1000
```

连接：

```bash
lsof -iTCP -p <pid>
ss -antp | grep <pid>
```

重点看：

- 线程是否卡在 HTTP Client。
- 连接池是否 pending。
- CLOSE_WAIT 是否多。
- 是否大量创建短连接。
- GC 是否导致暂停。
- Netty EventLoop 是否阻塞。

## 排查案例模板

描述问题时建议包含：

```text
时间范围：
调用方：
被调方：
域名/IP/端口：
错误类型：
错误率和 QPS：
P50/P95/P99：
是否发布或变更：
是否集中在机房/机器/实例：
curl/nc/dig 结果：
ss/tcpdump 证据：
应用日志 traceId：
```

## 常见追问

### ping 通为什么业务还不通

ping 使用 ICMP，只能说明部分网络可达。业务还依赖 DNS、TCP 端口、TLS、HTTP 路由、应用状态和权限。

### telnet 通为什么 HTTP 失败

telnet 或 nc 只验证 TCP 端口可连。HTTP 失败可能来自 TLS、Host、路径、Header、鉴权、网关路由或应用异常。

### 客户端超时但服务端没有日志

可能请求没到服务端：

- DNS 错。
- 网关没转发。
- TCP 没建连。
- TLS 握手失败。
- 请求在连接池等待。

也可能服务端入口日志采集漏了，需要抓包确认。

## 易错点

- 不区分 refused、timeout、reset。
- 只看应用日志，不看 TCP 状态和抓包。
- ping 通就认为网络没问题。
- 把所有 504 都归因于网络。
- 忽略连接池获取超时。
- 忽略客户端超时后服务端可能仍处理成功。

## 自检清单

- 能否按阶段定位网络失败。
- 能否区分连接拒绝、连接超时、连接重置、读超时。
- 能否用 `dig`、`nc`、`curl -v`、`ss`、`tcpdump` 做最小验证。
- 能否根据 RST 方向判断谁重置连接。
- 能否把网关 502、503、504 映射到排查路径。
