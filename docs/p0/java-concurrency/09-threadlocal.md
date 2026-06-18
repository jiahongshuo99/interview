# ThreadLocal 原理、内存泄漏、线程池场景风险

## 面试定位

`ThreadLocal` 是 Java 后端非常常见的上下文传递工具，常用于 traceId、用户信息、租户、事务资源等。面试重点不是会不会调用 `set/get`，而是：

- ThreadLocal 数据到底存在哪里。
- ThreadLocalMap 的 key 为什么是弱引用。
- 为什么会内存泄漏。
- 在线程池中为什么必须 remove。
- InheritableThreadLocal 有什么坑。
- 线程上下文如何在异步任务中传递。

核心答案：ThreadLocal 提供线程隔离变量，每个线程持有自己的 ThreadLocalMap；线程池线程会复用，使用后不清理会造成数据串扰和内存泄漏。

## 基本用法

```java
class TraceContext {
    private static final ThreadLocal<String> TRACE_ID = new ThreadLocal<>();

    static void setTraceId(String traceId) {
        TRACE_ID.set(traceId);
    }

    static String getTraceId() {
        return TRACE_ID.get();
    }

    static void clear() {
        TRACE_ID.remove();
    }
}
```

典型请求入口：

```java
try {
    TraceContext.setTraceId(request.getTraceId());
    doHandle(request);
} finally {
    TraceContext.clear();
}
```

## 核心原理

### 数据结构关系

不是 ThreadLocal 持有所有线程的数据，而是每个 Thread 持有一个 ThreadLocalMap。

简化关系：

```text
Thread
  -> threadLocals: ThreadLocalMap
       Entry(ThreadLocal<?> key, Object value)
```

每个线程有自己的 map：

```text
thread-1.threadLocals: tlA -> valueA1, tlB -> valueB1
thread-2.threadLocals: tlA -> valueA2, tlB -> valueB2
```

同一个 ThreadLocal 对象在不同线程中对应不同 value。

### get 流程

```text
ThreadLocal.get()
  -> 获取当前线程 Thread.currentThread()
  -> 获取当前线程的 ThreadLocalMap
  -> 以当前 ThreadLocal 对象为 key 查找 Entry
  -> 找到则返回 value
  -> 找不到则调用 initialValue 并 set
```

### set 流程

```text
ThreadLocal.set(value)
  -> 获取当前线程 ThreadLocalMap
  -> key 为当前 ThreadLocal 对象
  -> value 为传入值
  -> 放入当前线程自己的 map
```

### remove 流程

`remove()` 删除当前线程 ThreadLocalMap 中当前 ThreadLocal 对应的 Entry。

这一步在线程池中非常重要。

## 弱引用 key

ThreadLocalMap.Entry 的 key 是弱引用：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
}
```

为什么 key 是弱引用：

- 如果 ThreadLocal 对象本身不再被业务代码引用，GC 可以回收 key。
- 避免 ThreadLocal 对象因为线程长期存活而无法释放。

但 value 是强引用。如果 key 被回收，Entry 可能变成：

```text
key = null
value = largeObject
```

只要线程还活着，ThreadLocalMap 还活着，value 就可能继续被引用。

## 内存泄漏原因

典型泄漏链路：

```text
线程池 worker Thread
  -> Thread.threadLocals
  -> ThreadLocalMap
  -> Entry
  -> value 强引用大对象
```

即使 key 是弱引用被 GC 清掉，value 仍可能被 Entry 强引用。

ThreadLocalMap 在 `get/set/remove` 时会顺带清理一部分 stale entry，但不能依赖这种被动清理。线程池线程长期存活时，未 remove 的 value 可能长期驻留。

## 线程池场景风险

线程池中的线程会复用。如果请求 A 设置了 ThreadLocal 但没有清理，请求 B 复用同一个线程时可能读到请求 A 的数据。

错误示例：

```java
static final ThreadLocal<Long> USER_ID = new ThreadLocal<>();

void handle(Request request) {
    USER_ID.set(request.userId());
    doBusiness();
    // 忘记 remove
}
```

风险：

- 用户身份串扰。
- traceId 串扰。
- 租户串扰。
- 权限判断错误。
- 大对象长期无法释放。

正确示例：

```java
void handle(Request request) {
    try {
        USER_ID.set(request.userId());
        doBusiness();
    } finally {
        USER_ID.remove();
    }
}
```

## withInitial

可以提供初始值：

```java
private static final ThreadLocal<SimpleDateFormat> FORMATTER =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
```

`SimpleDateFormat` 非线程安全，过去常用 ThreadLocal 包装。现代 Java 更推荐使用线程安全的 `DateTimeFormatter`。

```java
private static final DateTimeFormatter FORMATTER =
        DateTimeFormatter.ofPattern("yyyy-MM-dd");
```

## InheritableThreadLocal

`InheritableThreadLocal` 允许子线程创建时继承父线程的值。

```java
InheritableThreadLocal<String> context = new InheritableThreadLocal<>();
context.set("trace-1");

