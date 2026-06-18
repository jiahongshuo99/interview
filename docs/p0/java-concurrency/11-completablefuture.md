# CompletableFuture 使用、异常处理、线程池选择

## 面试定位

`CompletableFuture` 是 Java 后端异步编排高频工具。面试常考：

- `Future` 和 `CompletableFuture` 有什么区别。
- `thenApply`、`thenCompose`、`thenCombine` 怎么选。
- `thenApply` 和 `thenApplyAsync` 区别。
- 异常如何处理。
- 默认线程池是什么，为什么不建议直接用。
- 多任务并发聚合如何写。
- 超时、取消、线程池隔离怎么处理。

核心答案：CompletableFuture 用于异步任务编排，必须明确线程池、异常边界和超时策略，否则容易把公共线程池打满或吞异常。

## 基本概念

`CompletableFuture` 同时实现了：

- `Future`
- `CompletionStage`

相比传统 `Future`：

- 支持手动完成。
- 支持链式回调。
- 支持任务组合。
- 支持异常恢复。
- 支持多个异步阶段编排。

基本示例：

```java
CompletableFuture<String> future = CompletableFuture
        .supplyAsync(() -> queryUser(userId), executor)
        .thenApply(user -> user.name());

String name = future.join();
```

## 创建方式

### completedFuture

已有结果：

```java
CompletableFuture<String> future = CompletableFuture.completedFuture("ok");
```

### supplyAsync

有返回值异步任务：

```java
CompletableFuture<User> future =
        CompletableFuture.supplyAsync(() -> userService.query(userId), executor);
```

### runAsync

无返回值异步任务：

```java
CompletableFuture<Void> future =
        CompletableFuture.runAsync(() -> sendLog(event), executor);
```

### 手动完成

```java
CompletableFuture<String> future = new CompletableFuture<>();
future.complete("ok");
future.completeExceptionally(new RuntimeException("failed"));
```

适合回调式 API 转换为 CompletableFuture。

## 同步回调和异步回调

### 不带 Async

```java
future.thenApply(this::convert);
```

回调通常由完成上一个阶段的线程执行。如果上一个阶段已经完成，可能由当前调用线程执行。

风险：回调如果很慢，可能拖住完成线程，例如 RPC 回调线程、Netty 事件循环线程、业务线程。

### 带 Async 但不传 executor

```java
future.thenApplyAsync(this::convert);
```

默认使用 `ForkJoinPool.commonPool()`。

风险：公共池被阻塞任务打满，影响同 JVM 内其他依赖 commonPool 的任务。

### 带 Async 且传 executor

```java
future.thenApplyAsync(this::convert, bizExecutor);
```

工程推荐。明确线程池归属，便于隔离和监控。

## 常用编排方法

### thenApply：同步转换

输入一个值，返回另一个值。

```java
CompletableFuture<String> nameFuture = userFuture
        .thenApply(User::name);
```

### thenAccept：消费结果

```java
userFuture.thenAccept(user -> log.info("user={}", user.id()));
```

返回 `CompletableFuture<Void>`。

### thenRun：不关心上一步结果

```java
future.thenRun(() -> log.info("done"));
```

### thenCompose：异步扁平化

用于一个异步结果依赖另一个异步任务，避免嵌套 future。

错误嵌套：

```java
CompletableFuture<CompletableFuture<Order>> nested =
        userFuture.thenApply(user -> queryOrderAsync(user.id()));
```

正确：

```java
CompletableFuture<Order> orderFuture =
        userFuture.thenCompose(user -> queryOrderAsync(user.id()));
```

### thenCombine：两个任务结果合并

```java
CompletableFuture<User> userFuture =
        CompletableFuture.supplyAsync(() -> queryUser(userId), executor);

CompletableFuture<Account> accountFuture =
        CompletableFuture.supplyAsync(() -> queryAccount(userId), executor);

CompletableFuture<UserProfile> profileFuture = userFuture.thenCombine(
        accountFuture,
        (user, account) -> new UserProfile(user, account)
);
```

### allOf：等待全部完成

