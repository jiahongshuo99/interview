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

G1 是 Garbage First，目标是在可控停顿时间内获得较好的吞吐。它不是按一整块连续新生代、老年代来管理堆，而是把堆切成多个大小相同的 Region，再按 Region 粒度做分配、统计和回收。

G1 的核心思路：

> 不一定一次回收整个老年代，而是估算每个 Region 的回收收益和回收成本，在停顿时间预算内优先回收收益最高的一批 Region。

核心特点：

- Region 化堆布局。
- 每个 Region 可扮演 Eden、Survivor、Old、Humongous 等角色。
- 年轻代和老年代在逻辑上仍然存在，但物理上不要求连续。
- 通过 CSet 选择本次回收的 Region，尽量满足停顿目标。
- 通过复制转移回收对象，回收后 Region 可整体复用，因此碎片问题比 CMS 少。
- 支持 Young GC、并发标记、Mixed GC，必要时也可能退化为 Full GC。

### Region 和对象分配

G1 会把堆划分为多个 Region，例如 1MB、2MB、4MB、8MB 等，具体大小由堆大小和参数决定。一个 Region 在某一时刻可以是：

- Eden Region：新对象优先分配到这里。
- Survivor Region：Young GC 后仍存活的年轻对象进入这里。
- Old Region：年龄足够或 Survivor 放不下的对象晋升到这里。
- Humongous Region：大对象专用 Region。

普通对象分配路径通常是：

1. 线程优先在 Eden 中的 TLAB 分配。
2. TLAB 不够时申请新的 TLAB 或走慢路径。
3. Eden Region 用到一定程度后触发 Young GC。
4. Young GC 把存活对象复制到 Survivor 或 Old Region。
5. 被回收的 Eden Region 整体变为空闲 Region。

所以 G1 不是在原地清理年轻代对象，而是把存活对象转移出去，再整块回收原 Region。这也是它相比标记-清除更不容易产生碎片的原因。

### Young GC

Young GC 是 STW。触发原因通常是 Eden 空间不足。它主要做：

1. 选择所有 Eden Region 和部分 Survivor Region 作为 CSet。
2. 扫描 GC Roots 和相关跨 Region 引用。
3. 把 CSet 中仍存活的对象复制到新的 Survivor 或 Old Region。
4. 回收整个 CSet 中原来的 Region。

G1 会根据停顿目标动态调整年轻代 Region 数量。年轻代越大，吞吐可能更好，但单次 Young GC 停顿也可能变长；年轻代越小，单次停顿可能更短，但 GC 更频繁。

### 并发标记周期

G1 需要知道老年代各 Region 的存活比例，才能判断哪些 Old Region 回收收益高。因此会周期性做并发标记。

主要阶段：

1. Initial Mark：STW，标记 GC Roots 直接可达对象，通常借一次 Young GC 顺带完成。
2. Root Region Scan：扫描 Survivor Region 中指向老年代的引用，不能和下一次 Young GC 并发。
3. Concurrent Mark：与用户线程并发，遍历对象图，统计各 Region 存活情况。
4. Remark：STW，处理并发标记期间的引用变化，完成标记修正。
5. Cleanup：统计 Region 回收收益，清理完全没有存活对象的 Region。

并发标记结束后，G1 才知道哪些 Old Region 垃圾比例高，后续才能进入 Mixed GC。

### Mixed GC

Mixed GC 也是 STW。它和 Young GC 的区别是：

- Young GC：只回收年轻 Region。
- Mixed GC：回收年轻 Region，同时挑选一部分高收益 Old Region。

G1 不会一次性把所有可回收 Old Region 都收掉，而是分多次 Mixed GC 慢慢回收。这样做是为了把每次停顿控制在目标范围内。

如果 Mixed GC 跟不上对象分配速度，或者老年代空间不足，可能出现 evacuation failure、to-space exhausted，严重时退化为 Full GC。

关键机制：

- Remembered Set：每个 Region 维护“有哪些其他 Region 引用了我”，避免回收一个 Region 时扫描全堆。
- Collection Set：本次计划回收的 Region 集合，Young GC 主要包含年轻 Region，Mixed GC 还包含部分 Old Region。
- Pause Prediction：根据历史回收速度、复制成本、RSet 扫描成本估算停顿时间，并选择合适的 CSet。
- SATB：Snapshot-At-The-Beginning，并发标记按“开始标记那一刻”的对象图近似做快照，写屏障会记录被覆盖前的旧引用。
- Humongous Object：超过 Region 一半的大对象会直接进入 Humongous Region，通常连续占用一个或多个 Region。

### RSet 为什么重要

Region 化以后，如果回收某个 Region，必须知道堆里哪些地方引用了这个 Region 中的对象。最粗暴的办法是扫描全堆，但这会让停顿不可控。

RSet 解决的是：

> 回收一个 Region 时，只扫描和它相关的跨 Region 引用，而不是扫描整个堆。

代价是：

- 写对象引用时需要写屏障维护 RSet。
- 跨 Region 引用很多时，RSet 会变大。
- RSet 扫描和维护都会消耗 CPU 和内存。

