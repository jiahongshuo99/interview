# STW、吞吐量、延迟、堆大小配置

## 面试定位

这类题常用于区分“会背 JVM 参数”和“能做线上调优”的候选人。面试官会问：什么是 STW，吞吐量和延迟怎么取舍，堆是不是越大越好，`Xms` 和 `Xmx` 怎么设，容器环境要注意什么，如何从 GC 日志判断调优方向。

回答主线：

1. STW 是 JVM 暂停用户线程执行某些全局操作。
2. 吞吐量关注应用运行时间占比，延迟关注单次请求停顿。
3. 堆越大不一定越好，可能降低 GC 频率但增加单次停顿和内存成本。
4. 调优要基于目标、指标、GC 日志、压测和容量模型，而不是套模板。

## STW

STW 是 Stop The World，指 JVM 暂停所有或大部分 Java 应用线程，执行 GC、类重定义、偏向锁撤销、线程栈采样等全局安全操作。

GC 中常见 STW 阶段：

- Young GC 暂停。
- CMS 初始标记、重新标记。
- G1 Young GC、Remark、Cleanup 中部分阶段。
- Full GC。
- ZGC 也有短暂停顿阶段，但大部分工作并发执行。

STW 影响：

- 请求响应时间上升。
- P99/P999 延迟抖动。
- 心跳、锁续期、任务调度延迟。
- RPC 超时、连接池堆积。

面试重点：低延迟优化不是追求“没有 GC”，而是控制停顿时间、停顿频率和尾延迟影响。

## 吞吐量

吞吐量通常表示应用线程运行时间占总时间的比例：

```text
吞吐量 = 应用运行时间 / (应用运行时间 + GC 时间)
```

吞吐优先适合：

- 离线批处理。
- 数据计算。
- 后台任务。
- 对单次延迟不敏感的吞吐型服务。

常见收集器：

- Parallel GC。
- 部分场景下 G1 也能兼顾吞吐，但不是纯吞吐优先。

常用参数：

```bash
-XX:+UseParallelGC
-XX:GCTimeRatio=99
-XX:MaxGCPauseMillis=500
```

注意：`MaxGCPauseMillis` 和 `GCTimeRatio` 可能互相牵制，不能同时幻想极低停顿和极高吞吐。

## 延迟

延迟关注请求响应时间，尤其是 P95、P99、P999。

延迟敏感场景：

- 在线 API。
- 交易链路。
- 网关。
- 推荐、搜索、广告等实时服务。
- 锁续期、调度、心跳敏感系统。

常见优化方向：

- 使用 G1、ZGC 等低停顿收集器。
- 降低分配速率。
- 降低对象存活率和晋升速率。
- 避免大对象。
- 控制堆大小和 Region 行为。
- 避免显式 Full GC。
- 做好限流和队列长度控制。

## 吞吐和延迟的取舍

典型矛盾：

- 堆大：GC 频率可能降低，但单次回收成本可能上升。
- 堆小：单次 GC 可能快，但 GC 更频繁，分配压力大时容易 Full GC。
- 并发 GC：停顿降低，但 GC 线程与应用线程争 CPU，吞吐可能下降。
- Parallel：吞吐好，但停顿可能长。
- ZGC：停顿低，但 CPU 和吞吐成本需要压测验证。

调优目标必须明确：

- 批任务：总完成时间和吞吐优先。
- 在线服务：P99/P999 和稳定性优先。
- 成本敏感服务：内存占用和 CPU 成本也要纳入。

## 堆大小配置

常用参数：

```bash
-Xms<size>   # 初始堆大小
-Xmx<size>   # 最大堆大小
-Xmn<size>   # 新生代大小，传统分代收集器常用
```

线上常见做法：

- `Xms` 和 `Xmx` 设置相同，避免运行时堆扩缩容带来的抖动。
- 保留足够 native memory 给线程栈、Metaspace、Direct Memory、JIT Code Cache、GC 结构、JNI、文件映射等。
- 容器环境中不能把容器内存几乎全部给堆。
- 大堆服务要关注压缩指针阈值、GC 停顿和 dump 成本。

## 堆是不是越大越好

不是。

堆变大的好处：

- 容纳更多对象。
- 降低 GC 频率。
- 缓解短期流量峰值。

堆变大的问题：

- 单次 GC 扫描和整理成本可能上升。
- Full GC 风险更大，停顿更长。
- heap dump 更大，排查成本更高。
- 可能影响压缩指针。
- 容器内 native memory 空间变少，可能出现线程、直接内存、Metaspace OOM。
- 内存成本增加。

合理堆大小来自容量评估：

```text
活跃对象大小 + 峰值临时对象 + 缓冲余量 + GC 策略需要的空闲空间
```

## 新生代和老年代比例

传统分代收集器中：

- 新生代太小：Young GC 频繁，对象可能过早晋升。
- 新生代太大：Young GC 单次停顿可能变长，老年代空间变小。
- Survivor 太小：对象容易提前晋升。
- 老年代太小：Full GC 风险增加。

相关参数：

```bash
-XX:NewRatio=2
-XX:SurvivorRatio=8
-XX:MaxTenuringThreshold=15
```

G1 中不建议过度手动固定年轻代大小，因为 G1 会根据停顿目标动态调整年轻代 Region 数量。重点关注：

```bash
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
-XX:G1HeapRegionSize=<size>
```

