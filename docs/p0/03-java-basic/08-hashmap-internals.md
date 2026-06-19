# HashMap 扩容、哈希冲突、红黑树化、线程安全问题

## 面试定位

`HashMap` 是 Java 后端面试最高频集合题。面试官通常会从 `put` 流程问起，继续追问数组、链表、红黑树、扩容、负载因子、哈希扰动、树化阈值、为什么线程不安全，以及 JDK 7 与 JDK 8 的差异。

回答时要避免只背零散数字，应围绕三个问题展开：

- 怎么定位：key 的 hash 如何映射到数组桶。
- 冲突怎么办：链表和红黑树如何存储同桶元素。
- 容量怎么变：什么时候扩容，扩容时元素如何迁移。

## 核心数据结构

JDK 8 `HashMap` 底层是：

```text
数组 table
  ├── bucket 0: Node / TreeNode
  ├── bucket 1: null
  ├── bucket 2: Node -> Node -> Node
  └── bucket n: TreeNode 红黑树
```

核心字段：

- `table`：桶数组，长度始终是 2 的幂。
- `size`：键值对数量。
- `threshold`：扩容阈值，通常为 `capacity * loadFactor`。
- `loadFactor`：负载因子，默认 `0.75`。
- `modCount`：结构性修改次数，用于 fail-fast。

节点包含：

- `hash`
- `key`
- `value`
- `next`

红黑树节点还包含父节点、左右子节点、颜色等信息。

## put 流程

简化流程：

1. 如果 table 未初始化，先初始化。
2. 计算 key 的 hash。
3. 根据 `(n - 1) & hash` 定位桶下标。
4. 如果桶为空，直接放新节点。
5. 如果桶不为空：
   - 如果首节点 key 相等，覆盖 value。
   - 如果桶是红黑树，按树节点逻辑插入或覆盖。
   - 如果桶是链表，遍历链表，找到相等 key 就覆盖，否则尾插新节点。
6. 插入后如果链表长度达到树化阈值，尝试树化。
7. size 超过 threshold 后扩容。

伪代码：

```java
V put(K key, V value) {
    int hash = hash(key);
    int index = (table.length - 1) & hash;
    Node<K, V> first = table[index];

    if (first == null) {
        table[index] = new Node<>(hash, key, value, null);
    } else {
        // 覆盖、链表插入或红黑树插入
    }

    if (++size > threshold) {
        resize();
    }
    return oldValue;
}
```

## get 流程

1. 计算 key 的 hash。
2. 根据桶数组长度定位下标。
3. 检查首节点。
4. 如果首节点不匹配：
   - 红黑树则按树查找。
   - 链表则顺序遍历。
5. 判断 key 相等时先比较 hash，再比较 `==` 或 `equals`。

```java
if (node.hash == hash &&
        (node.key == key || (key != null && key.equals(node.key)))) {
    return node.value;
}
```

## 哈希扰动与下标计算

HashMap 通过 key 的 `hashCode` 计算 hash。JDK 8 中会把高 16 位与低 16 位异或，让高位信息也参与低位寻址。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

下标计算：

```java
index = (n - 1) & hash;
```

这里要求数组长度 `n` 是 2 的幂。这样 `(n - 1) & hash` 等价于对 `n` 取模，但位运算更快。

为什么要扰动？如果数组长度较小，下标只依赖 hash 的低位。某些 key 的低位分布不好时会冲突严重。扰动把高位信息混入低位，降低冲突概率。

## 容量为什么是 2 的幂

好处：

- 下标计算可用 `(n - 1) & hash`，效率高。
- 扩容为 2 倍后，元素迁移可以根据旧容量对应的 bit 决定留在原位置还是移动到 `oldIndex + oldCap`。
- 分布更均匀，前提是 hash 扰动合理。

如果初始容量不是 2 的幂，HashMap 会调整到大于等于指定值的最小 2 的幂。

## 负载因子和扩容阈值

默认负载因子是 `0.75`，这是时间和空间的折中：

- 负载因子太低：空间浪费，扩容更频繁。
- 负载因子太高：冲突增加，查找性能下降。

阈值：

```text
threshold = capacity * loadFactor
```

默认构造的 HashMap 首次 put 时才初始化数组，默认容量通常是 16，阈值是 12。

## 扩容机制

