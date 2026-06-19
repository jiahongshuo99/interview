# AQS 原理

## 面试定位

AQS，全称 AbstractQueuedSynchronizer，是 JUC 中大量同步器的基础。面试通常不会要求逐行背源码，但会要求理解：

- AQS 解决什么问题。
- `state`、CLH 队列、独占模式、共享模式分别是什么。
- ReentrantLock、Semaphore、CountDownLatch 如何基于 AQS。
- 获取锁失败后线程如何排队、阻塞、唤醒。
- Condition 和 AQS 队列有什么关系。

回答时要抓住主线：AQS 用一个 volatile state 表示同步状态，用 FIFO 队列管理等待线程，用 CAS 修改状态，用 park/unpark 挂起和唤醒线程。

## AQS 是什么

AQS 是一个用于构建锁和同步器的框架，位于：

```java
java.util.concurrent.locks.AbstractQueuedSynchronizer
```

它把通用排队、阻塞、唤醒、状态管理抽出来，具体同步器只需要定义：

- 如何尝试获取同步状态。
- 如何尝试释放同步状态。

典型子类或使用者：

- `ReentrantLock`
- `ReentrantReadWriteLock`
- `Semaphore`
- `CountDownLatch`
- `FutureTask`
- `ThreadPoolExecutor.Worker`

## 核心组件

### state

AQS 内部有一个 volatile int `state`，表示同步状态。

不同同步器含义不同：

| 同步器 | state 含义 |
| --- | --- |
| ReentrantLock | 重入次数，0 表示未持有 |
| Semaphore | 剩余许可数 |
| CountDownLatch | 剩余计数 |
| ReentrantReadWriteLock | 高位读锁计数，低位写锁重入计数 |

修改 state 通常使用 CAS。

### 同步队列

AQS 维护一个 FIFO 双向队列，是 CLH 队列的变体。获取同步状态失败的线程会被封装为 Node 加入队列。

简化结构：

```text
head <-> node(thread-1) <-> node(thread-2) <-> tail
```

队列中线程通过 `LockSupport.park()` 阻塞，由前驱节点或释放线程唤醒。

### Node

Node 中关键信息包括：

- 当前线程。
- 前驱节点。
- 后继节点。
- 等待状态。
- 独占或共享模式标记。

等待状态用于表示节点是否取消、是否需要唤醒后继等。

## 独占模式

独占模式下，同一时刻只有一个线程能获取同步状态。

典型同步器：

- `ReentrantLock`

核心方法：

- `tryAcquire(int arg)`
- `tryRelease(int arg)`

AQS 负责排队和阻塞，子类负责判断能不能获取。

## ReentrantLock 获取锁流程

非公平锁简化流程：

```text
lock()
  -> CAS state 0 -> 1 成功，设置 owner 为当前线程，返回
  -> 失败则 tryAcquire
       -> 如果 state=0，继续 CAS 抢锁
       -> 如果 owner 是当前线程，state++，重入成功
       -> 否则失败
  -> acquire 入队
  -> 前驱是 head 时反复尝试获取
  -> 获取失败则 park
```

公平锁区别：

- 获取前会检查队列中是否已有前驱节点。
- 如果已有等待线程，新线程不插队。

## ReentrantLock 释放锁流程

简化流程：

```text
unlock()
  -> tryRelease
       -> state--
       -> 如果 state=0，清空 owner，释放成功
  -> 如果释放成功，唤醒 head 后继节点
```

可重入锁必须释放到 state 为 0，其他线程才有机会获得锁。

## 共享模式

共享模式下，多个线程可以同时获取同步状态。

典型同步器：

- `Semaphore`
- `CountDownLatch`
- `ReentrantReadWriteLock` 的读锁

核心方法：

- `tryAcquireShared(int arg)`
- `tryReleaseShared(int arg)`

返回值语义通常表示：

- 小于 0：获取失败。
- 等于 0：获取成功，但后续共享获取可能不能继续传播。
- 大于 0：获取成功，后续共享获取可继续传播。

## Semaphore 基于 AQS

`Semaphore` 的 state 表示许可数。

获取许可：

```text
acquire
  -> 读取 state
  -> next = state - permits
  -> next < 0 则失败入队
  -> CAS state 为 next 成功则获取成功
```

释放许可：

```text
release
  -> CAS state 增加 permits
  -> 唤醒等待线程
```

## CountDownLatch 基于 AQS

`CountDownLatch` 的 state 表示剩余计数。

- `await()`：共享获取，只有 state 为 0 才成功，否则入队等待。
- `countDown()`：共享释放，CAS 将 state 减一，减到 0 时唤醒所有等待线程。

`CountDownLatch` 一次性使用，state 到 0 后不能重置。

## Condition 原理

`ConditionObject` 是 AQS 的内部类。

它维护单独的条件队列：

```text
condition queue: firstWaiter -> waiter -> lastWaiter
sync queue:      head <-> node <-> tail
```

### await 流程

简化流程：

1. 当前线程必须持有锁。
2. 将当前线程加入条件队列。
3. 完全释放当前持有的锁。
4. park 当前线程。
5. 被 signal 后转移到同步队列。
6. 重新竞争锁。
7. 获取锁后从 `await()` 返回。

