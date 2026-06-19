# 缺页中断

## 面试定位

缺页中断是虚拟内存真正落地的关键机制。Java 后端面试中它常和 mmap、page cache、swap、内存抖动、冷启动延迟、主缺页和次缺页一起出现。

回答目标：

- 能解释什么是缺页。
- 能区分 minor fault 和 major fault。
- 能说明缺页不一定是异常错误。
- 能把 mmap、按需分配、swap、文件读写和缺页关联起来。
- 能用 `vmstat`、`pidstat`、`perf` 或 `/proc` 观察缺页。

## 核心概念

缺页中断，也叫 page fault，是 CPU 访问某个虚拟地址时，发现页表中没有可直接使用的物理页映射，于是触发异常进入内核处理。

常见原因：

- 该虚拟页还没有分配物理页，第一次访问。
- mmap 文件页尚未加载到内存。
- 页面被换出到 swap。
- Copy-On-Write 写入共享页。
- 访问权限不满足，例如写只读页。
- 访问非法地址。

缺页不一定是错误。很多缺页是操作系统按需分配和文件映射的正常机制。

## 缺页处理流程

```text
CPU 访问虚拟地址
  -> TLB 未命中或页表项无有效映射
  -> 触发 page fault
  -> CPU 切入内核
  -> 内核检查虚拟地址是否合法
  -> 如果合法，分配物理页或从文件/swap 读入
  -> 更新页表
  -> 返回用户态，重新执行触发缺页的指令
```

如果地址非法或权限不允许，内核会向进程发送信号，常见是 `SIGSEGV`，进程可能崩溃。

## Minor Fault 和 Major Fault

| 类型 | 含义 | 成本 |
| --- | --- | --- |
| Minor Fault | 不需要从磁盘读取数据即可建立映射 | 较低 |
| Major Fault | 需要从磁盘、文件系统或 swap 读取数据 | 较高 |

Minor Fault 示例：

- 匿名内存第一次写入，内核分配零页或物理页。
- COW 页面写入时复制已有内存页。
- 文件页已经在 page cache 中，只需建立页表映射。

Major Fault 示例：

- mmap 文件页不在 page cache，需要从磁盘读。
- 页面被换出到 swap，需要从 swap 读回。

工程上，minor fault 很常见，不一定是问题；major fault 多且延迟敏感时需要关注。

## mmap 与缺页

mmap 建立的是虚拟地址到文件的映射，不是立刻把文件全部读入内存。

流程：

```text
mmap(file)
  -> 建立虚拟地址区域
  -> 程序第一次访问某页
  -> 缺页
  -> 内核从 page cache 或磁盘加载对应文件页
  -> 建立页表映射
```

所以 mmap 大文件时：

- VIRT 可能明显增加。
- RSS 只随实际访问页增长。
- 首次访问可能因 major fault 变慢。
- 顺序访问可受预读优化。

Java 中 `MappedByteBuffer`、Lucene、RocksDB、部分消息队列和缓存系统都可能大量依赖 mmap。

## 匿名内存与缺页

进程申请堆内存后，物理页可能并未立即分配。第一次写入时触发缺页，内核分配物理页并清零。

这解释了：

- `malloc` 或 JVM 扩展虚拟内存不一定立刻增加 RSS。
- 大对象第一次写入可能产生缺页成本。
- 服务冷启动或预热阶段 page fault 较多。

## Copy-On-Write 缺页

`fork` 后父子进程共享物理页，页表标记只读。当任一方写入共享页时：

```text
写只读 COW 页
  -> 缺页异常
  -> 内核复制该物理页
  -> 写入方页表指向新页
  -> 恢复写入
```

这是正常缺页，不是非法访问。

## Swap 与缺页

内存压力大时，内核可能把不活跃匿名页换出到 swap。之后访问该页会触发 major fault，从 swap 读回，延迟明显增加。

Java 服务一般不希望频繁 swap：

- GC 延迟会变大。
- 请求尾延迟抖动。
- 线程栈或堆页被换出后恢复慢。
- 可能引发雪崩式超时。

