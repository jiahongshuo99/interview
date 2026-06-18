# Reactor 模型

## 面试定位

Reactor 是高性能网络框架的核心模型。Java 后端面试中常用于连接操作系统 IO 多路复用、Java NIO、Netty 线程模型和业务线程池隔离。面试官常问：Reactor 是什么，单 Reactor 和主从 Reactor 区别，为什么 event loop 不能阻塞。

回答目标：

- 能解释 Reactor 的职责和流程。
- 能区分 Reactor、Acceptor、Handler。
- 能说明单线程、多线程、主从 Reactor。
- 能把 Reactor 和 epoll/NIO 关联起来。
- 能说明线上 event loop 阻塞如何排查和修复。

## 核心概念

Reactor 模型是一种事件驱动网络编程模型。它把 IO 就绪事件分发给对应处理器。

核心组件：

| 组件 | 职责 |
| --- | --- |
| Reactor | 事件循环，等待 IO 事件并分发 |
| Acceptor | 处理新连接接入 |
| Handler | 处理连接上的读写事件 |
| Demultiplexer | IO 多路复用器，例如 select、poll、epoll |

基本流程：

```text
注册 fd 和感兴趣事件
  -> Reactor 调用 select/epoll_wait 等待事件
  -> 有连接到达，交给 Acceptor
  -> 有读写事件，交给对应 Handler
  -> Handler 执行 read/write 和协议处理
```

Reactor 通常基于同步非阻塞 IO 和 IO 多路复用，不是异步 IO。

## 单线程 Reactor

所有工作都在一个线程：

```text
一个 Reactor 线程
  -> accept 新连接
  -> read 请求
  -> decode
  -> 业务处理
  -> encode
  -> write 响应
```

优点：

- 模型简单。
- 没有多线程同步开销。

缺点：

- 无法利用多核。
- 任意业务处理阻塞都会阻塞所有连接。
- 连接多或业务稍重时不可扩展。

适合教学或简单场景，不适合复杂后端服务。

## 单 Reactor 多线程

一个 Reactor 线程负责 IO 事件，业务处理丢给工作线程池。

```text
Reactor 线程
  -> accept/read/write
  -> decode 简单协议
  -> 提交业务任务到 worker pool

Worker 线程池
  -> 执行业务逻辑
  -> 生成响应
  -> 交回 Reactor 写出
```

优点：

- IO 和业务分离。
- 业务可利用多线程。

缺点：

- 单 Reactor 仍可能成为 IO 瓶颈。
- 需要处理跨线程回写、队列积压、背压。

## 主从 Reactor

也叫多 Reactor。通常一个 boss Reactor 负责 accept，多个 worker Reactor 负责已建立连接的读写。

```text
Boss Reactor
  -> 监听 server socket
  -> accept 新连接
  -> 把连接注册到某个 Worker Reactor

Worker Reactor N
  -> 负责一批连接的 read/write
  -> 执行轻量编解码
  -> 必要时提交业务线程池
```

优点：

- accept 和连接读写分离。
- 多个 worker event loop 利用多核。
- 适合高并发网络服务。

Netty 常见服务端模型就是 boss group + worker group。

## Reactor 和 Proactor

Reactor：

- 关注“就绪事件”。
- 内核通知 fd 可读可写。
- 应用自己调用 read/write。
- 典型实现是 epoll + non-blocking socket。

Proactor：

- 关注“完成事件”。
- 应用提交异步 IO 请求。
- 内核或运行时完成 IO 后通知。
- 更接近异步 IO。

Java 后端常用 Netty 是 Reactor 模型，不是 Proactor。

## Handler 设计

Handler 通常包含：

- 读事件处理。
- 字节流到协议帧的解码。
- 协议对象到业务请求的转换。
- 业务处理或任务提交。
- 响应编码。
- 写事件处理。
- 异常和关闭处理。

重要原则：

- Event loop 中只做短、快、非阻塞操作。
- 慢业务放到业务线程池。
- 不要在 handler 中直接阻塞等待 Future。
- 写出要处理缓冲和背压。
- 连接关闭要释放资源。

## 背压

Reactor 模型中，如果读得太快、业务处理太慢或写出太慢，会产生队列积压。

常见背压点：

