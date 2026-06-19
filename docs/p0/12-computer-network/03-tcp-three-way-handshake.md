# TCP 三次握手

## 面试定位

TCP 三次握手是网络基础中的必考题。面试不仅会问流程，还会追问为什么是三次、两次行不行、初始序列号有什么用、SYN 队列和 accept 队列区别、SYN flood 怎么防。

回答时要把“建立连接”讲成双方确认收发能力和同步初始序列号的过程。

## 握手流程

```text
Client                                      Server
  | ---- SYN, seq = x --------------------> |
  |                                         | 进入 SYN_RECV
  | <--- SYN + ACK, seq = y, ack = x + 1 -- |
  |                                         |
  | ---- ACK, ack = y + 1 ----------------> |
  |                                         | 进入 ESTABLISHED
进入 ESTABLISHED
```

三次握手后，双方都确认：

- 客户端发送能力正常。
- 客户端接收能力正常。
- 服务端发送能力正常。
- 服务端接收能力正常。
- 双方初始序列号已经同步。

## 状态变化

客户端：

```text
CLOSED -> SYN_SENT -> ESTABLISHED
```

服务端：

```text
LISTEN -> SYN_RECV -> ESTABLISHED
```

服务端应用调用 `listen()` 后进入监听状态，调用 `accept()` 从已完成连接队列中取出连接。

## 为什么需要三次

两次握手只能让客户端确认服务端收发能力正常，但服务端无法确认客户端是否收到了自己的 SYN+ACK。

三次握手的关键是第三个 ACK：

- 服务端收到第三个 ACK 后，确认客户端能收到服务端数据。
- 可以避免历史重复 SYN 报文导致服务端误建立连接。
- 双方初始序列号都能被确认。

简化理解：

```text
第一次：客户端告诉服务端，我想连你，我的初始序列号是 x。
第二次：服务端告诉客户端，我收到了，你也要知道我的初始序列号 y。
第三次：客户端告诉服务端，我收到了你的 y。
```

## 初始序列号 ISN

TCP 是字节流协议，每个字节都有序号。握手中双方交换初始序列号：

- 客户端选择 `seq = x`。
- 服务端选择 `seq = y`。
- SYN 本身会消耗一个序号。

ISN 作用：

- 支撑可靠传输、确认和重传。
- 降低旧连接残留报文被新连接误收的概率。
- 配合四元组区分连接。

四元组：

```text
源 IP、源端口、目的 IP、目的端口
```

## 半连接队列和全连接队列

服务端内核维护两个队列：

| 队列 | 状态 | 含义 |
| --- | --- | --- |
| SYN 队列 | SYN_RECV | 收到 SYN，发出 SYN+ACK，等待最终 ACK |
| accept 队列 | ESTABLISHED | 三次握手完成，等待应用 `accept()` |

相关参数：

- `net.ipv4.tcp_max_syn_backlog`：半连接队列相关。
- `net.core.somaxconn`：listen backlog 上限。
- 应用传入的 backlog：例如 Netty、Tomcat 监听 backlog。

如果 accept 队列满，可能出现连接建立慢、丢包、客户端超时等现象。

## SYN flood

SYN flood 是攻击者发送大量 SYN 但不完成第三次握手，消耗服务端半连接队列。

常见防护：

- 开启 SYN cookies。
- 调大半连接队列。
- 限流和防火墙策略。
- LB 或网关层清洗。

查看参数：

```bash
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv4.tcp_max_syn_backlog
sysctl net.core.somaxconn
```

## 抓包示例

抓三次握手：

```bash
sudo tcpdump -i any host 203.0.113.20 and tcp port 8080 -nn -vv
```

典型输出中会看到：

```text
Flags [S]
Flags [S.]
Flags [.]
```

含义：

- `[S]`：SYN。
- `[S.]`：SYN + ACK。
- `[.]`：ACK。

用 `curl` 触发连接：

```bash
curl -v http://203.0.113.20:8080/health
```

查看 SYN 状态：

```bash
ss -ant state syn-sent
ss -ant state syn-recv
```

## 线上排查

### connect timeout

客户端长时间停在建连阶段，常见原因：

- 目标 IP 不可达。
- 防火墙丢弃 SYN。
- 服务端半连接队列满。
- 跨机房网络抖动。
- SYN 包或 SYN+ACK 包丢失。

排查：

```bash
nc -vz 203.0.113.20 8080
ss -ant state syn-sent
tcpdump -i any host 203.0.113.20 and port 8080 -nn
```

抓包判断：

- 只有 SYN，没有 SYN+ACK：网络或服务端入口问题。
- 有 SYN+ACK，客户端没发 ACK：客户端侧或回包路径问题。
- 三次握手完成后应用无响应：问题不在建连。

### connection refused

通常说明目标主机返回 RST：

- 端口没有监听。
- 服务刚重启。
- 本地防火墙主动拒绝。
- 访问了错误端口。

排查：

```bash
ss -lntp | grep 8080
nc -vz 203.0.113.20 8080
```

## 常见追问

### 第三次握手可以带数据吗

理论上 TCP 第三次 ACK 可以携带数据，部分场景还涉及 TCP Fast Open。但普通应用不需要依赖这个细节，主流程仍按三次握手理解。

### 服务端收到 SYN 后会立即创建 Socket 吗

内核会为半连接维护状态。三次握手完成后连接进入 accept 队列，应用调用 `accept()` 才拿到已连接 Socket。

### backlog 是不是越大越好

不是。backlog 太小容易溢出，太大也不能解决应用 accept 慢、线程池满、下游阻塞等根因。需要结合连接速率、服务处理能力和指标调整。

## 易错点

- 只背 SYN、SYN+ACK、ACK，不会解释为什么要三次。
- 把半连接队列和 accept 队列混为一谈。
- 认为 connect 成功就代表 HTTP 服务一定正常。
- 遇到 connect timeout 只查应用日志，不抓包看 SYN。
- 忽略安全组、防火墙、容器网络、LB 对握手的影响。

## 自检清单

- 能否画出三次握手报文和状态变化。
- 能否解释第三次 ACK 的必要性。
- 能否说明 SYN 队列和 accept 队列。
- 能否用 `tcpdump` 识别 `[S]`、`[S.]`、`[.]`。
- 能否区分 connect timeout 和 connection refused。
