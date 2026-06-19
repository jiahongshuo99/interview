# 线程调度、阻塞、唤醒

## 面试定位

线程调度、阻塞、唤醒是理解 Java 线程状态、锁、IO 和线程池的基础。面试官常问：线程从 `RUNNABLE` 到 `WAITING` 发生了什么，阻塞 IO 为什么不占 CPU，`notify` 后线程是否立刻执行，`park/unpark` 和 `wait/notify` 有什么差异。

回答要覆盖：

- OS 调度器如何选择线程运行。
- 阻塞和唤醒的状态变化。
- Java 状态和 OS 状态不是一一对应。
- `wait/notify`、`park/unpark`、IO 阻塞的差异。
- 线上如何看线程阻塞在哪里。

## 核心概念

线程在操作系统层面通常有几类状态：

- 运行中：正在 CPU 上执行。
- 可运行：具备运行条件，等待 CPU 调度。
- 阻塞或睡眠：等待某个事件，不参与 CPU 竞争。
- 终止：执行结束。

Java `Thread.State` 是 JVM 暴露的抽象：

| Java 状态 | 常见含义 |
| --- | --- |
| `NEW` | 创建未启动 |
| `RUNNABLE` | 正在运行或等待 OS 调度，也可能在 native IO |
| `BLOCKED` | 等待进入 synchronized monitor |
| `WAITING` | 无限期等待某个动作 |
| `TIMED_WAITING` | 限时等待 |
| `TERMINATED` | 已结束 |

注意：Java `RUNNABLE` 不等于一定正在占用 CPU。阻塞在某些 native IO 上时，线程 dump 也可能显示 `RUNNABLE`。

## 线程调度

调度器负责在可运行线程之间分配 CPU 时间。

调度考虑因素：

- 线程优先级和调度策略。
- 时间片。
- CPU 亲和性。
- 负载均衡。
- 交互性和睡眠时间。
- cgroup CPU 配额。

Java 层 `Thread.setPriority()` 只是提示，实际效果受操作系统、JVM、容器限制影响，工程中很少依赖它保证业务顺序。

## 阻塞

阻塞是线程因为等待某个条件无法继续执行，被移出可运行队列，暂时不消耗 CPU。

常见阻塞原因：

- 等待锁。
- 等待条件变量。
- 等待 socket 数据。
- 等待磁盘 IO。
- `sleep` 等待时间到期。
- 等待线程结束。
- 等待连接池、队列、信号量。

阻塞的价值是避免忙等。线程不满足执行条件时睡眠，条件满足后再唤醒。

## 唤醒

唤醒是把等待某个事件的线程重新变为可运行状态。

重点：

- 唤醒不等于立刻运行。
- 被唤醒后要进入可运行队列，等待 CPU 调度。
- 如果还需要锁，被唤醒后还要重新竞争锁。
- 条件可能已经不满足，所以等待条件必须用循环检查。

例如 `Object.wait()` 被 `notify()` 唤醒后，要重新竞争对象 monitor，拿到锁后才能从 `wait()` 返回。

## wait 和 notify

`wait/notify/notifyAll` 是 Object monitor 条件等待机制。

示例：

```java
class Buffer {
    private final Queue<String> queue = new ArrayDeque<>();
    private final int capacity = 10;

    synchronized void put(String value) throws InterruptedException {
        while (queue.size() == capacity) {
            wait();
        }
        queue.add(value);
        notifyAll();
    }

    synchronized String take() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();
        }
        String value = queue.remove();
        notifyAll();
        return value;
    }
}
```

关键点：

- 调用 `wait/notify` 必须持有对应对象 monitor。
- `wait()` 会释放 monitor。
- `notify()` 不释放 monitor，只是通知等待线程。
- 被通知线程必须等当前线程退出同步块后才能竞争锁。
- 必须用 `while` 检查条件，不能用 `if`。
- 多条件等待时优先考虑 `Condition` 拆分条件队列。

## park 和 unpark

`LockSupport.park/unpark` 是很多 Java 并发工具的底层阻塞原语。

许可模型：

- 每个线程最多持有一个许可。
- `unpark(thread)` 发放许可。
- `park()` 消耗许可；如果没有许可则阻塞。
- `unpark` 可以先于 `park` 调用。

示例：

```java
Thread worker = new Thread(() -> {
    LockSupport.park();
    doWork();
});

worker.start();
LockSupport.unpark(worker);
```

相比 `wait/notify`：

- 不要求持有 monitor。
- 不会自动释放某个锁。
- 不存在 `notify` 早于 `wait` 导致信号丢失的问题，因为许可可以先发。
- 仍可能被中断或伪唤醒，条件等待仍要循环检查。

