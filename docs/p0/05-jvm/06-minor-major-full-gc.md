# Minor GC、Major GC、Full GC

## 面试定位

Minor GC、Major GC、Full GC 是线上排查和 JVM 调优最常见的术语。面试官会看你是否能说明它们触发的区域、触发条件、日志表现、对 STW 的影响，以及不同收集器下术语差异。

回答时要避免绝对化：

- Minor GC 通常指新生代 GC。
- Major GC 常被用来指老年代 GC，但不同资料和收集器语义不完全一致。
- Full GC 通常指覆盖整个 Java 堆和方法区相关回收的全局 GC，通常 STW 时间更长。
- G1 中还有 Young GC、Mixed GC、Full GC，不能简单套传统分代术语。

## 基本概念

| 类型 | 常见回收范围 | 常见触发 | 影响 |
| --- | --- | --- | --- |
| Minor GC / Young GC | 新生代 | Eden 空间不足 | 通常较频繁，停顿较短 |
| Major GC / Old GC | 老年代 | 老年代空间压力 | 术语不统一，可能伴随 Full GC |
| Full GC | 整堆、方法区或元空间相关 | 老年代不足、元空间不足、显式 GC、晋升失败等 | 通常停顿更长，风险更高 |
| Mixed GC | G1 中新生代加部分老年代 Region | G1 并发标记后 | 目标是逐步回收高收益老年代 Region |

## Minor GC

Minor GC 通常发生在新生代，最常见触发条件是 Eden 区空间不足。

过程简化：

1. 新对象优先在 Eden 分配。
2. Eden 不足时触发 Young GC。
3. 存活对象复制到 Survivor 区。
4. 达到年龄阈值或 Survivor 放不下的对象晋升到老年代。
5. Eden 被清空，继续分配。

关键概念：

- 对象年龄：对象每经历一次 Young GC 并存活，年龄可能加一。
- 晋升阈值：`MaxTenuringThreshold` 控制对象晋升年龄上限。
- 动态年龄判断：如果 Survivor 中某年龄及以上对象总大小超过 Survivor 一半，这些对象可能提前晋升。
- 分配担保：老年代需要有足够空间容纳可能晋升的对象。

常见参数：

```bash
-Xmn<size>
-XX:SurvivorRatio=8
-XX:MaxTenuringThreshold=15
-XX:TargetSurvivorRatio=50
```

注意：在 G1、ZGC 等收集器中，不应机械使用所有传统新生代参数；要结合收集器模型。

## Major GC

Major GC 这个术语在不同资料中不完全统一：

- 有些语境指老年代 GC。
- 有些日志或工具中 Major GC 近似等于 Full GC。
- CMS 中有老年代并发回收过程，但失败后可能退化成 Full GC。
- G1 更常谈 Mixed GC 和 Full GC。

面试建议：先说明术语差异，再按“老年代回收”解释。

老年代 GC 常见触发：

- 老年代空间不足。
- 新生代对象晋升失败。
- 大对象直接分配失败。
- CMS/G1 并发周期达到触发阈值。

## Full GC

Full GC 通常是最需要警惕的 GC 类型，因为它经常意味着整堆或大范围 STW，可能造成明显请求延迟。

常见触发原因：

- 老年代空间不足。
- 方法区或元空间空间不足。
- Minor GC 后对象晋升失败。
- 大对象分配失败。
- 显式调用 `System.gc()`。
- CMS concurrent mode failure。
- G1 evacuation failure 或 to-space exhausted 后退化。
- 堆外或直接内存压力间接触发引用处理和 Full GC。

Full GC 的表现：

- GC 日志中出现 `Full GC`、`Pause Full`、`Metadata GC Threshold` 等字样。
- 应用停顿明显。
- Full GC 后老年代下降不明显时，可能存在内存泄漏或堆太小。
- Full GC 频繁时吞吐量和延迟都会恶化。

## 对象晋升与分配担保

Minor GC 前，JVM 需要判断老年代是否有足够空间容纳新生代可能晋升的对象。老年代空间不足时，可能触发 Full GC。

对象进入老年代的常见路径：

- 年龄达到阈值。
- 动态年龄判断提前晋升。
- Survivor 空间不足。
- 大对象直接进入老年代。
- 长期存活对象从新生代晋升。

排查 Young GC 后老年代快速增长时，要关注：

- 对象存活率是否过高。
- Survivor 是否太小。
- 是否有批量请求或缓存导致对象跨过多次 GC。
- 老年代是否存在泄漏。

## 大对象分配

大对象可能直接进入老年代，尤其是大数组、大字符串、大 byte buffer 等。

风险：

- 老年代快速增长。
- 连续空间不足。
- 空间碎片。
- Full GC 频繁。

优化方向：

- 避免一次性加载大文件或大集合。
- 流式处理。
- 分片处理。
- 限制请求体和批量大小。
- 对大缓存设置容量上限。

