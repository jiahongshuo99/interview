# CAS、ABA 问题、Atomic 类

## 面试定位

CAS 是 Java 原子类、AQS、并发容器和很多无锁算法的基础。面试常问：

- CAS 是什么，如何保证原子性。
- CAS 和锁有什么区别。
- ABA 问题是什么，怎么解决。
- AtomicInteger 为什么线程安全。
- AtomicStampedReference、AtomicMarkableReference 用在什么场景。
- LongAdder 为什么高并发下比 AtomicLong 更适合计数。

核心思路：CAS 是一种乐观并发控制，通过“比较期望值，匹配才更新”实现无锁原子更新。

## CAS 基本概念

CAS 全称 Compare-And-Swap 或 Compare-And-Set，包含三个值：

- 内存位置 V。
- 期望旧值 A。
- 新值 B。

语义：

```text
如果 V 当前值等于 A，则把 V 更新为 B；
否则不更新。
整个比较和更新是一个原子操作。
```

Java 中常见形式：

```java
AtomicInteger count = new AtomicInteger(0);
count.compareAndSet(0, 1);
```

## CAS 循环

CAS 更新失败时，通常需要重试。

```java
class CasCounter {
    private final AtomicInteger count = new AtomicInteger();

    int incrementAndGet() {
        while (true) {
            int current = count.get();
            int next = current + 1;
            if (count.compareAndSet(current, next)) {
                return next;
            }
        }
    }
}
```

`AtomicInteger.incrementAndGet()` 内部就是类似 CAS 循环。

## CAS 底层机制

CAS 依赖硬件原子指令支持。JVM 会通过 Unsafe、VarHandle 或内部 intrinsic 调用底层原子操作。

在 Java 层：

- 原子变量内部值通常是 volatile，保证可见性。
- CAS 保证比较和更新的原子性。
- CAS 失败说明期间有其他线程修改了值。

不要把 CAS 理解成纯 Java 代码里的 `if (value == expected) value = update`，那不是原子的。

## CAS 优点

- 不需要阻塞线程，避免线程挂起和唤醒成本。
- 低竞争下性能很好。
- 避免部分锁竞争带来的上下文切换。
- 是很多高性能并发组件的基础。

## CAS 缺点

### 1. 自旋开销

高竞争下大量线程反复 CAS 失败，会消耗 CPU。

```java
while (!atomic.compareAndSet(expect, update)) {
    // 高竞争时可能长时间自旋
}
```

### 2. 只能直接保护单个变量

单个 CAS 适合一个内存位置的更新。多个变量之间的不变量需要更复杂设计，例如封装为不可变对象后用 `AtomicReference` 整体替换。

### 3. ABA 问题

CAS 只比较当前值是否等于期望值，无法知道中间是否被改过又改回来。

## ABA 问题

### 问题示例

线程 T1 读取值 A，准备 CAS A -> C。

期间线程 T2 做了：

```text
A -> B -> A
```

T1 再执行 CAS 时发现当前仍是 A，于是更新成功。但 T1 不知道值曾被修改过。

### 为什么 ABA 有风险

如果只关心数值最终是否相同，ABA 可能无害。但如果值代表对象引用、链表节点、资源状态，ABA 可能导致严重问题。

典型无锁栈场景：

```text
栈顶 A -> B -> C
T1 读取 top=A, next=B
T2 pop A, pop B, push A
当前 top 又是 A，但 A.next 可能已变化
T1 CAS top 从 A 改为 B 成功，可能把已经弹出的 B 放回栈顶
```

## ABA 解决方式

### 1. 版本号

每次修改都增加版本号，CAS 同时比较值和版本。

Java 提供 `AtomicStampedReference`：

```java
AtomicStampedReference<String> ref =
        new AtomicStampedReference<>("A", 0);

int[] stampHolder = new int[1];
String current = ref.get(stampHolder);
int stamp = stampHolder[0];

boolean success = ref.compareAndSet(
        current,
        "B",
        stamp,
        stamp + 1
);
```

即使值从 A 到 B 再回 A，版本号也会变化。

### 2. 标记位

如果只需要标记是否被逻辑删除，可以用 `AtomicMarkableReference`。

```java
AtomicMarkableReference<Node> ref =
        new AtomicMarkableReference<>(node, false);

boolean[] markHolder = new boolean[1];
Node current = ref.get(markHolder);
boolean marked = markHolder[0];

ref.compareAndSet(current, current, marked, true);
```

### 3. 不复用节点或引用

在一些无锁数据结构中，通过避免对象复用、使用 GC 管理生命周期，可以降低 ABA 风险。但不能笼统认为有 GC 就没有 ABA，引用重新入栈仍可能出现逻辑 ABA。

## Atomic 类体系

### 基本类型

- `AtomicInteger`
- `AtomicLong`
- `AtomicBoolean`

常用方法：

```java
AtomicInteger value = new AtomicInteger(0);

value.get();
value.set(1);
value.compareAndSet(1, 2);
value.incrementAndGet();
value.getAndIncrement();
value.addAndGet(10);
value.updateAndGet(x -> x * 2);
```

### 引用类型

- `AtomicReference`
- `AtomicStampedReference`
- `AtomicMarkableReference`

`AtomicReference` 适合不可变快照整体替换：

```java
record Config(int timeoutMs, Set<String> allowList) {
}

class ConfigService {
    private final AtomicReference<Config> ref =
            new AtomicReference<>(new Config(1000, Set.of()));

    Config current() {
        return ref.get();
    }

    void update(Config next) {
        ref.set(next);
    }
}
```

### 数组类型

- `AtomicIntegerArray`
- `AtomicLongArray`
- `AtomicReferenceArray`

