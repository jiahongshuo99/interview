# Netty / NIO 与 OS IO 模型的关系

## 面试定位

这道题考察候选人能否把 Java 网络框架、JDK NIO 和 Linux IO 多路复用串起来。面试官常问：Netty 为什么高性能，NIO 是不是异步，Netty 和 epoll 什么关系，event loop 线程模型是什么，为什么不能在 ChannelHandler 里阻塞。

回答目标：

- 能说明 Java NIO 是同步非阻塞 IO 多路复用。
- 能说明 Netty 封装了 NIO/epoll/kqueue 等底层传输。
- 能解释 Channel、EventLoop、Pipeline、ByteBuf 的角色。
- 能说明 Netty 的线程模型和 OS epoll 的关系。
- 能给出线上排查 event loop 阻塞、连接泄漏、写缓冲堆积的方法。

## 分层关系

可以从上到下理解：

```text
业务代码
  -> Netty ChannelHandler / Pipeline
  -> Netty Channel / EventLoop / ByteBuf
  -> Java NIO Channel / Selector 或 Netty native epoll transport
  -> Linux epoll / socket 系统调用
  -> 网卡、内核协议栈
```

Netty 不是绕过操作系统 IO。它是在 JDK NIO 或 native transport 之上封装事件驱动网络框架。

## Java NIO 和 OS IO

Java NIO 核心组件：

- `Channel`：连接或文件通道。
- `Buffer`：读写缓冲。
- `Selector`：多路复用器。
- `SelectionKey`：fd 和事件注册关系。

Linux 上 Selector 通常基于 epoll：

```text
Selector.open()
  -> epoll_create
channel.register(selector, OP_READ)
  -> epoll_ctl ADD/MOD
selector.select()
  -> epoll_wait
channel.read(buffer)
  -> read/recv
channel.write(buffer)
  -> write/send
```

所以 NIO 是同步非阻塞 IO：

- 非阻塞：socket fd 设置非阻塞。
- 多路复用：selector/epoll 等待多个 fd 就绪。
- 同步：就绪后仍由应用线程调用 read/write。

## Netty 核心组件

### Channel

Netty 的连接抽象，封装底层 socket channel。

常见类型：

- `NioServerSocketChannel`。
- `NioSocketChannel`。
- `EpollServerSocketChannel`。
- `EpollSocketChannel`。

### EventLoop

EventLoop 是 Netty 的事件循环线程。

职责：

- 等待 IO 事件。
- 处理 accept/read/write。
- 执行注册到该 event loop 的任务。
- 触发 pipeline 事件。
- 处理定时任务。

一个 Channel 通常绑定到一个 EventLoop，生命周期内不迁移。这样可以减少锁竞争，保证同一连接上的事件串行执行。

### ChannelPipeline

Pipeline 是处理器链。

```text
入站：socket bytes -> decoder -> handler -> business
出站：business response -> encoder -> bytes -> socket
```

常见 handler：

- `ByteToMessageDecoder`。
- `MessageToByteEncoder`。
- 业务 handler。
- IdleStateHandler。
- SSL handler。

### ByteBuf

Netty 的字节缓冲抽象，相比 JDK ByteBuffer 更适合网络框架：

- 读写指针分离。
- 池化分配。
- 支持堆内和直接内存。
- 引用计数。

注意：使用引用计数 ByteBuf 时必须正确 release，否则会造成直接内存泄漏。

## Netty 线程模型

典型服务端：

```java
EventLoopGroup boss = new NioEventLoopGroup(1);
EventLoopGroup worker = new NioEventLoopGroup();

new ServerBootstrap()
    .group(boss, worker)
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        protected void initChannel(SocketChannel ch) {
            ch.pipeline().addLast(new Decoder());
            ch.pipeline().addLast(new Encoder());
            ch.pipeline().addLast(new BusinessHandler());
        }
    })
    .bind(8080);
```

职责：

- boss：监听端口，accept 新连接。
- worker：处理已建立连接的读写事件。
- business executor：可选，用于执行阻塞或耗时业务。

如果业务 handler 直接在 worker event loop 上执行阻塞操作，会拖慢该 event loop 管理的所有连接。

## Native epoll transport

Netty 在 Linux 上可使用 native epoll transport。

优势可能包括：

- 直接使用 Linux epoll 能力。
- 支持更多 Linux socket 选项。
- 减少 JDK Selector 某些限制。
- 在特定场景下性能和延迟更好。

常见类：

- `EpollEventLoopGroup`。
- `EpollServerSocketChannel`。
- `EpollSocketChannel`。

示例：

```java
EventLoopGroup boss = new EpollEventLoopGroup(1);
EventLoopGroup worker = new EpollEventLoopGroup();

new ServerBootstrap()
    .group(boss, worker)
    .channel(EpollServerSocketChannel.class);
```

是否使用 native epoll 要结合部署环境、依赖包、容器镜像、可观测性和团队维护成本。

