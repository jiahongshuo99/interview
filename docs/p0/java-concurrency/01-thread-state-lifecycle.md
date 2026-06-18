# 线程状态、线程生命周期

## 面试定位

线程状态和生命周期是 Java 并发的入口题。面试官通常不会只问 `Thread.State` 枚举，而是会继续追问：

- `start()` 和 `run()` 的区别。
- `sleep()`、`wait()`、`join()`、`park()` 分别让线程进入什么状态。
- `BLOCKED`、`WAITING`、`TIMED_WAITING` 怎么区分。
- 线上线程 dump 中大量 `WAITING`、`BLOCKED`、`RUNNABLE` 分别说明什么。
- 如何优雅停止线程。

回答时要把 Java 层状态、操作系统线程调度、锁竞争和线程 dump 分析串起来。

## 核心概念

Java 线程状态由 `java.lang.Thread.State` 定义，共 6 种：

| 状态 | 含义 | 常见来源 |
| --- | --- | --- |
| `NEW` | 线程对象已创建，但还没有调用 `start()` | `new Thread(...)` |
| `RUNNABLE` | 可运行或正在运行 | 已 `start()`，等待 CPU 调度，或正在执行 |
| `BLOCKED` | 等待进入 `synchronized` 临界区 | 竞争 monitor 锁 |
| `WAITING` | 无限期等待另一个线程动作 | `Object.wait()`、`Thread.join()`、`LockSupport.park()` |
| `TIMED_WAITING` | 带超时等待 | `sleep()`、`wait(timeout)`、`join(timeout)`、`parkNanos()` |
| `TERMINATED` | 执行结束 | `run()` 正常返回或异常退出 |

注意：Java 的 `RUNNABLE` 包含操作系统层面的 running 和 ready，也可能包含正在进行阻塞式 native I/O 的线程。看到 `RUNNABLE` 不等于一定在消耗 CPU。

## 生命周期主线

典型生命周期：

```text
NEW --start()--> RUNNABLE --run()结束/异常--> TERMINATED
                 |   ^  |
                 |   |  +-- sleep/join/wait/park --> WAITING/TIMED_WAITING
                 |   |
                 +-- 竞争 synchronized 失败 --> BLOCKED
```

关键点：

- `Thread` 对象只是 Java 对象，调用 `start()` 后才会创建并启动底层线程。
- `run()` 是普通方法，直接调用不会启动新线程。
- 一个 `Thread` 实例只能调用一次 `start()`，重复调用会抛 `IllegalThreadStateException`。
- 线程终止后不能重新启动，只能创建新的 `Thread` 对象。
- 守护线程不会阻止 JVM 退出；当所有非守护线程结束时，JVM 会退出。

## 常见状态转换

### NEW 到 RUNNABLE

```java
Thread t = new Thread(() -> System.out.println("run"));
// 此时 t.getState() 为 NEW
t.start();
// start 后进入 RUNNABLE，等待调度或正在运行
```

### RUNNABLE 到 TERMINATED

```java
Thread t = new Thread(() -> {
    // 正常执行完 run 方法
});
t.start();
t.join();
System.out.println(t.getState()); // TERMINATED
```

### RUNNABLE 到 BLOCKED

`BLOCKED` 只针对等待 `synchronized` monitor 锁，不包含 `ReentrantLock.lock()` 等 AQS 锁等待。

```java
Object lock = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lock) {
        sleepQuietly(10_000);
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lock) {
        System.out.println("acquired");
    }
});

t1.start();
Thread.sleep(100);
t2.start();
Thread.sleep(100);
System.out.println(t2.getState()); // 通常为 BLOCKED
```

### RUNNABLE 到 WAITING

```java
Object lock = new Object();

Thread t = new Thread(() -> {
    synchronized (lock) {
        try {
            lock.wait(); // 释放 monitor，进入 WAITING
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
});
```

`wait()` 必须在持有对象 monitor 时调用，否则抛 `IllegalMonitorStateException`。

### RUNNABLE 到 TIMED_WAITING

