# 用户态、内核态、系统调用

## 面试定位

用户态、内核态、系统调用是理解 IO、线程调度、内存管理和网络调用的基础。Java 后端面试中，它经常和 `read/write`、阻塞 IO、上下文切换、零拷贝、`strace` 排查一起问。

回答目标：

- 说清为什么要区分用户态和内核态。
- 说清用户程序如何请求内核能力。
- 能把 Java API 和底层系统调用关联起来。
- 能解释系统调用为什么有成本。
- 能用 `strace` 看线上进程做了什么系统调用。

## 核心概念

操作系统把 CPU 执行权限分为不同特权级。常见表达是：

- 用户态：普通应用程序运行的状态，权限受限。
- 内核态：操作系统内核运行的状态，权限高，可以访问硬件和内核数据结构。

用户态程序不能直接操作磁盘、网卡、物理内存、进程调度器等资源，必须通过系统调用向内核发起请求。

系统调用是用户态进入内核态的受控入口，例如：

- 文件：`open`、`read`、`write`、`close`。
- 网络：`socket`、`bind`、`listen`、`accept`、`connect`、`sendto`、`recvfrom`。
- 进程线程：`clone`、`fork`、`execve`、`wait`、`futex`。
- 内存：`mmap`、`brk`、`munmap`、`mprotect`。
- IO 多路复用：`select`、`poll`、`epoll_wait`。
- 时间：`clock_gettime`、`nanosleep`。

## 为什么要区分用户态和内核态

核心原因是保护和隔离：

- 防止应用直接破坏内核内存。
- 防止应用直接操作硬件造成系统崩溃。
- 防止应用绕过权限控制访问其他进程或文件。
- 统一管理 CPU、内存、文件、网络等共享资源。

如果用户程序可以直接执行任意硬件指令，一个普通 bug 就可能破坏整个机器。用户态和内核态的隔离把故障限制在进程内部，并让内核统一做权限检查和资源调度。

## 系统调用流程

以 `read(fd, buf, size)` 为例：

```text
用户代码调用 read
  -> C 库或 JDK native 方法准备参数
  -> 执行 syscall 指令陷入内核
  -> CPU 从用户态切到内核态
  -> 内核根据系统调用号找到处理函数
  -> 校验 fd、权限、用户缓冲区地址
  -> 从文件、socket 或页缓存读取数据
  -> 必要时阻塞当前线程，等待磁盘或网络数据
  -> 拷贝数据到用户缓冲区
  -> 返回用户态
```

系统调用的成本来自：

- 用户态和内核态切换。
- 参数校验和权限检查。
- 用户空间和内核空间数据拷贝。
- 可能触发线程阻塞和调度。
- CPU 缓存、TLB、分支预测等微架构影响。

所以高性能网络和存储系统会尽量减少系统调用次数、减少拷贝、批量处理。

## Java API 与系统调用关系

Java 代码本身运行在用户态，但很多 JDK API 底层会通过 native 方法调用系统调用。

| Java 操作 | 可能涉及的系统调用 |
| --- | --- |
| `new FileInputStream(path)` | `openat` |
| `InputStream.read()` | `read` |
| `OutputStream.write()` | `write` |
| `SocketChannel.open()` | `socket` |
| `ServerSocketChannel.bind()` | `bind`、`listen` |
| `ServerSocketChannel.accept()` | `accept` |
| `Selector.select()` | `epoll_wait` 或 `poll` |
| `Thread.sleep()` | `nanosleep` 或 futex 相关等待 |
| `LockSupport.park()` | `futex` |
| `FileChannel.map()` | `mmap` |

不同 JDK、不同 Linux 版本、不同参数下细节会变化，排查时以 `strace`、`perf`、JDK 源码和实际环境为准。

## 用户态缓冲和内核态缓冲

一次普通文件读取可能涉及：

```text
磁盘 -> 内核页缓存 -> 用户态 byte[] / ByteBuffer -> Java 对象处理
```

网络读取可能涉及：

```text
网卡 DMA -> 内核 socket 接收缓冲区 -> 用户态 ByteBuffer -> Java 业务对象
```

这里有两个常见优化方向：