## 容器环境注意事项

容器中 JVM 调优要关注：

- JVM 是否正确识别容器 CPU 和内存限制。
- `MaxRAMPercentage`、`InitialRAMPercentage`。
- 线程栈、Metaspace、Direct Memory、Code Cache 都在堆外。
- CPU limit 太低会影响并发 GC 线程和应用线程。
- OOMKilled 不一定有 Java 堆 OOM 日志。

常用参数：

```bash
-XX:MaxRAMPercentage=70
-XX:InitialRAMPercentage=70
-XX:MaxMetaspaceSize=<size>
-XX:MaxDirectMemorySize=<size>
-Xss<size>
```

经验：不要让 `-Xmx` 接近容器内存上限，通常需要给非堆内存留出 20% 到 40% 空间，具体取决于线程数、直接内存和框架使用情况。

## 调优方法论

### 1. 明确目标

先确定要优化什么：

- P99 延迟。
- Full GC 次数。
- Young GC 频率。
- CPU 使用率。
- 内存成本。
- 批处理完成时间。

### 2. 收集证据

需要的数据：

- GC 日志。
- JVM 参数。
- 机器或容器规格。
- QPS、请求耗时、错误率。
- 分配速率、晋升速率。
- heap dump 或对象直方图。
- JFR 分配和停顿事件。

### 3. 判断瓶颈

常见判断：

- Young GC 频繁：分配速率高或新生代太小。
- Full GC 频繁且回收明显：堆不足或回收触发太晚。
- Full GC 后不下降：泄漏或长生命周期对象过多。
- STW 长但频率低：单次回收成本高。
- STW 短但频率高：分配速率或堆划分问题。

### 4. 小步调整

一次只调整少数参数，压测对比：

- 堆大小。
- 收集器。
- G1 pause target。
- IHOP。
- 新生代比例。
- 代码分配热点。

不要在没有证据的情况下堆参数堆满。

## 常用命令和指标

查看 JVM 参数：

```bash
jcmd <pid> VM.flags
jcmd <pid> VM.command_line
```

观察 GC：

```bash
jstat -gcutil <pid> 1000 20
jstat -gccause <pid> 1000 20
```

开启日志：

```bash
-Xlog:gc*,safepoint:file=/path/gc.log:time,uptime,level,tags:filecount=10,filesize=100M
```

JFR：

```bash
jcmd <pid> JFR.start name=profile settings=profile duration=120s filename=/tmp/profile.jfr
```

重点指标：

- GC pause max、p95、p99。
- GC time ratio。
- Allocation rate。
- Promotion rate。
- Old usage after GC。
- Full GC count。
- Safepoint time。
- CPU steal 或 throttling。

## 常见追问

### `Xms` 和 `Xmx` 为什么经常设置一样？

避免 JVM 运行过程中动态扩容或缩容堆带来额外开销和延迟抖动，提升内存行为可预测性。

### 堆很大但还是 OOM，为什么？

可能是对象仍可达导致堆 OOM，也可能是 Metaspace、Direct Memory、线程栈、Code Cache 或容器内存触顶。要看具体 OOM 类型和系统日志。

### 低延迟服务是不是一定用 ZGC？

不一定。ZGC 适合低延迟和大堆场景，但需要现代 JDK、足够 CPU、压测验证。很多中等堆在线服务用 G1 就能满足目标。

### GC 调优优先调参数还是改代码？

先用证据判断。如果是分配热点、无界缓存、对象生命周期不合理，改代码收益更大。参数调优主要用于匹配业务负载和收集器策略。

### 为什么容器里明明 `Xmx` 没超过限制还被 OOMKilled？

因为容器限制的是进程总内存，不只是 Java 堆。Metaspace、Direct Memory、线程栈、JIT、GC 元数据、native 库都占内存。

## 易错点

- 把 STW 等同于 Full GC，忽略 Young GC 和其他安全点操作也会 STW。
- 认为堆越大越好。
- 只关注平均延迟，不看 P99/P999。
- 只调 `Xmx`，不关注非堆内存。
- 在 G1 下强行套 CMS 或 Parallel 的分代调参习惯。
- 容器环境不给 native memory 留空间。
- 没有压测对比就在线上大幅改 GC 参数。

## 项目表达

可以这样表达：

> 我们做 JVM 调优时先明确目标是降低接口 P99 抖动，而不是单纯减少 GC 次数。先打开 GC 日志和 JFR，发现 Young GC 单次不长但非常频繁，分配热点集中在请求转换和日志拼接。我们先减少临时对象，再适当调整堆和 G1 停顿目标。上线后用 GC pause、P99、CPU、Old after GC 一起验证，避免只看某一个指标。

容器场景：

> 另一次服务没有 Java heap OOM，但容器被 OOMKilled。排查发现 `Xmx` 设置接近容器上限，同时 Netty direct memory 和线程栈占用较高。后来降低堆比例，限制直接内存，收敛线程池，并补充容器 RSS 监控。

## 自检清单

- 能否解释 STW 的含义和影响？
- 能否说明吞吐量和延迟的取舍？
- 能否解释为什么堆不是越大越好？
- 能否给出 `Xms`、`Xmx`、非堆内存、容器限制的配置思路？
- 能否通过 GC 日志判断调优方向？
- 能否说明 G1 和 ZGC 的调优目标差异？