```java
CompletableFuture<User> userFuture = queryUserAsync(userId);
CompletableFuture<List<Order>> ordersFuture = queryOrdersAsync(userId);
CompletableFuture<Coupon> couponFuture = queryCouponAsync(userId);

CompletableFuture<UserView> viewFuture = CompletableFuture
        .allOf(userFuture, ordersFuture, couponFuture)
        .thenApply(ignored -> new UserView(
                userFuture.join(),
                ordersFuture.join(),
                couponFuture.join()
        ));
```

`allOf` 返回 `CompletableFuture<Void>`，需要再从原 future 中取结果。

### anyOf：任一完成

```java
CompletableFuture<Object> fastest =
        CompletableFuture.anyOf(cacheFuture, dbFuture);
```

适合竞速查询，但要处理未完成任务的取消和资源消耗。

## 异常处理

### exceptionally

只处理异常，返回 fallback。

```java
CompletableFuture<User> future = queryUserAsync(userId)
        .exceptionally(ex -> {
            log.warn("query user failed", ex);
            return User.EMPTY;
        });
```

### handle

同时处理成功和异常，并返回新结果。

```java
CompletableFuture<Result> future = queryAsync()
        .handle((value, ex) -> {
            if (ex != null) {
                return Result.fallback();
            }
            return Result.success(value);
        });
```

### whenComplete

观察结果，不改变原结果。

```java
CompletableFuture<User> future = queryUserAsync(userId)
        .whenComplete((value, ex) -> {
            if (ex != null) {
                log.error("query failed", ex);
            }
        });
```

注意：`whenComplete` 如果自己抛异常，也会影响后续阶段。

### exception 包装

`join()` 抛 `CompletionException`，`get()` 抛 `ExecutionException`。

```java
try {
    return future.join();
} catch (CompletionException e) {
    Throwable cause = e.getCause();
    throw convert(cause);
}
```

## 超时控制

Java 9 起常用：

```java
future.orTimeout(500, TimeUnit.MILLISECONDS);
```

超时后 future 以 `TimeoutException` 异常完成。

```java
future.completeOnTimeout(defaultValue, 500, TimeUnit.MILLISECONDS);
```

超时后使用默认值完成。

重要：CompletableFuture 超时完成不等于底层任务一定停止。底层 RPC、DB、HTTP 调用仍需要自己的超时设置。

## 线程池选择

### 不推荐默认 commonPool

不传 executor 时，async 阶段默认使用 `ForkJoinPool.commonPool()`。

风险：

- 公共池被阻塞 I/O 打满。
- 不同业务互相影响。
- 线程名不具备业务含义。
- 监控和隔离困难。

### 推荐业务自定义线程池

```java
ThreadPoolExecutor profileExecutor = new ThreadPoolExecutor(
        16,
        32,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(1000),
        new NamedThreadFactory("profile-cf"),
        new ThreadPoolExecutor.AbortPolicy()
);
```

异步任务：

```java
CompletableFuture.supplyAsync(() -> queryUser(userId), profileExecutor);
```

不同下游可以拆不同线程池，避免慢接口拖垮全部异步编排。

## 并发聚合示例

```java
class ProfileService {
    private final Executor executor;

    UserProfile queryProfile(long userId) {
        CompletableFuture<User> userFuture = CompletableFuture
                .supplyAsync(() -> queryUser(userId), executor)
                .orTimeout(300, TimeUnit.MILLISECONDS);

        CompletableFuture<List<Order>> orderFuture = CompletableFuture
                .supplyAsync(() -> queryOrders(userId), executor)
                .completeOnTimeout(List.of(), 500, TimeUnit.MILLISECONDS);

        CompletableFuture<Coupon> couponFuture = CompletableFuture
                .supplyAsync(() -> queryCoupon(userId), executor)
                .exceptionally(ex -> Coupon.EMPTY);

        return CompletableFuture.allOf(userFuture, orderFuture, couponFuture)
                .thenApply(ignored -> new UserProfile(
                        userFuture.join(),
                        orderFuture.join(),
                        couponFuture.join()
                ))
                .join();
    }
}
```

注意：

- 是否允许 fallback 要看业务语义。
- 核心信息失败可能应该整体失败。
- 非核心信息可以超时降级。

## 取消

```java
future.cancel(true);
```

CompletableFuture 的取消会让 future 进入取消状态，但不一定能中断底层正在执行的任务。尤其是已经提交到普通线程池的任务，是否停止取决于任务是否响应中断。

工程上不要把 `cancel(true)` 当成可靠杀死任务的手段。