## GC 日志观察

JDK 8 常见参数：

```bash
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/path/gc.log
```

JDK 9+ 统一日志：

```bash
-Xlog:gc*,safepoint:file=/path/gc.log:time,uptime,level,tags:filecount=10,filesize=100M
```

观察指标：

- Young GC 次数和停顿时间。
- Full GC 次数和停顿时间。
- GC 前后 Eden、Survivor、Old 使用量。
- 老年代是否持续增长。
- Full GC 后回收效果。
- 晋升速率。
- 元空间增长。
- 是否有 allocation failure、promotion failed、to-space exhausted。

## 常见现象与判断

### Young GC 频繁，但 Full GC 不频繁

可能原因：

- 分配速率高。
- 新生代太小。
- 请求瞬时对象多。
- 对象大多朝生夕死。

优化：

- 增大新生代或调整 G1 pause target。
- 减少临时对象分配。
- 批量和缓存优化。
- 使用 JFR 找分配热点。

### Full GC 频繁，Full GC 后老年代明显下降

可能原因：

- 堆偏小。
- 老年代触发阈值不合理。
- CMS/G1 启动回收太晚。
- 峰值流量导致对象短时间进入老年代。

优化：

- 合理增大堆。
- 调整触发阈值。
- 降低晋升速率。
- 削峰限流。

### Full GC 频繁，Full GC 后老年代不下降

可能原因：

- 内存泄漏。
- 缓存无界。
- 长生命周期对象过多。
- 类加载器泄漏。

优化：

- 导出 heap dump。
- 分析 dominator tree 和 GC Roots。
- 修复持有链。

## 常用命令

```bash
jstat -gcutil <pid> 1000 20
jstat -gccause <pid> 1000 20
jcmd <pid> GC.heap_info
jcmd <pid> GC.class_histogram
jcmd <pid> VM.flags
jcmd <pid> GC.heap_dump /tmp/heap.hprof
```

`jstat` 字段常见含义：

- `YGC`：Young GC 次数。
- `YGCT`：Young GC 总耗时。
- `FGC`：Full GC 次数。
- `FGCT`：Full GC 总耗时。
- `GCT`：GC 总耗时。
- `S0/S1/E/O/M`：Survivor、Eden、Old、Metaspace 使用率。

## 常见追问

### Minor GC 会 STW 吗？

通常会 STW，但停顿时间相对较短。不同收集器实现不同，现代低延迟收集器会尽量把部分工作并发化，但 Young GC 暂停仍是重要指标。

### Full GC 一定回收整个堆吗？

通常语义上是全局回收，但具体范围和日志含义与收集器有关。面试中要结合 HotSpot 和具体收集器说明，不要绝对化。

### Major GC 和 Full GC 是一回事吗？

不一定。Major GC 常被用来指老年代 GC，但很多工具和资料会混用。严谨回答应说明上下文差异。

### `System.gc()` 一定触发 Full GC 吗？

不一定。它只是建议 JVM 执行 GC。HotSpot 默认可能触发 Full GC，也可以通过参数禁用或改为并发处理，例如 `-XX:+DisableExplicitGC`。

### 为什么 Full GC 后内存还很高？

如果对象仍从 GC Roots 可达，GC 不能回收。常见原因是缓存、静态集合、ThreadLocal、队列、类加载器泄漏或业务确实需要大量常驻对象。

## 易错点

- 把 Major GC 和 Full GC 在所有场景下画等号。
- 只看 GC 次数，不看停顿时间和回收效果。
- 认为 Young GC 多一定有问题，忽略分配速率和延迟目标。
- 认为 Full GC 后老年代不下降就是 GC 没工作，实际可能是对象仍可达。
- 只调大堆，不分析对象生命周期和分配热点。
- 忽略容器内存限制和 JVM 参数不匹配。

## 项目表达

可以这样说：

> 我们线上有一次 P99 延迟抖动，GC 日志显示 Young GC 很频繁但每次停顿不长，Full GC 不明显。通过 JFR allocation profiling 看到热点是接口聚合时创建大量临时 DTO 和集合。优化后减少中间对象和重复转换，同时适当调大新生代，Young GC 次数下降，延迟抖动改善。

Full GC 场景：

> 另一次是 Full GC 频繁，且 Full GC 后老年代几乎不下降。我们导出 heap dump，用 MAT 看 Retained Size，发现静态缓存持有大量历史数据。修复时引入容量上限和过期策略，Full GC 次数恢复正常。

## 自检清单

- 能否区分 Minor GC、Major GC、Full GC、Mixed GC？
- 能否说明 Young GC 触发条件和对象晋升路径？
- 能否解释 Full GC 常见触发原因？
- 能否通过 GC 日志判断分配速率、晋升和老年代回收效果？
- 能否说明 Full GC 后内存不下降的排查方向？
- 能否给出减少 GC 压力的代码和参数优化思路？
