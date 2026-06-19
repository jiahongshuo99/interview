# 线程池参数、拒绝策略、队列选择、线程池隔离

## 面试定位

线程池是 Java 后端面试和线上治理的重点。常见问题：

- `ThreadPoolExecutor` 的核心参数是什么。
- 任务提交后线程池如何决策。
- corePoolSize、maximumPoolSize、workQueue 的关系。
- 为什么不推荐 `Executors` 快捷工厂。
- 队列怎么选，拒绝策略怎么选。
- CPU 密集型和 I/O 密集型线程数怎么估算。
- 什么是线程池隔离，为什么要隔离。

回答时一定要结合工程风险：无界队列 OOM、线程数爆炸、拒绝策略吞异常、业务混用线程池互相拖垮。

## ThreadPoolExecutor 核心参数

构造方法：

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
        corePoolSize,
        maximumPoolSize,
        keepAliveTime,
        TimeUnit.SECONDS,
        workQueue,
        threadFactory,
        rejectedExecutionHandler
);
```

参数说明：

| 参数 | 含义 |
| --- | --- |
| `corePoolSize` | 核心线程数 |
| `maximumPoolSize` | 最大线程数 |
| `keepAliveTime` | 非核心线程空闲存活时间 |
| `unit` | 时间单位 |
| `workQueue` | 任务队列 |
| `threadFactory` | 线程创建工厂 |
| `handler` | 拒绝策略 |

## 任务提交流程

`execute()` 简化流程：

```text
提交任务
  |
  | worker 数 < corePoolSize
  v
创建核心线程执行任务
  |
  | 否则尝试入队
  v
入队成功，等待 worker 消费
  |
  | 入队失败且 worker 数 < maximumPoolSize
  v
创建非核心线程执行任务
  |
  | 仍失败
  v
