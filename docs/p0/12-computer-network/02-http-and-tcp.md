# HTTP 与 TCP 的关系

## 面试定位

HTTP 和 TCP 的关系是网络面试高频题。面试官常追问：HTTP 是不是基于 TCP、HTTP 长连接和 TCP 长连接是不是一回事、HTTP/2 多路复用为什么还会有队头阻塞、为什么 HTTP/3 改用 QUIC。

回答时要明确：

- HTTP 是应用层协议，描述请求和响应语义。
- TCP 是传输层协议，提供面向连接的可靠字节流。
- 常见 HTTP/1.1、HTTP/2 运行在 TCP 上，HTTPS 是 HTTP over TLS over TCP。
- HTTP/3 运行在 QUIC 上，QUIC 基于 UDP 实现可靠传输和多路复用。

## 核心概念

HTTP 关心：

- 请求方法：GET、POST、PUT、DELETE。
- URI、Header、Body。
- 状态码。
- 缓存、Cookie、内容协商。
- 长连接、代理、网关语义。

TCP 关心：

- 连接建立和关闭。
- 端口。
- 字节序号和确认。
- 重传。
- 滑动窗口。
- 流量控制。
- 拥塞控制。

二者关系：

```text
HTTP 报文
  -> TLS 记录，HTTPS 场景
  -> TCP 字节流
  -> IP 包
  -> 链路层帧
```

## 请求生命周期

一次 HTTPS 请求简化流程：

```text
DNS 解析域名
  -> TCP connect
  -> TLS handshake
  -> 写 HTTP request
  -> 服务端读取请求并处理
  -> 服务端写 HTTP response
  -> 客户端读取响应
  -> 连接复用或关闭
```

如果连接池中已有可复用连接，则可以跳过 DNS、TCP 握手和 TLS 握手，直接写请求。

## HTTP 报文如何放进 TCP

HTTP/1.1 请求示例：

```http
GET /api/user?id=1 HTTP/1.1
Host: example.com
Connection: keep-alive

```

TCP 不知道这是一个 HTTP 请求，它只看到一串字节。应用层需要通过协议规则判断报文边界：

- 请求行和 Header 以空行结束。
- Body 长度可由 `Content-Length` 指定。
- 也可以使用 `Transfer-Encoding: chunked` 分块。
- 响应也遵循类似规则。

因此 TCP 粘包、拆包不是 TCP 错误，而是字节流协议的自然结果。应用协议必须自己处理边界。

## HTTP/1.1、HTTP/2、HTTP/3 对比

| 版本 | 传输基础 | 主要特性 | 典型问题 |
| --- | --- | --- | --- |
| HTTP/1.0 | TCP | 默认短连接 | 每次请求都要建连，开销大 |
| HTTP/1.1 | TCP | 默认长连接、管线化、chunked | 同一连接内响应有序，管线化实践少 |
| HTTP/2 | TCP | 二进制分帧、多路复用、Header 压缩 | TCP 层丢包会影响同连接所有流 |
| HTTP/3 | QUIC/UDP | 用户态拥塞控制、连接迁移、流级别多路复用 | 运维和网络设备支持要求更高 |

HTTP/2 解决了 HTTP/1.1 应用层队头阻塞，但仍受 TCP 层队头阻塞影响：同一个 TCP 连接丢了一个包，后续字节即使到达也不能交给上层。

## Java 后端视角

Java 应用里常见组件：

- Tomcat、Jetty、Undertow：服务端 HTTP 协议解析和 Socket 管理。
- OkHttp、Apache HttpClient、JDK HttpClient：客户端连接池和请求发送。
- Netty：异步网络框架，常用于网关、RPC、长连接服务。
- Nginx、Envoy、API Gateway：反向代理、TLS 终止、负载均衡。

HTTP Client 连接池通常管理的是 TCP 连接，核心参数包括：

- 最大连接数。
- 每个路由最大连接数。
- 连接空闲存活时间。
- 连接获取超时。
- 连接建立超时。
- 读写超时。

## 抓包和命令示例

查看 HTTP 过程：

```bash
curl -v http://example.com/
curl -vk https://example.com/
```

查看连接复用：

```bash
curl -v --http1.1 https://example.com/a https://example.com/b
```

抓 TCP 连接：

```bash
sudo tcpdump -i any host example.com and tcp port 443 -nn
```

查看连接状态：

```bash
ss -antp | grep ':443'
```

查看 TLS 和 HTTP 版本：

```bash
curl -v --http2 https://example.com/
openssl s_client -connect example.com:443 -servername example.com
```

## 线上排查

### HTTP 失败但 TCP 已通

表现：

- 能建立连接。
- 能完成 TLS 握手。
- 返回 4xx 或 5xx。

重点排查：

- 网关路由。
- 鉴权。
- Header 是否缺失。
- 请求体格式。
- 服务端异常。

### TCP 失败导致 HTTP 失败

表现：

- `connect timed out`
- `connection refused`
- `connection reset`
- 大量重传。

重点排查：

- 目标地址和端口。
- 监听状态。
- 安全组、防火墙、ACL。
- 连接池是否耗尽。
- 下游是否主动关闭空闲连接。

### 连接复用导致偶发失败

常见原因：

- 客户端复用了已经被服务端关闭的空闲连接。
- 代理或 LB 的 idle timeout 小于客户端连接池空闲时间。
- 服务端滚动重启时老连接被关闭。

治理方式：

- 设置连接最大空闲时间小于下游或 LB 的 idle timeout。
- 开启连接可用性检测，但注意检测开销。
- 对可重试且幂等请求做有限重试。

## 常见追问

### HTTPS 和 HTTP、TCP 的关系

HTTPS 不是新的应用语义，通常是 HTTP over TLS over TCP。TLS 提供加密、身份认证和完整性保护。

### HTTP 长连接是不是 TCP 长连接

HTTP 长连接指多个 HTTP 请求复用同一个 TCP 连接。底层确实是同一个 TCP 连接保持打开，但 HTTP 长连接强调应用层复用策略。

### TCP 粘包会影响 HTTP 吗

HTTP/1.1 有明确的报文边界规则，成熟 HTTP 解析器会处理 TCP 拆包和粘包。自定义 RPC 协议如果没有长度字段或边界标识，就容易出问题。

## 易错点

- 说 HTTP 负责可靠传输。可靠传输主要由 TCP 提供。
- 说 TCP 传输的是“一个个 HTTP 请求”。TCP 传输的是字节流。
- 认为 HTTP/2 多路复用后底层丢包就互不影响。HTTP/2 over TCP 仍受 TCP 队头阻塞影响。
- 只配置 read timeout，不配置 connect timeout 和连接池获取超时。
- 忽略代理、网关、LB 也可能主动关闭连接。

## 自检清单

- 能否说明 HTTP、TLS、TCP 的层次关系。
- 能否解释 TCP 字节流和 HTTP 报文边界。
- 能否区分 HTTP 长连接、TCP 连接、连接池。
- 能否说明 HTTP/1.1、HTTP/2、HTTP/3 的核心差异。
- 能否根据异常判断是 HTTP 语义问题还是 TCP 连接问题。