```java
Thread.sleep(1000);        // 不释放 synchronized 锁
lock.wait(1000);           // 释放 synchronized 锁
thread.join(1000);         // 等待另一个线程结束，最多等待指定时间
LockSupport.parkNanos(1L); // AQS 常用底层阻塞方式
```

## sleep、wait、join、yield、park 对比

| 方法 | 所属类 | 是否释放 monitor | 是否需要持有锁 | 状态 | 典型用途 |
| --- | --- | --- | --- | --- | --- |
| `sleep` | `Thread` | 否 | 否 | `TIMED_WAITING` | 暂停执行 |
| `wait` | `Object` | 是 | 是 | `WAITING` / `TIMED_WAITING` | 条件等待 |
| `join` | `Thread` | 底层基于 `wait` | 调用者无需显式持有目标线程锁 | `WAITING` / `TIMED_WAITING` | 等待线程结束 |
| `yield` | `Thread` | 否 | 否 | 仍是 `RUNNABLE` | 提示调度器让出 CPU |
| `park` | `LockSupport` | 否 | 否 | `WAITING` / `TIMED_WAITING` | 构建锁和同步器 |

重点区别：

- `sleep()` 不释放已经持有的 `synchronized` 锁。
- `wait()` 会释放调用对象的 monitor，唤醒后需要重新竞争锁。
- `yield()` 只是调度提示，不保证让出，也不保证其他线程马上执行。
- `park()` 和 `unpark()` 以许可为模型，`unpark()` 可以先于 `park()` 调用。

## interrupt 中断机制

Java 没有安全的强制停止线程能力，`interrupt()` 是协作式中断。

```java
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            doOneBatch();
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // 恢复中断标记
            break;
        }
    }
});
```

关键语义：

- `interrupt()` 设置目标线程的中断标记。
- 如果线程阻塞在 `sleep()`、`wait()`、`join()`，会抛 `InterruptedException` 并清除中断标记。
- `Thread.interrupted()` 会读取并清除当前线程中断标记。
- `isInterrupted()` 只读取，不清除。
- 很多阻塞 I/O 不一定响应 Java 中断，需要额外关闭 socket、channel 或使用带超时 API。

## 优雅停止线程

推荐方式：

```java
class Worker implements Runnable {
    private volatile boolean running = true;

    public void shutdown() {
        running = false;
    }

    @Override
    public void run() {
        while (running && !Thread.currentThread().isInterrupted()) {
            try {
                doWork();
            } catch (RuntimeException e) {
                handle(e);
            }
        }
    }
}
```

更常见的工程方式是使用线程池：

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

