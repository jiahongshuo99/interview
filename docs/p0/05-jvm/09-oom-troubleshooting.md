# OOM 类型与排查

## 面试定位

OOM 排查是 Java 后端面试中最贴近线上经验的 JVM 题。面试官通常会问：Java 里有哪些 OOM，堆 OOM 怎么查，Metaspace OOM 和 Direct Memory OOM 怎么区别，为什么容器 OOMKilled 没有 Java 异常，线上 OOM 前怎么保留现场。

回答时要先分类，再讲证据链：

1. OOM 不是只有 Java heap space。
2. 先看异常类型、GC 日志、JVM 参数、容器事件和系统日志。
3. 堆 OOM 优先保留 heap dump，分析大对象和 GC Roots。
4. 非堆 OOM 要分别看 Metaspace、Direct Memory、线程栈、native memory。
5. 解决方案要回到对象生命周期、容量限制、资源释放和参数边界。

## 常见 OOM 类型

| 类型 | 典型错误 | 常见原因 |
| --- | --- | --- |
| 堆 OOM | `java.lang.OutOfMemoryError: Java heap space` | 对象过多、内存泄漏、堆太小 |
| GC 开销过大 | `GC overhead limit exceeded` | GC 花大量时间但回收很少 |
| 数组过大 | `Requested array size exceeds VM limit` | 创建超大数组 |
| 元空间 OOM | `Metaspace` | 动态类过多、类加载器泄漏 |
| 直接内存 OOM | `Direct buffer memory` | NIO/Netty direct buffer 未释放或上限不足 |
| 无法创建线程 | `unable to create new native thread` | 线程过多、栈太大、系统限制 |
| 本地内存 OOM | `Native memory allocation failed` | 堆外、JNI、malloc、容器内存不足 |
| 容器 OOMKilled | 进程被杀，无 Java OOM | RSS 超过 cgroup 限制 |

## 堆 OOM

典型错误：

```text
java.lang.OutOfMemoryError: Java heap space
```

常见原因：

- 无界集合，例如静态 Map、List、队列。
- 本地缓存没有容量或过期策略。
- 一次性加载大量数据。
- 批处理分页过大。
- 大对象，例如大数组、大字符串、大 JSON。
- ThreadLocal 在线程池中未清理。
- 对象生命周期设计不合理。
- 堆配置太小。

优先保留现场参数：

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/to/dumps
-XX:ErrorFile=/path/to/hs_err_pid%p.log
```

排查步骤：

1. 确认 OOM 类型是 Java heap space。
2. 查看 OOM 前 GC 日志，判断 Full GC 后是否回收不动。
3. 分析 heap dump。
4. 看 Dominator Tree 和 Retained Size。
5. 找 GC Roots 引用链。
6. 结合代码确认持有者为什么没有释放。
7. 修复泄漏或优化容量，再通过压测验证。

常用命令：

```bash
jcmd <pid> GC.heap_dump /tmp/heap.hprof
jmap -dump:format=b,file=/tmp/heap.hprof <pid>
jcmd <pid> GC.class_histogram
jmap -histo <pid>
```

注意：进程已经 OOM 时未必还能稳定执行命令，最好提前配置自动 dump。

## GC overhead limit exceeded

典型错误：

```text
java.lang.OutOfMemoryError: GC overhead limit exceeded
```

含义：JVM 花了大量时间做 GC，但每次只回收很少空间。它通常是堆接近耗尽的保护性错误。

常见原因：

- 内存泄漏。
- 堆太小。
- 对象存活率过高。
- 业务短时间加载过多数据。

处理方式：

- 按堆 OOM 思路分析 dump。
- 不要只通过 `-XX:-UseGCOverheadLimit` 掩盖问题。
- 如果确实是正常大内存任务，重新评估堆大小和批处理粒度。

## Requested array size exceeds VM limit

典型错误：

```text
java.lang.OutOfMemoryError: Requested array size exceeds VM limit
```

含义：申请的数组长度超过 JVM 能支持的最大限制，通常与堆剩余空间无关。

常见原因：

- 错误计算容量，例如溢出。
- 一次性构造超大 byte 数组、字符串、集合。
- 文件或响应体未做大小限制。

优化：

- 流式处理。
- 分块处理。
- 校验输入大小。
- 防止 `int` 溢出导致容量异常。

## Metaspace OOM

典型错误：

```text
java.lang.OutOfMemoryError: Metaspace
```

常见原因：

- 动态生成类过多。
- CGLIB、Byte Buddy、Javassist、Groovy、脚本引擎使用不当。
- 热部署旧类加载器无法卸载。
- 插件类加载器泄漏。
- `MaxMetaspaceSize` 设置过小。

排查命令：

```bash
jcmd <pid> VM.classloader_stats
jcmd <pid> GC.class_histogram
jcmd <pid> VM.native_memory summary
jstat -gcutil <pid> 1000 10
```

排查重点：

- 类加载器数量是否异常增长。
- 某些动态类名前缀是否大量出现。
- 是否频繁创建代理类。
- 线程上下文类加载器、静态变量、ThreadLocal 是否持有旧类加载器。

参数：

```bash
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
```

注意：调大 `MaxMetaspaceSize` 只是缓解，类加载器泄漏必须修引用链。

## Direct Memory OOM

典型错误：

```text
java.lang.OutOfMemoryError: Direct buffer memory
```

常见原因：

- NIO `ByteBuffer.allocateDirect` 大量分配。
- Netty direct buffer 池配置不合理。
- 直接内存释放依赖 Cleaner，回收不及时。
- 堆外内存上限过小。
- 容器内存不足。

相关参数：

```bash
-XX:MaxDirectMemorySize=<size>
```

排查：

```bash
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.native_memory detail
jcmd <pid> VM.flags
```

需要先开启 NMT：

```bash
-XX:NativeMemoryTracking=summary
# 或 detail，开销更高
```

Netty 场景关注：

- `PlatformDependent.usedDirectMemory()`。
- pooled direct buffer 指标。
- 是否有 ByteBuf 未 release。
- 是否开启 leak detector。

## unable to create new native thread

典型错误：

```text
java.lang.OutOfMemoryError: unable to create new native thread
```

常见原因：

- 线程数过多。
- 每个线程栈过大。
- 系统线程数限制。
- 容器 PID 限制。
- native memory 不足。

排查：

```bash
jcmd <pid> Thread.print
jstack <pid>
ps -eLf | grep <pid> | wc -l
ulimit -u
ulimit -s
```

优化：

- 收敛线程池。
- 设置合理队列和拒绝策略。
- 降低 `-Xss`，但不能盲目过小。
- 检查线程泄漏。
- 检查容器 pids limit。

## Native Memory OOM

典型表现：

```text
Out of Memory Error (os_linux.cpp:...)
Native memory allocation failed
```

可能来源：

- Direct Memory。
- Metaspace。
- 线程栈。
- JIT Code Cache。
- GC 内部结构。
- JNI 或 native 库。
- mmap 文件映射。
- 压缩、加密、图像处理等 native 分配。

排查工具：

```bash
jcmd <pid> VM.native_memory summary
pmap -x <pid>
top -p <pid>
cat /proc/<pid>/status
```

macOS 或本地环境可以使用 `vmmap` 等系统工具，生产 Linux 更常用 NMT、pmap、proc 文件。

## 容器 OOMKilled

容器 OOMKilled 可能没有 Java OOM 异常，因为进程是被内核或容器运行时直接杀掉。

判断方式：

- Kubernetes Pod 状态 `OOMKilled`。
- `Last State: Terminated Reason: OOMKilled`。
- 容器 RSS 达到 limit。
- JVM 日志里没有 `OutOfMemoryError`。

常见原因：

- `Xmx` 太接近容器 limit。
- Direct Memory、Metaspace、线程栈等非堆内存占用高。
- native 库泄漏。
- sidecar 或同容器进程占用内存。

优化：

- 堆只占容器内存的一部分。
- 限制直接内存和元空间。
- 控制线程数。
- 开启容器内存监控。
- 使用 `MaxRAMPercentage` 时确认 JVM 版本和识别情况。

## 线上 OOM 预案

建议配置：

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump
-XX:ErrorFile=/data/logs/hs_err_pid%p.log
-Xlog:gc*,safepoint:file=/data/logs/gc.log:time,uptime,level,tags:filecount=10,filesize=100M
```

