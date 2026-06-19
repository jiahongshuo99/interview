# 内存管理、虚拟内存、页表

## 面试定位

内存管理是理解 JVM 堆、直接内存、mmap、缺页中断、OOM Killer 和容器内存限制的基础。Java 后端面试中经常会问：虚拟内存是什么，为什么进程看到的是连续地址，页表有什么用，RSS 和 VIRT 区别是什么，JVM OOM 和 Linux OOM 有什么区别。

回答目标：

- 能解释虚拟地址和物理地址。
- 能说明页、页表、TLB、缺页的关系。
- 能把 Linux 进程内存和 JVM 内存对应起来。
- 能用 `/proc`、`top`、`pmap` 看进程内存。
- 能区分 Java 堆 OOM、直接内存 OOM、系统 OOM。

## 核心概念

现代操作系统通常给每个进程提供独立虚拟地址空间。进程使用虚拟地址，CPU 和内核通过页表把虚拟地址转换为物理地址。

关键术语：

| 概念 | 含义 |
| --- | --- |
| 虚拟地址 | 进程看到的地址 |
| 物理地址 | 实际内存硬件地址 |
| 页 | 虚拟内存和物理内存管理的固定大小块，常见 4KB |
| 页框 | 物理内存中的页大小块 |
| 页表 | 虚拟页到物理页框的映射 |
| TLB | CPU 中缓存地址翻译结果的高速缓存 |
| 缺页 | 访问的虚拟页当前没有有效物理映射 |
| mmap | 把文件或匿名内存映射到进程虚拟地址空间 |

## 为什么需要虚拟内存

虚拟内存解决了几个问题：

- 进程隔离：每个进程有独立地址空间，不能随意访问其他进程内存。
- 简化编程：进程看到连续地址，不必关心物理内存碎片。
- 按需分配：申请虚拟地址不一定立刻分配物理内存。
- 内存保护：页表记录读、写、执行权限。
- 文件映射：文件可映射到内存，提高访问效率。
- 共享内存：多个进程可映射同一物理页。
- 换页：内存不足时可把部分页换出到磁盘。

## 进程虚拟地址空间

典型 Linux 进程虚拟地址空间包括：

```text
高地址
  栈 stack
  mmap 区：共享库、直接内存、文件映射、线程栈
  堆 heap
  BSS：未初始化全局变量
  数据段：已初始化全局变量
  代码段 text
低地址
```

Java 进程中常见内存区域：

- Java 堆：`-Xmx` 控制主要上限。
- 元空间：类元数据，使用 native memory。
- 线程栈：`-Xss` 乘以线程数。
- 直接内存：`DirectByteBuffer`、NIO、Netty。
- Code Cache：JIT 编译代码。
- GC 内部结构。
- JNI/native 库分配。
- mmap 文件映射。

所以 JVM 进程 RSS 不等于 Java 堆大小。堆外内存、线程栈、元空间、mmap 都会贡献 RSS 或虚拟地址空间。

## 页表

页表负责虚拟页到物理页框的映射。每个页表项通常包含：

- 物理页框号。
- present 位，表示是否在内存。
- 读写权限。
- 用户态或内核态权限。
- dirty 位，表示是否被修改。
- accessed 位，表示是否被访问。
- 可执行权限。
- copy-on-write 等标志。

多级页表用于节省页表内存。虚拟地址会被拆成多段索引，逐级查表，最终找到物理页框。

## TLB

每次内存访问都查多级页表太慢，所以 CPU 使用 TLB 缓存虚拟页到物理页框的翻译结果。

TLB 命中：

```text
虚拟地址 -> TLB -> 物理地址 -> 访问缓存/内存
```

TLB 未命中：

```text
虚拟地址 -> 页表遍历 -> 更新 TLB -> 访问内存
```

上下文切换、地址空间切换、页表变化都可能影响 TLB 命中率。这也是进程切换通常比线程切换重的原因之一。

## 按需分配和 overcommit

Linux 可能允许进程申请大量虚拟内存，但不立即分配物理内存。只有真正访问页面时，才通过缺页分配物理页。

例如：

```c
char *p = malloc(1024 * 1024 * 1024);
```

申请成功不代表 1GB 物理内存已全部占用。只有写入对应页面时，RSS 才会上涨。

这解释了：

- `VIRT` 可能很大，但 RSS 不一定大。
- mmap 大文件后虚拟地址空间变大，但常驻内存取决于实际访问。
- 系统内存不足时可能触发 OOM Killer。

## Copy-On-Write

`fork` 创建子进程时，如果立刻复制父进程全部内存会很昂贵。Linux 通常使用 Copy-On-Write：

- 父子进程初始共享物理页。
- 页表标记为只读。
- 某一方写入时触发缺页异常。
- 内核复制该页，写入方获得私有副本。

