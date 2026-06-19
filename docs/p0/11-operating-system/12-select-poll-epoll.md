# select、poll、epoll

## 面试定位

`select`、`poll`、`epoll` 是 Linux IO 多路复用的核心。Java 后端面试中常和 NIO Selector、Netty、Reactor、高并发连接、水平触发/边缘触发、惊群问题一起问。

回答目标：

- 能解释 IO 多路复用解决什么问题。
- 能比较 select、poll、epoll 的数据结构和复杂度差异。
- 能说明 epoll 的 LT 和 ET。
- 能把 Java Selector 和 Linux epoll 关联起来。
- 能指出 epoll 不是异步 IO。

## IO 多路复用解决的问题

阻塞 IO 模型下，一个线程阻塞等待一个 fd。连接数很大时，线程数也很大。

IO 多路复用允许一个线程同时等待多个 fd：

```text
多个 socket fd
  -> 注册到 select/poll/epoll
  -> 一个线程等待就绪事件
  -> 哪些 fd 可读/可写，就处理哪些 fd
```

这让少量事件循环线程可以管理大量连接。

## select

`select` 使用 fd_set 位图表示关注的 fd 集合。

特点：

- 每次调用都要把 fd 集合从用户态拷贝到内核态。
- 内核遍历 fd 集合找就绪 fd。
- 返回后应用还要遍历 fd 集合判断哪些就绪。
- 受 `FD_SETSIZE` 限制，常见默认 1024。

伪代码：

```c
fd_set readfds;
FD_ZERO(&readfds);
FD_SET(fd1, &readfds);
FD_SET(fd2, &readfds);

int n = select(maxfd + 1, &readfds, NULL, NULL, &timeout);
if (FD_ISSET(fd1, &readfds)) {
    read(fd1, buf, size);
}
```

缺点：

- fd 数量限制。
- 每次重复拷贝 fd 集合。
- 每次线性扫描。
- 大量连接少量活跃时效率差。

## poll

`poll` 使用 `pollfd` 数组表示 fd 和事件。

特点：

- 没有 select 那种固定 1024 位图限制。
- 每次仍要把数组从用户态拷贝到内核态。
- 内核仍要线性扫描数组。
- 返回后应用也要遍历数组找就绪 fd。

伪代码：

```c
struct pollfd fds[2];
fds[0].fd = fd1;
fds[0].events = POLLIN;
fds[1].fd = fd2;
fds[1].events = POLLIN;

int n = poll(fds, 2, timeout);
if (fds[0].revents & POLLIN) {
    read(fd1, buf, size);
}
```

相比 select：

- 解决 fd 数量固定限制。
- 表达事件更方便。
- 但没有解决每次拷贝和线性扫描问题。

## epoll

epoll 是 Linux 提供的更适合大量 fd 的 IO 多路复用机制。

核心 API：

```c
int epfd = epoll_create1(0);
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);
int n = epoll_wait(epfd, events, maxevents, timeout);
```

关键特点：

- 使用 epoll 实例保存关注的 fd，避免每次传入完整 fd 集合。
- 通过 `epoll_ctl` 增删改关注事件。
- 内核维护就绪队列，`epoll_wait` 返回就绪事件。
- 适合大量连接、少量活跃的场景。

简化流程：

```text
epoll_create 创建 epoll fd
epoll_ctl 注册 socket fd 和事件
内核在 fd 就绪时加入 ready list
epoll_wait 返回就绪 fd 列表
应用 read/write 处理
```

## select、poll、epoll 对比

| 维度 | select | poll | epoll |
| --- | --- | --- | --- |
| fd 表达 | fd_set 位图 | pollfd 数组 | 内核 epoll 实例 |
| fd 数量限制 | 受 FD_SETSIZE 限制 | 理论上无固定小限制 | 受系统资源限制 |
| 每次拷贝 | 拷贝 fd_set | 拷贝 pollfd 数组 | 注册后不需要每次拷贝全部 |
| 查找就绪 | 线性扫描 | 线性扫描 | ready list 返回就绪 |
| 适合场景 | 少量 fd | 中等 fd | 大量连接少量活跃 |
| 平台 | 跨 Unix 较通用 | 跨 Unix 较通用 | Linux 特有 |

注意：epoll 不是所有场景都绝对更快。fd 很少时差异不大；事件处理逻辑、网络栈、业务耗时、锁竞争往往才是瓶颈。

## 水平触发和边缘触发

### LT 水平触发

Level Triggered 是默认模式。

只要 fd 仍处于可读或可写状态，每次 `epoll_wait` 都会返回该事件。

