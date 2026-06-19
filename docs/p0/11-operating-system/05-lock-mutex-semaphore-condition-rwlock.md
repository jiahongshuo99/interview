# 锁、互斥量、信号量、条件变量、读写锁

## 面试定位

同步原语是并发题的核心。面试官通常会从 Java 的 `synchronized`、`ReentrantLock` 问到底层互斥、信号量、条件变量，再追问读写锁适用场景、锁竞争排查和死锁风险。

回答目标：

- 区分互斥、计数、条件等待、读写分离。
- 能把 Java 同步工具和 OS 同步原语关联起来。
- 能说明阻塞、唤醒、伪唤醒、锁公平性。
- 能用线程 dump、`jstack`、`strace` 分析锁问题。

## 总览

| 原语 | 核心语义 | Java 对应 | 典型场景 |
| --- | --- | --- | --- |
| 锁 | 保护临界区 | `synchronized`、`Lock` | 共享变量互斥访问 |
| 互斥量 | 同一时刻只允许一个线程进入 | monitor、`ReentrantLock` | 修改共享状态 |
| 信号量 | 控制同时访问资源的数量 | `Semaphore` | 限流、连接数控制 |
| 条件变量 | 等待某个条件成立 | `Object.wait`、`Condition` | 生产消费、状态等待 |
| 读写锁 | 多读单写 | `ReadWriteLock` | 读多写少共享数据 |

## 锁和互斥量

锁的本质是保护临界区，保证共享状态在并发访问下满足一致性。

互斥量是最典型的锁：同一时刻最多一个线程持有。

Java 中常见互斥锁：

- `synchronized`：JVM monitor，语法级支持，自动释放。
- `ReentrantLock`：JUC 显式锁，基于 AQS，支持中断、超时、公平锁、多个条件队列。

`synchronized` 示例：

```java
class Counter {
    private int count;

    synchronized int incrementAndGet() {
        return ++count;
    }
}
```

`ReentrantLock` 示例：

```java
class Counter {
    private final ReentrantLock lock = new ReentrantLock();
    private int count;

    int incrementAndGet() {
        lock.lock();
        try {
            return ++count;
        } finally {
            lock.unlock();
        }
    }
}
```

工程选择：

- 简单互斥优先 `synchronized`，代码简洁，异常自动释放。
- 需要可中断、超时、公平、多条件队列时用 `ReentrantLock`。
- 高并发计数可考虑 `LongAdder`、原子类、分片聚合。

## 自旋锁和阻塞锁

自旋锁获取失败时不立即睡眠，而是在 CPU 上循环等待。

优点：

- 避免短临界区下线程阻塞和唤醒成本。

缺点：

- 持锁时间长时浪费 CPU。
- 单核或 CPU 紧张时效果差。

阻塞锁获取失败时让线程睡眠，等待唤醒。

Java 锁实现通常会结合自旋、队列、阻塞等策略。面试中不要把锁简单理解成只有“睡眠等待”。

## 信号量

信号量维护一组许可，线程进入前获取许可，退出后释放许可。

Java 示例：

```java
class ClientLimiter {
    private final Semaphore semaphore = new Semaphore(100);

    void call() throws InterruptedException {
        if (!semaphore.tryAcquire(200, TimeUnit.MILLISECONDS)) {
            throw new RejectedExecutionException("too many concurrent calls");
        }
        try {
            doRpc();
        } finally {
            semaphore.release();
        }
    }
}
```

适用场景：

- 限制同时访问下游的请求数。
- 控制本地昂贵资源并发度。
- 模拟连接池许可。
- 批处理并发保护。

注意：

- `release()` 必须放在 `finally`。
- 许可数量不等于线程数量，要按资源容量设置。
- `Semaphore` 可以公平或非公平，公平性会影响吞吐。
- 不要在未成功 `acquire` 时误 `release`。

## 条件变量

条件变量用于“在持锁保护的共享状态上等待某个条件成立”。

`wait/notify` 示例：

```java
class BoundedQueue<E> {
    private final Queue<E> queue = new ArrayDeque<>();
    private final int capacity;

    BoundedQueue(int capacity) {
        this.capacity = capacity;
    }

    synchronized void put(E e) throws InterruptedException {
        while (queue.size() == capacity) {
            wait();
        }
        queue.add(e);
        notifyAll();
    }

    synchronized E take() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();
        }
        E e = queue.remove();
        notifyAll();
        return e;
    }
}
```

`Condition` 示例：

```java
class BoundedQueue<E> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();
    private final Queue<E> queue = new ArrayDeque<>();
    private final int capacity;

    BoundedQueue(int capacity) {
        this.capacity = capacity;
    }

    void put(E e) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();
            }
            queue.add(e);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    E take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }
            E e = queue.remove();
            notFull.signal();
            return e;
        } finally {
            lock.unlock();
        }
    }
}
```

关键规则：

- 条件判断必须和状态修改在同一把锁保护下。
- 等待条件必须用 `while`，不能用 `if`。
- `await/wait` 会释放锁，返回前会重新获取锁。
- `signal/notify` 只是通知，不立即释放锁。
- 多个条件队列时 `Condition` 比 `wait/notifyAll` 更清晰。

## 读写锁

读写锁把访问分为读锁和写锁：

- 多个读线程可以同时持有读锁。
- 写线程独占。
- 写锁和读锁互斥。

Java 示例：

