# CPU 飙高、死锁、内存泄漏排查

## 面试定位

CPU 飙高、死锁、内存泄漏是 Java 后端线上排障三件套。面试官看重的是你能否在压力场景下按步骤保留现场、缩小范围、定位线程或对象、解释根因并提出修复方案。

回答原则：

1. 先保留现场，不要直接重启。
2. CPU 问题定位到线程，再定位到代码栈。
3. 死锁问题通过线程 dump 找锁等待环。
4. 内存泄漏通过 GC 日志、heap dump、GC Roots 引用链定位。
5. 修复后要补监控、限流、超时、容量和回归验证。

## CPU 飙高排查

### 常见原因

- 死循环或大循环。
- 正则表达式灾难性回溯。
- JSON 序列化或反序列化大对象。
- 加解密、压缩、图片处理等 CPU 密集操作。
- GC 频繁导致 CPU 被 GC 线程占用。
- 锁竞争、自旋、CAS 重试。
- 线程池过大导致上下文切换。
- 日志量暴增。
- 热点接口流量突增。

### 排查步骤

1. 查看进程 CPU。

```bash
top
ps -ef | grep java
```

2. 查看 Java 进程内线程 CPU。

```bash
top -H -p <pid>
```

3. 将高 CPU 线程 id 转成十六进制。

```bash
printf "%x\n" <tid>
```

4. 导出线程栈并搜索十六进制 nid。

```bash
jstack -l <pid> > /tmp/jstack.txt
grep -n "nid=0x<hex>" -A 40 /tmp/jstack.txt
```

5. 连续抓取多次线程栈。

```bash
jstack -l <pid> > /tmp/jstack1.txt
sleep 5
jstack -l <pid> > /tmp/jstack2.txt
sleep 5
jstack -l <pid> > /tmp/jstack3.txt
```

如果同一个线程持续停在同一段业务代码，可能是死循环或长耗时计算；如果大量 GC 线程占 CPU，需要看 GC 日志。

### Arthas 定位

```bash
dashboard
thread -n 5
thread <threadId>
trace com.demo.Service method '#cost > 100'
watch com.demo.Service method '{params, returnObj}' -x 2
```

注意：`trace`、`watch`、`monitor` 在线上要控制匹配范围和执行次数，避免放大问题。

### async-profiler / JFR

采样 CPU 火焰图：

```bash
profiler start
profiler stop --format html --file /tmp/cpu.html
```

JFR：

```bash
jcmd <pid> JFR.start name=cpu settings=profile duration=120s filename=/tmp/cpu.jfr
```

火焰图适合定位 CPU 热点方法，比单次 jstack 更适合分析“忙但没有明显卡住”的问题。

## GC 导致 CPU 高

判断方法：

- `top -H` 中 GC 线程 CPU 高。
- GC 日志显示 GC 次数密集。
- `jstat -gcutil` 中 YGC/FGC 快速增加。
- 应用吞吐下降，响应变慢。

命令：

```bash
jstat -gcutil <pid> 1000 20
jstat -gccause <pid> 1000 20
```

处理：

- 如果 Young GC 频繁，查分配热点。
- 如果 Full GC 频繁且回收少，查内存泄漏。
- 如果堆太小，评估扩容。
- 如果对象晋升过快，调新生代或优化对象生命周期。

## 死锁排查

### 死锁定义

死锁是多个线程互相等待对方持有的资源，导致所有相关线程都无法继续执行。

必要条件：

- 互斥。
- 持有并等待。
- 不可剥夺。
- 循环等待。

### Java 死锁示例

```java
Object a = new Object();
Object b = new Object();

new Thread(() -> {
    synchronized (a) {
        synchronized (b) {
            // do something
        }
    }
}).start();

new Thread(() -> {
    synchronized (b) {
        synchronized (a) {
            // do something
        }
    }
}).start();
```

### 排查命令

```bash
jstack -l <pid>
jcmd <pid> Thread.print -l
```

如果存在 Java 层死锁，线程 dump 通常会出现：

```text
Found one Java-level deadlock:
```

重点看：

- 哪些线程互相等待。
- 持有了哪个锁。
- 等待哪个锁。
- 锁对应的对象地址。
- 业务代码栈位置。

### 死锁修复思路

- 固定加锁顺序。
- 减小锁粒度。
- 避免持锁调用外部接口、RPC、IO。
- 使用 `tryLock` 加超时。
- 用更高层并发工具替代手写锁。
- 对资源分配做排序。
- 增加超时、降级和监控。

## 锁竞争和线程阻塞

不一定是死锁，但也会导致性能问题。

线程状态：

- `RUNNABLE`：正在运行或等待 CPU，也可能在 native IO。
- `BLOCKED`：等待进入 `synchronized`。
- `WAITING`：无限等待，例如 `Object.wait`、`LockSupport.park`。
- `TIMED_WAITING`：限时等待，例如 `sleep`、带超时的 `wait`。

排查：

```bash
jstack -l <pid> | grep -E "BLOCKED|WAITING|parking|locked" -n
```

Arthas：

```bash
thread -b
thread --state BLOCKED
```

修复：

- 缩短临界区。
- 避免锁内慢操作。
- 使用读写锁、分段锁、无锁队列。
- 限制并发度。
- 检查连接池、线程池、下游超时。

## 内存泄漏排查

### 判断是否泄漏

内存泄漏通常表现为：

