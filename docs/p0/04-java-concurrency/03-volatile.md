# volatile 的语义与限制

## 面试定位

`volatile` 是 Java 并发高频题，常用于考察候选人是否真正理解 JMM。面试官一般会从一个停止标记、双重检查锁定或 `volatile int count++` 切入，追问：

- `volatile` 保证什么。
- 为什么 `volatile` 不能保证自增安全。
- `volatile` 和 `synchronized` 的区别。
- `volatile` 如何禁止指令重排。
- 哪些场景适合用 `volatile`，哪些不适合。

核心答案：`volatile` 保证可见性和有序性，不保证复合操作的原子性。

## 核心语义

### 可见性

对一个 `volatile` 变量的写，对后续读取该变量的线程立即可见。

```java
class StopFlag {
    private volatile boolean stopped;

    void stop() {
        stopped = true;
    }

    void run() {
        while (!stopped) {
            doWork();
        }
    }
}
```

如果 `stopped` 不是 `volatile`，运行线程可能一直读不到主线程写入的 `true`。

### 有序性

`volatile` 会建立内存屏障，限制特定方向的指令重排。

典型发布模式：

```java
class ConfigHolder {
    private Map<String, String> config;
    private volatile boolean ready;

    void init() {
        config = loadConfig();
        ready = true; // volatile 写
    }

    String get(String key) {
        if (!ready) { // volatile 读
            throw new IllegalStateException("not ready");
        }
        return config.get(key);
    }
}
```

如果线程 B 读到 `ready=true`，则线程 A 在写 `ready=true` 之前对 `config` 的写入对线程 B 可见。

### 单次读写原子性

`volatile` 变量的单次读或单次写具有原子性。对 `long`、`double` 这类 64 位变量，`volatile` 也能避免非原子读写问题。

但复合操作不原子。

## volatile 不保证复合操作原子性

错误示例：

```java
class Counter {
    private volatile int count;

    void incr() {
        count++; // 非线程安全
    }

    int get() {
        return count;
    }
}
```

`count++` 包含读取、加一、写回。多个线程可能同时读到相同旧值，最终丢失更新。

正确做法：

```java
class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger();

    void incr() {
        count.incrementAndGet();
    }

    int get() {
        return count.get();
    }
}
```

或者：

```java
class LockedCounter {
    private int count;

    synchronized void incr() {
        count++;
    }

    synchronized int get() {
        return count;
    }
}
```

## happens-before 规则

对同一个 `volatile` 变量：

- 写操作 happens-before 后续读操作。
- 线程读到 volatile 写后，也能看到写线程在 volatile 写之前的普通写。

示例：

```java
int data;
volatile boolean ready;

// 线程 A
data = 42;
ready = true;

// 线程 B
if (ready) {
    System.out.println(data); // 保证看到 42
}
```

这不是因为 `data` 也是 volatile，而是因为 volatile 写读建立了 happens-before。

## 底层机制

### 内存屏障

JVM 会围绕 volatile 读写插入内存屏障，约束编译器和处理器重排。

粗略理解：

- volatile 写：写之前的普通写不能重排到 volatile 写之后。
- volatile 读：读之后的普通读写不能重排到 volatile 读之前。
- volatile 写会促使值对其他线程可见。
- volatile 读会促使当前线程获取最新值。

不同 CPU 架构和 JVM 版本实现细节不同，面试时以 JMM 语义为准，不要把答案绑定到某条固定机器指令。

### 缓存一致性

volatile 依赖 JVM 和硬件提供的缓存一致性、内存屏障能力。它不是简单地“每次都直接读主内存”，这种说法过于简化。更准确的说法是：JMM 保证 volatile 读能看到之前 volatile 写建立的可见结果。

## 适合使用 volatile 的场景

### 1. 状态标记

```java
class Worker implements Runnable {
    private volatile boolean running = true;

    void shutdown() {
        running = false;
    }

    @Override
    public void run() {
        while (running) {
            doWork();
        }
    }
}
```

前提：循环内没有长期阻塞。若可能阻塞，需要配合 `interrupt()` 或超时机制。

### 2. 一次性发布

```java
class Holder {
    private volatile Config config;

    void refresh() {
        config = loadConfig();
    }

    Config current() {
        return config;
    }
}
```

适合引用整体替换，不适合原地修改共享对象。

### 3. 双重检查锁定

```java
class ServiceRegistry {
    private static volatile ServiceRegistry instance;

    static ServiceRegistry getInstance() {
        ServiceRegistry local = instance;
        if (local == null) {
            synchronized (ServiceRegistry.class) {
                local = instance;
                if (local == null) {
                    local = new ServiceRegistry();
                    instance = local;
                }
            }
        }
        return local;
    }
}
```

`volatile` 防止引用赋值与对象初始化重排，并保证可见性。

### 4. 低成本读多写少配置

