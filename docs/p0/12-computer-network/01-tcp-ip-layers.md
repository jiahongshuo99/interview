# TCP/IP 分层模型

## 面试定位

TCP/IP 分层模型是网络题的入口。面试官通常不是只考“有几层”，而是看你能不能把一次请求从业务代码、Socket、TCP、IP、网卡到对端服务的路径讲清楚，并且能在排查超时、连接拒绝、DNS 异常、TLS 失败时定位问题落在哪一层。

回答时建议抓住三点：

- 分层是为了隔离职责，每层只关心自己的协议语义。
- Java 后端多数代码运行在应用层，但故障可能发生在任意层。
- 排查要按“域名解析、路由连通、端口监听、TCP 建连、TLS、HTTP/RPC 协议、应用处理”逐层收敛。

## 核心概念

TCP/IP 常见四层模型：

| 层次 | 典型协议 | 主要职责 | 后端常见问题 |
| --- | --- | --- | --- |
| 应用层 | HTTP、HTTPS、DNS、RPC、MQ 协议 | 业务语义、报文格式、状态码、鉴权 | 4xx/5xx、序列化失败、网关限流 |
| 传输层 | TCP、UDP | 端到端传输、端口、可靠性、流控、拥塞控制 | 连接超时、重置、TIME_WAIT、丢包重传 |
| 网络层 | IP、ICMP、ARP 邻近链路配合 | 跨网段寻址和路由 | 路由不通、NAT、跨机房访问异常 |
| 网络接口层 | Ethernet、Wi-Fi、物理网卡 | 帧传输、MAC、MTU | 网卡丢包、MTU 黑洞、链路抖动 |

OSI 七层模型更细：

```text
应用层
表示层
会话层
传输层
网络层
数据链路层
物理层
```

面试中通常可以说明：实际工程排查更常用 TCP/IP 四层或五层模型，OSI 七层用于表达职责边界。

## 封装与解封装

一次 HTTP 请求在发送端会逐层封装：

```text
HTTP 请求报文
  -> TCP 段：加源端口、目的端口、序号、确认号
  -> IP 包：加源 IP、目的 IP、TTL
  -> Ethernet 帧：加源 MAC、目的 MAC、FCS
  -> 物理链路传输
```

接收端反向解封装：

```text
网卡收到帧
  -> 校验并交给 IP
  -> 根据目标 IP 和协议号交给 TCP
  -> 根据端口找到 Socket
  -> 应用读取 HTTP/RPC 数据
```

关键点：

- IP 负责把包送到目标主机，不保证可靠。
- TCP 负责端到端可靠字节流，不理解 HTTP 语义。
- HTTP 负责请求方法、路径、Header、Body、状态码，不关心路由细节。
- DNS 是应用层协议，但它服务于网络访问的第一步。

## 一次 Java HTTP 调用经过哪些组件

以 Java 服务调用下游 HTTPS 接口为例：

```text
业务代码
  -> HTTP Client 连接池
  -> DNS 解析
  -> Socket connect
  -> TCP 三次握手
  -> TLS 握手和证书校验
  -> 写 HTTP 请求
  -> 服务端处理
  -> 读 HTTP 响应
  -> 连接复用或关闭
```

每一步都有独立超时和故障表现：

- DNS 慢：表现为请求还没开始建连就耗时很长。
- connect 慢：常见为跨网络不通、防火墙丢包、SYN 重试。
- TLS 慢或失败：证书不可信、SNI 不匹配、协议版本不兼容。
- read 慢：对端处理慢、响应大、网络抖动、线程池阻塞。
- 连接池等待慢：连接池太小、连接泄漏、下游慢导致占满。

## 常用命令

查看本机地址和路由：

```bash
ip addr
ip route
ifconfig
netstat -rn
```

查看端口监听和连接状态：

```bash
ss -lntp
ss -antp
lsof -iTCP -sTCP:LISTEN
```

验证连通性：

```bash
ping example.com
traceroute example.com
nc -vz example.com 443
curl -v https://example.com/
```

抓包：

```bash
sudo tcpdump -i any host 203.0.113.10 and port 443 -nn -vv
sudo tcpdump -i any tcp port 8080 -w /tmp/http.pcap
```

DNS 排查：

```bash
dig example.com
nslookup example.com
```

## 线上排查思路

当请求失败时，按层定位：

1. 应用层：看 HTTP 状态码、RPC 错误码、业务日志、网关日志。
2. TLS 层：看证书、SNI、协议版本、握手失败原因。
3. 传输层：看 TCP 状态、重传、RST、TIME_WAIT、CLOSE_WAIT。
4. 网络层：看路由、ACL、安全组、NAT、跨机房路径。
5. 链路层：看网卡丢包、MTU、宿主机网络、容器网络。

常见判断：

- `connection refused`：通常目标主机可达，但端口无人监听或被主动拒绝。
- `connection timed out`：通常 SYN 没有收到回应，可能网络不通、防火墙丢弃、路由异常。
- `connection reset`：TCP 已建立后被 RST，可能应用主动关闭、代理重置、协议不匹配。
- HTTP 5xx：TCP 和 HTTP 通常已经通了，重点看服务端和网关。

## 常见追问

### TCP/IP 四层和 OSI 七层怎么对应

应用层大致对应 OSI 的应用层、表示层、会话层；传输层对应传输层；网络层对应网络层；网络接口层对应数据链路层和物理层。

### IP 层可靠么

不可靠。IP 只尽力投递，可能丢包、乱序、重复。TCP 在 IP 之上实现可靠传输。

### TCP 是消息协议么

不是。TCP 是面向连接的可靠字节流，没有消息边界。HTTP、RPC 协议需要自己定义报文边界，例如 Content-Length、chunked、固定头长度字段等。

### 为什么网络排查不能只看应用日志

应用日志通常只能看到请求结果，看不到 DNS、SYN 重试、TLS 握手、TCP 重传、RST 包等底层细节。复杂故障需要结合指标、Socket 状态和抓包。

## 易错点

- 把 TCP/IP 四层和 OSI 七层混着背，但说不清职责。
- 认为 HTTP 长连接等于 TCP 永远不断。
- 认为 ping 通就代表端口可访问。ping 用 ICMP，业务端口可能被防火墙拦截。
- 认为 TCP 可靠等于请求一定成功。TCP 只保证字节流传输，应用处理仍可能失败。
- 忽略 DNS、TLS、连接池这些应用和传输层之间的关键环节。

## 自检清单

- 能否画出一次 HTTP 请求从代码到网卡的完整路径。
- 能否区分 DNS、TCP、TLS、HTTP 各自失败的表现。
- 能否说明 IP 不可靠、TCP 可靠、HTTP 有业务语义。
- 能否用 `ss`、`curl -v`、`tcpdump` 做最小排查。
- 能否把 `refused`、`timeout`、`reset` 对应到不同层次。
