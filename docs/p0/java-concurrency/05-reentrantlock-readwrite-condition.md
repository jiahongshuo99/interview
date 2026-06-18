# ReentrantLock、读写锁、条件变量

## 面试定位

`ReentrantLock` 是显式锁代表，常用于和 `synchronized` 对比。读写锁和条件变量则考察候选人是否会根据读写比例、等待条件拆分同步策略。

高频追问：

- `ReentrantLock` 相比 `synchronized` 多了哪些能力。
- 公平锁和非公平锁有什么区别。
- 为什么必须在 `finally` 里 `unlock()`。
- `Condition` 和 `wait/notify` 有什么区别。
- `ReadWriteLock` 适合什么场景，有什么坑。
- 锁降级是什么，锁升级为什么危险。

## ReentrantLock 核心概念

`ReentrantLock` 位于 `java.util.concurrent.locks` 包，是基于 AQS 实现的可重入互斥锁。

主要特性：

- 可重入。
- 可选择公平或非公平。
- 支持可中断获取锁。
- 支持超时获取锁。
- 支持非阻塞尝试获取锁。
- 支持多个 `Condition` 条件队列。

基本模板：

```java
class Counter {
    private final ReentrantLock lock = new ReentrantLock();
    private int value;

    int incrementAndGet() {
        lock.lock();
        try {
            return ++value;
        } finally {
            lock.unlock();
        }
    }
}
```

`unlock()` 必须放在 `finally`，否则业务异常会导致锁永远不释放。

## 可重入性

同一个线程可以多次获得同一把锁，锁内部会维护 hold count。

```java
class Service {
    private final ReentrantLock lock = new ReentrantLock();

    void outer() {
        lock.lock();
        try {
            inner();
        } finally {
            lock.unlock();
        }
    }

    void inner() {
        lock.lock();
        try {
            doWork();
        } finally {
            lock.unlock();
        }
    }
}
```

加锁几次就要解锁几次，否则锁仍被当前线程持有。

## 公平锁和非公平锁

### 非公平锁

默认构造：

```java
ReentrantLock lock = new ReentrantLock();
```

非公平锁允许新来的线程直接尝试抢锁，即插队。优点是吞吐高，缺点是可能造成等待线程饥饿。

### 公平锁

```java
ReentrantLock lock = new ReentrantLock(true);
```

公平锁尽量按等待队列顺序获取锁，减少饥饿，但吞吐通常更低，因为线程切换和队列维护成本更高。

工程中默认优先非公平锁，只有明确需要等待顺序或避免饥饿时才考虑公平锁。

## 获取锁方式

### lock

一直等待，直到获得锁。等待期间不响应中断退出。

```java
lock.lock();
try {
    doWork();
} finally {
    lock.unlock();
}
```

### lockInterruptibly

等待锁时可响应中断。

```java
try {
    lock.lockInterruptibly();
    try {
        doWork();
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    handleCancel();
}
```

适合可取消任务、避免死等。

### tryLock

立即尝试获取锁。

```java
if (lock.tryLock()) {
    try {
        doWork();
    } finally {
        lock.unlock();
    }
} else {
    fallback();
}
```

### tryLock(timeout)

在指定时间内等待锁。

```java
if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
    try {
        doWork();
    } finally {
        lock.unlock();
    }
} else {
    degrade();
}
```

适合线上保护，避免无限等待。

## Condition 条件变量

`Condition` 类似 `wait/notify`，但一把锁可以创建多个条件队列。

```java
class BoundedBuffer<E> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();
    private final Queue<E> queue = new ArrayDeque<>();
    private final int capacity;

    BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }

    void put(E e) throws InterruptedException {
        lock.lockInterruptibly();
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
        lock.lockInterruptibly();
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

重点：

- `await()` 必须在持有对应锁时调用。
- `await()` 会释放锁并进入条件队列。
- 被 `signal()` 唤醒后，需要重新竞争锁。
- 等待条件必须用 `while` 检查。

## Condition 和 wait/notify 对比

| 维度 | Condition | wait/notify |
| --- | --- | --- |
| 所属 | `Lock` | `Object` monitor |
| 条件队列数量 | 一把锁可多个 | 一个对象一个 WaitSet |
| 等待方法 | `await` | `wait` |
| 唤醒方法 | `signal` / `signalAll` | `notify` / `notifyAll` |
| 可中断等待 | 支持 | 支持 |
| 超时等待 | 支持 | 支持 |
| 搭配锁 | `ReentrantLock` 等 | `synchronized` |

多个条件队列可以减少无效唤醒，例如生产者只唤醒消费者，消费者只唤醒生产者。

## ReadWriteLock

`ReadWriteLock` 将读和写分离：

- 读锁：多个读线程可同时持有。
- 写锁：独占，与读锁和其他写锁互斥。

常用实现是 `ReentrantReadWriteLock`。

```java
class ConfigCache {
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    private Map<String, String> config = Map.of();

    String get(String key) {
        readLock.lock();
        try {
            return config.get(key);
        } finally {
            readLock.unlock();
        }
    }