触发条件：插入新节点后 `size > threshold`。

扩容通常把容量扩大为原来的 2 倍，并重新计算阈值。JDK 8 中迁移元素时，不需要重新完整计算每个 key 的下标。因为容量翻倍后，元素的新位置只可能是：

- 原下标 `oldIndex`
- 原下标 + 旧容量 `oldIndex + oldCap`

判断依据是：

```java
(hash & oldCap) == 0
```

如果为 0，留在原位置；否则移动到 `oldIndex + oldCap`。

示意：

```text
oldCap = 16
oldIndex = hash & 15
newCap = 32
newIndex = hash & 31

newIndex 只可能是 oldIndex 或 oldIndex + 16
```

这也是容量为 2 的幂带来的优化。

## 哈希冲突

不同 key 可能映射到同一个桶，称为哈希冲突。HashMap 使用链地址法解决冲突。

JDK 8 之前，桶内主要是链表。JDK 8 后，当链表过长且数组容量足够大时，会转为红黑树，以降低极端冲突下的查找成本。

冲突判断中的 key 相等逻辑：

- hash 值相等只是候选。
- 还要用 `==` 或 `equals` 判断 key 是否逻辑相等。

因此自定义 key 必须正确实现 `equals` 和 `hashCode`。

## 红黑树化

JDK 8 引入树化机制，主要阈值：

- `TREEIFY_THRESHOLD = 8`：链表长度达到 8 时尝试树化。
- `UNTREEIFY_THRESHOLD = 6`：树节点数量降到 6 附近时退化为链表。
- `MIN_TREEIFY_CAPACITY = 64`：数组容量至少 64 才树化；否则优先扩容。

为什么链表长度到 8 不一定立刻树化？因为如果数组太小，冲突可能主要来自容量不足，扩容后元素会分散，没必要立刻引入红黑树的复杂结构。

### 为什么是红黑树

红黑树是近似平衡二叉搜索树，能把极端冲突下的查找从链表 `O(n)` 降到 `O(log n)`。它不像 AVL 树那样追求严格平衡，插入删除时旋转调整成本更低，适合工程折中。

### 树化一定提升性能吗

不一定。红黑树节点更大，维护成本更高。只有冲突较严重时才值得使用。普通情况下链表短，链表更简单更快。

## JDK 7 与 JDK 8 差异

常被问到的差异：

- JDK 7：数组 + 链表。
- JDK 8：数组 + 链表 + 红黑树。
- JDK 7 扩容迁移链表时常见头插法，并发扩容可能形成环链。
- JDK 8 链表插入通常为尾插法，减少了并发扩容环链问题，但 HashMap 仍然不是线程安全的。

不要把“JDK 8 不会死循环”理解成“JDK 8 HashMap 线程安全”。并发 put 仍可能导致数据覆盖、丢失、size 不准、结构异常等问题。

## 线程安全问题

`HashMap` 不是线程安全的。并发读写会出现：

- 数据丢失：多个线程同时 put，覆盖彼此结果。
- size 不准确：`size++` 不是原子操作。
- 扩容时结构不一致。
- 读到中间状态。
- fail-fast 异常。

错误示例：

```java
Map<Long, User> map = new HashMap<>();

// 多线程同时执行
map.put(user.getId(), user);
```

并发场景应使用：

```java
ConcurrentHashMap<Long, User> map = new ConcurrentHashMap<>();
```

或者在外部加锁，或使用不可变 Map，取决于场景。

## null key 和 null value

`HashMap` 允许一个 null key，null key 的 hash 按 0 处理，通常放在 0 号桶。允许多个 null value。

```java
Map<String, String> map = new HashMap<>();
map.put(null, "x");
map.put("a", null);
```

注意 `ConcurrentHashMap` 不允许 null key 和 null value。

## 初始容量计算

如果预计要放 `expectedSize` 个元素，为避免扩容，可以设置初始容量为大于 `expectedSize / loadFactor` 的 2 的幂。

```java
int expectedSize = 1000;
int capacity = (int) (expectedSize / 0.75f) + 1;
Map<Long, User> map = new HashMap<>(capacity);
```

实际 HashMap 会把容量调整到 2 的幂。Guava、Hutool 等工具库也提供过类似容量计算工具。重点是别把 `new HashMap<>(1000)` 误认为“最多放 1000 个不扩容”，默认负载因子下阈值约为容量的 75%。