执行拒绝策略
```

重点：

- 只有队列满了，才会继续创建超过 core 的线程。
- 如果使用无界队列，`maximumPoolSize` 通常不会生效。
- 如果使用 `SynchronousQueue`，任务不排队，倾向于直接扩线程。

## 线程池状态

`ThreadPoolExecutor` 内部用一个原子整数同时表示 runState 和 workerCount。

常见状态：

| 状态 | 含义 |
| --- | --- |
| RUNNING | 接收新任务并处理队列任务 |
| SHUTDOWN | 不接收新任务，但处理队列任务 |
| STOP | 不接收新任务，不处理队列任务，中断运行中任务 |
| TIDYING | 所有任务结束，workerCount 为 0 |
| TERMINATED | terminated hook 执行完成 |

`shutdown()` 进入 SHUTDOWN；`shutdownNow()` 尝试进入 STOP。

## 队列选择

### ArrayBlockingQueue

有界数组队列。

优点：

- 容量固定，便于做背压。
- 内存可控。

缺点：

- 需要预估容量。
- 入队出队竞争同一把锁，吞吐可能受限。

适合大多数业务线程池。

### LinkedBlockingQueue

链表队列，可有界也可无界。默认构造是近似无界，容量为 `Integer.MAX_VALUE`。

风险：

- 无界队列可能堆积大量任务导致 OOM。
- 使用无界队列时 maximumPoolSize 基本失效。

建议显式指定容量。

### SynchronousQueue

不存储任务，提交任务必须直接交给 worker。

特点：

- 队列容量为 0。
- 容易触发扩线程。
- 适合任务短、希望快速移交、不希望排队的场景。

`Executors.newCachedThreadPool()` 使用它，但最大线程数很大，线上要谨慎。

### PriorityBlockingQueue

优先级队列，通常无界。

风险：

- 低优先级任务可能饥饿。
- 无界堆积风险。
- 任务必须可比较或提供 comparator。

### DelayQueue

延迟队列，适合延迟任务，不是通用业务线程池队列。

## 拒绝策略

JDK 内置策略：

### AbortPolicy

默认策略，抛 `RejectedExecutionException`。

适合希望调用方感知失败的场景。

### CallerRunsPolicy

由提交任务的线程自己执行任务。

优点：

- 形成一定反压，提交方被拖慢。

风险：

- 如果提交线程是请求线程，会增加请求耗时。
- 如果提交线程持有锁，可能引发复杂阻塞。

### DiscardPolicy

直接丢弃任务，不抛异常。

风险极高，除非任务天然可丢，并有监控。

### DiscardOldestPolicy

丢弃队列中最旧任务，再尝试提交新任务。

风险：

- 可能丢关键任务。
- 与优先级队列结合时语义更复杂。

### 自定义拒绝策略

常见做法：

- 打点监控。
- 记录日志但限频。
- 快速失败。
- 降级到同步执行。
- 投递到备用队列。

示例：

```java
RejectedExecutionHandler handler = (task, executor) -> {
    Metrics.counter("biz_pool_rejected").increment();
    throw new RejectedExecutionException("biz pool overloaded");
};
```

不要在拒绝策略里做慢操作。

## Executors 快捷工厂风险

### newFixedThreadPool

使用无界 `LinkedBlockingQueue`，任务堆积可能 OOM。

### newSingleThreadExecutor

也是无界队列，单线程处理慢时任务无限堆积。

### newCachedThreadPool

最大线程数接近无限，突发流量下可能创建大量线程，导致 OOM 或系统线程耗尽。

### newScheduledThreadPool

延迟队列也可能堆积大量任务。周期任务异常处理不当会停止后续调度。

建议直接使用 `ThreadPoolExecutor` 明确参数。

## 线程数估算

### CPU 密集型

任务主要消耗 CPU，例如计算、加密、压缩。

常用估算：

```text
线程数 = CPU 核数 或 CPU 核数 + 1
```

过多线程只会增加上下文切换。

### I/O 密集型

任务大量等待 I/O，例如 RPC、数据库、文件、网络。

估算公式：

```text
线程数 = CPU 核数 * (1 + 等待时间 / 计算时间)
```

这只是起点，最终要通过压测和线上指标调整。

### 混合型

拆分线程池：

- CPU 计算池。
- RPC 调用池。
- DB 访问池。
- 定时任务池。

不要让慢 I/O 占满 CPU 任务线程池。

## 线程池隔离

线程池隔离是指不同业务、不同下游、不同优先级任务使用不同线程池，避免互相拖垮。

典型隔离维度：

- 核心链路和非核心链路。
- 不同下游服务。
- 实时请求和异步补偿。
- 用户请求和后台任务。
- 快任务和慢任务。

示例：

```java
ThreadPoolExecutor paymentPool = new ThreadPoolExecutor(...);
ThreadPoolExecutor notificationPool = new ThreadPoolExecutor(...);
ThreadPoolExecutor reportPool = new ThreadPoolExecutor(...);
```

如果通知下游变慢，只应拖慢 notificationPool，不应耗尽支付主链路线程。

## 线程工厂

线程命名非常重要：

```java
class NamedThreadFactory implements ThreadFactory {
    private final AtomicInteger index = new AtomicInteger();
    private final String prefix;

    NamedThreadFactory(String prefix) {
        this.prefix = prefix;
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r);
        t.setName(prefix + "-" + index.incrementAndGet());
        t.setUncaughtExceptionHandler((thread, ex) ->
                log.error("uncaught exception in {}", thread.getName(), ex));
        return t;
    }
}
```

线程名要能定位业务池，方便线程 dump 和监控。

## submit 和 execute 区别

### execute

- 接收 `Runnable`。
- 没有返回值。
- 任务异常会交给线程的 UncaughtExceptionHandler，worker 可能退出后被补充。

### submit

- 返回 `Future`。
- 任务异常会被封装在 `Future` 中。
- 如果不调用 `future.get()`，异常可能被静默吞掉。

```java
Future<?> future = executor.submit(() -> {
    throw new RuntimeException("failed");
});