```java
class ConfigCache {
    private final ReadWriteLock rw = new ReentrantReadWriteLock();
    private volatile Map<String, String> config = Map.of();

    String get(String key) {
        rw.readLock().lock();
        try {
            return config.get(key);
        } finally {
            rw.readLock().unlock();
        }
    }

    void reload(Map<String, String> newConfig) {
        rw.writeLock().lock();
        try {
            config = Map.copyOf(newConfig);
        } finally {
            rw.writeLock().unlock();
        }
    }
}
```

适用场景：

- 读多写少。
- 读操作耗时足够长，读并发能带来收益。
- 数据结构不适合简单 volatile 快照。

不适合：

- 写很多。
- 临界区很短，读写锁开销超过收益。
- 读锁内存在慢 IO，导致写锁长期饥饿。

锁降级：

- 线程持有写锁时，再获取读锁，然后释放写锁，保留读锁。
- 常用于更新缓存后继续以读视角使用数据。

锁升级：

- 持有读锁时再获取写锁。
- 容易死锁，因为多个读线程都等写锁，但彼此持有读锁不释放。
- `ReentrantReadWriteLock` 不支持安全锁升级。

## 底层机制

锁实现通常包含：

- 快速路径：无竞争时用户态或 JVM 内部快速获取。
- 竞争路径：入队、挂起、等待唤醒。
- 原子操作：CAS、内存屏障。
- 等待队列：管理阻塞线程。
- 内核协作：严重竞争时通过 futex 等机制睡眠和唤醒。

Java AQS 的核心思想：

- 用一个 `state` 表示同步状态。
- 获取失败的线程进入 FIFO 等待队列。
- 阻塞线程通过 `LockSupport.park` 挂起。
- 释放锁时唤醒后继节点。

## 命令和排查

### 线程 dump

```bash
jstack -l <pid> > /tmp/jstack.txt
```

关注：

- `BLOCKED`：等待 `synchronized` monitor。
- `waiting to lock`：等待哪把 monitor。
- `locked`：谁持有 monitor。
- `parking to wait for`：AQS、LockSupport 等等待。

### Arthas

```bash
thread --state BLOCKED
thread -b
```

### strace

```bash
strace -f -e trace=futex -p <pid>
```

大量 `futex` 可能说明锁等待、条件等待或线程 park/unpark 很多，需要结合线程 dump 判断。

### perf

```bash
perf top -p <pid>
```

如果看到锁、futex、调度相关函数占比高，可能存在严重同步竞争。

## 线上问题和修复

### 锁竞争严重

现象：

- 响应时间变长。
- CPU 不一定高，但吞吐下降。
- 大量线程 `BLOCKED` 或 `WAITING parking`。
- 上下文切换升高。

修复：

- 缩短临界区。
- 锁内不要做 RPC、DB、文件 IO。
- 拆分锁粒度。
- 用不可变快照加 volatile 替代读写锁。
- 使用并发集合。
- 用队列串行化热点更新。

### 信号量许可泄漏

现象：

- 并发越来越低。
- 请求大量等待许可。
- 重启后恢复。

原因：

- 异常路径没有 `release`。
- 超时、取消、线程中断后没有正确释放。
- acquire 失败仍 release 导致许可数量异常。

修复：

- 成功 acquire 后才进入 `try/finally`。
- 记录成功标记。
- 加监控：可用许可、等待线程数、拒绝数。

### 条件变量信号丢失

常见原因：

- 条件检查和等待没有放在同一把锁里。
- 用 `if` 而不是 `while`。
- `notify` 唤醒了不满足条件的线程。
- 多条件共享一个 wait set 导致混乱。

修复：

- 用 `while`。
- 必要时用 `notifyAll`。
- 多条件用 `Condition` 拆分。
- 优先使用 `BlockingQueue` 等成熟工具。

## 常见追问

### mutex 和 semaphore 区别

mutex 是互斥，通常同一时刻只允许一个线程进入，且强调持有者释放。semaphore 是计数许可，可允许 N 个线程同时进入，也常用于资源限流。

### condition 为什么要配合锁

条件变量等待的是共享状态，状态检查和状态修改必须在同一把锁下完成，否则会出现竞态和信号丢失。

### 读写锁一定比互斥锁快吗

不一定。只有读多写少、读临界区足够长时才可能收益明显。写多、临界区短、锁竞争不高时，读写锁额外开销可能更大。

### 公平锁为什么吞吐低

公平锁要按队列顺序唤醒和获取，减少插队，但会增加调度和上下文切换，降低局部性。非公平锁允许刚运行的线程快速再次获取锁，吞吐通常更高。

## 易错点

- 忘记在 `finally` 中释放 `Lock` 或 `Semaphore`。
- 条件等待用 `if`。
- 在锁内调用下游 RPC 或执行慢 IO。
- 认为读写锁在所有场景都更快。
- 尝试读锁升级写锁。
- 用 `notify` 处理多个条件队列，导致唤醒错误线程。
- 只看 Java 状态，不结合栈顶和锁对象判断。

## 自检清单

- 能否区分互斥量、信号量、条件变量、读写锁。
- 能否写出正确的 `Lock` 和 `Condition` 模板。
- 能否解释 `wait/await` 为什么要放在 while 中。
- 能否说明读写锁适用和不适用场景。
- 能否用 `jstack` 判断 monitor 锁竞争和 AQS 等待。
- 能否给出锁竞争、信号量泄漏、条件等待错误的修复方案。
