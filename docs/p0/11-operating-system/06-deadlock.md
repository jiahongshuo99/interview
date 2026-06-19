# 死锁条件、死锁排查

## 面试定位

死锁是并发排障高频题。Java 后端面试通常会从“四个必要条件”问起，再追问如何用 `jstack` 定位、如何修复、数据库死锁和线程死锁有什么不同、线上能不能直接重启。

回答目标：

- 能说清死锁定义和四个必要条件。
- 能写出最小死锁示例。
- 能看懂线程 dump 中的锁等待环。
- 能给出预防、避免、检测、恢复手段。
- 能把 Java 锁、数据库锁、连接池等待区分开。

## 核心概念

死锁是多个执行单元互相等待对方持有的资源，导致所有相关执行单元都无法继续推进。

四个必要条件：

| 条件 | 含义 | 破坏方式 |
| --- | --- | --- |
| 互斥 | 资源同一时刻只能被一个线程持有 | 减少互斥资源，使用无锁或共享读 |
| 持有并等待 | 持有已有资源时继续等待新资源 | 一次性申请资源，或等待前释放已有资源 |
| 不可剥夺 | 资源不能被强制抢走，只能持有者释放 | 引入超时、取消、回滚 |
| 循环等待 | 存在资源等待环 | 固定加锁顺序，资源排序 |

这四个条件同时成立才可能死锁。预防死锁通常就是破坏其中一个条件，最常用的是破坏循环等待和持有并等待。

## Java 死锁示例

```java
public class DeadlockDemo {
    private static final Object A = new Object();
    private static final Object B = new Object();

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (A) {
                sleep(100);
                synchronized (B) {
                    System.out.println("t1 done");
                }
            }
        }, "t1").start();

        new Thread(() -> {
            synchronized (B) {
                sleep(100);
                synchronized (A) {
                    System.out.println("t2 done");
                }
            }
        }, "t2").start();
    }

    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

`t1` 持有 A 等 B，`t2` 持有 B 等 A，形成循环等待。

## 线程 dump 中怎么看

导出线程栈：

```bash
jstack -l <pid> > /tmp/jstack.txt
jcmd <pid> Thread.print -l > /tmp/thread.txt
```

Java monitor 死锁通常会看到：

```text
Found one Java-level deadlock:
```

重点看：

- 线程名。
- Java 线程状态。
- `waiting to lock` 等待哪个锁。
- `locked` 已经持有哪些锁。
- 锁对象地址是否构成等待环。
- 业务代码栈位置。

典型片段：

```text
"t1" BLOCKED
  waiting to lock <0x0000000712345678> (a java.lang.Object)
  locked <0x0000000787654321> (a java.lang.Object)

"t2" BLOCKED
  waiting to lock <0x0000000787654321> (a java.lang.Object)
  locked <0x0000000712345678> (a java.lang.Object)
```

这说明 `t1` 等 `t2` 持有的锁，`t2` 等 `t1` 持有的锁。

## ReentrantLock 死锁

`ReentrantLock` 也可能死锁，但线程 dump 展示方式和 monitor 不完全一样，常见为 `WAITING parking`。

示例：

```java
ReentrantLock a = new ReentrantLock();
ReentrantLock b = new ReentrantLock();