future.get(); // 这里抛 ExecutionException
```

## shutdown

优雅关闭：

```java
executor.shutdown();
if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
    List<Runnable> dropped = executor.shutdownNow();
    log.warn("force shutdown, dropped={}", dropped.size());
}
```

注意：

- `shutdown()` 不再接收新任务，会处理队列中已有任务。
- `shutdownNow()` 尝试中断运行中任务，并返回未执行队列任务。
- 任务必须正确响应中断，否则 shutdownNow 也无法快速停止。

## 监控指标

关键指标：

- activeCount：活跃线程数。
- poolSize：当前线程数。
- corePoolSize / maximumPoolSize。
- queueSize：队列长度。
- completedTaskCount：完成任务数。
- taskCount：任务总数。
- rejectedCount：拒绝次数。
- largestPoolSize：历史最大线程数。
- task latency：排队耗时、执行耗时。

排队耗时比执行耗时更能反映线程池是否饱和。

## 代码示例：可控业务线程池

```java
ThreadPoolExecutor orderExecutor = new ThreadPoolExecutor(
        16,
        32,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(1000),
        new NamedThreadFactory("order-worker"),
        (task, executor) -> {
            Metrics.counter("order_worker_rejected").increment();
            throw new RejectedExecutionException("order worker overloaded");
        }
);
```

根据业务延迟要求，拒绝后可以快速失败、降级、返回稍后重试，不要无限等待。

## 工程实践

- 禁止无脑使用 `Executors` 快捷方法，显式设置队列容量和拒绝策略。
- 线程池必须命名。
- 队列必须有界，除非有非常明确的容量控制。
- 拒绝必须可观测，不要静默丢弃关键任务。
- 不同业务、不同下游做线程池隔离。
- 线程池参数要通过压测和线上指标调优。
- 任务内部要设置 RPC/DB 超时，线程池不是超时控制的替代品。
- 异步任务异常要被记录或上报。
- 上下文 ThreadLocal/MDC 在线程池任务中要复制并清理。

## 常见追问

### 1. corePoolSize 和 maximumPoolSize 如何配合队列？

先创建核心线程；核心满后优先入队；队列满后才创建非核心线程；达到最大线程数后执行拒绝策略。

### 2. 为什么无界队列危险？

生产速度大于消费速度时，任务会无限堆积，占用内存，最终 OOM。并且 maximumPoolSize 难以发挥作用。

### 3. CallerRunsPolicy 有什么风险？

会让提交任务的线程执行任务，形成反压。但如果提交线程是请求线程，会拉长响应；如果持有锁或在事件循环中执行，可能造成更大问题。

### 4. 如何设置线程池大小？

先根据任务类型估算：CPU 密集接近核数，I/O 密集结合等待/计算比例。再通过压测和线上监控调优。

### 5. submit 为什么可能吞异常？

`submit` 会把异常封装进 `Future`，如果不调用 `get()`，异常不会直接暴露到日志。

## 线上风险

- 无界队列导致 OOM。
- 最大线程数过大导致线程爆炸、上下文切换严重。
- 队列过长导致请求排队超时，用户侧延迟雪崩。
- 多业务共用线程池，一个慢下游拖垮全部任务。
- 拒绝策略静默丢任务，数据不一致。
- 异步任务异常未处理，业务失败无人感知。
- ThreadLocal 未清理导致上下文串扰。
- shutdown 不完整导致应用停机卡住或任务丢失。

## 易错点

- 认为 maximumPoolSize 一定会生效，忽略无界队列。
- 使用 `Executors.newFixedThreadPool` 却不知道它是无界队列。
- 只监控活跃线程数，不监控队列长度和拒绝次数。
- `submit` 后不保存 Future，也不处理异常。
- 拒绝策略使用 DiscardPolicy 丢关键业务。
- 所有异步任务共用一个公共线程池。
- 线程名使用默认值，线上无法定位。
- 任务没有超时，线程被慢调用长期占用。

## 自检清单

- 能否写出 ThreadPoolExecutor 七个核心参数。
- 能否说明 execute 提交流程。
- 能否解释不同队列对 maximumPoolSize 的影响。
- 能否列出四种内置拒绝策略及风险。
- 能否说明为什么不推荐 Executors 快捷工厂。
- 能否根据 CPU/I/O 类型估算线程数。
- 能否设计线程池隔离方案。
- 能否列出线程池线上监控指标。