```java
final class RuleSnapshot {
    private final Set<String> blockedUsers;

    RuleSnapshot(Set<String> blockedUsers) {
        this.blockedUsers = Set.copyOf(blockedUsers);
    }

    boolean blocked(String userId) {
        return blockedUsers.contains(userId);
    }
}

class RuleService {
    private volatile RuleSnapshot snapshot = new RuleSnapshot(Set.of());

    void reload() {
        snapshot = new RuleSnapshot(loadBlockedUsers());
    }

    boolean blocked(String userId) {
        return snapshot.blocked(userId);
    }
}
```

用不可变快照加 volatile 引用，是高并发读配置的常用方式。

## 不适合使用 volatile 的场景

### 1. 依赖当前值的更新

```java
balance = balance - amount;
count++;
max = Math.max(max, value);
```

这类读改写需要锁或 CAS。

### 2. 多变量不变量

```java
volatile int lower;
volatile int upper;

void setRange(int l, int u) {
    if (l > u) {
        throw new IllegalArgumentException();
    }
    lower = l;
    upper = u;
}
```

读线程可能看到新 `lower` 和旧 `upper` 的组合，破坏不变量。应该使用锁或不可变对象整体替换。

### 3. 对象内部可变状态

```java
private volatile List<String> users = new ArrayList<>();

void add(String user) {
    users.add(user); // ArrayList 本身仍不线程安全
}
```

volatile 只保护引用读写，不保护对象内部操作。

## volatile 和 synchronized 对比

| 维度 | volatile | synchronized |
| --- | --- | --- |
| 可见性 | 保证 | 保证 |
| 有序性 | 保证一定范围 | 保证临界区内外顺序语义 |
| 原子性 | 只保证单次读写 | 保证临界区互斥 |
| 阻塞 | 不阻塞 | 竞争时可能阻塞 |
| 使用范围 | 变量修饰符 | 方法或代码块 |
| 适合场景 | 状态标记、发布快照 | 复合操作、保护不变量 |

总结：volatile 是轻量同步机制，但不是锁。

## volatile 和 Atomic 类对比

`AtomicInteger` 内部的值通常使用 volatile 保证可见性，并通过 CAS 保证复合更新的原子性。

```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet(); // CAS 循环，线程安全
```

如果只是发布状态，用 volatile；如果要并发计数、累加、条件更新，用 atomic 或锁。

## 工程实践

- 用 volatile 保存“最新快照引用”，快照对象尽量不可变。
- 用 volatile 做停止标记时，循环体要能及时返回。
- 多字段一致性用不可变对象整体替换，不要多个 volatile 字段分散维护。
- 计数器优先用 `LongAdder` 或 `AtomicLong`，不要用 `volatile long` 加 `++`。
- 缓存刷新时，先构建完整新对象，再一次性赋给 volatile 字段。
- 不要用 volatile 包装非线程安全集合后继续并发修改集合内部。

## 常见追问

### 1. volatile 能保证线程安全吗？

只能在特定场景保证，例如单个状态标记、单引用发布。对复合操作、多变量约束、对象内部可变状态，volatile 不足以保证线程安全。

### 2. volatile 为什么能修复双重检查锁定？

它禁止对象引用赋值与构造初始化发生对其他线程可见的危险重排，并保证后续读取 `instance` 的线程能看到构造完成的对象状态。

### 3. volatile 变量读写性能一定比锁好吗？

不一定。volatile 不会阻塞，但会带来内存屏障和缓存一致性成本；高频写 volatile 变量可能造成缓存行竞争。锁在低竞争下也可能很快。

### 4. volatile 数组元素有 volatile 语义吗？

`volatile int[] arr` 只表示数组引用是 volatile，不表示 `arr[0]` 是 volatile。需要使用 `AtomicIntegerArray` 或 VarHandle 等工具。

### 5. volatile 可以替代数据库锁或分布式锁吗？

不能。volatile 只在同一个 JVM 进程内生效，不跨进程、不跨机器。

## 线上风险

- 用 `volatile int` 做 QPS 统计，结果丢失。
- volatile 引用指向 `HashMap`，刷新线程原地修改，读线程并发遍历触发异常或脏读。
- 停止标记是 volatile，但线程阻塞在队列、I/O 或远程调用，无法及时退出。
- 多个 volatile 字段表达一个配置版本，读线程看到混合版本。
- 高频 volatile 写造成缓存行抖动，吞吐下降。

## 易错点

- 说 volatile 保证线程安全但不限定场景。
- 认为 volatile 等于直接读写主内存。
- 认为 volatile 能保证 `++` 原子。
- 忘记 volatile 只保护引用本身，不保护对象内部。
- 用多个 volatile 字段维护一个不变量。
- volatile 数组引用和数组元素语义混淆。
- 忽略 volatile 不跨 JVM。

## 自检清单

- 能否说清 volatile 的两大核心语义。
- 能否解释 `volatile int count; count++` 为什么不安全。
- 能否写出 volatile 停止标记示例，并说明阻塞场景限制。
- 能否解释 volatile 在双重检查锁定中的作用。
- 能否区分 volatile 引用和引用对象内部状态。
- 能否说明 volatile 与 synchronized、Atomic 类的边界。
- 能否给出不可变快照加 volatile 引用的工程用法。