    void reload() {
        Map<String, String> newConfig = loadConfig();
        writeLock.lock();
        try {
            config = newConfig;
        } finally {
            writeLock.unlock();
        }
    }
}
```

## 读写锁适用场景

适合：

- 读多写少。
- 读操作持锁时间明显大于锁开销。
- 读操作之间没有互斥要求。
- 写操作不频繁。

不适合：

- 写很多，读写竞争频繁。
- 临界区非常短，读写锁管理成本超过收益。
- 读操作中会升级写锁。
- 业务要求非常低延迟且写锁可能被大量读长期阻塞。

## 锁降级和锁升级

### 锁降级

写锁降级为读锁是允许的：持有写锁时获取读锁，再释放写锁。

```java
writeLock.lock();
try {
    updateData();
    readLock.lock();
} finally {
    writeLock.unlock();
}

try {
    readData();
} finally {
    readLock.unlock();
}
```

这样可以保证从写到读期间数据不被其他写线程修改。

### 锁升级

持有读锁时再获取写锁通常危险，可能死锁。多个读线程都试图升级写锁时，大家都持有读锁并等待其他读锁释放，无法推进。

正确方式通常是释放读锁后再获取写锁，并重新检查条件。

```java
readLock.lock();
try {
    if (cacheValid()) {
        return value;
    }
} finally {
    readLock.unlock();
}

writeLock.lock();
try {
    if (!cacheValid()) {
        reload();
    }
    return value;
} finally {
    writeLock.unlock();
}
```

## StampedLock 简述

`StampedLock` 提供：

- 写锁。
- 悲观读锁。
- 乐观读。

乐观读适合读多写少且允许校验失败重试的场景。

```java
class Point {
    private final StampedLock lock = new StampedLock();
    private double x;
    private double y;

    double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();
        double currentX = x;
        double currentY = y;
        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

注意：`StampedLock` 不可重入，使用复杂度更高，不要无脑替换读写锁。

## 底层机制

`ReentrantLock` 和 `ReentrantReadWriteLock` 底层都基于 AQS：

- 使用 volatile `state` 表示同步状态。
- 使用 CAS 修改 state。
- 获取失败的线程进入 CLH 变体同步队列。
- 通过 `LockSupport.park/unpark` 阻塞和唤醒线程。

`Condition` 有独立条件队列。线程 `await` 后从同步队列转移到条件队列；`signal` 后再转回同步队列竞争锁。

## 工程实践

- 显式锁必须成对释放，固定模板是 `lock.lock(); try {...} finally { lock.unlock(); }`。
- 可取消任务优先使用 `lockInterruptibly()` 或带超时 `tryLock()`。
- 非公平锁吞吐通常更高，公平锁只在明确需要时使用。
- 多条件等待优先用多个 `Condition` 表达不同条件。
- 读写锁保护的数据，读和写必须都走同一把读写锁。
- 锁内不要做不可控慢调用。
- 读写锁写锁饥饿、读锁长期占用需要监控。
- 简单互斥优先用 `synchronized`，除非确实需要显式锁能力。

## 常见追问

### 1. ReentrantLock 一定比 synchronized 快吗？

不一定。现代 JVM 对 `synchronized` 做了大量优化。`ReentrantLock` 的优势主要是功能更丰富，而不是绝对性能更好。

### 2. 为什么 unlock 要放 finally？

因为临界区代码可能抛异常。如果不在 finally 释放锁，其他线程会永久等待。

### 3. Condition 为什么要 while 判断？

因为存在虚假唤醒，也可能多个线程被唤醒后条件被其他线程先消费。

### 4. 读写锁为什么可能比互斥锁更慢？

如果临界区很短或写竞争频繁，读写锁的状态维护、队列管理、锁降级等成本可能超过并发读收益。

### 5. ReentrantReadWriteLock 的读锁可以升级为写锁吗？

不推荐，容易死锁。通常释放读锁后获取写锁，并重新检查条件。

## 线上风险

- 忘记 `unlock()` 导致线程永久阻塞。
- 锁对象粒度过大导致吞吐下降。
- 公平锁在高并发下吞吐明显降低。
- 读写锁中读操作太慢导致写线程饥饿或配置迟迟无法更新。
- 读锁升级写锁造成死锁。
- `Condition.signal()` 唤醒错误条件或漏唤醒，任务永久等待。
- `tryLock()` 失败路径没有降级策略，直接丢业务。

## 易错点

- `tryLock()` 返回 false 时仍执行 `unlock()`。
- `lockInterruptibly()` 捕获中断后不恢复中断标记。
- 读写锁只给写加锁，不给读加锁。
- 用 `if` 而不是 `while` 判断 `Condition` 等待条件。
- 误以为公平锁能严格保证绝对 FIFO。
- 忘记 `StampedLock` 不可重入。
- 持有读锁时直接申请写锁。

## 自检清单

- 能否写出 ReentrantLock 标准 try/finally 模板。
- 能否说明公平锁和非公平锁差异。
- 能否解释 `lock`、`tryLock`、`lockInterruptibly` 的使用场景。
- 能否写出两个 Condition 的生产者消费者模型。
- 能否说明 Condition 和 wait/notify 区别。
- 能否判断读写锁是否适合某个业务场景。
- 能否解释锁降级和读锁升级风险。
- 能否从 AQS 角度概括 ReentrantLock 底层原理。