Java 服务中不常直接 `fork` 大进程，但理解 COW 有助于理解进程创建、容器和一些工具行为。

## mmap

`mmap` 把文件或匿名内存映射到进程虚拟地址空间。

用途：

- 文件映射读写。
- 共享内存。
- 直接内存分配。
- 动态库加载。
- JVM heap 或 native 区域管理。

Java 中：

```java
try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
    MappedByteBuffer buffer = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());
    byte b = buffer.get(0);
}
```

注意：

- mmap 不等于文件全部读入内存。
- 实际访问页时才可能缺页加载。
- mmap 区域释放和 GC、Cleaner、引用生命周期有关。
- mmap 文件可能导致 page cache 压力。

## 命令和示例

### top

```bash
top -p <pid>
```

常见字段：

- `VIRT`：虚拟地址空间大小。
- `RES`：常驻物理内存，近似 RSS。
- `SHR`：共享内存部分。

### /proc

```bash
cat /proc/<pid>/status | egrep "VmSize|VmRSS|VmData|VmStk|VmExe|VmLib|Threads"
cat /proc/<pid>/smaps_rollup
cat /proc/<pid>/maps | head
```

`smaps_rollup` 常见字段：

- `Rss`：常驻内存。
- `Pss`：按共享比例摊分后的内存。
- `Private_Clean`、`Private_Dirty`：私有页。
- `Shared_Clean`、`Shared_Dirty`：共享页。

### pmap

```bash
pmap -x <pid> | tail -20
```

用于看进程虚拟内存映射分布。

### JVM native memory

如果开启 NMT：

```bash
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.native_memory detail
```

启动参数：

```bash
-XX:NativeMemoryTracking=summary
```

NMT 有额外开销，生产是否开启要按环境评估。

## 线上排查

### RSS 持续上涨但堆不高

排查方向：

- 直接内存泄漏。
- 线程数过多导致栈内存增加。
- 元空间增长。
- mmap 文件映射。
- JNI/native 内存泄漏。
- glibc arena 碎片。
- page cache 和共享内存观察误判。

命令：

```bash
jcmd <pid> VM.native_memory summary
cat /proc/<pid>/smaps_rollup
pmap -x <pid> | sort -k3 -nr | head
cat /proc/<pid>/status | egrep "VmRSS|Threads"
```

### VIRT 很大是否有问题

不一定。VIRT 是虚拟地址空间，可能包含未实际占用物理内存的映射。重点看 RSS、PSS、容器内存、swap、OOM 记录。

```bash
dmesg -T | grep -i "killed process"
```

### 容器内存问题

容器中要看 cgroup 限制和当前使用：

```bash
cat /sys/fs/cgroup/memory.max
cat /sys/fs/cgroup/memory.current
```

不同 cgroup 版本路径可能不同。Java 容器内存要同时考虑堆和堆外。

## 常见追问

### 虚拟内存是不是磁盘上的内存

不是。虚拟内存是地址空间抽象。它可以映射到物理内存、文件、swap，也可以暂时没有物理页。

### VIRT 大是不是内存泄漏

不一定。VIRT 大只说明虚拟地址空间映射大。判断泄漏要看 RSS/PSS 是否持续上涨、业务对象或 native 内存是否持续增长。

### 页表为什么需要多级

虚拟地址空间很大，如果每个虚拟页都用连续页表项表示，页表本身会非常大。多级页表只为实际使用的地址范围分配下级页表，节省内存。

### TLB 有什么用

TLB 缓存虚拟地址到物理地址的翻译结果，避免每次内存访问都进行多级页表遍历。

### Java 堆 OOM 和 Linux OOM 有什么区别

Java 堆 OOM 是 JVM 在 Java 堆内无法分配对象，抛 `OutOfMemoryError: Java heap space`。Linux OOM 是系统或 cgroup 内存不足，内核选择进程杀掉，进程通常直接退出。

## 易错点

- 把虚拟内存等同于 swap。
- 看到 VIRT 大就判断内存泄漏。
- 只关注 `-Xmx`，忽略直接内存、线程栈、元空间、Code Cache、JNI。
- 认为 mmap 会立刻把整个文件读入物理内存。
- 忽略容器内存限制和 JVM 堆外内存。
- 把 RSS 和 Java 堆使用量混为一谈。

## 自检清单

- 能否解释虚拟地址、物理地址、页、页表、TLB。
- 能否说明为什么进程看到独立连续地址空间。
- 能否解释 VIRT、RSS、PSS 的区别。
- 能否说出 JVM 进程中除 Java 堆外还有哪些内存。
- 能否用 `/proc/<pid>/smaps_rollup`、`pmap`、NMT 排查堆外内存。
- 能否区分 Java OOM 和 Linux OOM Killer。
