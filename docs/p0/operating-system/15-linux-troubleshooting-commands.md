# top、ps、lsof、ss、netstat、vmstat、iostat、strace、tcpdump

## 面试定位

Linux 排障命令是 Java 后端线上基本功。面试官一般不满足于“我会 top”，而是会追问 CPU 高、线程阻塞、fd 泄漏、连接异常、IO 慢、系统调用慢、网络包异常时具体怎么定位。

回答目标：

- 知道每个命令看什么。
- 能按场景组合命令。
- 能解释关键字段。
- 知道哪些命令线上有开销。
- 能把 OS 现象和 Java 线程栈、JVM 工具关联起来。

## 总览

| 命令 | 主要用途 | 高频场景 |
| --- | --- | --- |
| `top` | CPU、内存、线程概览 | CPU 高、负载高 |
| `ps` | 进程和线程快照 | 找进程、看启动参数、线程数 |
| `lsof` | 打开的 fd | fd 泄漏、文件占用、socket |
| `ss` | socket 状态 | 连接数、监听端口、CLOSE_WAIT |
| `netstat` | 网络状态，较老 | 兼容老环境 |
| `vmstat` | CPU、内存、swap、上下文切换 | 系统整体负载 |
| `iostat` | 磁盘 IO | 磁盘慢、await 高 |
| `strace` | 系统调用跟踪 | 调用卡在哪里、fd 操作 |
| `tcpdump` | 抓包 | 网络协议、连接、重传排查 |

## top

常用：

```bash
top
top -p <pid>
top -H -p <pid>
```

关键字段：

- `load average`：系统负载。
- `%Cpu(s) us`：用户态 CPU。
- `%Cpu(s) sy`：内核态 CPU。
- `%Cpu(s) wa`：IO wait。
- `%Cpu(s) id`：空闲。
- `VIRT`：虚拟内存。
- `RES`：常驻内存。
- `SHR`：共享内存。

`top -H -p <pid>` 看进程内线程 CPU。Java CPU 高定位链路：

```bash
top -H -p <pid>
printf "%x\n" <tid>
jstack -l <pid> > /tmp/jstack.txt
grep -n "nid=0x<hex>" -A 50 /tmp/jstack.txt
```

判断：

- 单线程 CPU 高：死循环、热点计算、正则、序列化等。
- 多线程 CPU 高：流量大、GC、锁自旋、线程池过大。
- `sy` 高：系统调用、网络栈、锁、内核开销。
- `wa` 高：磁盘或存储等待。

## ps

查进程：

```bash
ps -ef | grep java
ps -fp <pid>
```

看完整启动参数：

```bash
ps -ww -fp <pid>
```

看线程：

```bash
ps -T -p <pid>
ps -eLf | grep <pid>
```

自定义输出：

```bash
ps -o pid,ppid,stat,etime,pcpu,pmem,nlwp,cmd -p <pid>
```

字段：

- `STAT`：进程状态。
- `NLWP`：线程数。
- `ETIME`：运行时长。
- `%CPU`、`%MEM`：资源占用。

## lsof

查看进程打开的 fd：

```bash
lsof -p <pid>
lsof -p <pid> | wc -l
```

按类型聚合：

```bash
lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -nr
```

看 TCP：

```bash
lsof -p <pid> -a -iTCP
lsof -iTCP:<port> -sTCP:LISTEN
```

查 deleted 文件：

```bash
lsof -p <pid> | grep deleted
```

典型用途：

- fd 泄漏。
- 端口被谁占用。
- 文件被哪个进程打开。
- 删除文件后磁盘空间未释放。
- socket 连接来源和状态。

注意：`lsof` 在 fd 很多时可能较慢，生产要控制频率。

## ss

`ss` 是查看 socket 的首选工具，通常比 `netstat` 更快。

监听端口：

```bash
ss -lntp
```

TCP 连接：

```bash
ss -antp
ss -ant state established
ss -ant state close-wait
```

按端口：

```bash
ss -antp 'sport = :8080'
ss -antp 'dport = :3306'
```

