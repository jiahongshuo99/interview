# jstack、jmap、jstat、jcmd、arthas

## 面试定位

JVM 工具题考察的是线上排查基本功。面试官常问：CPU 高怎么用 `jstack`，内存泄漏怎么用 `jmap`，GC 怎么用 `jstat`，为什么推荐 `jcmd`，Arthas 能做什么，哪些命令生产环境要慎用。

回答时要按场景组织：

- 线程和死锁：`jstack`、`jcmd Thread.print`、Arthas `thread`。
- 堆和对象：`jmap`、`jcmd GC.heap_dump`、Arthas `heapdump`。
- GC 观察：`jstat`、GC 日志、JFR。
- JVM 运行信息：`jcmd`。
- 动态诊断：Arthas。

## 先理解这些工具是什么

这些工具不是新的 JVM 组件，而是“观察正在运行的 JVM”的诊断入口。它们通常通过进程号 `pid` 附加到目标 Java 进程，读取或触发 JVM 暴露的诊断能力。

可以先把它们分成五类：

1. 线程快照：看每个 Java 线程此刻在执行什么、卡在哪里、拿着什么锁。
2. 堆快照：看堆里有哪些对象、对象数量和大小、对象之间如何引用。
3. 统计计数器：看 GC 次数、GC 时间、各内存区使用率等趋势。
4. 运行配置：看启动参数、系统属性、类加载器、native memory、JFR 等运行信息。
5. 动态诊断：不重启进程，临时观察方法参数、返回值、耗时、调用栈。

几个基础概念：

- `pid`：Java 进程号，很多命令都需要它。
- Thread dump：线程栈快照，适合排查 CPU 高、死锁、阻塞。
- Heap dump：堆内存快照，适合排查内存泄漏和 OOM，但文件大、可能停顿。
- Class histogram：对象直方图，只看各类对象数量和大小，不展示完整引用链。
- JFR：Java Flight Recorder，记录一段时间内的 JVM 事件，适合看 CPU、分配、GC、锁、IO。
- NMT：Native Memory Tracking，看堆外的 JVM native memory，例如 Metaspace、线程栈、Direct Memory、Code Cache。

不要把工具当成互相替代关系。更准确的关系是：

- `jstack` 专门看线程。
- `jmap` 专门看堆和对象。
- `jstat` 专门看 JVM 统计趋势。
- `jcmd` 是统一入口，能做很多 `jstack`、`jmap`、JFR、NMT 相关事情。
- Arthas 更偏在线动态诊断，适合看方法级行为。

## 工具总览

| 工具 | 主要用途 | 常见场景 | 风险 |
| --- | --- | --- | --- |
| `jstack` | 线程栈、死锁 | CPU 高、线程阻塞、死锁 | 输出瞬时状态，需要多次采样 |
| `jmap` | 堆 dump、对象直方图 | 内存泄漏、堆 OOM | dump 和 live histo 可能 STW |
| `jstat` | GC 和类加载统计 | GC 频率、堆使用趋势 | 只有统计值，不给根因 |
| `jcmd` | 综合诊断命令 | 参数、线程、堆、NMT、JFR | 部分命令有停顿或开销 |
| Arthas | 在线动态诊断 | 方法耗时、参数、线程、类加载 | 命令使用不当会影响线上 |

## jstack

`jstack` 用于打印 Java 线程栈，常用于排查：

- CPU 飙高。
- 死锁。
- 线程阻塞。
- 线程池耗尽。
- 请求卡住。

常用命令：

```bash
jstack <pid> > /tmp/jstack.txt
jstack -l <pid> > /tmp/jstack-lock.txt
```

`-l` 会打印锁相关信息，排查死锁和锁竞争时更有用。

### CPU 高定位链路

```bash
top -H -p <pid>
printf "%x\n" <tid>
jstack -l <pid> > /tmp/jstack.txt
grep -n "nid=0x<hex>" -A 40 /tmp/jstack.txt
```

建议连续抓多次：

```bash
jstack -l <pid> > /tmp/jstack1.txt
sleep 5
jstack -l <pid> > /tmp/jstack2.txt
sleep 5
jstack -l <pid> > /tmp/jstack3.txt
```

判断：

- 同一线程多次停在同一业务方法：可能死循环或长耗时计算。
- 大量线程 `BLOCKED`：锁竞争。
- 大量线程等待连接池：下游或资源池瓶颈。
- 出现 `Found one Java-level deadlock`：Java 层死锁。

### 常见线程状态

- `RUNNABLE`：正在运行或等待 OS 调度，也可能在 native IO。
- `BLOCKED`：等待进入 synchronized。
- `WAITING`：无限等待。
- `TIMED_WAITING`：限时等待。
- `NEW`：新建未启动。
- `TERMINATED`：已结束。

