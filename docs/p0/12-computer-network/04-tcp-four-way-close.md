# TCP 四次挥手

## 面试定位

TCP 四次挥手常和 TIME_WAIT、CLOSE_WAIT 一起考。面试官会问：为什么建立连接是三次而关闭连接通常是四次，谁进入 TIME_WAIT，CLOSE_WAIT 为什么会堆积，应用层怎么避免连接泄漏。

回答要强调：TCP 是全双工连接，两个方向的字节流需要分别关闭。

## 关闭流程

假设客户端主动关闭：

```text
Client                                      Server
  | ---- FIN, seq = u --------------------> |
  |                                         | 进入 CLOSE_WAIT
  | <--- ACK, ack = u + 1 ---------------- |
  | 进入 FIN_WAIT_2                        |
  |                                         | 应用处理完后 close
  | <--- FIN, seq = v -------------------- |
  | ---- ACK, ack = v + 1 ----------------> |
  | 进入 TIME_WAIT                         | 进入 CLOSED
  |
等待 2MSL 后进入 CLOSED
```

主动关闭方状态：

```text
ESTABLISHED -> FIN_WAIT_1 -> FIN_WAIT_2 -> TIME_WAIT -> CLOSED
```

被动关闭方状态：

```text
ESTABLISHED -> CLOSE_WAIT -> LAST_ACK -> CLOSED
```

## 为什么通常是四次

TCP 连接是全双工的：

- 一方发送 FIN，只表示自己不再发送数据。
- 另一方仍然可能还有数据要发送。
- 被动方先 ACK，表示收到关闭请求。
- 被动方等应用处理完并调用 close 后，再发送 FIN。

如果被动方收到 FIN 后也立刻关闭，ACK 和 FIN 可能合并，看起来像三次挥手。但逻辑上两个方向仍是独立关闭。

## 半关闭

TCP 支持半关闭：

- 调用 `shutdownOutput()`：不再写，但仍可读。
- 调用 `shutdownInput()`：不再读。
- 调用 `close()`：通常关闭整个 Socket。

Java 示例：

```java
socket.shutdownOutput();
// 仍可读取对端响应
```

HTTP/RPC 日常开发中较少显式使用半关闭，但理解它有助于解释四次挥手。

## SO_LINGER 与 RST

正常 close 通常走 FIN 流程。某些场景会发送 RST：

- 设置 `SO_LINGER` 为 0 后关闭。
- 应用关闭连接时接收缓冲区还有未读数据。
- 代理、网关或服务主动重置连接。
- 协议错误或连接异常。

RST 表示异常终止，不需要完整四次挥手。客户端常看到 `connection reset`。

## 抓包示例

抓关闭过程：

```bash
sudo tcpdump -i any host 203.0.113.20 and tcp port 8080 -nn -vv
```

典型标志：

```text
Flags [F.]
Flags [.]
Flags [F.]
Flags [.]
```

查看状态：

```bash
ss -ant state fin-wait-1
ss -ant state fin-wait-2
ss -ant state time-wait
ss -ant state close-wait
```

按端口统计：

```bash
ss -ant | awk '{print $1}' | sort | uniq -c | sort -nr
```

## 线上排查

### 大量 TIME_WAIT

通常出现在主动关闭方。比如客户端频繁短连接请求下游，或者服务端主动关闭连接。

关注：

- 是否没有使用连接池。
- HTTP keep-alive 是否生效。
- 空闲连接超时是否过短。
- QPS 是否过高导致短连接创建频繁。

### 大量 CLOSE_WAIT

通常是本端被动收到 FIN 后，应用没有及时 close。

常见原因：

- Java 客户端响应体没有关闭。
- 服务端代码没有释放 Socket。
- 线程阻塞导致关闭逻辑迟迟不执行。
- 连接池泄漏。

排查：

```bash
ss -antp state close-wait
lsof -iTCP -sTCP:CLOSE_WAIT
jstack <pid>
```

需要结合堆栈看哪些线程持有连接或卡在读写。

### FIN_WAIT_2 堆积

主动关闭方已收到 FIN 的 ACK，但迟迟等不到对端 FIN。

常见原因：

- 对端应用没有 close。
- 中间代理异常。
- 对端进入 CLOSE_WAIT 堆积。

## Java 工程注意点

HTTP 客户端必须关闭响应体：

```java
try (Response response = client.newCall(request).execute()) {
    return response.body().string();
}
```

Apache HttpClient 也要消费或关闭响应实体，否则连接无法归还连接池。

服务端要避免：

- 请求处理线程长时间阻塞。
- 异常路径未关闭流。
- 自定义 Socket 服务没有 finally close。

## 常见追问

### 谁进入 TIME_WAIT

主动关闭连接的一方进入 TIME_WAIT。

### 为什么要有 TIME_WAIT

为了确保最后一个 ACK 能被对端收到，并让旧连接残留报文在网络中自然消失，避免影响后续相同四元组连接。

### close 和 shutdown 有什么区别

`shutdown` 可以关闭单向读或写，保留另一方向；`close` 关闭整个 Socket 并释放资源。

## 易错点

- 认为四次挥手必须严格四个包。ACK 和 FIN 可能合并，但语义仍是双向关闭。
- 认为 CLOSE_WAIT 是网络问题。大量 CLOSE_WAIT 往往是应用没有 close。
- 认为 TIME_WAIT 都是坏事。少量 TIME_WAIT 是 TCP 正常机制。
- 只调大系统参数，不治理短连接和连接泄漏。

## 自检清单

- 能否画出主动关闭方和被动关闭方状态。
- 能否解释为什么 TCP 关闭通常需要四次。
- 能否说明 TIME_WAIT 和 CLOSE_WAIT 分别出现在哪一方。
- 能否用 `ss` 找出连接状态和进程。
- 能否说出 Java HTTP 客户端如何避免连接泄漏。
