# 文件描述符、socket fd

## 面试定位

文件描述符是 Linux “一切皆文件”思想在进程内的句柄抽象。Java 后端面试里，它常和 socket、连接数、fd 泄漏、`Too many open files`、`lsof`、`ulimit`、epoll 一起问。

回答目标：

- 能解释 fd 是什么。
- 能说明文件 fd 和 socket fd 的共同点与差异。
- 能讲清进程 fd 表、系统打开文件表、inode/socket 结构之间的关系。
- 能用 `lsof`、`/proc/<pid>/fd`、`ss` 排查。
- 能理解为什么 fd 耗尽会影响网络、文件、日志、类加载。

## 核心概念

文件描述符是进程内的非负整数句柄。进程通过 fd 引用内核中的打开文件对象。

常见 fd：

- `0`：标准输入。
- `1`：标准输出。
- `2`：标准错误。
- 普通文件 fd。
- 目录 fd。
- socket fd。
- pipe fd。
- eventfd、timerfd、signalfd。
- epoll fd。

用户态程序不直接操作内核对象，而是把 fd 传给系统调用：

```c
read(fd, buf, size);
write(fd, buf, size);
close(fd);
```

## fd 相关内核结构

可以简化理解为三层：

```text
进程 fd 表
  fd -> 打开文件对象 file
        -> inode / socket / pipe 等内核对象
```

进程 fd 表：

- 每个进程独立。
- fd 是数组下标或类似索引。
- `open`、`socket` 返回最小可用 fd。

打开文件对象：

- 记录当前文件偏移。
- 记录打开模式和 flags。
- 记录引用计数。

inode 或 socket：

- inode 表示文件系统对象。
- socket 表示网络连接或监听端点。

多个 fd 可能指向同一个打开文件对象，例如 `dup`。多个打开文件对象也可能指向同一个 inode，例如同一个文件被多次 open。

## 普通文件 fd

打开文件：

```c
int fd = open("/tmp/a.log", O_WRONLY | O_CREAT | O_APPEND, 0644);
write(fd, buf, len);
close(fd);
```

特点：

- 读写可能经过 page cache。
- 文件偏移记录在打开文件对象中。
- `O_APPEND` 保证追加语义。
- `fsync` 控制刷盘。

Java 对应：

```java
try (FileInputStream in = new FileInputStream("/tmp/a.txt")) {
    byte[] buf = in.readAllBytes();
}
```

`try-with-resources` 会自动关闭底层 fd。

## socket fd

socket 也是 fd。网络系统调用通过 socket fd 操作连接。

服务端 TCP 流程：

```text
socket()
  -> bind()
  -> listen()
  -> accept() 返回 connected socket fd
  -> read/write 或 recv/send
  -> close()
```

监听 socket 和连接 socket 是不同 fd：

- 监听 fd 用于接收新连接。
- `accept` 每成功一次返回一个新的连接 fd。
- 每个 TCP 连接通常在服务端占一个 socket fd。

Java NIO 示例：

```java
try (ServerSocketChannel server = ServerSocketChannel.open()) {
    server.bind(new InetSocketAddress(8080));
    try (SocketChannel client = server.accept()) {
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        client.read(buffer);
    }
}
```

## fd 限制

Linux 对 fd 有多层限制：

- 进程级限制：`ulimit -n`，软限制和硬限制。
- 系统级最大打开文件数：`/proc/sys/fs/file-max`。
- 当前系统已分配文件对象：`/proc/sys/fs/file-nr`。
- systemd service 的 `LimitNOFILE`。
- 容器运行参数和基础镜像配置。

查看当前 shell：

```bash
ulimit -n
ulimit -Hn
```

查看进程限制：

```bash
cat /proc/<pid>/limits | grep "open files"
```

## 命令和示例

### 查看进程 fd

```bash
ls -l /proc/<pid>/fd | head
ls /proc/<pid>/fd | wc -l
```

查看 fd 信息：

```bash
readlink /proc/<pid>/fd/<fd>
cat /proc/<pid>/fdinfo/<fd>
```

### lsof