所以 G1 在对象引用关系复杂、跨 Region 引用特别多的场景下，效果可能变差。

### Humongous Object

对象大小超过一个 Region 的一半时，会被 G1 作为 Humongous Object 处理。例如 Region 是 4MB，大于 2MB 的对象就属于 Humongous Object。

特点：

- 会占用连续的 Humongous Region。
- 生命周期较短的大对象会增加 GC 压力。
- 连续 Region 不够时可能更容易触发 Full GC。
- 常见来源包括大数组、大字符串、大 byte buffer、一次性加载大响应。

排查时如果 GC 日志中 Humongous 分配频繁，需要关注大对象来源，而不是只调 `MaxGCPauseMillis`。

常用参数：

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200              # 停顿目标，不是硬保证
-XX:G1HeapRegionSize=<size>           # Region 大小
-XX:InitiatingHeapOccupancyPercent=45  # 触发并发标记的老年代占用阈值
-XX:G1ReservePercent=10                # 预留空闲空间，降低转移失败风险
-XX:ParallelGCThreads=<n>              # STW 阶段并行 GC 线程数
-XX:ConcGCThreads=<n>                  # 并发标记线程数
```

优点：

- 适合多核、大堆、在线服务。
- 停顿目标相对可控。
- 相比 CMS 更少碎片问题。
- 不需要像 CMS 那样依赖标记-清除后的空闲列表处理大量碎片。
- JDK 9 以后是 HotSpot 默认收集器，生产资料和工具支持丰富。

缺点：

- 维护 Remembered Set 有额外开销。
- 参数和日志理解复杂。
- 超大对象、跨 Region 引用多时效果可能变差。
- 停顿目标不是硬保证。
- 并发标记和写屏障会带来 CPU 成本。
- 堆太小、分配速率过高或存活对象过多时，仍可能 Full GC。

常见排查点：

- Young GC 频繁：看分配速率、Eden 大小、对象创建热点。
- Mixed GC 频繁：看老年代增长、对象存活率、缓存和集合持有。
- Evacuation Failure / To-space Exhausted：看堆是否太小、预留空间是否不足、晋升压力是否过大。
- Humongous Allocation：看大对象来源，优先从业务对象大小和批量处理方式入手。
- RSet 扫描耗时高：看跨 Region 引用是否复杂，例如大 Map、大缓存、长生命周期对象引用大量短生命周期对象。

## ZGC

ZGC 是低延迟收集器，目标是在大堆场景下也保持极短停顿。它的核心目标不是追求最高吞吐，而是把标记、整理、对象移动等大部分 GC 工作与用户线程并发执行，让 STW 阶段只做很少的根处理和同步工作。

可以先抓住一句话：

> G1 通过 Region 和停顿预测尽量控制停顿，ZGC 则进一步把对象标记和对象搬迁都做成并发，从机制上减少长时间 STW。

核心特点：

- 低停顿，停顿时间通常不随堆大小线性增长。
- 支持大堆。
- 并发标记、并发重定位、并发引用修正。
- 使用染色指针和读屏障跟踪对象状态。
- 现代 JDK 中可使用分代 ZGC，进一步改善年轻对象回收效率。
- 需要额外 CPU 和内存余量支撑并发回收。

### 为什么停顿低

传统压缩整理型 GC 如果要移动对象，通常需要 STW：

```text
暂停用户线程
标记存活对象
移动对象
更新所有引用
恢复用户线程
```

ZGC 的思路是：

```text
短暂停顿处理根
并发标记对象
并发选择要搬迁的区域
并发移动对象
用户线程访问对象时通过读屏障修正引用
```

因此，堆越大时，ZGC 的总后台工作量仍然会变大，但 STW 时间不会像传统 Full GC 那样随堆大小明显线性增长。

### 染色指针

染色指针可以理解为：对象引用里除了对象地址，还带了一些状态位。ZGC 利用这些状态位表示对象当前处于什么状态，例如是否已标记、是否需要重映射等。

这解决的是：

> GC 和用户线程并发运行时，JVM 需要快速判断一个引用是不是当前阶段可直接使用，还是需要修正。

注意：这是 HotSpot 实现层面的引用表示，不是 Java 语言层面的对象引用。

### 读屏障

读屏障是在读取对象引用时插入的一小段检查逻辑。用户线程每次从对象字段、数组元素等位置读取引用时，ZGC 都有机会检查这个引用是否需要处理。

典型作用：

- 并发标记时，保证对象可达性信息正确。
- 并发重定位时，如果对象已经被移动，读屏障可以把旧引用修正到新地址。
- 让引用修正工作分摊到用户线程访问过程中，而不是一次性 STW 扫描全堆。

代价是：每次读取引用都有额外屏障成本，因此 ZGC 可能牺牲一部分吞吐来换低延迟。

### 主要阶段

ZGC 的具体阶段随 JDK 版本会有演进，面试中按下面的抽象链路理解即可：

1. Pause Mark Start：短暂 STW，处理根对象，启动标记。
2. Concurrent Mark：并发遍历对象图，标记存活对象。
3. Pause Mark End：短暂 STW，完成标记收尾。
4. Concurrent Prepare for Relocation：并发选择需要整理搬迁的内存区域。
5. Pause Relocate Start：短暂 STW，处理根引用重定位。
6. Concurrent Relocate：并发移动对象，并通过读屏障逐步修正引用。

重点不是背阶段名，而是说明：标记和搬迁这些重活基本并发，STW 阶段很短。

### 分代 ZGC

早期 ZGC 是非分代收集器，年轻对象和老对象在同一套机制下处理。现代 JDK 中可使用分代 ZGC，把堆逻辑上分成年轻代和老年代。

分代的意义仍然来自分代假说：

- 大多数对象朝生夕死。
- 年轻代可以更频繁、更便宜地回收。
- 老年代可以降低扫描和搬迁频率。

所以分代 ZGC 的目标是：在保持低停顿的同时，减少处理全堆对象的成本，改善吞吐和内存使用效率。具体参数和默认行为会随 JDK 版本变化，生产使用前要以当前 JDK 文档和压测为准。

### 什么时候可能出问题

ZGC 低停顿不代表没有成本。常见风险：

- 分配速率太高，GC 并发回收追不上，可能出现 allocation stall。
- CPU 核数太少或容器 CPU 限额太紧，并发 GC 线程会和业务线程竞争。
- 堆预留空间不足，对象还没搬完就没有足够空间继续分配。
- 读屏障和并发线程带来吞吐损耗，吞吐优先场景不一定适合。
- JDK 版本差异较大，需要验证当前版本的稳定性和参数支持。

常用参数：

```bash
-XX:+UseZGC
-Xms<size>
-Xmx<size>
-XX:SoftMaxHeapSize=<size>
```

参数理解：

- `-XX:+UseZGC`：启用 ZGC。
- `-Xms` / `-Xmx`：设置堆初始和最大大小，低延迟服务通常会固定成相同值减少运行期扩缩容影响。
- `-XX:SoftMaxHeapSize`：软堆上限，ZGC 会尽量把堆使用控制在该值附近，但必要时仍可增长到 `Xmx`。

优点：

- 延迟极低。
- 适合大堆、低延迟服务。
- 减少长时间 STW 风险。
- 支持并发整理，碎片风险低。
- 在现代 JDK 上适合对 P99/P999 延迟敏感的服务评估。

缺点：

- 吞吐可能略低于吞吐优先收集器。
- 对 JDK 版本要求较高。
- 需要结合业务、JDK 版本、容器环境压测验证。
- 需要 CPU 和内存余量，资源紧张时优势会下降。

常见排查点：

- STW 仍然抖动：看 GC Roots 数量、线程数、JNI、类加载、系统 safepoint。
- Allocation Stall：看分配速率、堆大小、CPU 是否足够、并发 GC 是否追不上。
- CPU 上升：看读屏障成本、并发 GC 线程、对象分配热点。
- 堆占用高：看 `SoftMaxHeapSize`、对象存活率、缓存和大对象。

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

### G1 为什么比 CMS 更不容易碎片化？

CMS 主要是标记-清除，清理后对象不移动，容易留下不连续空闲块。G1 回收 CSet 时会把存活对象复制到其他 Region，再把原 Region 整体释放，因此天然带有整理效果。

### G1 的 RSet 解决什么问题？

RSet 记录跨 Region 引用。回收某个 Region 时，G1 只需要扫描 GC Roots 和相关 RSet，不必每次扫描全堆。代价是写屏障、RSet 维护和扫描会带来额外 CPU 与内存开销。

### G1 的 Mixed GC 回收整个老年代吗？

不是。Mixed GC 会回收所有年轻 Region，再选择一部分垃圾比例高、回收收益大的 Old Region。它通常分多次完成老年代回收，用多次较短停顿替代一次长时间 Full GC。

### G1 的停顿目标是硬限制吗？

不是。`MaxGCPauseMillis` 是目标，JVM 会尽量满足，但对象分配过快、存活率高、堆太小、跨 Region 引用多等都可能导致超过目标。

### ZGC 为什么停顿低？

ZGC 将标记、转移、重定位等大量工作并发化，通过染色指针和读屏障保证应用线程访问对象时能处理对象移动和状态变化，STW 阶段只做少量根处理等工作。

### ZGC 和 G1 的核心区别是什么？

G1 仍然主要在 STW 的 Young GC / Mixed GC 中复制转移 CSet 里的存活对象，只是通过 Region 选择和停顿预测控制单次停顿。ZGC 则把对象移动和引用修正也尽量并发化，通过读屏障让用户线程访问对象时协助修正引用，所以更偏低延迟。

### ZGC 一定比 G1 好吗？

不一定。ZGC 的优势是低停顿，代价是读屏障、并发 GC 线程和额外内存余量。吞吐优先、堆不大、延迟目标没那么极端的服务，用 G1 可能更稳、更简单。

## 易错点

- 只背收集器名字，不说适用场景。
- 认为 G1 一定比 Parallel 吞吐高。
- 认为 ZGC 一定比 G1 吞吐高。
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