### signal 流程

简化流程：

1. 当前线程必须持有锁。
2. 从条件队列取出等待节点。
3. 将节点转移到 AQS 同步队列。
4. 等待它重新竞争锁。

`signal()` 不等于立即执行，被唤醒线程仍要等当前线程释放锁。

## AQS 为什么需要队列

如果获取失败的线程一直自旋，会浪费 CPU。AQS 用队列把竞争失败线程管理起来：

- 保证等待线程有序排队。
- 避免无限自旋。
- 支持取消、中断、超时。
- 释放时精准唤醒合适后继。

这也是很多 JUC 同步器比手写 wait/notify 更可靠的原因。

## LockSupport

AQS 底层通过 `LockSupport.park()` 和 `LockSupport.unpark(thread)` 阻塞和唤醒线程。

特点：

- 基于许可模型。
- `unpark` 可以先于 `park`。
- 每个线程最多一个许可。
- `park` 可能虚假返回，因此上层必须循环检查条件。

## 自定义 AQS 示例

一个简单一次性门闩：

```java
class OneShotLatch {
    private final Sync sync = new Sync();

    void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    void signal() {
        sync.releaseShared(1);
    }

    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected int tryAcquireShared(int ignored) {
            return getState() == 1 ? 1 : -1;
        }

        @Override
        protected boolean tryReleaseShared(int ignored) {
            setState(1);
            return true;
        }
    }
}
```

实际业务中很少需要自定义 AQS，但理解这个模板有助于掌握 AQS 分工。

## 公平与非公平

AQS 支持同步器自行决定公平策略。

非公平锁：

- 新线程可直接 CAS 抢锁。
- 吞吐高。
- 可能插队。

公平锁：

- 获取前检查是否有排队前驱。
- 等待更有序。
- 吞吐较低。

方法 `hasQueuedPredecessors()` 常用于公平性判断。

## 中断和超时

AQS 提供多类获取方法：

- 不响应中断的 acquire。
- 响应中断的 acquireInterruptibly。
- 带超时的 tryAcquireNanos。
- 共享模式对应 shared 版本。

这就是 `ReentrantLock.lockInterruptibly()`、`tryLock(timeout)` 等能力的基础。

## 工程实践

- 一般不要自定义 AQS，优先使用 JDK 已提供同步器。
- 理解 AQS 有助于排查线程 dump 中的 `AbstractQueuedSynchronizer` 栈。
- 看到大量线程 park 在 AQS，要找具体是哪把锁、哪个同步器、owner 线程是谁。
- 使用 `Condition` 时必须循环检查条件。
- 选择公平锁要有明确理由，默认非公平吞吐更好。
- 自定义同步器时，`tryAcquire` / `tryRelease` 必须非常小心状态一致性和异常路径。

## 常见追问

### 1. AQS 的 state 是什么？

`state` 是一个 volatile int，同步器用它表达自己的同步状态。不同同步器含义不同，例如锁重入次数、信号量许可数、倒计时剩余数。

### 2. AQS 获取失败后怎么处理？

线程被封装成 Node 加入同步队列，在合适条件下 park 阻塞。前驱节点释放后唤醒它，它再尝试获取同步状态。

### 3. 独占模式和共享模式区别？

独占模式同一时刻只允许一个线程成功获取；共享模式允许多个线程同时成功获取，例如信号量多个许可、读写锁多个读锁。

### 4. Condition 队列和同步队列区别？

Condition 队列存放等待某个条件的线程；同步队列存放等待获取锁的线程。`signal` 会把节点从条件队列转移到同步队列，不是直接运行。

### 5. 为什么 AQS 用 park/unpark 不直接用 wait/notify？

`park/unpark` 不要求持有对象 monitor，许可可以先发放，适合构建更灵活的同步器。AQS 自己维护队列和状态。

## 线上风险

- AQS 锁等待大量堆积，线程池耗尽。
- 忘记释放 ReentrantLock，AQS 队列永久阻塞。
- Condition 漏 signal，线程一直停在条件队列。
- 高竞争非公平锁导致个别线程饥饿。
- 自定义 AQS 状态更新错误，出现多线程同时进入临界区或永远不唤醒。
- 误读线程状态，把 AQS 的 `WAITING` 都当成死锁。

## 易错点

- 认为 AQS 只用于锁，不知道 CountDownLatch、Semaphore 也基于它。
- 把 Condition 队列和同步队列混为一谈。
- 认为 `signal()` 后等待线程立即持有锁。
- 忽略 `state` 在不同同步器中含义不同。
- 认为公平锁绝对没有插队或性能无损。
- 自定义 AQS 时在 `tryAcquire` 里阻塞线程。
- 忘记 park 可能虚假返回，上层必须循环检查。

## 自检清单

- 能否概括 AQS 的三件套：state、队列、CAS。
- 能否说明 ReentrantLock 如何用 state 表示重入。
- 能否说明 Semaphore 和 CountDownLatch 的 state 含义。
- 能否画出同步队列和条件队列的关系。
- 能否解释 await/signal 的节点转移过程。
- 能否区分独占模式和共享模式。
- 能否解释公平锁如何通过队列前驱判断实现。
- 能否根据线程 dump 识别 AQS 等待。
