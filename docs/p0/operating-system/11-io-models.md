# 阻塞 IO、非阻塞 IO、同步 IO、异步 IO

## 面试定位

IO 模型是网络编程、Netty、NIO、epoll、Reactor 的前置知识。Java 后端面试中常见陷阱是把阻塞/非阻塞和同步/异步混为一谈，或者说 NIO 就是异步 IO。

回答目标：

- 区分阻塞和非阻塞关注的是调用是否等待。
- 区分同步和异步关注的是结果由谁完成、谁通知。
- 能说明 BIO、NIO、AIO 的关系。
- 能解释同步非阻塞为什么还需要事件循环。
- 能把 Linux IO 多路复用和 Java NIO 关联起来。

## 两个维度

IO 模型至少要从两个维度理解：

| 维度 | 问题 | 典型分类 |
| --- | --- | --- |
| 阻塞 / 非阻塞 | 调用发起后，如果条件不满足，调用线程是否等待 | blocking / non-blocking |
| 同步 / 异步 | IO 完成过程由调用线程自己推进，还是由内核或运行时完成后通知 | synchronous / asynchronous |

阻塞和同步不是一回事。同步 IO 也可以是非阻塞的，例如非阻塞 socket 加 epoll。异步 IO 通常表现为提交请求后立即返回，完成后回调或通知。

## 阻塞 IO

阻塞 IO 指调用线程发起 IO 操作后，如果数据没准备好，就一直等待。

以 socket read 为例：

```text
应用线程 read()
  -> 内核检查 socket 接收缓冲区
  -> 没有数据，线程睡眠
  -> 数据到达，内核唤醒线程
  -> 拷贝数据到用户态
  -> read 返回
```

Java BIO 示例：

```java
try (ServerSocket server = new ServerSocket(8080)) {
    while (true) {
        Socket socket = server.accept(); // 阻塞等待连接
        new Thread(() -> handle(socket)).start();
    }
}
```

优点：

- 编程模型简单。
- 代码顺序直观。
- 适合连接数少、并发低或虚拟线程场景。

缺点：

- 一个阻塞连接通常占用一个平台线程。
- 大量连接会造成线程数膨胀。
- 线程切换和栈内存成本高。
- 慢连接会占住线程。

## 非阻塞 IO

非阻塞 IO 指调用线程发起 IO 操作后，如果条件不满足，系统调用立即返回错误或特殊状态，而不是让线程睡眠。

以 non-blocking socket read 为例：

```text
应用线程 read()
  -> 内核检查接收缓冲区
  -> 没有数据，立即返回 EAGAIN / EWOULDBLOCK
  -> 应用稍后重试
```

Java NIO 示例：

```java
SocketChannel channel = SocketChannel.open();
channel.configureBlocking(false);
int n = channel.read(buffer);
if (n == 0) {
    // 当前没有数据，稍后再读
}
```

如果单纯 while 重试，会变成忙轮询，浪费 CPU。因此非阻塞 IO 通常要配合 IO 多路复用：

```text
注册 fd 到 selector/epoll
  -> 事件循环等待就绪事件
  -> fd 可读时再 read
  -> fd 可写时再 write
```

优点：

- 少量线程管理大量连接。
- 避免每个连接一个线程。
- 适合高并发长连接。

缺点：

- 编程复杂。
- 需要处理半包、粘包、写缓冲、背压。
- 业务不能阻塞事件循环。

## 同步 IO

同步 IO 指 IO 操作完成需要调用方主动参与。即使调用是非阻塞的，只要数据拷贝和完成语义由调用线程通过 `read/write` 推进，仍属于同步 IO。

常见同步 IO：

- 阻塞 `read/write`。
- 非阻塞 `read/write`。
- `select/poll/epoll` 事件就绪后自己调用 `read/write`。

epoll 是同步 IO 多路复用：epoll 告诉你 fd 可读或可写，但真正读取数据仍由应用线程调用 `read` 完成。

## 异步 IO

异步 IO 指应用提交 IO 请求后立即返回，内核或运行时负责完成 IO，完成后通过回调、事件、Future 等方式通知应用。

理想流程：

```text
应用提交异步读请求
  -> 立即返回
  -> 内核等待数据并完成拷贝
  -> 完成后通知应用
  -> 应用处理结果
```

Java AIO 示例：

```java
AsynchronousSocketChannel channel = AsynchronousSocketChannel.open();
ByteBuffer buffer = ByteBuffer.allocate(1024);

channel.read(buffer, null, new CompletionHandler<Integer, Void>() {
    @Override
    public void completed(Integer result, Void attachment) {
        handle(buffer);
    }

    @Override
    public void failed(Throwable exc, Void attachment) {
        handleError(exc);
    }
});
```

注意：不同操作系统对真正异步网络 IO 的支持差异较大，Java 后端高性能网络框架主流仍是 NIO + epoll + Reactor，而不是 AIO。