- socket 接收缓冲区。
- event loop 任务队列。
- 业务线程池队列。
- outbound 写缓冲区。
- 下游连接池。

处理方式：

- 控制自动读，例如 Netty `autoRead=false`。
- 限制业务线程池队列。
- 高水位暂停读，低水位恢复读。
- 对慢客户端断开或降级。
- 设置请求超时和最大包大小。

## Java NIO 简化示例

```java
Selector selector = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.configureBlocking(false);
server.bind(new InetSocketAddress(8080));
server.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();
    Iterator<SelectionKey> it = selector.selectedKeys().iterator();
    while (it.hasNext()) {
        SelectionKey key = it.next();
        it.remove();

        if (key.isAcceptable()) {
            SocketChannel ch = server.accept();
            ch.configureBlocking(false);
            ch.register(selector, SelectionKey.OP_READ, new ConnectionState());
        } else if (key.isReadable()) {
            readAndDecode(key);
        } else if (key.isWritable()) {
            flushPendingWrites(key);
        }
    }
}
```

真实框架还要处理：

- 半包粘包。
- 连接生命周期。
- 写半包。
- 异常关闭。
- TLS。
- 空轮询 bug。
- 跨线程任务唤醒 selector。

## 线上排查

### Event loop 阻塞

现象：

- 全部连接延迟升高。
- 单个 event loop 上连接都受影响。
- Netty pending task 增长。
- 心跳超时。

排查：

```bash
jstack -l <pid> > /tmp/jstack.txt
grep -n "EventLoop\|nioEventLoop\|epollEventLoop" -A 50 /tmp/jstack.txt
top -H -p <pid>
```

看 event loop 栈是否在：

- DB 查询。
- RPC 调用。
- 文件 IO。
- `Thread.sleep`。
- `Future.get`。
- 大 JSON 序列化。
- 正则或压缩。

修复：

- 阻塞操作移出 event loop。
- 拆分业务线程池。
- 对慢任务限流。
- 设置超时。
- 加 event loop 延迟监控。

### 写缓冲堆积

现象：

- 内存上涨。
- 出站缓冲区高。
- 慢客户端导致整体压力。

修复：

- 设置高低水位。
- 慢连接断开。
- 大响应分片。
- 限制单连接 outstanding 请求。
- 禁止无限堆积写任务。

### 单个 worker 热点

连接分配不均或某些连接流量特别大，可能导致某个 event loop CPU 高。

排查：

```bash
top -H -p <pid>
jstack -l <pid>
ss -antp | grep <pid>
```

修复：

- 检查连接分配策略。
- 拆分热点连接。
- 增加 worker 数要谨慎，避免上下文切换过多。
- 优化热点协议处理。

## 常见追问

### Reactor 为什么适合高并发连接

它用少量线程基于 IO 多路复用等待大量连接事件，避免一个连接一个线程的成本。活跃连接才被处理，适合大量连接少量活跃的场景。

### Event loop 为什么不能阻塞

一个 event loop 负责一批连接。如果它被阻塞，这批连接的 accept、read、write、定时任务、关闭事件都会延迟，造成局部甚至整体不可用。

### Reactor 和观察者模式有什么关系

都有事件通知和分发思想。Reactor 更偏网络 IO 事件驱动架构，底层依赖 IO 多路复用。

### Netty 的 boss 和 worker 对应什么

boss group 类似主 Reactor，负责 accept。worker group 类似从 Reactor，负责已建立连接的读写和 pipeline 事件传播。

## 易错点

- 认为 Reactor 是异步 IO。
- 在 event loop 中执行阻塞业务。
- 忽略写缓冲和背压。
- 只配置 worker 线程数，不看 CPU 核数和业务阻塞。
- 所有业务共享一个无界线程池。
- 不处理半包、粘包、写半包。
- 忽略连接关闭和资源释放。

## 自检清单

- 能否画出 Reactor 基本流程。
- 能否区分单 Reactor、多线程 Reactor、主从 Reactor。
- 能否解释 Reactor 和 Proactor 差异。
- 能否说明 event loop 不能阻塞的原因。
- 能否讲清 Netty boss/worker 和 Reactor 的对应关系。
- 能否给出 event loop 阻塞和写缓冲堆积的排查方式。