## jmap

`jmap` 用于查看堆信息、对象直方图、导出 heap dump。

常用命令：

```bash
jmap -heap <pid>
jmap -histo <pid> | head -50
jmap -histo:live <pid> | head -50
jmap -dump:format=b,file=/tmp/heap.hprof <pid>
```

风险：

- `jmap -dump` 会带来明显停顿，大堆尤其明显。
- `jmap -histo:live` 会触发 Full GC，生产环境慎用。
- dump 文件可能很大，要注意磁盘空间和敏感数据。

推荐：

- OOM 自动 dump 优先。
- 生产手动 dump 前评估流量和实例冗余。
- 很多场景可用 `jcmd` 替代。

Heap dump 分析工具：

- Eclipse MAT。
- JProfiler。
- VisualVM。
- YourKit。

分析重点：

- Dominator Tree。
- Retained Size。
- Path to GC Roots。
- 大集合。
- ThreadLocalMap。
- ClassLoader。

## jstat

`jstat` 用于观察 JVM 统计信息，最常用的是 GC。

常用命令：

```bash
jstat -gc <pid> 1000 10
jstat -gcutil <pid> 1000 10
jstat -gccause <pid> 1000 10
jstat -class <pid> 1000 10
```

`jstat -gcutil` 常见字段：

- `S0`、`S1`：Survivor 使用率。
- `E`：Eden 使用率。
- `O`：Old 使用率。
- `M`：Metaspace 使用率。
- `CCS`：Compressed Class Space 使用率。
- `YGC`：Young GC 次数。
- `YGCT`：Young GC 总耗时。
- `FGC`：Full GC 次数。
- `FGCT`：Full GC 总耗时。
- `GCT`：GC 总耗时。

使用方式：

- 看 YGC/FGC 是否快速增长。
- 看 Old 在 Full GC 后是否下降。
- 看 Metaspace 是否持续增长。
- 配合 GC 日志判断具体原因。

限制：

- 只提供统计，不提供对象引用链。
- 不适合替代 GC 日志。
- 容器和权限环境下可能不可用。

## jcmd

`jcmd` 是 JDK 提供的综合诊断工具，很多场景比 `jmap`、`jstack` 更统一。

查看可用命令：

```bash
jcmd <pid> help
```

常用命令：

```bash
jcmd <pid> VM.command_line
jcmd <pid> VM.flags
jcmd <pid> VM.system_properties
jcmd <pid> Thread.print -l
jcmd <pid> GC.heap_info
jcmd <pid> GC.class_histogram
jcmd <pid> GC.heap_dump /tmp/heap.hprof
jcmd <pid> VM.classloader_stats
jcmd <pid> VM.native_memory summary
```

### JFR

启动 JFR：

```bash
jcmd <pid> JFR.start name=profile settings=profile duration=120s filename=/tmp/profile.jfr
```

查看 JFR：

```bash
jcmd <pid> JFR.check
jcmd <pid> JFR.stop name=profile filename=/tmp/profile.jfr
```

JFR 适合分析：

- CPU 热点。
- 分配热点。
- GC 事件。
- 锁竞争。
- IO。
- 异常频率。

### NMT

Native Memory Tracking 用于观察 JVM native memory。

启动参数：

```bash
-XX:NativeMemoryTracking=summary
# 或 detail
```

查看：

```bash
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.native_memory detail
```

适合排查：

- Direct Memory。
- Metaspace。
- Thread stack。
- Code Cache。
- GC native 结构。
- JNI 或 native 库内存。

注意：NMT 需要启动时开启，`detail` 开销更高。

## Arthas

Arthas 是在线 Java 诊断工具，适合在不重启应用的情况下观察线程、方法调用、类加载、JVM 状态。

常用命令：

```bash
dashboard
thread -n 5
thread -b
jvm
memory
sysprop
vmoption
sc -d com.demo.UserService
sm com.demo.UserService
jad com.demo.UserService
watch com.demo.UserService query '{params, returnObj}' -x 2 -n 5
trace com.demo.UserService query '#cost > 100' -n 5
monitor com.demo.UserService query -c 10
stack com.demo.UserService query -n 5
heapdump /tmp/heap.hprof
```

### thread

用于看线程状态和 CPU：

```bash
thread -n 10
thread <id>
thread -b
```

`thread -b` 可以找当前阻塞其他线程的线程，适合排查锁竞争。

### watch

观察方法入参、返回值、异常：

```bash
watch com.demo.Service method '{params, returnObj, throwExp}' -x 2 -n 5
```

注意：

- 表达式要控制复杂度。
- 必须限制次数 `-n`。
- 高 QPS 方法慎用。
- 不要打印巨大对象。

### trace

观察方法内部调用耗时：

