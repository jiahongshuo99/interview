# 常见 GC：Serial、Parallel、CMS、G1、ZGC

## 面试定位

GC 收集器是 JVM 面试的高频题，重点不是背诵名字，而是能按“目标、算法、适用场景、优缺点、关键参数、版本变化”比较。Java 后端面试常见追问包括：为什么 CMS 会产生浮动垃圾和碎片，G1 为什么可预测停顿，ZGC 为什么低延迟，生产环境怎么选择收集器。

建议回答：

1. 先说基础算法：标记-清除、标记-复制、标记-整理、分代收集。
2. 再比较收集器：Serial 简单单线程，Parallel 追求吞吐，CMS 追求低停顿但有碎片，G1 面向可预测停顿，ZGC 面向超低延迟和大堆。
3. 结合 JDK 版本：CMS 已被移除，G1 是 JDK 9 后默认收集器，ZGC 在现代 JDK 中成熟度更高。

## 基础回收算法

### 标记-清除

过程：

1. 标记存活对象。
2. 清除未标记对象。

优点：

- 实现相对简单。
- 不需要移动对象。

缺点：

- 容易产生内存碎片。
- 分配大对象时可能因连续空间不足触发 GC。
- 清除效率和对象数量有关。

### 标记-复制

过程：

1. 将内存分为两块。
2. 每次只使用一块。
3. GC 时把存活对象复制到另一块。
4. 清空原区域。

优点：

- 没有碎片。
- 适合存活率低的区域，例如新生代。

缺点：

- 浪费部分空间。
- 存活对象多时复制成本高。

### 标记-整理

过程：

1. 标记存活对象。
2. 将存活对象向一端移动。
3. 清理边界外空间。

优点：

- 避免碎片。
- 适合老年代。

缺点：

- 移动对象成本较高。
- 通常需要更长停顿。

### 分代收集

基于多数对象朝生夕死的经验，将堆划分为新生代和老年代：

- 新生代对象存活率低，适合复制算法。
- 老年代对象存活率高，适合标记-清除或标记-整理。

现代收集器即使物理布局不同，也常保留类似“年轻对象”和“老对象”的生命周期管理思想。

## Serial GC

Serial 是最基础的收集器，使用单线程完成 GC 工作，GC 时会 STW。

特点：

- 单线程。
- 新生代常用复制算法。
- 老年代 Serial Old 使用标记-整理。
- 实现简单，额外线程开销小。

适用场景：

- 客户端程序。
- 单核或小内存环境。
- 对吞吐和延迟要求不高的后台工具。

常用参数：

```bash
-XX:+UseSerialGC
```

优点：

- 简单稳定。
- 小堆场景开销低。

缺点：

- 多核服务器资源利用差。
- 停顿时间随堆增长明显。

## Parallel GC

Parallel 又称吞吐量优先收集器。它关注单位时间内应用线程运行时间占比，适合后台计算、批处理等吞吐敏感场景。

特点：

- 多线程 GC。
- 新生代 Parallel Scavenge。
- 老年代 Parallel Old。
- 目标是高吞吐。

常用参数：

```bash
-XX:+UseParallelGC
-XX:ParallelGCThreads=<n>
-XX:MaxGCPauseMillis=<ms>
-XX:GCTimeRatio=<n>
-XX:+UseAdaptiveSizePolicy
```

优点：

- 吞吐量高。
- 多核利用好。
- 自适应调节策略便于配置。

缺点：

- 停顿时间可能较长。
- 不适合对 P99 延迟敏感的在线服务。

## CMS

CMS 是 Concurrent Mark Sweep，目标是降低老年代回收停顿。它主要使用标记-清除算法。

主要阶段：

1. 初始标记：STW，标记 GC Roots 直接关联对象。
2. 并发标记：与用户线程并发，遍历对象图。
3. 重新标记：STW，修正并发期间引用变化。
4. 并发清除：与用户线程并发，清除垃圾对象。

优点：

- 老年代大部分标记和清除工作并发执行。
- 停顿时间比传统 Full GC 低。

缺点：

- 对 CPU 资源敏感，并发阶段会抢占用户线程。
- 无法处理浮动垃圾，需要预留空间。
- 标记-清除产生内存碎片。
- 可能出现 Concurrent Mode Failure，退化为 Serial Old Full GC。
- CMS 已在较新 JDK 中移除，不适合作为新项目默认选择。

常用参数：

```bash
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=70
-XX:+UseCMSInitiatingOccupancyOnly
-XX:+UseCMSCompactAtFullCollection
```

## G1

G1 是 Garbage First，目标是在可控停顿时间内获得较好的吞吐。G1 将堆划分为多个 Region，不再是传统连续新生代和老年代布局。

核心特点：

- Region 化堆布局。
- 每个 Region 可扮演 Eden、Survivor、Old、Humongous 等角色。
- 通过回收价值选择垃圾最多、收益最高的 Region。
- 支持 Young GC、并发标记、Mixed GC。
- 面向可预测停顿。

主要阶段：

1. Young GC：回收年轻 Region。
2. 并发标记：识别老年代 Region 存活情况。
3. Remark：STW，完成标记修正。
4. Cleanup：统计和清理。
5. Mixed GC：回收年轻 Region 加部分老年代高收益 Region。

关键机制：

- Remembered Set：记录跨 Region 引用，避免每次扫描全堆。
- Collection Set：本次计划回收的 Region 集合。
- SATB：Snapshot-At-The-Beginning，并发标记时维持快照语义。
- Humongous Object：超过 Region 一半的大对象会特殊处理，可能影响 GC。