汇总：

```bash
ss -s
```

常见状态：

- `LISTEN`：监听。
- `ESTAB`：连接已建立。
- `SYN-SENT`：主动连接中。
- `SYN-RECV`：收到 SYN，等待握手完成。
- `CLOSE-WAIT`：对端关闭，本端未关闭。
- `TIME-WAIT`：主动关闭后等待。

判断：

- `CLOSE-WAIT` 多：应用未关闭连接。
- `SYN-SENT` 多：下游不可达、网络或防火墙问题。
- `SYN-RECV` 多：握手积压或攻击。
- `ESTAB` 远超预期：连接池或长连接异常。

## netstat

老环境可能没有 `ss`，用 `netstat`：

```bash
netstat -antp
netstat -lntp
netstat -s
```

连接状态统计：

```bash
netstat -ant | awk '{print $6}' | sort | uniq -c | sort -nr
```

如果有 `ss`，优先用 `ss`。

## vmstat

系统整体观察：

```bash
vmstat 1
```

关键字段：

- `r`：可运行队列长度。
- `b`：不可中断睡眠任务数。
- `swpd`：swap 使用。
- `free`：空闲内存。
- `si`：swap in。
- `so`：swap out。
- `bi`：块设备读。
- `bo`：块设备写。
- `in`：中断。
- `cs`：上下文切换。
- `us`：用户态 CPU。
- `sy`：内核态 CPU。
- `id`：空闲。
- `wa`：IO wait。

判断：

- `r` 长期大于 CPU 核数很多：CPU 排队。
- `b` 高：可能 IO 等不可中断等待。
- `si/so` 持续非 0：swap 抖动。
- `cs` 很高：线程切换频繁、锁竞争或线程过多。
- `wa` 高：磁盘或存储慢。
- `sy` 高：内核开销高，可能系统调用或网络栈压力。

## iostat

查看磁盘 IO：

```bash
iostat -x 1
```

关键字段随版本不同略有差异，常见：

- `r/s`、`w/s`：每秒读写请求。
- `rkB/s`、`wkB/s`：吞吐。
- `await`：平均 IO 等待时间。
- `r_await`、`w_await`：读写等待。
- `%util`：设备忙碌程度。
- `aqu-sz` 或 `avgqu-sz`：平均队列长度。

判断：

- `%util` 接近 100 且 await 高：磁盘繁忙。
- await 高但 util 不高：可能存储网络、虚拟化、队列或瞬时抖动。
- 写高：日志、刷盘、临时文件、数据库。
- 读高：冷数据、page cache miss、批处理。

配合进程：

```bash
pidstat -d -p <pid> 1
```

## strace

跟踪系统调用：

```bash
strace -p <pid>
strace -f -p <pid>
strace -tt -T -f -p <pid>
strace -c -p <pid>
```

按类型过滤：

```bash
strace -f -e trace=network -p <pid>
strace -f -e trace=openat,read,write,close -p <pid>
strace -f -e trace=futex -p <pid>
strace -f -e trace=epoll_wait,epoll_ctl -p <pid>
```

用途：

- 看进程卡在哪个系统调用。
- 看系统调用耗时。
- 看是否频繁打开文件。
- 看 socket connect/read/write。
- 看 futex 等待。
- 看 epoll_wait 是否空转。

风险：

- `strace` 会明显影响被跟踪进程性能。
- `-f` 跟踪所有线程开销更大。
- 生产要短时间、带过滤、保留输出后及时停止。

## tcpdump

抓包：

```bash
tcpdump -i eth0 host 203.0.113.10
tcpdump -i eth0 port 8080
tcpdump -i eth0 tcp and host 203.0.113.10 and port 443
```

保存到文件：

```bash
tcpdump -i eth0 -s 0 -w /tmp/capture.pcap host 203.0.113.10 and port 8080
```

常用参数：

- `-i`：网卡。
- `-nn`：不解析域名和服务名。
- `-s 0`：抓完整包。
- `-w`：写 pcap 文件。
- `-r`：读取 pcap。
- `host`、`port`、`tcp`、`udp`：过滤条件。