## 阻塞 IO 的线程状态

阻塞 socket read 的流程大致是：

```text
Java read
  -> native read/recvfrom
  -> 内核检查 socket 接收缓冲区
  -> 没有数据则线程睡眠
  -> 网卡收到数据，中断和网络栈处理
  -> 数据进入 socket 缓冲区
  -> 唤醒等待线程
  -> 线程等待调度，返回用户态
```

阻塞期间线程通常不占 CPU，但占用线程资源、栈内存和连接上下文。大量阻塞线程会导致线程数、内存和调度压力上升。

## 命令和示例

### 看线程状态

```bash
jstack -l <pid> > /tmp/jstack.txt
grep -n "java.lang.Thread.State" /tmp/jstack.txt | sort | uniq -c
```

查看阻塞线程：

```bash
grep -n "BLOCKED" -A 20 /tmp/jstack.txt
grep -n "WAITING" -A 20 /tmp/jstack.txt
```

### Arthas

```bash
thread
thread -n 5
thread --state BLOCKED
thread -b
```

`thread -b` 常用于找当前阻塞其他线程的锁持有者。

### OS 线程观察

```bash
top -H -p <pid>
pidstat -w -t -p <pid> 1
```

结合 `cswch/s` 和 `nvcswch/s` 判断线程是主动阻塞多，还是被抢占多。

## 线上排查

### 大量 BLOCKED

通常是 `synchronized` 锁竞争。

排查：

- 找 `waiting to lock`。
- 找 `locked` 同一个对象的线程。
- 看锁持有者是否执行慢操作。
- 检查是否在锁内做 RPC、DB、IO、日志。

修复：

- 缩短临界区。
- 拆锁。
- 用并发集合或 AQS 锁。
- 避免锁内慢调用。

### 大量 WAITING parking

可能是：

- 线程池工作线程空闲。
- AQS 锁等待。
- 队列等待。
- 连接池等待。
- `CompletableFuture` 或异步任务等待。

要结合栈顶判断，不要只看状态。

### 大量 TIMED_WAITING

可能是：

- `sleep` 定时任务。
- 带超时的队列 poll。
- 连接池等待。
- HTTP 客户端超时等待。

如果大量线程都在同一下游调用超时，要查下游延迟、连接池、DNS、网络。

### RUNNABLE 很多但 CPU 不高

可能是线程 dump 显示 `RUNNABLE`，但实际在 native IO 等待。要结合：

```bash
top -H -p <pid>
strace -tt -T -f -p <pid>
ss -antp
```

## 常见追问

### notify 后线程会立刻执行吗

不会。`notify` 只是把一个等待线程从 wait set 移到锁竞争队列，被唤醒线程还要等当前线程释放 monitor，并等待调度器分配 CPU。

### 为什么 wait 要放在 while 中

因为可能存在伪唤醒，也可能多个线程被唤醒后条件已被其他线程消费。用 `while` 可以在真正继续执行前重新检查条件。

### sleep 会释放锁吗

不会。`Thread.sleep()` 只让当前线程休眠，不释放已经持有的 `synchronized` monitor 或 `ReentrantLock`。

### blocked 和 waiting 有什么区别

`BLOCKED` 是等待进入 `synchronized` monitor。`WAITING` 是等待某个动作，例如 `wait`、`join`、`park`。`ReentrantLock` 等待通常表现为 `WAITING parking`，不是 `BLOCKED`。

### 线程阻塞为什么不占 CPU

阻塞线程被移出可运行队列，CPU 会调度其他可运行线程。它仍占用线程栈、内核任务结构、Java 对象等资源。

## 易错点

- 认为唤醒等于立即运行。
- 用 `if` 包裹 `wait`。
- 调用 `wait/notify` 时没有持有 monitor。
- 认为 `sleep` 会释放锁。
- 看到 `WAITING` 就认为有问题，线程池空闲线程等待很常见。
- 把 `BLOCKED` 泛化为所有锁等待，忽略 AQS 锁通常是 `WAITING parking`。
- 忽略 Java `RUNNABLE` 可能包括 native IO 等待。

## 自检清单

- 能否解释线程从运行到阻塞再到唤醒的完整路径。
- 能否区分 `wait/notify` 和 `park/unpark`。
- 能否说明 `notify` 后为什么还要竞争锁。
- 能否根据线程 dump 判断 `BLOCKED`、`WAITING`、`TIMED_WAITING` 的原因。
- 能否解释阻塞 IO 为什么不占 CPU但占线程资源。
- 能否给出大量线程阻塞的线上排查步骤。