优点：

- 编程简单。
- 不容易漏事件。

缺点：

- 如果没有把数据读完，会反复通知。

### ET 边缘触发

Edge Triggered 只在状态从不可读变为可读、不可写变为可写等边缘变化时通知。

要求：

- fd 必须设置为非阻塞。
- 收到可读事件后要循环读，直到返回 `EAGAIN`。
- 收到可写事件后要尽量写，直到写完或 `EAGAIN`。

伪代码：

```c
while (true) {
    n = read(fd, buf, size);
    if (n > 0) {
        handle(buf, n);
    } else if (n == -1 && errno == EAGAIN) {
        break;
    } else {
        close(fd);
        break;
    }
}
```

ET 优点是减少重复通知，缺点是更容易写错，读写不彻底会导致事件丢失或连接卡住。

## epoll 事件

常见事件：

- `EPOLLIN`：可读。
- `EPOLLOUT`：可写。
- `EPOLLERR`：错误。
- `EPOLLHUP`：挂起。
- `EPOLLET`：边缘触发。
- `EPOLLONESHOT`：事件触发一次后禁用，需要重新 arm。

可写事件要谨慎注册。如果长期关注 `EPOLLOUT`，大多数 socket 经常可写，可能导致事件循环频繁被唤醒。常见做法是只有写缓冲区有待发送数据时才关注可写事件。

## Java Selector 与 epoll

Java NIO `Selector` 是跨平台抽象。在 Linux 上，主流 JDK 通常使用 epoll 实现。

Java 示例：

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
            accept(key);
        } else if (key.isReadable()) {
            read(key);
        }
    }
}
```

JDK Selector 屏蔽了底层 select、poll、epoll、kqueue 的差异，但线上排查时可以通过 `strace` 看到 `epoll_wait`。

## 命令和排查

### strace 看 epoll

```bash
strace -f -e trace=epoll_create1,epoll_ctl,epoll_wait -p <pid>
```

现象：

- event loop 空闲时大量 `epoll_wait` 是正常的。
- `epoll_wait` 很快返回且 CPU 高，可能有 busy loop、空轮询、不断就绪的 fd。
- `epoll_ctl` 很频繁，可能连接频繁建立关闭或注册事件抖动。

### 查看 fd 中 epoll

```bash
ls -l /proc/<pid>/fd | grep eventpoll
cat /proc/<pid>/fdinfo/<epoll_fd>
```

`fdinfo` 可能显示 epoll 关注的 fd 信息，适合排查复杂 fd 注册问题。

### 网络连接

```bash
ss -antp | grep <pid>
ss -s
```

结合连接状态判断是否大量连接积压、CLOSE_WAIT 或连接风暴。

## 常见追问

### epoll 为什么比 select/poll 适合高并发

epoll 把关注的 fd 保存在内核 epoll 实例中，不需要每次传入全部 fd 集合；内核维护就绪队列，`epoll_wait` 返回就绪事件，避免每次线性扫描全部 fd。

### epoll 一定是 O(1) 吗

不能绝对这么说。epoll 避免了按全部 fd 线性扫描，返回和处理主要与就绪事件数量相关，但真实性能还受事件数量、回调、锁、网络栈、用户态处理、内存分配等影响。

### LT 和 ET 怎么选

LT 更安全，适合大多数场景。ET 通知更少，但要求非阻塞 fd 并且读写到 `EAGAIN`，代码更容易出错。框架内部可以使用 ET，业务侧一般不直接处理。

### epoll 是异步 IO 吗

不是。epoll 只是通知 fd 就绪，应用仍需自己调用 `read/write`。它是同步非阻塞 IO 多路复用。

## 易错点

- 认为 epoll 会把数据读到用户态。
- 认为 epoll 是异步 IO。
- ET 模式没有循环读到 `EAGAIN`。
- 长期注册可写事件导致 CPU 空转。
- 忘记处理 `EPOLLERR`、`EPOLLHUP`。
- 以为 epoll 在所有 fd 数量下都明显优于 select/poll。
- 看到 `epoll_wait` 就认为线程卡死，事件循环等待是正常状态。

## 自检清单

- 能否解释 select、poll、epoll 的核心差异。
- 能否说明 epoll 的 `epoll_create`、`epoll_ctl`、`epoll_wait` 分工。
- 能否区分 LT 和 ET。
- 能否解释为什么 ET 要非阻塞并读到 `EAGAIN`。
- 能否把 Java Selector 和 Linux epoll 对应起来。
- 能否用 `strace`、`/proc/<pid>/fdinfo` 排查 epoll 相关问题。
