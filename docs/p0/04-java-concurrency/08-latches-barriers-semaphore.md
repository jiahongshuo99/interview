# CountDownLatch、CyclicBarrier、Semaphore

## 面试定位

这三个同步工具常用于考察候选人是否能根据业务协作模型选择合适工具：

- `CountDownLatch`：一个或多个线程等待其他任务完成。
- `CyclicBarrier`：一组线程互相等待，到齐后一起继续。
- `Semaphore`：限制同时访问某资源的并发数。

面试官常追问：

- CountDownLatch 和 CyclicBarrier 区别。
- CountDownLatch 能不能复用。
- CyclicBarrier 的 barrier action 在哪个线程执行。
- Semaphore 公平和非公平区别。
- 这些工具底层是否基于 AQS。

## CountDownLatch

### 核心语义

倒计时门闩。初始化一个计数，调用 `countDown()` 计数减一，调用 `await()` 的线程等待计数归零。

```java
CountDownLatch latch = new CountDownLatch(3);

for (int i = 0; i < 3; i++) {
    pool.submit(() -> {
        try {
            doWork();
        } finally {
            latch.countDown();
        }
    });
}

latch.await();
mergeResult();
```

### 特点

- 一次性使用，计数到 0 后不能重置。
- `countDown()` 不会阻塞。
- `await()` 会阻塞直到计数为 0、中断或超时。
- 底层基于 AQS 共享模式，state 表示剩余计数。

### 常见场景

- 主线程等待多个子任务完成。
- 服务启动时等待多个组件初始化完成。
- 压测时让多个线程同时起跑。

同时起跑示例：

```java
CountDownLatch ready = new CountDownLatch(workerCount);
CountDownLatch start = new CountDownLatch(1);
CountDownLatch done = new CountDownLatch(workerCount);

for (int i = 0; i < workerCount; i++) {
    pool.submit(() -> {
        try {
            ready.countDown();
            start.await();
            doRequest();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            done.countDown();
        }
    });
}

ready.await();
start.countDown();
done.await();
```

## CyclicBarrier

### 核心语义

循环屏障。一组线程都调用 `await()` 到达屏障后，屏障打开，所有线程继续执行。打开后可以进入下一轮。

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("all arrived");
});

for (int i = 0; i < 3; i++) {
    pool.submit(() -> {
        doPhaseOne();
        barrier.await();
        doPhaseTwo();
        return null;
    });
}
```

### 特点

- 可复用。
- 参与线程互相等待。
- 可设置 barrier action。
- 任一等待线程被中断、超时或屏障动作失败，屏障会 broken，其他等待线程收到异常。
- 底层使用 `ReentrantLock` 和 `Condition`，不是直接基于 AQS 自定义同步状态。

### barrier action

barrier action 由最后一个到达屏障的线程执行。执行完成后，其他线程继续。

如果 barrier action 抛异常，屏障进入 broken 状态。

### 常见场景

- 多线程分阶段计算。
- 每轮任务都要等所有参与者完成后再进入下一轮。
- 并行模拟、游戏回合、批处理阶段同步。

## Semaphore

### 核心语义

信号量，用许可数限制同时访问某资源的线程数。

```java
Semaphore semaphore = new Semaphore(10);

