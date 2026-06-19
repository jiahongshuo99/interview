# fd 泄漏问题

## 面试定位

fd 泄漏是 Java 后端线上常见故障。它可能表现为 `Too many open files`、新连接失败、日志写失败、类加载失败、临时文件无法打开。面试官看重的是你能否从症状定位到 fd 类型，再回到代码关闭资源。

回答目标：

- 能说明 fd 泄漏是什么。
- 能识别 socket 泄漏、文件流泄漏、Selector 泄漏、临时文件泄漏。
- 能用 `lsof`、`/proc/<pid>/fd`、`ss` 定位增长类型。
- 能给出 Java 代码层修复方式。
- 能说明调大 `ulimit` 只是缓解，不是根因修复。

## 核心概念

fd 泄漏是进程打开了文件、socket、pipe、eventfd、epoll fd 等资源，但使用完后没有关闭，导致 fd 数持续增长。

最终结果：

- 达到进程 `open files` 上限。
- 新文件或新 socket 无法打开。
- `accept` 失败。
- 下游连接失败。
- 日志、临时文件、配置加载异常。
- 进程可用性下降。

常见异常：

```text
java.net.SocketException: Too many open files
java.io.FileNotFoundException: ... (Too many open files)
java.nio.channels.SocketChannel.open failed
accept failed: Too many open files
```

## fd 泄漏类型

### 文件流未关闭

错误示例：

```java
InputStream in = new FileInputStream(path);
byte[] data = in.readAllBytes();
return data; // 异常或正常路径都没有 close
```

正确示例：

```java
try (InputStream in = new FileInputStream(path)) {
    return in.readAllBytes();
}
```

### socket 未关闭

错误示例：

```java
Socket socket = new Socket(host, port);
socket.getOutputStream().write(data);
// 异常路径未关闭
```

正确示例：

```java
try (Socket socket = new Socket(host, port)) {
    socket.getOutputStream().write(data);
}
```

工程中通常不直接手写 socket，而是使用 HTTP 客户端、数据库连接池、RPC 框架。此时要正确关闭 response、连接、channel 或释放回连接池。

### HTTP 响应未关闭

以 OkHttp 为例：

```java
try (Response response = client.newCall(request).execute()) {
    return response.body().string();
}
```

如果 response body 不关闭，连接可能不能归还连接池，最终表现为连接泄漏或 fd 增长。

### 数据库连接未归还