## 五种经典 IO 模型

经典 Unix 网络编程常讲五种模型：

| 模型 | 等待数据阶段 | 数据拷贝阶段 | Java / Linux 对应 |
| --- | --- | --- | --- |
| 阻塞 IO | 阻塞 | 阻塞 | BIO |
| 非阻塞 IO | 立即返回，应用轮询 | 阻塞或立即返回 | non-blocking socket |
| IO 多路复用 | 阻塞在 select/poll/epoll | 应用 read 阻塞或非阻塞读取 | Java NIO Selector |
| 信号驱动 IO | 内核发信号通知可读 | 应用 read | 较少用于 Java 后端 |
| 异步 IO | 内核等待并完成 | 内核完成后通知 | AIO / io_uring 场景 |

IO 多路复用看似在 `epoll_wait` 阻塞，但阻塞的是少量事件循环线程，不是每个连接一个线程。

## Java BIO、NIO、AIO

| Java 模型 | 核心 API | OS 模型 | 特点 |
| --- | --- | --- | --- |
| BIO | `Socket`、`ServerSocket`、流 API | 阻塞 IO | 简单，一个连接常占一个线程 |
| NIO | `Channel`、`Buffer`、`Selector` | 同步非阻塞 + IO 多路复用 | 少量线程管理大量连接 |
| AIO | `AsynchronousSocketChannel` | 异步 IO 或运行时模拟 | 回调式，主流服务较少直接用 |

NIO 不是异步 IO。NIO 中 `Selector` 只负责就绪通知，真正 `read/write` 仍在应用线程执行。

## 线上排查

### 阻塞 IO 线程过多

现象：

- 线程数多。
- 大量线程卡在 socket read、HTTP client、DB driver。
- 线程池耗尽。

命令：

```bash
jstack -l <pid> > /tmp/jstack.txt
grep -n "socketRead\|SocketInputStream\|NioSocketImpl" -A 20 /tmp/jstack.txt
top -H -p <pid>
```

修复：

- 设置连接、读、写、请求总超时。
- 控制线程池和队列。
- 控制连接池大小。
- 使用 NIO/Netty 或虚拟线程。
- 对慢下游做隔离、限流、熔断。

### NIO 事件循环阻塞

现象：

- 少量 event loop 线程 CPU 高或卡住。
- 大量连接延迟上升。
- Netty 报 event loop blocked 或 pending tasks 增长。

排查：

```bash
jstack -l <pid> | grep -n "nioEventLoop\|epollEventLoop" -A 40
```

修复：

- event loop 中不要做阻塞 DB/RPC/文件 IO。
- 业务逻辑丢到业务线程池。
- 大计算任务拆分或异步化。
- 写出时处理 backpressure。

### 空轮询或 busy loop

非阻塞 IO 如果错误地无限重试 read/write，可能 CPU 高。

排查：

```bash
top -H -p <pid>
jstack -l <pid>
perf top -p <pid>
```

看线程是否持续在 selector、read、业务循环中。

## 常见追问

### 阻塞和同步有什么区别

阻塞描述调用没准备好时是否等待。同步描述 IO 完成是否由调用线程主动参与。阻塞 IO 是同步阻塞；NIO + epoll 是同步非阻塞；异步 IO 是提交后由内核或运行时完成并通知。

### epoll 是异步 IO 吗

不是。epoll 是 IO 多路复用，通知 fd 就绪。应用仍要自己调用 `read/write` 完成数据读写，所以通常归为同步非阻塞 IO。

### 非阻塞 IO 为什么还需要 selector

如果没有 selector，应用只能循环尝试每个 fd 是否可读可写，会浪费 CPU。selector/epoll 帮应用等待大量 fd 的就绪事件。

### 虚拟线程会替代 NIO 吗

虚拟线程让阻塞式代码在大量 IO 等待场景下更轻量，能简化很多业务服务代码。但高性能网络框架、协议栈、极致连接规模和事件驱动场景仍会大量使用 NIO/Netty。

## 易错点

- 把 NIO 说成异步 IO。
- 把非阻塞 IO 理解为不会消耗 CPU，忽略忙轮询风险。
- 认为异步一定更快，忽略回调复杂度和调度成本。
- 在 Netty event loop 中执行阻塞操作。
- 只有 socket connect 超时，没有 read/request 总超时。
- IO 线程池和业务线程池混用导致互相拖垮。

## 自检清单

- 能否用两个维度解释阻塞/非阻塞和同步/异步。
- 能否说明 epoll 为什么是同步非阻塞 IO。
- 能否区分 BIO、NIO、AIO。
- 能否解释非阻塞 IO 为什么需要事件循环。
- 能否从线程 dump 看阻塞 IO 和 event loop 阻塞。
- 能否给出 Java 后端 IO 模型选型建议。