```bash
lsof -p <pid>
lsof -p <pid> | wc -l
lsof -p <pid> -a -iTCP
lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -nr | head
```

常见 TYPE：

- `REG`：普通文件。
- `DIR`：目录。
- `IPv4` / `IPv6`：网络 socket。
- `FIFO`：管道。
- `anon_inode`：epoll、eventfd 等匿名 inode。

### ss

```bash
ss -antp | grep <pid>
ss -lntp
ss -ant state established
```

查看 socket 状态：

- `LISTEN`：监听。
- `ESTAB`：已建立连接。
- `TIME-WAIT`：主动关闭后等待。
- `CLOSE-WAIT`：对端关闭，本端未关闭。
- `SYN-SENT`：正在连接。

## Java 中 fd 来源

常见 fd 来源：

- HTTP/TCP 连接。
- 数据库连接。
- Redis 连接。
- Kafka、MQ 连接。
- 日志文件。
- 配置文件。
- jar、class 文件加载。
- 临时文件。
- NIO Selector 的 epoll fd。
- pipe、eventfd。

一个高并发 Java 服务 fd 数通常由以下因素决定：

- 入站连接数。
- 出站连接池大小。
- 下游数量。
- 日志和文件句柄。
- Selector、epoll、eventfd。
- 临时文件和 mmap 文件。

## 线上排查

### Too many open files

异常可能包括：

```text
java.io.FileNotFoundException: ... (Too many open files)
java.net.SocketException: Too many open files
accept failed: Too many open files
```

排查：

```bash
cat /proc/<pid>/limits | grep "open files"
ls /proc/<pid>/fd | wc -l
lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -nr
lsof -p <pid> -a -iTCP | awk '{print $9,$10}' | sort | uniq -c | sort -nr | head
```

判断：

- 如果 socket fd 多，看连接池、长连接、CLOSE_WAIT。
- 如果 REG 文件多，看日志、临时文件、文件流未关闭。
- 如果 anon_inode 多，看 epoll/eventfd 是否重复创建。

### 连接数与 fd 数

入站一个 TCP 连接会占服务端一个 socket fd。出站连接到 MySQL、Redis、HTTP 下游也各占 fd。

不要只按 QPS 估算 fd，要按并发连接数、长连接池、下游数量和实例数估算。

## 常见追问

### fd 是全局唯一的吗

不是。fd 是进程内句柄。不同进程的 fd 3 可以指向完全不同的内核对象。

### socket fd 和文件 fd 有什么区别

从进程接口看都是 fd，都可以传给 `read/write/close` 等系统调用。但内核对象不同，socket 对应网络协议栈状态，普通文件对应文件系统 inode 和页缓存。

### close(fd) 做了什么

`close` 从进程 fd 表移除引用，减少打开文件对象引用计数。引用计数归零后，内核释放打开文件对象；如果是 socket，还会触发协议关闭流程。

### 为什么 accept 返回新 fd

监听 fd 代表监听端口，连接 fd 代表具体客户端连接。一个监听 fd 可以 accept 出很多连接 fd。

### fd 耗尽为什么会影响 accept

`accept` 需要为新连接分配 fd。进程 fd 达到上限后无法分配，accept 失败，服务端不能接收新连接。

## 易错点

- 认为 fd 是系统全局唯一。
- 忘记 socket、pipe、epoll 也是 fd。
- 只调大 `ulimit`，不找 fd 泄漏根因。
- 忽略 `CLOSE_WAIT` 代表本端未关闭连接。
- 估算 fd 时只看入站连接，忽略出站连接池和日志文件。
- 以为连接关闭后 fd 一定马上消失，忽略引用计数和应用未 close。

## 自检清单

- 能否解释 fd、打开文件对象、inode/socket 的关系。
- 能否说明监听 socket fd 和连接 socket fd 的区别。
- 能否用 `/proc/<pid>/fd`、`lsof`、`ss` 查看 fd。
- 能否解释 `Too many open files` 的影响面。
- 能否估算 Java 服务 fd 来源。
- 能否区分 fd 耗尽、连接池耗尽和端口耗尽。