排查场景：

- 三次握手是否成功。
- 是否有重传。
- 谁主动发 FIN/RST。
- 请求是否发出。
- 响应是否回来。
- TLS 握手是否中断。

示例：

```bash
tcpdump -i eth0 -nn 'tcp and host 203.0.113.10 and port 8080'
```

注意：

- 抓包可能包含敏感数据。
- 高流量机器抓包要加过滤条件。
- pcap 文件可能很大。
- 容器环境要确认抓宿主机网卡还是容器网卡。

## 场景组合

### CPU 高

```bash
top
top -H -p <pid>
printf "%x\n" <tid>
jstack -l <pid> > /tmp/jstack.txt
grep -n "nid=0x<hex>" -A 50 /tmp/jstack.txt
```

补充：

```bash
vmstat 1
pidstat -u -t -p <pid> 1
```

### 线程多或上下文切换高

```bash
cat /proc/<pid>/status | egrep "Threads|VmRSS"
pidstat -w -t -p <pid> 1
vmstat 1
jstack -l <pid> > /tmp/jstack.txt
```

### fd 泄漏

```bash
cat /proc/<pid>/limits | grep "open files"
ls /proc/<pid>/fd | wc -l
lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -nr
lsof -p <pid> -a -iTCP
```

### 网络连接异常

```bash
ss -lntp
ss -antp | grep <pid>
ss -s
tcpdump -i eth0 -nn 'host <ip> and port <port>'
```

### 磁盘 IO 慢

```bash
vmstat 1
iostat -x 1
pidstat -d -p <pid> 1
strace -tt -T -e trace=read,write,fsync -p <pid>
```

### 系统调用慢

```bash
strace -c -p <pid>
strace -tt -T -f -p <pid>
```

结合 Java：

```bash
jstack -l <pid> > /tmp/jstack.txt
```

## 常见追问

### load average 高一定是 CPU 高吗

不一定。Linux load 包括可运行任务和不可中断睡眠任务。IO 等待多时 load 也会高。要结合 `top`、`vmstat` 的 `r`、`b`、`wa` 判断。

### top 里 CPU 的 us、sy、wa 怎么理解

`us` 是用户态 CPU，常见业务代码、JIT 后代码。`sy` 是内核态 CPU，常见系统调用、网络栈、锁、内核处理。`wa` 是等待 IO 的时间比例。

### ss 和 netstat 选哪个

优先 `ss`，速度更快，信息更丰富。老环境没有 `ss` 时用 `netstat`。

### strace 会不会影响线上

会。它拦截系统调用，尤其 `-f` 多线程和无过滤跟踪开销明显。生产应短时间、带过滤、必要时在低峰或单实例上执行。

### tcpdump 抓不到包说明什么

可能是过滤条件错、网卡错、容器网络位置错、流量没到本机、防火墙或路由问题。抓不到包不能直接断定应用没发，要确认抓包点。

## 易错点

- 只看 top，不看线程级 CPU 和 Java 栈。
- load 高就断定 CPU 高。
- fd 泄漏只调大 ulimit。
- CLOSE_WAIT 和 TIME_WAIT 混淆。
- `strace -f` 长时间挂在线上核心实例。
- tcpdump 不加过滤导致 pcap 巨大。
- iostat 只看 util，不看 await、队列和业务延迟。
- 容器环境忽略 cgroup 限制和网卡命名空间。

## 自检清单

- 能否用 top + jstack 定位 Java 高 CPU 线程。
- 能否用 lsof 和 ss 排查 fd 与连接问题。
- 能否解释 vmstat 的 r、b、cs、si、so、wa。
- 能否用 iostat 判断磁盘 IO 压力。
- 能否安全使用 strace 观察系统调用。
- 能否用 tcpdump 判断请求是否发出、响应是否回来、谁主动断开。
- 能否按 CPU、内存、fd、网络、磁盘场景组合命令。
