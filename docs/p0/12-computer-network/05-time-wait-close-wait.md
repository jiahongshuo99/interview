# TIME_WAIT、CLOSE_WAIT

## 面试定位

TIME_WAIT 和 CLOSE_WAIT 是网络基础与线上排查结合最紧的题。面试官通常会给现象：机器上 TIME_WAIT 很多、CLOSE_WAIT 很多、端口耗尽、连接池打满，然后看你是否能判断是哪一方主动关闭、是正常现象还是泄漏、应该改代码还是调内核参数。

核心结论：

- TIME_WAIT 在主动关闭方，是 TCP 正常状态。
- CLOSE_WAIT 在被动关闭方，长时间大量存在通常是应用未关闭连接。
- TIME_WAIT 多不一定是故障，CLOSE_WAIT 多通常要优先查代码和线程。

## TIME_WAIT

主动关闭方在发送最后一个 ACK 后进入 TIME_WAIT，等待 2MSL 后关闭。

作用：

- 确保最后一个 ACK 丢失时还能重发。
- 等待旧连接报文在网络中消失。
- 避免相同四元组的新连接收到旧报文。

MSL 是 Maximum Segment Lifetime，表示报文在网络中可能存活的最长时间。TIME_WAIT 通常等待 2MSL。

## CLOSE_WAIT

被动关闭方收到对端 FIN 后，内核回复 ACK，并把连接交给应用关闭。此时连接进入 CLOSE_WAIT。

如果应用迟迟不调用 close，连接会一直停留在 CLOSE_WAIT。

常见原因：

- HTTP 响应体没有关闭。
- 输入流、输出流异常路径没有 close。
- 业务线程阻塞，无法执行关闭逻辑。
- 连接池使用不当，连接没有释放。
- 自定义 TCP 服务没有 finally 清理 Socket。

## 状态对比

| 状态 | 出现位置 | 是否正常 | 常见根因 | 处理方向 |
| --- | --- | --- | --- | --- |
| TIME_WAIT | 主动关闭方 | 少量或短时间大量可正常 | 短连接多、主动关闭多 | 连接复用、减少短连接、谨慎调参 |
| CLOSE_WAIT | 被动关闭方 | 长时间大量通常异常 | 应用未 close、线程阻塞 | 查代码、查堆栈、修复资源释放 |

## 命令示例

统计连接状态：

```bash
ss -ant | awk 'NR>1 {state[$1]++} END {for (s in state) print s, state[s]}'
```

查看 TIME_WAIT：

```bash
ss -ant state time-wait
ss -ant state time-wait '( sport = :8080 or dport = :8080 )'
```

查看 CLOSE_WAIT 和进程：

```bash
ss -antp state close-wait
lsof -iTCP -sTCP:CLOSE_WAIT
```

查看端口范围：

```bash
sysctl net.ipv4.ip_local_port_range
```

查看 TCP 相关参数：

```bash
sysctl net.ipv4.tcp_fin_timeout
sysctl net.ipv4.tcp_tw_reuse
```

macOS 上可以用：

```bash
netstat -anv | grep TIME_WAIT
netstat -anv | grep CLOSE_WAIT
lsof -iTCP -sTCP:CLOSE_WAIT
```

## Java 连接泄漏示例

错误示例：

```java
Response response = client.newCall(request).execute();
String body = response.body().string();
return body;
```

如果异常发生在读取过程中，响应可能没有关闭。

推荐：

```java
try (Response response = client.newCall(request).execute()) {
    if (!response.isSuccessful()) {
        throw new IOException("unexpected code " + response);
    }
    return response.body().string();
}
```

对输入输出流也要使用 try-with-resources：

```java
try (InputStream in = socket.getInputStream();
     OutputStream out = socket.getOutputStream()) {
    // read and write
}
```

## TIME_WAIT 过多的影响

可能影响：

- 临时端口耗尽，客户端无法建立新连接。
- 连接表占用内核资源。
- 短连接 QPS 高时增加 CPU 和网络开销。

但 TIME_WAIT 本身不是错误，看到很多 TIME_WAIT 要先看：

- 是否有短连接高峰。
- 是否连接池未生效。
- 是否服务端或客户端主动关闭策略过于激进。
- 是否连接空闲时间配置过短。
- 端口耗尽是否真实发生。

## 处理 TIME_WAIT 的优先级

优先工程治理：

- 使用 HTTP/RPC 连接池。
- 开启 keep-alive。
- 合理设置连接池大小和空闲时间。
- 避免每次请求创建新客户端。
- 降低无意义探活频率。

再考虑系统参数：

- 扩大本地临时端口范围。
- 合理启用复用能力。
- 调整 FIN 等待相关参数。

不要一上来就改内核参数。错误调参可能引入旧报文串扰、NAT 场景异常等风险。

## CLOSE_WAIT 过多的排查路径

1. 找到进程：

```bash
lsof -iTCP -sTCP:CLOSE_WAIT
```

2. 看连接对端：

```bash
ss -antp state close-wait
```

3. 抓 Java 堆栈：

```bash
jstack <pid> > /tmp/jstack.txt
```

4. 查 HTTP Client 使用方式：

- 是否关闭 response。
- 是否消费 response body。
- 是否异常路径跳过 close。
- 是否连接池参数过小。

5. 查线程池：

- 业务线程是否卡在下游。
- close 回调是否执行不到。
- Netty EventLoop 是否被阻塞。

## 常见追问

### 服务端会有 TIME_WAIT 吗

会。谁主动关闭谁进入 TIME_WAIT。HTTP 服务端如果主动关闭连接，也会产生 TIME_WAIT。

### TIME_WAIT 能不能直接杀掉

不能像进程一样直接杀。应通过连接复用、主动关闭策略和参数优化降低数量。

### CLOSE_WAIT 是否能靠调参数解决

通常不能。CLOSE_WAIT 是应用没有关闭连接，核心是修代码或释放资源。

### 为什么连接池泄漏会导致 CLOSE_WAIT

连接池认为连接还被业务占用，无法关闭或复用。对端关闭后，本端没有释放，状态就停在 CLOSE_WAIT。

## 易错点

- 看到 TIME_WAIT 就认为服务异常。
- 认为 CLOSE_WAIT 是对端没关闭。其实对端已经发 FIN，本端应用没 close。
- 在不知道主动关闭方的情况下盲目判断。
- 忽略连接池空闲超时与 LB idle timeout 的关系。
- 只看数量，不看持续时间、QPS、端口耗尽和错误率。

## 自检清单

- 能否说明 TIME_WAIT 和 CLOSE_WAIT 分别在哪一方。
- 能否解释 TIME_WAIT 的 2MSL 作用。
- 能否用 `ss`、`lsof` 定位 CLOSE_WAIT 对应进程。
- 能否说出 Java HTTP 响应体未关闭的后果。
- 能否给出 TIME_WAIT 多时的工程治理优先级。
