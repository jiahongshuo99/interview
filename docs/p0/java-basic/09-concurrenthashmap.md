# ConcurrentHashMap 原理

## 面试定位

`ConcurrentHashMap` 是 Java 并发集合面试核心题。面试官通常会问它与 `HashMap`、`Hashtable` 的区别，JDK 7 和 JDK 8 实现差异，为什么不允许 null，put/get 是否加锁，扩容如何协作，size 为什么不精确，`computeIfAbsent` 有什么坑。

回答要抓住主线：

- 它是线程安全的高并发 Map。
- JDK 8 主要使用 CAS + synchronized + volatile 保证并发安全。
- 读操作多数无锁，写操作只锁局部桶，扩容可多线程协助。
- 它不允许 null key/value，以避免并发语义歧义。

## 与其他 Map 的区别

| 类型 | 线程安全 | null key/value | 典型特点 |
| --- | --- | --- | --- |
| `HashMap` | 否 | 允许 null key 和 null value | 单线程高效，非并发安全 |
| `Hashtable` | 是 | 不允许 null | 方法级 synchronized，并发性能差，遗留类 |
| `Collections.synchronizedMap` | 是 | 取决于被包装 Map | 外层同步包装，粒度粗 |
| `ConcurrentHashMap` | 是 | 不允许 null | 分段或桶级并发控制，高并发性能好 |

## JDK 7 与 JDK 8 差异

### JDK 7

JDK 7 的 `ConcurrentHashMap` 主要结构是 Segment 分段锁：

```text
ConcurrentHashMap
└── Segment[]
    ├── Segment 0 -> HashEntry[]
    ├── Segment 1 -> HashEntry[]
    └── Segment n -> HashEntry[]
```

每个 Segment 类似一个小 HashMap，并继承 `ReentrantLock`。不同 Segment 可以并发写，降低锁竞争。并发级别 `concurrencyLevel` 影响 Segment 数量。

### JDK 8

JDK 8 取消 Segment 数组作为主要并发控制结构，改为：

```text
Node[] table
  ├── null
  ├── Node 链表
  ├── TreeBin 红黑树
  └── ForwardingNode 扩容迁移标记
```

主要手段：

- `volatile` 保证 table 和节点 value、next 的可见性。
- CAS 初始化 table、插入空桶、更新计数。
- `synchronized` 锁定桶首节点处理冲突写入。
- 链表过长树化为红黑树。
- 多线程协助扩容。

面试中若没有特别说明，通常默认讨论 JDK 8 及之后实现。

## 核心字段与节点

JDK 8 核心概念：

- `table`：主桶数组，volatile。
- `nextTable`：扩容时的新数组。
- `sizeCtl`：控制初始化和扩容的重要状态字段。
- `baseCount`：基础计数。
- `counterCells`：高并发下分散计数，类似 LongAdder 思路。
- `Node`：普通链表节点。
- `TreeNode` / `TreeBin`：红黑树相关节点和容器。
- `ForwardingNode`：扩容时表示该桶已迁移，引导线程去新表或协助扩容。

`Node` 中 key 和 hash 通常是 final，value 和 next 使用 volatile，保证读线程能看到更新。

## put 流程

JDK 8 `put` 简化流程：

1. 检查 key 和 value，任何一个为 null 都抛 `NullPointerException`。
2. 计算 hash。
3. 如果 table 未初始化，CAS 初始化。
4. 根据 hash 定位桶。
5. 如果桶为空，CAS 插入新 Node，成功则结束。
6. 如果桶是 `ForwardingNode`，说明正在扩容，当前线程协助迁移。
7. 如果桶不为空，使用 `synchronized` 锁住桶首节点：
   - 链表中查找相同 key，找到则更新 value。
   - 没找到则尾插新节点。
   - 如果是树结构，则按树逻辑插入或更新。
8. 插入后更新计数。
9. 判断是否需要树化或扩容。

关键点：

- 空桶插入用 CAS，不加锁。
- 冲突桶写入只锁当前桶首节点，不锁整张表。
- 读操作通常不加锁，依赖 volatile 可见性。

## get 流程

`get` 基本无锁：

1. 计算 hash。
2. 定位桶。
3. 如果首节点匹配，直接返回。
4. 如果遇到特殊节点，例如树或 ForwardingNode，走对应查找逻辑。
5. 链表则遍历查找。

为什么读可以不加锁？

- table 引用是 volatile。
- 节点 value、next 是 volatile。
- key 和 hash 是 final。
- 写入时通过 CAS、volatile 写和 synchronized 建立可见性。

这不意味着读取过程看到的是“全局瞬时一致快照”，而是保证不会读到破坏内存安全的数据结构，且能按并发集合语义工作。

## sizeCtl

`sizeCtl` 是 JDK 8 `ConcurrentHashMap` 的关键控制字段，语义随状态变化：