## 常见追问

### HashMap 的 put 过程是什么？

计算 hash，定位桶。如果桶为空直接插入；如果桶不为空，先判断首节点，再根据桶结构遍历链表或红黑树，key 相等则覆盖 value，不相等则插入新节点。插入后如果链表过长且容量足够会树化，如果 size 超过阈值会扩容。

### 为什么 HashMap 容量是 2 的幂？

为了用位运算快速定位桶，并在扩容时利用容量翻倍的性质快速判断元素新位置是原下标还是原下标加旧容量。

### 负载因子为什么默认 0.75？

这是空间利用率和哈希冲突概率之间的折中。太低浪费空间，太高冲突多、查找慢。

### 链表什么时候树化？

链表长度达到 8 且数组容量至少 64 时树化。若容量不足 64，会优先扩容。树节点数量减少到 6 附近时可能退化为链表。

### HashMap 为什么线程不安全？

内部没有同步控制。并发 put、resize、修改链表或树时可能发生数据覆盖、丢失、size 错误、读到中间状态等问题。JDK 8 改进了部分扩容行为，但不改变非线程安全结论。

### HashMap 和 Hashtable 区别？

`Hashtable` 方法基本使用 synchronized，线程安全但并发性能差，不允许 null key/value，属于早期遗留类。现代代码一般用 `HashMap` 或 `ConcurrentHashMap`。

## 代码示例

### 自定义 key

```java
public final class UserCacheKey {
    private final long tenantId;
    private final long userId;

    public UserCacheKey(long tenantId, long userId) {
        this.tenantId = tenantId;
        this.userId = userId;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof UserCacheKey)) {
            return false;
        }
        UserCacheKey that = (UserCacheKey) o;
        return tenantId == that.tenantId && userId == that.userId;
    }

    @Override
    public int hashCode() {
        return Objects.hash(tenantId, userId);
    }
}
```

### 统计计数

```java
Map<String, Integer> countMap = new HashMap<>();
for (String type : types) {
    countMap.merge(type, 1, Integer::sum);
}
```

### 按 key 分组

```java
Map<Long, List<Order>> ordersByUser = new HashMap<>();
for (Order order : orders) {
    ordersByUser
            .computeIfAbsent(order.getUserId(), key -> new ArrayList<>())
            .add(order);
}
```

注意：`computeIfAbsent` 在 `HashMap` 中不是并发安全的；并发场景用 `ConcurrentHashMap` 并理解其函数执行语义。

## 工程实践

- 单线程普通 key-value 查找默认使用 `HashMap`。
- 并发读写使用 `ConcurrentHashMap`，不要用 `HashMap` 加侥幸心理。
- 大批量导入数据时设置合理初始容量，减少扩容。
- 自定义 key 使用不可变字段，并正确实现 `equals` 和 `hashCode`。
- 不要依赖 `HashMap` 遍历顺序。如果需要顺序，用 `LinkedHashMap` 或 `TreeMap`。
- 避免把可变对象作为 key，尤其不要修改参与哈希计算的字段。
- 对外返回 Map 时考虑不可变拷贝，避免调用方修改内部状态。
- 关注恶意哈希冲突攻击场景，JDK 8 树化能缓解但不是所有问题的完整答案。

## 易错点

- 把初始容量当成最多可存元素数，忽略负载因子。
- 认为 hashCode 相等就代表 key 相等。
- 认为 JDK 8 HashMap 已经线程安全。
- 忘记链表树化需要容量至少 64。
- 依赖 HashMap 的遍历顺序。
- 自定义 key 只重写 equals 不重写 hashCode。
- 用可变对象做 key 后修改字段，导致 get 不到。
- 在多线程场景用 HashMap 做缓存。

## 自检清单

- 能说出 HashMap 的数组、链表、红黑树结构。
- 能完整描述 put/get 流程。
- 能解释哈希扰动和 `(n - 1) & hash`。
- 能说明为什么容量是 2 的幂。
- 能解释负载因子、阈值和扩容。
- 能说明树化阈值 8、退化阈值 6、最小树化容量 64。
- 能说明 HashMap 线程不安全的具体表现。
- 能根据元素规模设置初始容量。