```bash
trace com.demo.Service method '#cost > 100' -n 5
```

适合找慢调用链，但对高频方法要控制范围。

### jad / sc / sm

查看线上实际加载的类和方法：

```bash
sc -d com.demo.Service
sm com.demo.Service
jad com.demo.Service
```

适合排查：

- 线上代码是否是预期版本。
- 类由哪个 ClassLoader 加载。
- 方法签名是否匹配。

## 工具场景速查

### CPU 高

```bash
top -H -p <pid>
printf "%x\n" <tid>
jstack -l <pid>
# 或 Arthas
thread -n 5
profiler start
```

### 死锁

```bash
jstack -l <pid>
jcmd <pid> Thread.print -l
# 或 Arthas
thread -b
```

### Young GC 频繁

```bash
jstat -gcutil <pid> 1000 20
jcmd <pid> JFR.start name=alloc settings=profile duration=120s filename=/tmp/alloc.jfr
```

### Full GC 频繁

```bash
jstat -gccause <pid> 1000 20
jcmd <pid> GC.heap_info
jcmd <pid> GC.heap_dump /tmp/heap.hprof
```

### 堆内存泄漏

```bash
jcmd <pid> GC.class_histogram
jcmd <pid> GC.heap_dump /tmp/heap.hprof
```

### Metaspace 增长

```bash
jcmd <pid> VM.classloader_stats
jstat -gcutil <pid> 1000 20
```

### Direct Memory 或 native memory

```bash
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.flags
```

## 生产环境注意事项

- `jmap -dump`、`jcmd GC.heap_dump` 可能造成明显停顿。
- `jmap -histo:live` 会触发 Full GC，慎用。
- Arthas `watch`、`trace` 高 QPS 方法要限制条件和次数。
- heap dump 可能包含用户数据、密钥、Token，要保护文件权限。
- 大 dump 可能打满磁盘。
- 线上诊断前优先摘流或选择单个实例。
- 对关键链路命令执行要有回滚和中止方案。

## 常见追问

### `jstack` 和 `jcmd Thread.print` 有什么区别？

二者都能打印线程栈。`jcmd` 是更统一的诊断入口，支持更多命令；`jstack` 更专注于线程栈。实际工作中二者都常用。

### `jmap -histo` 和 `jmap -histo:live` 区别？

`histo` 打印对象直方图；`histo:live` 会先触发 Full GC，只统计存活对象，因此线上慎用。

### 为什么有了 `jstat` 还要 GC 日志？

`jstat` 是实时统计快照，看趋势方便；GC 日志包含每次 GC 的原因、阶段、前后内存和停顿细节，更适合复盘和根因分析。

### Arthas 会不会影响线上？

会有潜在影响，尤其是增强高频方法、打印大对象、复杂 OGNL 表达式、heapdump 等操作。必须限制范围、次数和条件。

### heap dump 应该什么时候导？

最好在 OOM 自动导出，或在内存持续上涨且实例可摘流时手动导出。导出前要确认磁盘空间、停顿影响和数据安全。

## 易错点

- CPU 高时忘记把线程 id 转十六进制。
- 只抓一次 jstack 就下结论。
- 在线上随意执行 `jmap -histo:live`。
- 以为 `jstat` 能直接告诉哪个对象泄漏。
- Arthas `watch` 不限制次数，打印大量对象。
- 忽略 heap dump 敏感数据。
- 未开启 NMT 时临时想查 native memory 细节。

## 项目表达

可以这样说：

> 我平时会按场景选工具。CPU 高先用 `top -H -p` 找线程，再把 tid 转十六进制到 `jstack` 里看栈；GC 问题先用 `jstat` 看趋势，再结合 GC 日志；内存泄漏用 `jcmd GC.heap_dump` 或 OOM 自动 dump 后用 MAT 分析；线上方法级问题会用 Arthas 的 `trace`、`watch`，但一定限制次数和条件，避免影响高 QPS 接口。

更完整的排障表达：

> 有次接口延迟抖动，我先用 `jstat` 排除 Full GC，再用 Arthas `thread -n` 和 JFR 看 CPU 热点，发现主要耗时在大对象 JSON 序列化。随后通过 `trace` 限定慢请求验证调用链，最终优化返回字段和分页大小，并补充了慢调用监控。

## 自检清单

- 能否说出 CPU 高的 `top -H`、tid 转换、`jstack` 链路？
- 能否解释 `jmap -histo:live` 为什么慎用？
- 能否读懂 `jstat -gcutil` 的核心字段？
- 能否列出常用 `jcmd` 命令？
- 能否说明 Arthas `watch`、`trace`、`thread` 的适用场景和风险？
- 能否按 CPU、死锁、GC、OOM、Metaspace、Direct Memory 选择工具？