- 减少用户态和内核态之间的数据拷贝，例如 `sendfile`、`mmap`。
- 减少系统调用次数，例如批量读写、缓冲、事件驱动。

## 命令和示例

### strace 查看系统调用

跟踪进程：

```bash
strace -p <pid>
```

统计系统调用耗时：

```bash
strace -c -p <pid>
```

只看网络相关：

```bash
strace -f -e trace=network -p <pid>
```

只看文件读写：

```bash
strace -f -e trace=openat,read,write,close -p <pid>
```

输出带时间：

```bash
strace -tt -T -f -p <pid>
```

常见现象：

- 大量 `futex`：锁等待、条件等待、线程 park/unpark。
- 大量 `epoll_wait`：事件循环等待 IO，可能正常。
- 大量短小 `read/write`：可能缺少缓冲或批量。
- `connect` 超时：下游网络或服务不可达。
- `accept` 返回 `EMFILE`：文件描述符耗尽。

### 查看系统调用文档

```bash
man 2 read
man 2 epoll_wait
man 2 futex
```

`man 2` 表示系统调用章节，`man 3` 多数是 C 库函数。

## 线上排查

### 请求卡住

可以用线程栈和系统调用结合分析：

```bash
jstack -l <pid> > /tmp/jstack.txt
strace -tt -T -f -p <pid>
```

判断：

- Java 栈在 socket read，`strace` 看到 `recvfrom` 长时间不返回，通常是网络读等待。
- Java 栈在锁等待，`strace` 大量 `futex`，通常是锁或条件等待。
- Java 栈在文件 IO，`strace` 看到 `read`、`write` 慢，可能是磁盘或文件系统问题。

### CPU 高

CPU 高不一定系统调用多。可以先用：

```bash
top -H -p <pid>
perf top -p <pid>
strace -c -p <pid>
```

如果 `strace -c` 显示系统调用占用大量时间，重点看系统调用类型；如果系统调用不多，更多是用户态 CPU 热点，需要火焰图或 Java 栈分析。

### IO 慢

```bash
iostat -x 1
pidstat -d -p <pid> 1
strace -tt -T -e trace=read,write,fsync -p <pid>
```

如果 `fsync`、`write` 慢，要结合磁盘 util、await、应用日志和文件系统状态判断。

## 常见追问

### 用户态能不能直接读磁盘

不能。用户程序通过系统调用请求内核读磁盘。即使用了 mmap，建立映射和缺页处理也由内核完成。

### 系统调用和函数调用有什么区别

普通函数调用仍在用户态，主要是栈帧和跳转；系统调用要从用户态切到内核态，进入内核受控入口，执行权限检查和内核逻辑，成本更高。

### 内核态一定比用户态快吗

不是。内核态权限高，但进入内核有切换成本。能在用户态安全完成的事情通常不应频繁进入内核。

### Java 为什么也需要系统调用

Java 应用运行在 JVM 上，但 JVM 本身也是用户态进程。文件、网络、线程阻塞、内存映射等都需要通过操作系统能力实现。

### futex 是什么

`futex` 是 Linux 的快速用户态互斥机制。无竞争时尽量在用户态完成，竞争或等待时通过系统调用进入内核睡眠和唤醒。Java 锁、AQS、park/unpark 的底层等待常能看到 futex。

## 易错点

- 认为 Java 层没有系统调用。
- 把系统调用和上下文切换完全等同。系统调用会发生特权级切换，但不一定发生线程调度切换。
- 认为 `epoll_wait` 出现在 `strace` 中就是异常，事件循环空闲等待时很常见。
- 线上直接长时间 `strace -f`，可能引入明显开销。
- 看到大量 `futex` 就断定死锁，还需要结合线程 dump 判断。
- 忽略用户态缓冲和内核页缓存的区别。

## 自检清单

- 能否解释用户态和内核态为什么要隔离。
- 能否画出一次 `read` 的系统调用路径。
- 能否说出 Java 常见 API 对应的底层系统调用。
- 能否解释系统调用成本来自哪里。
- 能否用 `strace -c`、`strace -tt -T` 分析系统调用耗时。
- 能否区分系统调用、线程上下文切换和普通函数调用。