可选动作：

```bash
-XX:OnOutOfMemoryError="sh /path/oom-hook.sh %p"
```

注意：

- dump 文件可能很大，要保证磁盘空间。
- OOM hook 不应做复杂阻塞操作。
- 生产环境导出 dump 可能带有敏感数据，要控制权限。
- 堆很大时 dump 会造成额外停顿或失败。

## 常见追问

### OOM 后服务还能继续运行吗？

不可靠。部分 OOM 可能被捕获后局部恢复，但 JVM 或业务状态可能已经不稳定。生产系统通常应快速保留现场并重启实例，同时修复根因。

### `jmap` 导 dump 和自动 dump 哪个更好？

自动 dump 能在 OOM 发生时保留现场；`jmap` 适合进程仍存活且允许手动操作时使用。大堆线上手动 dump 要谨慎。

### 堆 OOM 一定是内存泄漏吗？

不一定。也可能是正常数据量超过容量、堆设置太小、批量任务过大、瞬时峰值。是否泄漏要看 Full GC 后对象是否持续不可下降，以及引用链是否业务上无效。

### Full GC 后对象很多，为什么不能回收？

因为这些对象仍然从 GC Roots 可达。GC 判断的是可达性，不判断业务是否还需要。

### 增大堆能解决 OOM 吗？

如果是容量不足，可以缓解；如果是泄漏，只会延迟 OOM。必须结合 dump 和对象增长趋势判断。

## 易错点

- 一看到 OOM 就只查堆。
- 没有提前配置 `HeapDumpOnOutOfMemoryError`。
- 只看 Shallow Size，不看 Retained Size。
- 对容器 OOMKilled 仍然只查 Java 异常。
- 用禁用 GC overhead limit 掩盖问题。
- Metaspace OOM 只调大参数，不查类加载器泄漏。
- 线程 OOM 只调大内存，不收敛线程池。

## 项目表达

可以这样说：

> 我排查 OOM 会先按错误类型分类。堆 OOM 会优先拿 OOM 时的 heap dump 和 GC 日志，看 Full GC 后老年代是否下降，再用 MAT 的 Dominator Tree 找 Retained Size 最大的对象和 GC Roots 路径。之前有一次是静态缓存没有淘汰策略，Full GC 后仍被静态 Map 强引用，修复时改为有容量和过期策略的缓存，并补充缓存大小监控。

容器场景：

> 也遇到过没有 Java OOM 但 Pod 被 OOMKilled 的问题。最后发现堆配置接近容器 limit，Netty direct memory 和线程栈叠加后 RSS 超限。解决方式是降低 `Xmx` 占比，限制 direct memory，收敛线程池并补齐容器 RSS 告警。

## 自检清单

- 能否列出堆、Metaspace、Direct Memory、线程、Native、容器 OOM 的区别？
- 能否说明堆 OOM 的 dump 分析流程？
- 能否解释 Full GC 后内存不下降意味着什么？
- 能否排查 Metaspace OOM 的类加载器泄漏？
- 能否说明 Direct Memory OOM 和容器 OOMKilled 的关系？
- 能否给出线上 OOM 预案参数？