常用参数：

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=<size>
-XX:InitiatingHeapOccupancyPercent=45
```

优点：

- 适合多核、大堆、在线服务。
- 停顿目标相对可控。
- 相比 CMS 更少碎片问题。

缺点：

- 维护 Remembered Set 有额外开销。
- 参数和日志理解复杂。
- 超大对象、跨 Region 引用多时效果可能变差。
- 停顿目标不是硬保证。

## ZGC

ZGC 是低延迟收集器，目标是在大堆场景下也保持极短停顿。它通过并发标记、并发转移、染色指针、读屏障等机制，将大部分 GC 工作与应用线程并发执行。

核心特点：

- 低停顿，停顿时间通常不随堆大小线性增长。
- 支持大堆。
- 并发标记、并发整理、并发重定位。
- 使用读屏障和指针元数据跟踪对象状态。
- 现代 JDK 中已有分代 ZGC，进一步改善年轻对象回收效率。

常用参数：

```bash
-XX:+UseZGC
-Xms<size>
-Xmx<size>
-XX:SoftMaxHeapSize=<size>
```

优点：

- 延迟极低。
- 适合大堆、低延迟服务。
- 减少长时间 STW 风险。

缺点：

- 吞吐可能略低于吞吐优先收集器。
- 对 JDK 版本要求较高。
- 需要结合业务、JDK 版本、容器环境压测验证。

## 收集器选择

常见选择思路：

- 小内存、简单工具：Serial。
- 批处理、离线任务、吞吐优先：Parallel。
- 老项目 JDK 8、低停顿诉求：可能见到 CMS，但新项目不建议首选。
- 通用在线 Java 服务：G1 是常见默认选择。
- 大堆、低延迟、现代 JDK：ZGC 可以重点考虑。

生产选择不要只看理论：

- 看 JDK 版本。
- 看堆大小。
- 看 P99/P999 延迟目标。
- 看吞吐目标。
- 看对象分配速率和存活率。
- 看容器 CPU 和内存限制。
- 用压测和 GC 日志验证。

## 排查和调优方法

### GC 日志

JDK 9+ 推荐：

```bash
-Xlog:gc*,safepoint:file=/path/gc.log:time,uptime,level,tags:filecount=10,filesize=100M
```

关注：

- Pause 类型。
- 停顿时间。
- GC 前后堆变化。
- 晋升速率。
- 并发阶段耗时。
- 是否退化 Full GC。
- 是否有 evacuation failure、to-space exhausted、concurrent mode failure。

### JFR

```bash
jcmd <pid> JFR.start name=profile settings=profile duration=120s filename=/tmp/app.jfr
```

关注事件：

- Garbage Collection。
- Allocation in new TLAB。
- Allocation outside TLAB。
- Object Count after GC。
- Safepoint。

### 指标

线上应监控：

- GC 次数和耗时。
- Young/Old 使用量。
- 分配速率。
- 晋升速率。
- Full GC 次数。
- P99/P999 延迟。
- 容器内存和 CPU。

## 常见追问

### CMS 为什么会产生浮动垃圾？

CMS 并发标记和并发清除时，用户线程仍在运行，会继续产生新垃圾。这些在本轮标记后产生或状态变化导致未能回收的垃圾称为浮动垃圾，需要等下一轮 GC。

### CMS 为什么会碎片化？

CMS 主要使用标记-清除，不移动存活对象，清理后会留下不连续空闲块。大对象分配可能找不到连续空间，触发 Full GC 压缩整理。

### G1 为什么叫 Garbage First？

G1 会根据 Region 的垃圾比例、回收成本和预期收益选择回收集合，优先回收收益高的 Region，而不是一次性回收整个老年代。

### G1 的停顿目标是硬限制吗？

不是。`MaxGCPauseMillis` 是目标，JVM 会尽量满足，但对象分配过快、存活率高、堆太小、跨 Region 引用多等都可能导致超过目标。

### ZGC 为什么停顿低？

ZGC 将标记、转移、重定位等大量工作并发化，通过染色指针和读屏障保证应用线程访问对象时能处理对象移动和状态变化，STW 阶段只做少量根处理等工作。

## 易错点

- 只背收集器名字，不说适用场景。
- 认为 G1 一定比 Parallel 吞吐高。
- 认为 CMS 没有 STW，实际初始标记和重新标记需要 STW。
- 认为 G1 停顿目标是绝对保证。
- 忽略 CMS 已被移除的版本事实。
- 只调 GC 参数，不分析分配热点和对象生命周期。
- 在容器里忽略 CPU 限额，导致 GC 线程数不合理。

## 项目表达

可以这样说：

> 我们服务从 Parallel 切到 G1 的原因不是追求最高吞吐，而是接口 P99 对停顿更敏感。切换前先开启 GC 日志和 JFR，确认对象分配速率和老年代增长模式，再在压测环境调整 `MaxGCPauseMillis`、堆大小和 IHOP。上线后重点观察 Young GC、Mixed GC、Full GC 次数和 P99 延迟，确认没有 evacuation failure。

ZGC 场景：

> 对大堆低延迟服务，我们评估过 ZGC。重点不是直接换参数，而是先确认 JDK 版本、容器 CPU、堆大小、分配速率和业务延迟目标，再压测对比 G1 和 ZGC 的 P99/P999、吞吐和 CPU 开销。

## 自检清单

- 能否解释标记-清除、复制、标记-整理的优缺点？
- 能否比较 Serial、Parallel、CMS、G1、ZGC 的目标和适用场景？
- 能否说清 CMS 的浮动垃圾、碎片和 concurrent mode failure？
- 能否解释 G1 的 Region、RSet、CSet、Mixed GC？
- 能否说明 ZGC 低延迟的核心机制？
- 能否结合业务延迟和吞吐目标选择收集器？