pool.shutdown();
if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
    pool.shutdownNow();
}
```

不要使用：

- `Thread.stop()`：可能在任意位置释放锁，破坏对象不变量。
- `Thread.suspend()` / `resume()`：容易死锁。
- 忽略 `InterruptedException`：会导致调用方无法停止任务。

## 线程 dump 分析

线程 dump 中常见状态解释：

### 大量 BLOCKED

通常说明某个 `synchronized` 锁竞争严重。

排查方向：

- 找 `waiting to lock <...>`。
- 找持有该锁的线程 `locked <...>`。
- 看持锁线程是否在执行慢 I/O、远程调用、复杂计算。
- 检查是否把大段业务逻辑放进了 `synchronized`。

### 大量 WAITING

不一定是问题。线程池空闲 worker、消息消费者等待队列、定时调度线程都可能 `WAITING`。

排查方向：

- 栈顶是否是 `LockSupport.park`、`AbstractQueuedSynchronizer`。
- 是否所有业务线程都等待同一个条件。
- 是否存在漏 `notify`、漏 `signal`、任务没有提交等问题。

### 大量 TIMED_WAITING

常见于 `sleep`、带超时的 `poll`、连接池等待、定时任务。

风险点：

- 业务代码用 `sleep` 做轮询，导致延迟和资源浪费。
- 超时时间设置过大，故障时恢复慢。
- 定时任务堆积但线程仍显示等待。

### RUNNABLE 但 CPU 不高

可能是 native I/O、socket read、文件读写、JVM 内部等待。需要结合栈和 CPU 采样，不要只看状态。

## 底层机制

### Java 线程和 OS 线程

HotSpot 中传统平台线程通常映射到操作系统原生线程，由 OS 调度。Java 的 `Thread.State` 是 JVM 对状态的抽象，不完全等同 OS 状态。

### monitor

`synchronized` 基于对象 monitor。线程进入同步块时需要获取 monitor，获取失败时进入 `BLOCKED`。

### wait set 和 entry set

对同一个 monitor：

- 竞争锁失败的线程在 entry set，表现为 `BLOCKED`。
- 调用 `wait()` 的线程进入 wait set，表现为 `WAITING` 或 `TIMED_WAITING`。
- `notify()` / `notifyAll()` 只负责从 wait set 唤醒线程，被唤醒线程还要重新竞争 monitor。

## 工程实践

- 线程命名必须带业务含义，方便线程 dump 排查，例如 `order-reconcile-worker-1`。
- 不要直接无限创建线程，优先使用受控线程池。
- 后台线程要定义明确的退出条件。
- 捕获 `InterruptedException` 后要恢复中断标记或明确结束任务。
- 锁内不要做远程调用、慢 SQL、大文件 I/O。
- 生产环境保留线程 dump 获取手段，例如 `jstack`、Arthas `thread`、容器诊断命令。
- 线程池要监控活跃线程数、队列长度、拒绝次数、任务耗时。

## 常见追问

### 1. `start()` 和 `run()` 有什么区别？

`start()` 会请求 JVM 创建并启动新线程，新线程回调 `run()`；直接调用 `run()` 只是当前线程执行普通方法，不会并发执行。

### 2. 为什么 `wait()` 要放在循环里？

因为可能存在虚假唤醒，也可能被唤醒后条件已经被其他线程改变。正确写法：

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
    doSomething();
}
```

### 3. `notify()` 和 `notifyAll()` 怎么选？

如果等待条件单一且能确定唤醒一个线程足够，可以用 `notify()`；如果有多个条件或不确定唤醒哪个线程，优先 `notifyAll()`，避免唤醒错误线程导致全部沉睡。

### 4. `BLOCKED` 和 `WAITING` 区别？

`BLOCKED` 是等待进入 `synchronized`；`WAITING` 是线程主动等待某个条件或许可，例如 `wait()`、`join()`、`park()`。

### 5. 线程池里的线程为什么长期 `WAITING`？

空闲 worker 通常在任务队列上等待，这是正常现象。需要看队列长度、提交速率、任务耗时和业务是否停滞。

## 线上风险

- 死锁：多个线程循环等待锁，业务完全停住。
- 活锁：线程一直响应对方变化但无法推进。
- 饥饿：低优先级或竞争弱的线程长期拿不到资源。
- 线程泄漏：请求级任务创建线程但不关闭，最终耗尽系统线程。
- 中断丢失：任务无法被线程池 `shutdownNow()` 停止。
- 锁内慢调用：一个慢依赖拖住大量 `BLOCKED` 线程。
- 无界线程创建：高峰期触发 OOM 或系统无法创建新线程。

## 易错点

- 认为 `RUNNABLE` 就一定正在跑 CPU。
- 认为 `sleep()` 会释放锁。
- 用 `if` 判断等待条件，而不是 `while`。
- 捕获 `InterruptedException` 后什么都不做。
- 重复调用同一个线程对象的 `start()`。
- 使用 `stop()` 强制结束线程。
- 把 `wait()`、`notify()` 用在没有持有 monitor 的对象上。
- 误以为 `notify()` 会立即释放锁；实际要等同步块退出。

## 自检清单

- 能否画出 6 种线程状态转换图。
- 能否解释 `BLOCKED`、`WAITING`、`TIMED_WAITING` 的区别。
- 能否说明 `sleep()` 和 `wait()` 的锁释放差异。
- 能否写出正确的 `wait` 循环模板。
- 能否解释协作式中断，并写出不中断丢失的代码。
- 能否根据线程 dump 找出锁竞争和死锁线索。
- 能否说明为什么不应该使用 `Thread.stop()`。
- 能否解释线程池空闲线程为什么常见为 `WAITING`。