## 上下文传递

CompletableFuture 跨线程后，ThreadLocal、MDC 不会自动传递。

```java
String traceId = TraceContext.getTraceId();
CompletableFuture.supplyAsync(() -> {
    try {
        TraceContext.setTraceId(traceId);
        MDC.put("traceId", traceId);
        return query();
    } finally {
        MDC.clear();
        TraceContext.clear();
    }
}, executor);
```

更好的方式是封装上下文感知 executor。

## 常见陷阱

### join 阻塞导致死锁

如果在同一个小线程池里提交任务并等待同池其他任务，可能线程饥饿。

```java
CompletableFuture.supplyAsync(() -> {
    return CompletableFuture.supplyAsync(this::queryOther, sameSmallPool).join();
}, sameSmallPool);
```

外层任务占满线程，内层任务没有线程执行，形成线程池饥饿。

### allOf 中异常传播

任一 future 异常，`allOf` 会异常完成。后续 `thenApply` 不会执行。

如果希望每个子任务独立降级，要在子 future 上先 `exceptionally`。

### 回调线程不明确

不带 Async 的回调可能在完成线程执行。不要在 RPC 回调线程或事件循环线程上执行耗时逻辑。

## 工程实践

- 所有 async 方法显式传 executor。
- 不同业务、不同下游做线程池隔离。
- 每个异步任务定义超时和异常策略。
- 核心任务失败要失败，非核心任务可以降级，不能一律吞异常。
- `allOf` 前对子任务分别处理异常，避免一个非核心失败拖垮整体。
- 避免在异步任务中阻塞等待同线程池任务。
- 不要在 commonPool 中跑阻塞 I/O。
- 上下文传递要显式复制并清理。
- 对异步任务耗时、失败、超时、拒绝做监控。

## 常见追问

### 1. thenApply 和 thenCompose 区别？

`thenApply` 做普通值转换；如果转换函数返回的是另一个 CompletableFuture，会得到嵌套 future。`thenCompose` 用于异步任务串联，会把嵌套 future 扁平化。

### 2. thenApply 和 thenApplyAsync 区别？

不带 Async 通常由完成前一阶段的线程执行；带 Async 会提交到线程池执行。不传 executor 时使用 commonPool。

### 3. exceptionally、handle、whenComplete 区别？

`exceptionally` 只处理异常并返回替代值；`handle` 同时处理成功和失败并转换结果；`whenComplete` 只观察结果，不改变正常返回值。

### 4. allOf 如何拿到结果？

`allOf` 只表示全部完成，本身结果是 Void。需要在后续阶段从原始 future 中 `join()` 获取结果。

### 5. CompletableFuture 超时会停止底层任务吗？

不一定。超时只是让 future 以超时结果完成，底层任务可能仍在运行。底层调用必须配置自己的超时和取消机制。

## 线上风险

- 使用 commonPool 跑阻塞 I/O，公共池被打满。
- 异步任务没有超时，请求线程最终仍长时间等待。
- `exceptionally` 吞掉核心异常，返回错误默认值。
- `submit` 或 CompletableFuture 异常未观察，业务静默失败。
- allOf 一个非核心任务失败导致整个接口失败。
- 同线程池嵌套 join，线程饥饿死锁。
- ThreadLocal/MDC 未传递或未清理，日志和权限上下文错乱。
- 异步任务拒绝未处理，接口偶现失败。

## 易错点

- async 不传 executor。
- 认为 `orTimeout` 能终止底层 RPC。
- 在 `whenComplete` 中以为可以恢复异常。
- `allOf` 后忘记处理子 future 异常。
- `join()` 到处调用，把异步又串行化。
- 异步任务和请求主流程共用过小线程池。
- 回调中执行慢操作但使用非 Async 方法。
- 取消 future 后以为任务一定停止。

## 自检清单

- 能否说明 CompletableFuture 相比 Future 的增强点。
- 能否区分 thenApply、thenCompose、thenCombine。
- 能否解释 Async 后缀和 executor 选择。
- 能否写出 allOf 聚合并获取结果的代码。
- 能否说明 exceptionally、handle、whenComplete 区别。
- 能否说明 commonPool 风险。
- 能否解释超时不等于底层任务停止。
- 能否识别同线程池嵌套 join 的饥饿风险。