注意：`volatile int[]` 不能让数组元素具备 volatile/CAS 语义。

### 字段更新器

- `AtomicIntegerFieldUpdater`
- `AtomicLongFieldUpdater`
- `AtomicReferenceFieldUpdater`

适合在大量对象上节省 Atomic 包装对象开销，但要求字段：

- `volatile`
- 非 `static`
- 非 `final`
- 可访问性满足 updater 创建位置要求

示例：

```java
class Task {
    volatile int state;

    private static final AtomicIntegerFieldUpdater<Task> STATE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(Task.class, "state");

    boolean markRunning() {
        return STATE_UPDATER.compareAndSet(this, 0, 1);
    }
}
```

### 累加器

- `LongAdder`
- `LongAccumulator`
- `DoubleAdder`
- `DoubleAccumulator`

`LongAdder` 适合高并发统计计数。

```java
LongAdder qps = new LongAdder();
qps.increment();
long current = qps.sum();
```

## AtomicLong 和 LongAdder

### AtomicLong

所有线程竞争同一个 value，CAS 失败会重试。低竞争下很好，高竞争下热点严重。

### LongAdder

把计数分散到多个 Cell 中，不同线程更新不同 Cell，最后 `sum()` 汇总。

优点：

- 高并发写入吞吐更好。

缺点：

- `sum()` 不是强一致快照。
- 占用更多内存。
- 不适合需要精确即时值参与判断的场景。

适合：

- QPS 统计。
- 命中次数。
- 监控指标。

不适合：

- 库存扣减。
- 金额余额。
- 需要基于当前值做强一致决策。

## CAS 和锁对比

| 维度 | CAS | 锁 |
| --- | --- | --- |
| 思路 | 乐观并发 | 悲观互斥 |
| 阻塞 | 通常不阻塞 | 竞争时可能阻塞 |
| 低竞争性能 | 通常较好 | 现代锁也较好 |
| 高竞争性能 | 可能大量自旋 | 可能阻塞排队 |
| 多变量一致性 | 实现复杂 | 更自然 |
| 编程难度 | 较高 | 较低 |
| ABA 风险 | 有 | 通常无 |

工程上不要迷信无锁。复杂不变量用锁更清晰、更安全。

## 代码示例：不可变对象整体 CAS

```java
record Range(int lower, int upper) {
    Range {
        if (lower > upper) {
            throw new IllegalArgumentException();
        }
    }
}

class RangeHolder {
    private final AtomicReference<Range> ref =
            new AtomicReference<>(new Range(0, 100));

    boolean updateUpper(int newUpper) {
        while (true) {
            Range current = ref.get();
            Range next = new Range(current.lower(), newUpper);
            if (ref.compareAndSet(current, next)) {
                return true;
            }
        }
    }
}
```

把多个字段放进不可变对象，使用 `AtomicReference` 整体替换，可以维护多字段一致性。

## 工程实践

- 简单计数：低竞争用 `AtomicLong`，高竞争指标用 `LongAdder`。
- 余额、库存、额度这类强一致更新，不要只靠本地 Atomic，通常需要数据库事务、分布式锁或服务端原子操作。
- CAS 循环中不要放慢操作或有副作用的逻辑，因为可能多次重试。
- `updateAndGet` 传入函数必须无副作用。
- 多变量一致性优先考虑锁或不可变快照整体替换。
- 对 ABA 敏感的数据结构使用版本号或标记位。
- 高竞争 CAS 失败率高时，要考虑退避、分段或换锁。

## 常见追问

### 1. CAS 为什么是原子的？

因为比较和更新由底层硬件原子指令支持，JVM 将 Java 原子操作映射到对应平台能力。

### 2. CAS 一定比 synchronized 快吗？

不一定。低竞争下 CAS 通常快；高竞争下大量自旋会浪费 CPU，锁阻塞排队可能更稳定。

### 3. AtomicInteger 的 value 为什么还要 volatile？

volatile 保证可见性，CAS 保证更新原子性。二者结合才能让其他线程及时看到原子更新结果。

### 4. LongAdder 能替代 AtomicLong 吗？

不能完全替代。LongAdder 适合统计类高并发累加，不适合需要精确即时值和条件更新的场景。

### 5. ABA 一定会导致 bug 吗？

不一定。如果业务只关心当前值是否等于期望值，ABA 可能无害。但在节点引用、资源生命周期、状态版本敏感场景中必须处理。

## 线上风险

- 高并发 CAS 自旋导致 CPU 飙高。
- CAS 循环中调用远程服务或写日志，失败重试造成副作用重复。
- 用 LongAdder 做库存判断，读到非强一致值导致超卖。
- 忽略 ABA，导致无锁队列或栈结构损坏。
- AtomicReference 指向可变对象，内部仍被并发修改。
- `updateAndGet` lambda 有副作用，被多次执行造成重复处理。

## 易错点

- 把 CAS 写成普通 if 比较赋值。
- 认为 Atomic 类所有组合操作都天然满足业务一致性。
- 认为有 GC 就完全没有 ABA。
- 用 `volatile` 数组代替 Atomic 数组。
- LongAdder 的 `sum()` 当成线性一致读。
- CAS 失败后不重试也不处理失败路径。
- 在 CAS 循环里使用非幂等逻辑。

## 自检清单

- 能否解释 CAS 三个操作数和原子语义。
- 能否手写 CAS 循环实现自增。
- 能否说明 CAS 的优缺点。
- 能否讲清 ABA 问题及版本号解决方案。
- 能否区分 AtomicLong 和 LongAdder 适用场景。
- 能否解释 AtomicReference 如何维护不可变快照。
- 能否说明为什么 volatile 数组元素不具备 volatile 语义。
- 能否指出 CAS 循环中副作用的风险。