生产环境通常会谨慎配置 swap 和 swappiness，并通过容器限制、堆大小、堆外内存控制避免系统进入持续换页。

## 命令和示例

### vmstat

```bash
vmstat 1
```

关注：

- `si`：每秒从 swap 读入。
- `so`：每秒写入 swap。
- `wa`：IO wait。
- `free`：空闲内存。

持续 `si/so` 不为 0 通常要关注。

### ps 查看缺页

部分系统支持：

```bash
ps -o pid,comm,min_flt,maj_flt -p <pid>
```

字段：

- `min_flt`：minor fault 次数。
- `maj_flt`：major fault 次数。

### pidstat

```bash
pidstat -r -p <pid> 1
```

可观察内存使用和缺页相关指标，字段随 sysstat 版本略有差异。

### /proc

```bash
cat /proc/<pid>/stat
cat /proc/<pid>/status | egrep "VmRSS|VmSize|RssAnon|RssFile|VmSwap"
```

`/proc/<pid>/stat` 中包含 minor fault 和 major fault 计数，但字段位置不直观，脚本解析要谨慎。

### perf

```bash
perf stat -e page-faults,minor-faults,major-faults -p <pid> sleep 10
```

也可以看缺页热点：

```bash
perf record -e page-faults -p <pid> sleep 30
perf report
```

### time

运行命令时看缺页：

```bash
/usr/bin/time -v java -jar app.jar
```

关注：

- Minor page faults。
- Major page faults。

## 线上排查

### 服务冷启动慢

可能原因：

- 类加载和 JIT。
- 堆内存首次触碰。
- mmap 文件首次访问。
- page cache 未预热。
- major fault 多。

排查：

```bash
pidstat -r -p <pid> 1
perf stat -e page-faults,minor-faults,major-faults -p <pid> sleep 30
```

优化：

- 预热接口和热点数据。
- 控制启动期加载量。
- 对大 mmap 文件做顺序预读或业务预热。
- 避免启动后立刻承接满流量。

### 延迟偶发抖动

如果系统有 swap：

```bash
vmstat 1
cat /proc/<pid>/status | grep VmSwap
```

若 `VmSwap` 增长且 `si/so` 不为 0，可能存在换页导致的尾延迟。

### mmap 文件访问慢

排查：

```bash
vmtouch <file>
mincore 工具或业务侧统计
iostat -x 1
```

如果首次访问慢、后续快，可能是文件页第一次从磁盘加载到 page cache。

## 常见追问

### 缺页中断是不是异常

从 CPU 机制上是异常处理路径，但从业务视角不一定是错误。按需分配、mmap 加载、COW 都依赖缺页完成。

### minor fault 会不会很慢

相对 major fault 快很多，但仍有内核处理和页表更新成本。大量 minor fault 也可能影响性能，例如频繁分配和首次触碰大量内存。

### major fault 为什么慢

因为需要访问磁盘或 swap。磁盘 IO 延迟远高于内存访问，尤其在随机 IO 或磁盘压力大时。

### Java 堆访问会发生缺页吗

会。JVM 申请或保留堆地址空间后，物理页可能在首次访问时才分配。堆页也可能在内存压力下被换出，后续访问触发缺页。

### 缺页和 OOM 有什么关系

缺页时内核可能需要分配物理页。如果内存不足且回收失败，可能触发 OOM Killer。Java 堆内对象分配失败则可能由 JVM 抛 Java OOM。

## 易错点

- 认为缺页一定是程序 bug。
- 认为 mmap 会立即加载整个文件。
- 只关注 page fault 总数，不区分 major 和 minor。
- 忽略 swap 对 Java 服务尾延迟的影响。
- 把 VIRT 增长等同于物理内存增长。
- 冷启动性能不做预热，直接满流量。

## 自检清单

- 能否解释缺页中断处理流程。
- 能否区分 minor fault 和 major fault。
- 能否说明 mmap 为什么首次访问可能变慢。
- 能否解释匿名内存按需分配和 COW。
- 能否用 `vmstat`、`pidstat`、`perf stat` 观察缺页。
- 能否判断 swap 和 major fault 对线上延迟的影响。