- `0`：table 未初始化，使用默认容量。
- 正数：未初始化时表示初始容量；已初始化时表示下一次扩容阈值。
- `-1`：正在初始化。
- 小于 `-1`：正在扩容，包含扩容标识和参与扩容线程数信息。

面试不必背完整位运算细节，但要知道它用于协调初始化和扩容，避免多个线程重复初始化或无序扩容。

## 扩容机制

触发扩容一般是元素数量超过阈值。扩容时：

1. 创建 `nextTable`，容量通常翻倍。
2. 多个线程可以领取不同桶区间协助迁移。
3. 桶迁移完成后，旧桶位置放置 `ForwardingNode`。
4. 后续访问旧桶的线程看到 `ForwardingNode`，会去新表查找或参与迁移。
5. 所有桶迁移完成后，`table` 指向 `nextTable`，清理扩容状态。

这种设计避免扩容全部由一个线程承担，降低大 Map 扩容时的长时间阻塞。

## 哈希冲突与树化

JDK 8 `ConcurrentHashMap` 与 `HashMap` 类似，也有链表转红黑树机制。

常见阈值：

- 链表长度达到 8 时尝试树化。
- 数组容量小于 64 时优先扩容。
- 树节点减少到一定程度时退化为链表。

不同点在于并发场景下红黑树被包装在 `TreeBin` 中，并通过内部同步控制树操作。

## 计数机制

并发环境下维护准确 size 成本较高。JDK 8 `ConcurrentHashMap` 使用类似 `LongAdder` 的分散计数：

- 竞争低时更新 `baseCount`。
- 竞争高时更新不同 `CounterCell`。
- 求 size 时汇总 `baseCount` 和所有 `CounterCell`。

因此 `size()` 在并发修改时只是一个瞬时估计，不适合用作强一致控制条件。

```java
if (map.size() == 0) {
    // 并发场景下这不是可靠的“之后仍为空”判断
}
```

需要判断是否为空时可用 `isEmpty()`，但并发修改下同样只能表示调用时附近的状态，不是长期保证。

## 为什么不允许 null

`ConcurrentHashMap` 不允许 null key 和 null value。核心原因是避免并发语义歧义。

对于 `HashMap`：

```java
map.get(key) == null
```

可能表示：

- key 不存在。
- key 存在，但 value 是 null。

单线程下可以再 `containsKey` 判断。但并发环境中，`get` 和 `containsKey` 之间 Map 可能被其他线程修改，无法可靠区分。因此 `ConcurrentHashMap` 禁止 null，`get` 返回 null 就明确表示当前没有这个 key 对应的 value。

## 原子复合操作

`ConcurrentHashMap` 提供了一些原子复合方法：

- `putIfAbsent`
- `remove(key, value)`
- `replace`
- `compute`
- `computeIfAbsent`
- `computeIfPresent`
- `merge`

这些方法避免 “先 get 再 put” 的竞态。

错误写法：

```java
List<Order> list = map.get(userId);
if (list == null) {
    list = new ArrayList<>();
    map.put(userId, list);
}
list.add(order); // ArrayList 仍然非线程安全
```

更好：

```java
ConcurrentMap<Long, Queue<Order>> map = new ConcurrentHashMap<>();
map.computeIfAbsent(userId, key -> new ConcurrentLinkedQueue<>()).add(order);
```

注意：`computeIfAbsent` 能保证映射插入的原子性，但返回的 value 对象本身是否线程安全要自己保证。

## computeIfAbsent 注意点

`computeIfAbsent` 的 mapping function 应该：

- 快速完成。
- 不做长时间阻塞。
- 不对同一个 map 做复杂递归更新。
- 不依赖被调用次数精确为一次。
- 不抛出非预期异常。

不同 JDK 版本和竞争情况下，函数调用、锁粒度和异常处理细节可能影响性能。工程中不要把远程调用、数据库查询等重逻辑直接塞进高竞争 key 的 `computeIfAbsent` 中。

## 迭代器一致性

`ConcurrentHashMap` 的迭代器是弱一致的，不是 fail-fast。

弱一致含义：

- 迭代时可以并发修改 Map。
- 不会抛出 `ConcurrentModificationException`。
- 迭代可能看到部分修改，也可能看不到部分修改。
- 不保证遍历的是某一时刻的全量一致快照。

如果需要快照，可以复制：

```java
Map<K, V> snapshot = new HashMap<>(concurrentMap);
```

但复制期间仍可能与并发写交错，只能得到复制过程中的一个近似快照。

## 常见追问

### ConcurrentHashMap 是怎么保证线程安全的？

JDK 8 中主要通过 volatile 保证可见性，CAS 处理无锁初始化、空桶插入和计数更新，冲突写入时用 synchronized 锁住桶首节点，扩容时使用 ForwardingNode 和 sizeCtl 协调多线程迁移。