- 老年代使用持续上升。
- Full GC 后老年代不明显下降。
- heap dump 中某类对象数量持续增长。
- 业务上已不需要的对象仍从 GC Roots 可达。

判断命令：

```bash
jstat -gcutil <pid> 1000 20
jcmd <pid> GC.class_histogram
jcmd <pid> GC.heap_dump /tmp/heap.hprof
```

### 分析路径

1. 查看 GC 日志确认趋势。
2. 在不同时间点导出对象直方图或 heap dump。
3. 对比对象数量增长。
4. 用 MAT 看 Dominator Tree。
5. 找 Retained Size 大的对象。
6. 查看 Path to GC Roots。
7. 回到代码确认为什么引用没有释放。

### 常见泄漏来源

- 静态集合。
- 无界缓存。
- `ThreadLocal` 未清理。
- 监听器未注销。
- 线程池队列积压。
- 定时任务保留历史上下文。
- WebSocket、连接、会话未关闭。
- 类加载器泄漏。
- 日志 MDC 未清理。
- 异步回调持有大对象。

## ThreadLocal 泄漏

线程池中最常见：

```java
private static final ThreadLocal<UserContext> CTX = new ThreadLocal<>();

try {
    CTX.set(context);
    doWork();
} finally {
    CTX.remove();
}
```

原因：

- `ThreadLocalMap` 的 key 是弱引用。
- value 是强引用。
- 线程池线程长期存活。
- 不 `remove` 时 value 可能一直挂在线程上。

排查 heap dump 时可以搜索：

- `java.lang.Thread`
- `threadLocals`
- `ThreadLocalMap$Entry`

## 线程池问题

CPU、内存、死锁经常和线程池有关。

常见问题：

- 无界队列导致内存增长。
- 线程数过大导致 CPU 上下文切换。
- 线程数过小导致任务堆积。
- 拒绝策略不合理。
- 没有设置任务超时。
- Future 阻塞等待造成线程池饥饿。
- 业务线程池和回调线程池互相等待。

排查指标：

- 活跃线程数。
- 队列长度。
- 拒绝次数。
- 任务耗时。
- 线程池创建数量。

## 线上处置顺序

推荐顺序：

1. 保存现场：jstack、jstat、GC 日志、必要时 heap dump/JFR。
2. 降低影响：限流、摘流、扩容、重启单实例。
3. 定位根因：线程、锁、对象、GC。
4. 修复代码或参数。
5. 压测验证。
6. 补监控和告警。

注意：如果服务已经严重影响可用性，可以先摘流或重启部分实例，但要尽量保留至少一个问题现场。

## 常见追问

### CPU 高但 jstack 看不出业务栈怎么办？

可能是 native 方法、JIT、GC、系统调用或采样时机问题。可以连续抓栈、使用 JFR/async-profiler、看 GC 日志和系统指标。

### `RUNNABLE` 一定是在消耗 CPU 吗？

不一定。Java 线程状态的 `RUNNABLE` 包括正在运行和等待操作系统资源，也可能阻塞在 native IO。

### 死锁和长时间阻塞区别？

死锁存在循环等待，相关线程无法继续；长时间阻塞可能是等待锁、IO、连接池或下游响应，资源释放后仍可能继续。

### 内存泄漏一定会 OOM 吗？

不一定。泄漏对象增长慢或有上限时可能只是内存水位高、GC 频繁、性能下降。无界增长最终可能 OOM。

### 线程数越多吞吐越高吗？

不是。CPU 密集任务线程数接近 CPU 核数更合理；IO 密集可以更多，但也要受下游、连接池、内存和上下文切换限制。

## 易错点

- CPU 高直接重启，不保留线程栈。
- 只抓一次 jstack，错过动态问题。
- 看到 `RUNNABLE` 就认为一定是死循环。
- 看到 `WAITING` 就认为一定异常。
- 内存泄漏只看堆使用率，不看 Full GC 后趋势。
- 只看对象数量，不看 Retained Size 和引用链。
- 在线上无限制使用 Arthas `watch` 或 heap dump。

## 项目表达

CPU 场景：

> 我排查 CPU 飙高时会先用 `top -H -p` 找到高 CPU 线程，把 tid 转成十六进制后到 jstack 里找对应 `nid`。之前一次问题定位到正则匹配在异常输入下发生回溯，线程多次采样都卡在同一个 Pattern 调用栈。修复时改写正则并增加输入长度限制。

死锁场景：

> 死锁问题我会抓 `jstack -l`，看是否有 `Found one Java-level deadlock`，然后根据线程持有和等待的锁对象找到业务代码。修复通常是统一加锁顺序，或者用 `tryLock` 加超时，避免锁内调用外部服务。

内存泄漏场景：

> 内存泄漏我会先看 GC 日志确认 Full GC 后老年代是否下降，再 dump 堆分析 Dominator Tree 和 GC Roots。之前有个 ThreadLocal 泄漏，线程池线程长期持有上下文对象，最终在 finally 里统一 remove，并加了线程池任务包装器做兜底清理。

## 自检清单

- 能否完整说出 CPU 高的定位命令链路？
- 能否把十进制 tid 转换为 jstack 中的十六进制 nid？
- 能否解释 Java 线程状态和锁等待？
- 能否用 jstack 定位死锁？
- 能否说明内存泄漏的 heap dump 分析路径？
- 能否讲出 ThreadLocal 泄漏的根因和修复方式？