连接池场景中 `Connection.close()` 通常不是关闭物理连接，而是归还连接池。

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    handle(rs);
}
```

不关闭连接会导致连接池耗尽，也可能伴随 fd 数较高。

### Selector 或 event loop 重复创建

每个 Java NIO Selector 在 Linux 上可能对应 epoll fd 和 eventfd。如果反复创建 Selector 或 Netty EventLoopGroup 不关闭，可能出现 `anon_inode:[eventpoll]` 增长。

正确关闭：

```java
EventLoopGroup boss = new NioEventLoopGroup(1);
EventLoopGroup worker = new NioEventLoopGroup();
try {
    startServer(boss, worker);
} finally {
    boss.shutdownGracefully();
    worker.shutdownGracefully();
}
```

### 临时文件泄漏

临时文件未删除不一定是 fd 泄漏，但如果文件流未关闭，会同时占 fd。即使文件已删除，只要 fd 仍打开，磁盘空间也可能不释放。

`lsof` 中可能看到：

```text
/tmp/xxx (deleted)
```

## 排查步骤

### 1. 确认 fd 使用量

```bash
cat /proc/<pid>/limits | grep "open files"
ls /proc/<pid>/fd | wc -l
```

连续观察：

```bash
watch -n 1 'ls /proc/<pid>/fd | wc -l'
```

如果 fd 数只涨不降，基本可以怀疑泄漏。

### 2. 看 fd 类型分布

```bash
lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -nr
```

常见判断：

- `IPv4` / `IPv6` 多：socket 连接。
- `REG` 多：普通文件。
- `FIFO` 多：管道。
- `anon_inode` 多：epoll、eventfd、timerfd。

### 3. 看具体 fd 指向

```bash
ls -l /proc/<pid>/fd | head
readlink /proc/<pid>/fd/<fd>
cat /proc/<pid>/fdinfo/<fd>
```

### 4. socket 细分

```bash
lsof -p <pid> -a -iTCP
ss -antp | grep <pid>
ss -antp | awk '{print $1}' | sort | uniq -c | sort -nr
```

重点状态：

- `CLOSE-WAIT`：对端已关闭，本端未关闭，强烈指向应用未 close。
- `ESTAB`：已建立连接，可能是正常长连接，也可能连接池过大。
- `TIME-WAIT`：通常是主动关闭后的内核状态，不一定占进程 fd。
- `SYN-SENT`：连接建立中，可能下游不可达。

### 5. 找增长对象

每隔一段时间采样：

```bash
lsof -p <pid> > /tmp/lsof-1.txt
sleep 60
lsof -p <pid> > /tmp/lsof-2.txt
```

对比新增远端地址、文件路径、fd 类型。socket 泄漏可以按远端 IP:PORT 聚合：

```bash
lsof -p <pid> -a -iTCP | awk '{print $9,$10}' | sort | uniq -c | sort -nr | head
```

## Java 侧定位

### 线程栈

如果 fd 正在快速增长，抓线程栈：

```bash
jstack -l <pid> > /tmp/jstack.txt
```

搜索：

- 创建 socket 的业务方法。
- 上传下载文件方法。
- HTTP 客户端调用。
- 数据库操作。
- 定时任务。

### 代码搜索

重点搜索：

```bash
rg "new FileInputStream|new FileOutputStream|Files.newInputStream|Socket\\(|SocketChannel.open|Selector.open|HttpClient|OkHttp|CloseableHttpClient|getConnection"
```

检查：

- 是否 try-with-resources。
- catch 分支是否 close。
- response body 是否 close。
- 异步回调是否释放资源。
- 连接是否归还池。
- Netty channel 是否关闭。

### 连接池监控

看：

- active 连接数。
- idle 连接数。
- pending 等待数。
- acquire timeout。
- max pool size。
- 连接创建和销毁速率。

fd 泄漏和连接池耗尽常常一起出现，但根因可能是连接未归还而不是 fd 上限太低。

## 修复方式

### 使用 try-with-resources

所有实现 `AutoCloseable` 的资源优先这样写：

```java
try (InputStream in = Files.newInputStream(path);
     OutputStream out = Files.newOutputStream(target)) {
    in.transferTo(out);
}
```

### 成功获取后再释放

对需要手动释放的资源，记录是否成功获取：

```java
Resource r = acquire();
try {
    use(r);
} finally {
    r.close();
}
```

### 异步流程显式释放

异步回调中要覆盖成功、失败、取消、超时路径。

```java
future.whenComplete((result, error) -> {
    try {
        handle(result, error);
    } finally {
        resource.close();
    }
});
```

### 设置超时

连接、读、写、请求总耗时都要有超时。无超时会导致 socket 长期占用。

### 关闭不再使用的客户端

服务关闭时释放：

- HTTP client。
- 数据源。
- Redis client。
- MQ producer/consumer。
- Netty EventLoopGroup。
- Scheduler。

## 应急处理

如果线上 fd 即将耗尽：

1. 先摘流或降流，避免继续打开新连接。
2. 保留现场：

```bash
date
cat /proc/<pid>/limits > /tmp/limits.txt
ls -l /proc/<pid>/fd > /tmp/fd.txt
lsof -p <pid> > /tmp/lsof.txt
ss -antp > /tmp/ss.txt
jstack -l <pid> > /tmp/jstack.txt
```

3. 如果服务已不可用，按预案重启单实例恢复。
4. 调大 fd 上限只能作为临时缓解，必须修复泄漏。

## 常见追问

### CLOSE_WAIT 多说明什么

说明对端已经关闭连接，本端还没有调用 close。通常是应用没有正确关闭 socket、response body、连接对象，或者某个框架释放路径有 bug。

### TIME_WAIT 多是 fd 泄漏吗

通常不是进程 fd 泄漏。TIME_WAIT 是 TCP 状态，连接已经关闭，主要占内核连接状态和端口资源。要结合 `lsof -p` 看进程是否仍持有 fd。

### 只调大 ulimit 可以吗

不可以。调大上限可以延缓故障，但泄漏会继续增长，最终仍耗尽 fd、内存或下游资源。必须找到未关闭资源。

### 为什么文件删除了磁盘空间还没释放

如果进程仍打开该文件 fd，文件目录项删除了，但 inode 引用仍存在。只有所有 fd 关闭后空间才释放。`lsof | grep deleted` 可排查。

## 易错点

- 把连接池耗尽和 fd 泄漏混为一谈。
- 只看 `ss` 不看进程 `lsof`。
- 忽略 response body 必须关闭。
- catch 分支提前 return 导致 close 未执行。
- 异步超时和取消路径忘记释放。
- 看到 TIME_WAIT 多就认为应用 fd 泄漏。
- 只调 `ulimit` 不修代码。

## 自检清单

- 能否解释 fd 泄漏的定义和影响。
- 能否用 `/proc/<pid>/fd` 和 `lsof` 判断 fd 是否持续增长。
- 能否按类型区分 socket、普通文件、anon_inode。
- 能否解释 CLOSE_WAIT 和 TIME_WAIT 的差异。
- 能否写出 Java try-with-resources 修复模板。
- 能否给出线上保留现场和应急恢复步骤。