void handle() throws InterruptedException {
    semaphore.acquire();
    try {
        callLimitedResource();
    } finally {
        semaphore.release();
    }
}
```

最多 10 个线程同时进入临界区。

### 特点

- 可以有多个许可。
- `acquire()` 获取许可，许可不足则等待。
- `release()` 释放许可。
- 支持公平和非公平。
- 底层基于 AQS 共享模式，state 表示剩余许可。

### 常见场景

- 限制访问第三方接口并发。
- 限制数据库批处理并发。
- 限制本地稀缺资源使用量。
- 实现简单连接池。

### 公平与非公平

```java
Semaphore nonFair = new Semaphore(10);
Semaphore fair = new Semaphore(10, true);
```

非公平吞吐更高，但可能插队；公平信号量按等待顺序发放许可，但成本更高。

## 三者对比

| 工具 | 核心作用 | 是否可复用 | 等待模型 | 底层 |
| --- | --- | --- | --- | --- |
| CountDownLatch | 等待计数归零 | 否 | 一个或多个线程等待其他任务完成 | AQS 共享模式 |
| CyclicBarrier | 多线程互相等待到齐 | 是 | 参与线程彼此等待 | ReentrantLock + Condition |
| Semaphore | 限制并发许可 | 是 | 获取许可失败的线程等待 | AQS 共享模式 |

一句话区分：

- `CountDownLatch` 是“别人做完我再走”。
- `CyclicBarrier` 是“大家到齐一起走”。
- `Semaphore` 是“最多 N 个一起进”。

## 异常和中断处理

### CountDownLatch

子任务必须在 `finally` 中 `countDown()`，否则主线程可能永远等待。

```java
try {
    doWork();
} finally {
    latch.countDown();
}
```

主线程等待建议使用超时：

```java
boolean completed = latch.await(30, TimeUnit.SECONDS);
if (!completed) {
    handleTimeout();
}
```

### CyclicBarrier

`await()` 可能抛：

- `InterruptedException`
- `BrokenBarrierException`
- `TimeoutException`

如果一个线程等待超时，其他等待线程也会因为屏障 broken 被唤醒。

```java
try {
    barrier.await(5, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    barrier.reset();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
} catch (BrokenBarrierException e) {
    handleBroken();
}
```

### Semaphore

只有成功 acquire 后才能 release。避免 acquire 失败却 release 增加额外许可。

```java
boolean acquired = semaphore.tryAcquire(100, TimeUnit.MILLISECONDS);
if (!acquired) {
    return fallback();
}
try {
    return doWork();
} finally {
    semaphore.release();
}
```

## 代码示例：第三方接口并发保护

```java
class ThirdPartyClient {
    private final Semaphore limit = new Semaphore(50);

    Response call(Request request) throws InterruptedException {
        if (!limit.tryAcquire(200, TimeUnit.MILLISECONDS)) {
            throw new RejectedExecutionException("too many concurrent calls");
        }
        try {
            return doHttpCall(request);
        } finally {
            limit.release();
        }
    }
}
```

这类保护要配合超时、熔断、监控，否则只是把问题从下游转移到等待队列。

## 代码示例：批量查询聚合

```java
class Aggregator {
    private final ExecutorService pool = Executors.newFixedThreadPool(8);

    List<Result> queryAll(List<Query> queries) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(queries.size());
        ConcurrentLinkedQueue<Result> results = new ConcurrentLinkedQueue<>();

        for (Query query : queries) {
            pool.submit(() -> {
                try {
                    results.add(queryOne(query));
                } finally {
                    latch.countDown();
                }
            });
        }

        if (!latch.await(3, TimeUnit.SECONDS)) {
            throw new TimeoutException("query timeout");
        }
        return List.copyOf(results);
    }
}
```

实际代码中 `TimeoutException` 是受检异常，需要在方法签名或内部转换中处理。也可以用 `CompletableFuture` 简化聚合。

## 工程实践

- `CountDownLatch` 的 `countDown()` 放在 finally。
- 等待类方法尽量使用超时，避免永久挂死。
- `CyclicBarrier` 要处理 broken 状态，必要时 `reset()`。
- `Semaphore` release 必须与成功 acquire 匹配。
- 限流场景中，Semaphore 不替代线程池、熔断、限速器，它只是并发数控制。
- 不要把 Semaphore 许可数设置过大导致保护失效。
- 不要在线程池任务数小于 barrier parties 时使用 CyclicBarrier，否则可能永远等不齐。

## 常见追问

### 1. CountDownLatch 能复用吗？

不能。计数到 0 后不能重置。需要复用的阶段同步可以考虑 `CyclicBarrier` 或 `Phaser`。

### 2. CyclicBarrier 和 CountDownLatch 最大区别？

CountDownLatch 是一个或多个线程等待计数归零，通常任务线程不等待；CyclicBarrier 是参与线程互相等待，到齐后一起继续，并且可复用。

### 3. Semaphore 可以当锁用吗？

许可数为 1 时可以实现互斥效果，但不具备可重入语义，也没有 owner 概念，误 release 可能破坏互斥。一般互斥用 Lock 或 synchronized。

### 4. CyclicBarrier 的 barrier action 谁执行？

最后一个到达屏障的线程执行。

### 5. Semaphore 的 release 可以不由 acquire 的线程调用吗？

可以。Semaphore 没有所有者概念。但工程上要非常小心，错误 release 会增加许可数，破坏并发限制。

## 线上风险

- 子任务异常后没 countDown，主线程永久等待。
- await 不设超时，故障时请求线程堆积。
- barrier parties 大于实际可用线程数，线程池死锁。
- CyclicBarrier 某个参与者超时，整个屏障 broken，未正确处理导致后续轮次失败。
- Semaphore acquire 后异常路径未 release，许可泄漏。
- acquire 失败仍 release，许可膨胀，并发限制失效。
- 用 Semaphore 限制下游但没有快速失败策略，导致本服务线程池被等待耗尽。

## 易错点

- CountDownLatch 和 CyclicBarrier 复用性混淆。
- `countDown()` 写在 try 末尾而不是 finally。
- Semaphore 当作可重入锁使用。
- `tryAcquire()` 返回 false 仍进入临界区。
- barrier action 执行耗时过长，拖住所有等待线程。
- 忽略 `BrokenBarrierException`。
- 线程池大小小于 CyclicBarrier 参与方数量。

## 自检清单

- 能否用一句话区分三者。
- 能否写出 CountDownLatch 等待多个任务完成模板。
- 能否说明 CountDownLatch 底层 state 含义。
- 能否说明 CyclicBarrier broken 的触发条件。
- 能否说明 Semaphore acquire/release 匹配风险。
- 能否判断某个限流场景是否适合 Semaphore。
- 能否解释为什么等待操作建议设置超时。
- 能否说明 parties 大于线程池大小的死锁风险。