new Thread(() -> {
    System.out.println(context.get()); // trace-1
}).start();
```

线程池中有严重限制：

- 继承发生在线程创建时。
- 线程池 worker 通常早已创建。
- 后续提交任务时不会自动继承提交线程上下文。
- 还可能继承到创建 worker 时的旧上下文。

所以不要依赖 `InheritableThreadLocal` 在线程池中传递请求上下文。

## 异步任务上下文传递

ThreadLocal 不会自动跨线程传递。

错误理解：

```java
TRACE_ID.set("t1");
CompletableFuture.runAsync(() -> {
    System.out.println(TRACE_ID.get()); // 通常拿不到 t1
});
```

常见解决方式：

### 1. 显式传参

最可靠。

```java
String traceId = TraceContext.getTraceId();
pool.submit(() -> doAsync(traceId));
```

### 2. 包装 Runnable/Callable

```java
class ContextAwareRunnable implements Runnable {
    private final String traceId;
    private final Runnable delegate;

    ContextAwareRunnable(Runnable delegate) {
        this.traceId = TraceContext.getTraceId();
        this.delegate = delegate;
    }

    @Override
    public void run() {
        String old = TraceContext.getTraceId();
        try {
            TraceContext.setTraceId(traceId);
            delegate.run();
        } finally {
            if (old == null) {
                TraceContext.clear();
            } else {
                TraceContext.setTraceId(old);
            }
        }
    }
}
```

### 3. 使用框架能力

例如日志 MDC、链路追踪框架、TransmittableThreadLocal 等。使用时仍要理解清理边界，避免上下文污染。

## ThreadLocal 和 MDC

日志框架的 MDC 通常基于 ThreadLocal。请求入口设置 traceId，出口清理。

```java
try {
    MDC.put("traceId", traceId);
    chain.doFilter(request, response);
} finally {
    MDC.clear();
}
```

异步任务中 MDC 也不会天然传递，需要包装 executor 或显式复制上下文。

## 代码示例：过滤器中使用

```java
class TraceFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request,
                         ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        String traceId = extractTraceId(request);
        try {
            TraceContext.setTraceId(traceId);
            MDC.put("traceId", traceId);
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
            TraceContext.clear();
        }
    }
}
```

清理顺序通常放在 finally 中，确保业务异常也能清理。

## 底层细节

### ThreadLocalMap 哈希

ThreadLocalMap 不是普通 HashMap。它使用开放寻址法处理冲突，Entry 数组中 key 是 ThreadLocal 弱引用。

ThreadLocal 内部有 threadLocalHashCode，并使用固定增量生成，以减少连续创建 ThreadLocal 时的哈希冲突。

### stale entry 清理

当 key 为 null 时，Entry 称为 stale entry。ThreadLocalMap 在 set/get/remove 时会做局部清理，但不是实时全量清理。

因此工程上必须主动 remove。

## 工程实践

- ThreadLocal 通常定义为 `private static final`。
- 每次请求入口 set，出口 finally remove。
- 不要存放大对象、连接、文件句柄等重资源。
- 不要把 ThreadLocal 当全局参数传递工具滥用，优先显式传参。
- 线程池、异步任务、CompletableFuture 场景必须显式处理上下文传播。
- MDC 使用后必须 clear。
- 框架封装 ThreadLocal 时，要明确生命周期边界。
- 单元测试中也要清理 ThreadLocal，避免测试间污染。

## 常见追问

### 1. ThreadLocal 是怎么做到线程隔离的？

每个线程对象内部都有自己的 ThreadLocalMap。同一个 ThreadLocal 在不同线程的 map 中对应不同 value，所以线程之间互不影响。

### 2. key 为什么是弱引用？

为了当业务不再持有 ThreadLocal 对象时，ThreadLocal key 可以被 GC 回收，避免 key 因线程长期存活而泄漏。

### 3. key 是弱引用为什么还会泄漏？

value 是强引用。key 被回收后 Entry 的 value 仍可能被线程的 ThreadLocalMap 引用。线程池线程长期存活时，value 也长期不释放。

### 4. 为什么线程池中必须 remove？

线程池线程复用，不 remove 会造成下一次任务读到上一次任务的数据，也可能导致 value 长期驻留内存。

### 5. InheritableThreadLocal 能解决线程池上下文传递吗？

不能可靠解决。它只在线程创建时继承，线程池 worker 复用时不会按每次任务提交动态继承。

## 线上风险

- 用户上下文串扰导致越权。
- traceId 串扰导致日志链路错误。
- 大对象放入 ThreadLocal 未清理，引发内存泄漏。
- 异步任务拿不到上下文，导致审计字段、租户字段丢失。
- 单元测试之间 ThreadLocal 未清理导致偶现失败。
- MDC 未 clear，日志显示错误请求信息。

## 易错点

- 认为 ThreadLocal 对象自己保存 value。
- 认为 key 是弱引用就不会内存泄漏。
- 在线程池任务中 set 后不 remove。
- 用 InheritableThreadLocal 传递线程池任务上下文。
- ThreadLocal 保存可变集合并在多层业务中随意修改。
- 把 ThreadLocal 当成方法参数的替代品。
- 异步回调中直接读取请求线程 ThreadLocal。

## 自检清单

- 能否画出 Thread、ThreadLocalMap、Entry 的引用关系。
- 能否解释弱引用 key 和强引用 value 的泄漏链路。
- 能否说明线程池复用导致的数据串扰。
- 能否写出 try/finally remove 模板。
- 能否说明 InheritableThreadLocal 的线程池限制。
- 能否给出异步任务上下文传递方案。
- 能否解释 MDC 和 ThreadLocal 的关系。
- 能否说明为什么不建议 ThreadLocal 存重资源。