new Thread(() -> {
    a.lock();
    try {
        sleep(100);
        b.lock();
        try {
            doWork();
        } finally {
            b.unlock();
        }
    } finally {
        a.unlock();
    }
}).start();
```

排查时关注：

- 栈中是否出现 `AbstractQueuedSynchronizer`、`LockSupport.park`。
- 业务栈持锁位置。
- 是否有 `ReentrantLock` 对象和 owner 信息。
- Arthas `thread -b` 对 monitor 更直接，对 AQS 需要结合栈分析。

## 不是死锁但很像死锁的情况

### 锁竞争

大量线程等待同一把锁，但锁持有者仍在执行，只是很慢。这不是死锁，是锁竞争或临界区过长。

### 线程池饥饿

任务 A 在线程池中等待任务 B 的结果，但 B 也提交到同一个已满线程池，导致无法执行。

```java
Future<String> f = pool.submit(() -> query());
return f.get(); // 如果发生在线程池工作线程内，可能导致饥饿
```

这类问题线程 dump 可能显示大量 `WAITING`，但不一定有 Java-level deadlock。

### 连接池耗尽

线程都在等待数据库连接或 HTTP 连接，像“卡死”，但根因是资源池耗尽。

### 下游无超时

线程阻塞在 RPC、HTTP、Redis、DB 调用上，长期不返回。不是锁死，但业务不可用。

## 死锁预防

### 固定加锁顺序

对资源排序，所有线程按同一顺序加锁。

```java
void transfer(Account from, Account to, long amount) {
    Account first = from.id() < to.id() ? from : to;
    Account second = from.id() < to.id() ? to : from;

    synchronized (first) {
        synchronized (second) {
            doTransfer(from, to, amount);
        }
    }
}
```

### tryLock 加超时

```java
if (a.tryLock(100, TimeUnit.MILLISECONDS)) {
    try {
        if (b.tryLock(100, TimeUnit.MILLISECONDS)) {
            try {
                doWork();
            } finally {
                b.unlock();
            }
        } else {
            degrade();
        }
    } finally {
        a.unlock();
    }
}
```

超时失败后释放已有资源，破坏不可剥夺或持有并等待。

### 减少锁内操作

锁内不要做：

- RPC。
- DB 慢查询。
- 文件 IO。
- 日志大量输出。
- 阻塞队列等待。
- 调用未知外部回调。

### 使用更高层并发工具

优先使用：

- `BlockingQueue`。
- `ConcurrentHashMap`。
- `Semaphore`。
- `CountDownLatch`。
- `CompletableFuture`。
- 不可变快照。

减少手写多锁组合。

## 数据库死锁

数据库死锁是事务之间互相等待行锁、间隙锁或表锁。

常见原因：

- 多个事务更新相同多行但顺序不同。
- 范围更新触发间隙锁。
- 缺少索引导致锁范围扩大。
- 大事务持锁时间长。
- 先查后改和批量更新顺序不一致。

排查：

```sql
SHOW ENGINE INNODB STATUS;
```

关注：

- LATEST DETECTED DEADLOCK。
- 两个事务执行的 SQL。
- 等待和持有的锁。
- 索引和锁范围。

修复：

- 固定更新顺序。
- 补充合适索引，缩小锁范围。
- 拆小事务。
- 避免长事务。
- 失败后按幂等语义重试。

Java 线程死锁和数据库死锁可能互相影响，但排查工具不同。线程死锁看线程 dump，数据库死锁看数据库锁等待和死锁日志。

## 线上排查步骤

1. 保留现场：

```bash
date
jstack -l <pid> > /tmp/jstack-1.txt
sleep 5
jstack -l <pid> > /tmp/jstack-2.txt
sleep 5
jstack -l <pid> > /tmp/jstack-3.txt
```

2. 查看阻塞线程：

```bash
grep -n "BLOCKED" -A 30 /tmp/jstack-1.txt
grep -n "Found one Java-level deadlock" -A 80 /tmp/jstack-1.txt
```

3. 看锁等待对象和业务栈。

4. 如果无 Java-level deadlock，继续看：

- 是否线程池耗尽。
- 是否连接池耗尽。
- 是否下游无超时。
- 是否数据库锁等待。
- 是否大量 `WAITING parking`。

5. 恢复策略：

- 如果确认死锁且无自动恢复能力，通常需要重启实例或隔离流量。
- 如果是连接池或下游卡住，优先摘流、限流、熔断、恢复下游。
- 恢复后补根因修复和监控。

## 常见追问

### 死锁和饥饿有什么区别

死锁是多个线程形成等待环，彼此都无法推进。饥饿是某个线程长期得不到资源，但系统整体仍可能有其他线程在推进。

### 死锁和活锁有什么区别

死锁是大家都不动。活锁是线程一直在响应对方、改变状态，但始终没有实际进展，例如两个人同时让路又同时让回。

### synchronized 死锁 JVM 能自动解除吗

不能。JVM 可以检测并在线程 dump 中报告 Java-level deadlock，但不会自动释放锁，因为强制释放可能破坏对象状态。

### tryLock 为什么能降低死锁风险

`tryLock` 可以在等待失败或超时后放弃，并释放已持有资源，破坏无限等待，避免永久等待环。

## 易错点

- 只背四个条件，不会结合代码和线程 dump。
- 把所有卡住都叫死锁。
- 忽略线程池饥饿和连接池耗尽。
- 多把锁没有固定顺序。
- 锁内调用外部接口。
- 使用 `tryLock` 但失败路径没有释放已持有资源。
- 数据库死锁不做幂等重试。

## 自检清单

- 能否说清死锁四个必要条件。
- 能否写出 Java 死锁示例。
- 能否从线程 dump 中找出等待环。
- 能否区分死锁、锁竞争、线程池饥饿、连接池耗尽。
- 能否给出固定加锁顺序和 `tryLock` 超时方案。
- 能否说明数据库死锁的常见原因和排查方式。