## Netty 为什么高性能

主要原因不是某一个点，而是一组工程设计：

- 基于非阻塞 IO 和多路复用，少量线程管理大量连接。
- EventLoop 单线程串行处理单连接事件，减少锁竞争。
- ByteBuf 池化减少分配和 GC。
- 直接内存减少部分堆拷贝和 GC 压力。
- Pipeline 清晰拆分协议处理。
- 支持批量 flush、写缓冲和高低水位。
- 支持 native epoll、零拷贝相关能力。

但 Netty 高性能的前提是业务不阻塞 event loop，并且正确处理背压、内存和连接生命周期。

## 零拷贝相关

Netty 中常被称为零拷贝的点有多个层次：

- `FileRegion` 使用 `sendfile`，文件从 page cache 直接发送到 socket，减少用户态拷贝。
- `CompositeByteBuf` 逻辑组合多个 ByteBuf，减少内存复制。
- `slice`、`duplicate` 共享底层内存，避免复制。
- DirectByteBuf 用堆外内存，减少 JVM 堆和 native IO 的拷贝。

面试中要说清具体是哪种零拷贝，不要笼统说“Netty 零拷贝所以快”。

## 线上排查

### EventLoop 阻塞

命令：

```bash
jstack -l <pid> > /tmp/jstack.txt
grep -n "nioEventLoop\|epollEventLoop" -A 60 /tmp/jstack.txt
top -H -p <pid>
```

风险栈：

- JDBC 查询。
- Redis/HTTP/RPC 同步调用。
- 文件读写。
- `Thread.sleep`。
- `Future.get`。
- 大 JSON 序列化。
- 压缩、加密、大循环。

修复：

- 阻塞业务提交到独立业务线程池。
- 给下游设置超时。
- 使用异步客户端。
- 拆分慢 handler。
- 增加 event loop 延迟监控。

### ByteBuf 泄漏

现象：

- 直接内存上涨。
- 日志出现 leak detector 警告。
- `OutOfDirectMemoryError`。

排查：

- 开启 Netty leak detector，注意线上开销。
- 看 direct memory 指标。
- 检查自定义 handler 是否 release。

修复：

- 遵守引用计数规则。
- 入站 ByteBuf 消费后释放或交给下游所有权。
- 异常路径也 release。
- 避免缓存 ByteBuf 不释放。

### 写缓冲堆积

现象：

- 内存上涨。
- `ChannelOutboundBuffer` 增长。
- 慢客户端或下游网络差。

处理：

- 配置 write buffer high/low watermark。
- `channel.isWritable()` 做背压。
- 慢连接断开。
- 限制单连接未完成请求。
- 大包分片。

### 连接泄漏

命令：

```bash
ss -antp | grep <pid>
lsof -p <pid> -a -iTCP
```

重点看：

- `CLOSE-WAIT`。
- ESTAB 数是否持续增长。
- 是否连接关闭事件未处理。
- handler 异常路径是否关闭 channel。

## 常见追问

### Netty 是异步 IO 吗

Netty 对业务暴露异步事件和 Future 编程模型，但底层主流实现是基于同步非阻塞 IO 多路复用。epoll 通知就绪后，event loop 自己调用 read/write。

### NIO 和 epoll 什么关系

NIO 是 Java 的跨平台 IO API。Linux 上 Selector 通常通过 epoll 实现。epoll 是 OS 级 IO 多路复用机制。

### 为什么 Channel 绑定一个 EventLoop

同一连接事件由同一线程串行处理，可以避免很多锁，保证事件顺序，减少上下文切换和并发复杂度。

### 业务耗时为什么不能放 event loop

event loop 负责一批连接的 IO。如果被一个业务调用阻塞，该 event loop 上所有连接的读写、心跳、关闭事件都会延迟。

### Netty worker 线程数是不是越多越好

不是。worker 线程主要处理 IO 事件，通常和 CPU 核数相关。过多会增加上下文切换。业务阻塞应该放业务线程池，而不是盲目增加 worker。

## 易错点

- 把 Netty 底层说成纯异步 IO。
- 在 ChannelHandler 中直接调用阻塞 RPC 或 DB。
- 忽略 ByteBuf 引用计数，导致直接内存泄漏。
- 所有 write 都立即 flush，导致系统调用和网络包过多。
- 长期忽略 `isWritable`，导致写缓冲无限膨胀。
- worker 线程和业务线程池不隔离。
- 没有处理 channel inactive、exceptionCaught 的资源释放。

## 自检清单

- 能否画出 Netty 到 Java NIO 到 epoll 的分层关系。
- 能否解释 Channel、EventLoop、Pipeline、ByteBuf。
- 能否说明 NIO 是同步非阻塞 IO。
- 能否说明 Netty 高性能的关键设计。
- 能否排查 event loop 阻塞、ByteBuf 泄漏、写缓冲堆积。
- 能否说明 boss、worker、business executor 的职责边界。