### get 要加锁吗？

通常不加锁。`get` 依赖 volatile 和节点不可变字段保证可见性与安全遍历。遇到树或扩容节点时走特殊逻辑，但不按普通写操作那样锁整张表。

### put 会锁整张表吗？

不会。空桶插入用 CAS；桶内已有节点时锁当前桶首节点。不同桶之间可以并发写。

### JDK 7 和 JDK 8 最大区别？

JDK 7 使用 Segment 分段锁；JDK 8 取消 Segment 主结构，使用 Node 数组，结合 CAS、synchronized、volatile 和红黑树实现更细粒度并发。

### ConcurrentHashMap 为什么比 Hashtable 并发性能好？

`Hashtable` 多数方法直接 synchronized，锁住整个对象；`ConcurrentHashMap` 锁粒度更细，读多为无锁，不同桶写可并发，扩容也可协助完成。

### size 准确吗？

没有并发修改时可以认为准确；并发修改中只能表示近似瞬时结果，不适合作为强一致条件。需要精确容量控制时应使用额外同步、信号量、限流器或数据库约束等机制。

### ConcurrentHashMap 能代替所有加锁吗？

不能。它只能保证 Map 自身结构和单个方法或特定复合方法的线程安全。如果 value 是非线程安全对象，或者业务操作跨多个 key、多个数据结构，仍需要额外同步或事务。

## 代码示例

### 计数器

高并发计数建议 value 使用 `LongAdder`。

```java
ConcurrentHashMap<String, LongAdder> counters = new ConcurrentHashMap<>();

public void incr(String key) {
    counters.computeIfAbsent(key, k -> new LongAdder()).increment();
}

public long get(String key) {
    LongAdder adder = counters.get(key);
    return adder == null ? 0L : adder.sum();
}
```

### 本地缓存

```java
private final ConcurrentHashMap<Long, User> cache = new ConcurrentHashMap<>();

public User getUser(long userId) {
    return cache.computeIfAbsent(userId, this::loadUser);
}

private User loadUser(long userId) {
    // 真实项目要考虑异常、空值缓存、过期、击穿和限流
    return userRepository.findById(userId)
            .orElseThrow(() -> new BizException("USER_NOT_FOUND", "用户不存在"));
}
```

注意：这个例子只是展示 API。生产缓存还需要过期、最大容量、刷新、穿透保护等能力，通常优先使用 Caffeine 或成熟缓存框架。

### 跨 key 操作需要额外控制

```java
// 只用 ConcurrentHashMap 不能保证 from/to 两个账户扣加的整体原子性
public void transfer(long from, long to, long amount) {
    balances.compute(from, (key, value) -> value - amount);
    balances.compute(to, (key, value) -> value + amount);
}
```

这类跨 key 业务需要更高层锁、数据库事务或按账户顺序加锁，不能只依赖 ConcurrentHashMap。

## 工程实践

- 并发读写 Map 默认优先考虑 `ConcurrentHashMap`。
- 不允许 null key/value，空值场景用 Optional、哨兵对象或单独记录不存在。
- 使用 `putIfAbsent`、`computeIfAbsent`、`merge` 等原子复合方法，避免 get-then-put 竞态。
- value 如果是集合、对象或计数器，要确认 value 自身线程安全。
- `size()` 不适合做并发精确控制条件。
- 高并发计数用 `LongAdder`，不要频繁 `get` 后装箱加一再 `put`。
- `computeIfAbsent` 中避免慢调用和复杂递归更新。
- 本地缓存需求复杂时，不要自己用 ConcurrentHashMap 重新造缓存系统，优先使用 Caffeine。
- 跨多个 key 的一致性需要额外同步或事务。

## 易错点

- 认为 ConcurrentHashMap 的 value 对象也自动线程安全。
- 用 `get` 后判断 null 再 `put`，产生竞态。
- 在 `computeIfAbsent` 中执行耗时远程调用，造成桶锁竞争或请求堆积。
- 用 `size()` 做强一致限流。
- 认为 ConcurrentHashMap 迭代器会 fail-fast。
- 忘记它不允许 null key/value。
- 把 JDK 7 Segment 分段锁答案直接套到 JDK 8。
- 认为有 ConcurrentHashMap 就不需要业务锁或事务。

## 自检清单

- 能比较 HashMap、Hashtable、ConcurrentHashMap。
- 能说明 JDK 7 Segment 和 JDK 8 CAS + synchronized 的差异。
- 能描述 JDK 8 put/get 大致流程。
- 能解释为什么 get 多数无锁。
- 能解释 ForwardingNode 和多线程协助扩容的作用。
- 能说明为什么不允许 null。
- 能解释 size 的弱一致特性。
- 能正确使用 putIfAbsent、computeIfAbsent、merge。
- 能识别 value 非线程安全和跨 key 操作的风险。
