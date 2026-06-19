# Java 集合：ArrayList、LinkedList、HashMap、LinkedHashMap、TreeMap、HashSet

## 面试定位

Java 集合是后端面试高频 P0 内容。它不仅考 API 使用，还会追到底层数据结构、复杂度、扩容、排序、去重、线程安全和工程选型。面试中常见问题包括：`ArrayList` 和 `LinkedList` 怎么选，`HashMap` 和 `TreeMap` 区别，`LinkedHashMap` 有什么用，`HashSet` 底层是什么，集合遍历时修改为什么会异常。

这篇总览关注“怎么选”和“基本原理”。`HashMap` 和 `ConcurrentHashMap` 的深层机制单独看后续文档。

## 集合框架概览

常见接口：

```text
Iterable
└── Collection
    ├── List
    │   ├── ArrayList
    │   └── LinkedList
    ├── Set
    │   ├── HashSet
    │   ├── LinkedHashSet
    │   └── TreeSet
    └── Queue / Deque

Map
├── HashMap
├── LinkedHashMap
├── TreeMap
├── Hashtable
└── ConcurrentHashMap
```

注意 `Map` 不继承 `Collection`，它是键值对结构的独立接口。

## ArrayList

### 核心概念

`ArrayList` 底层是动态数组，支持按下标随机访问。

特点：

- 查询快：按下标访问时间复杂度 `O(1)`。
- 尾部追加通常快：均摊 `O(1)`。
- 中间插入、删除较慢：需要移动元素，`O(n)`。
- 允许 `null`。
- 非线程安全。

### 扩容机制

`ArrayList` 内部维护对象数组。元素数量超过容量时会扩容，JDK 8 中常见扩容为原容量的 1.5 倍左右，然后复制旧数组到新数组。

```java
List<String> list = new ArrayList<>(1000);
```

如果能预估数量，指定初始容量可以减少扩容和数组复制。

### 适用场景

- 读多写少。
- 需要按下标访问。
- 主要在尾部追加。
- 数据量可预估。

## LinkedList

### 核心概念

`LinkedList` 底层是双向链表，同时实现了 `List` 和 `Deque`。

特点：

- 按下标访问慢：需要从头或尾遍历，`O(n)`。
- 已定位节点后的插入删除快：改指针即可。
- 每个节点额外保存前后指针，内存开销大。
- CPU 缓存局部性差，实际性能经常不如直觉。
- 可作为队列、双端队列使用。

### 适用场景

`LinkedList` 并不适合大量随机访问，也不一定适合普通插入删除。只有在明确需要双端队列能力，或已经持有节点位置的链式结构中才有优势。Java 标准 `LinkedList` 没有暴露节点引用，所以“中间插入 O(1)”在很多业务场景并不成立，因为定位节点仍要 `O(n)`。

工程中如果需要队列，通常优先考虑 `ArrayDeque`；并发队列用 `ConcurrentLinkedQueue`、`LinkedBlockingQueue` 等。

## HashMap

### 核心概念

`HashMap` 是基于哈希表的 key-value 容器。

特点：

- 平均 `put/get/remove` 为 `O(1)`。
- 不保证遍历顺序。
- key 和 value 都允许 `null`，但 null key 只有一个。
- 非线程安全。
- JDK 8 后桶内链表过长会转为红黑树。

### 适用场景

- 按 key 快速查找。
- 对顺序没有要求。
- 单线程或外部已同步场景。

```java
Map<Long, User> userMap = new HashMap<>();
userMap.put(user.getId(), user);
User user = userMap.get(userId);
```

## LinkedHashMap

### 核心概念

`LinkedHashMap` 继承自 `HashMap`，在哈希表基础上维护一条双向链表，用来记录元素顺序。

顺序有两种：

- 插入顺序：默认模式。
- 访问顺序：构造时传入 `accessOrder = true`。

```java
Map<String, Integer> map = new LinkedHashMap<>();
map.put("b", 2);
map.put("a", 1);
map.put("c", 3);

System.out.println(map.keySet()); // [b, a, c]
```

### LRU 缓存

`LinkedHashMap` 可通过访问顺序和 `removeEldestEntry` 实现简单 LRU。

```java
public class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;

    public LruCache(int maxSize) {
        super(16, 0.75f, true);
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }
}
```

注意：这个简单实现本身不是线程安全的。生产级本地缓存通常使用 Caffeine。

### 适用场景

- 需要稳定遍历顺序。
- 需要按访问顺序淘汰。
- JSON 输出或测试断言中需要可预测顺序。

## TreeMap

### 核心概念

`TreeMap` 基于红黑树实现，按 key 的自然顺序或自定义比较器排序。

特点：

- `put/get/remove` 时间复杂度 `O(log n)`。
- key 必须可比较，或者构造时提供 `Comparator`。
- 不允许无法比较的 key。
- 默认不允许 null key，因为 null 无法自然比较；如果自定义 Comparator 支持 null，可处理 null。
- 适合范围查询、排序遍历。

```java
NavigableMap<Integer, String> map = new TreeMap<>();
map.put(3, "c");
map.put(1, "a");
map.put(2, "b");

System.out.println(map.keySet()); // [1, 2, 3]
System.out.println(map.subMap(1, true, 3, false)); // 1 到 3 左闭右开
```

### NavigableMap 能力

`TreeMap` 实现 `NavigableMap`，支持：

- `firstKey()` / `lastKey()`
- `floorKey(key)`：小于等于 key 的最大 key。
- `ceilingKey(key)`：大于等于 key 的最小 key。
- `lowerKey(key)`：严格小于 key 的最大 key。
- `higherKey(key)`：严格大于 key 的最小 key。
- `subMap` / `headMap` / `tailMap` 范围视图。

### 适用场景

- 需要有序 key。
- 需要范围查询。
- 需要找前驱、后继。
- 排行、时间窗口、区间映射等。

## HashSet

### 核心概念

`HashSet` 底层基于 `HashMap`，元素作为 HashMap 的 key，value 使用一个固定哨兵对象。

```java
Set<String> set = new HashSet<>();
set.add("a");
set.add("a");
System.out.println(set.size()); // 1
```

特点：

- 不保证顺序。
- 允许一个 `null`。
- 平均添加、删除、查找为 `O(1)`。
- 去重依赖元素的 `equals` 和 `hashCode`。
- 非线程安全。

### 适用场景

- 去重。
- 快速判断是否存在。
- 交集、并集、差集等集合运算。

```java
Set<Long> existingIds = new HashSet<>(queryExistingIds());
for (Long id : requestIds) {
    if (!existingIds.contains(id)) {
        // new id
    }
}
```

## 常见集合对比

### ArrayList vs LinkedList

| 维度 | ArrayList | LinkedList |
| --- | --- | --- |
| 底层结构 | 动态数组 | 双向链表 |
| 随机访问 | 快，`O(1)` | 慢，`O(n)` |
| 尾部追加 | 均摊 `O(1)` | `O(1)` |
| 中间插入删除 | 需要移动元素 | 定位慢，改指针快 |
| 内存开销 | 较低 | 较高，每个节点有指针 |
| 缓存友好 | 好 | 差 |
| 常用程度 | 高 | 较低 |

多数业务中，`ArrayList` 是默认选择。

### HashMap vs LinkedHashMap vs TreeMap

| 维度 | HashMap | LinkedHashMap | TreeMap |
| --- | --- | --- | --- |
| 底层 | 哈希表 | 哈希表 + 双向链表 | 红黑树 |
| 顺序 | 不保证 | 插入顺序或访问顺序 | key 排序 |
| get/put | 平均 `O(1)` | 平均 `O(1)` | `O(log n)` |
| null key | 允许一个 | 允许一个 | 默认不允许 |
| 典型场景 | 快速查找 | 稳定顺序、LRU | 排序、范围查询 |

### HashSet vs TreeSet vs LinkedHashSet

| 维度 | HashSet | LinkedHashSet | TreeSet |
| --- | --- | --- | --- |
| 底层 | HashMap | LinkedHashMap | TreeMap |
| 顺序 | 不保证 | 插入顺序 | 排序 |
| 查找 | 平均 `O(1)` | 平均 `O(1)` | `O(log n)` |
| 去重依据 | equals/hashCode | equals/hashCode | compareTo/Comparator |

## Fail-Fast

很多集合的迭代器是 fail-fast 的。迭代过程中如果通过集合自身方法结构性修改集合，迭代器会尽力抛出 `ConcurrentModificationException`。

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String value : list) {
    if ("b".equals(value)) {
        list.remove(value); // 可能 ConcurrentModificationException
    }
}
```

正确方式：

```java
Iterator<String> iterator = list.iterator();
while (iterator.hasNext()) {
    String value = iterator.next();
    if ("b".equals(value)) {
        iterator.remove();
    }
}
```

或使用：

```java
list.removeIf("b"::equals);
```

fail-fast 只是错误检测机制，不是线程安全保证。不能依赖它做并发控制。

## 线程安全

常见集合大多不是线程安全的：

- `ArrayList` 非线程安全。
- `LinkedList` 非线程安全。
- `HashMap` 非线程安全。
- `LinkedHashMap` 非线程安全。
- `TreeMap` 非线程安全。
- `HashSet` 非线程安全。

线程安全替代：

- 并发 Map：`ConcurrentHashMap`。
- 写少读多 List：`CopyOnWriteArrayList`。
- 阻塞队列：`ArrayBlockingQueue`、`LinkedBlockingQueue`。
- 外部同步：`Collections.synchronizedList`、`Collections.synchronizedMap`。

注意：同步包装集合在迭代时仍需要手动同步。

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());
synchronized (list) {
    for (String value : list) {
        // iterate safely
    }
}
```

## 常见追问

### ArrayList 初始容量为什么重要？

因为扩容需要创建新数组并复制旧元素。大量元素写入前如果能预估容量，指定初始容量可以减少扩容次数和内存复制成本。

### LinkedList 插入删除一定比 ArrayList 快吗？

不一定。链表在已定位节点时插入删除是 `O(1)`，但 Java `LinkedList` 按下标定位节点仍需要 `O(n)`。同时链表节点内存分散、缓存不友好，实际性能经常不如 `ArrayList`。

### HashSet 如何去重？

`HashSet` 底层用 `HashMap`，元素作为 key。先根据 `hashCode` 定位桶，再用 `equals` 判断是否已有相同元素。

### TreeMap 和 HashMap 怎么选？

只需要按 key 快速查找，用 `HashMap`；需要按 key 排序、范围查询、找前驱后继，用 `TreeMap`。

### LinkedHashMap 为什么能保持顺序？

它在 HashMap 的节点基础上维护双向链表。遍历时按链表顺序遍历，而不是按哈希桶顺序。

## 代码示例

### 去重并保留原顺序

```java
List<Long> ids = List.of(3L, 1L, 3L, 2L);
Set<Long> unique = new LinkedHashSet<>(ids);
System.out.println(unique); // [3, 1, 2]
```

### 分组计数

```java
Map<String, Integer> countMap = new HashMap<>();
for (String type : types) {
    countMap.merge(type, 1, Integer::sum);
}
```

### TreeMap 范围查询

```java
TreeMap<Long, String> events = new TreeMap<>();
events.put(1000L, "start");
events.put(2000L, "pay");
events.put(3000L, "finish");

SortedMap<Long, String> window = events.subMap(1000L, 3000L);
```

## 工程实践

- 默认 List 选 `ArrayList`，除非明确需要双端队列语义。
- 默认 Map 选 `HashMap`，需要稳定顺序选 `LinkedHashMap`，需要排序和范围查询选 `TreeMap`。
- 默认 Set 选 `HashSet`，需要保留插入顺序选 `LinkedHashSet`，需要排序选 `TreeSet`。
- 能预估容量时，为 `ArrayList`、`HashMap` 指定合理初始容量。
- 不要在 foreach 中直接调用集合自身 remove，使用 iterator、removeIf 或收集后批量删除。
- 集合中放自定义对象作为 key 或 set 元素时，必须正确实现 `equals` 和 `hashCode`。
- 对外返回集合时考虑不可变视图或拷贝，避免内部状态被外部修改。
- 并发场景不要使用普通集合硬扛，选择并发集合或外部同步策略。

## 易错点

- 认为 `LinkedList` 在所有插入删除场景都优于 `ArrayList`。
- 认为 `HashMap` 遍历顺序稳定。
- 用 `HashSet` 存自定义对象但没重写 `equals` 和 `hashCode`。
- 在 foreach 中删除元素导致 `ConcurrentModificationException`。
- 把 fail-fast 当成线程安全机制。
- `TreeSet` 去重依赖比较器，不是 `equals`。
- 使用 `Arrays.asList` 返回的固定大小列表后调用 add/remove。
- `List.of`、`Set.of` 返回不可变集合，不能修改，也不允许 null 元素。

## 自检清单

- 能画出 List、Set、Map 的常见实现关系。
- 能比较 ArrayList 和 LinkedList。
- 能比较 HashMap、LinkedHashMap、TreeMap。
- 能解释 HashSet 底层和去重依据。
- 能说明 fail-fast 的触发原因和正确删除方式。
- 能根据业务场景选择合适集合。
- 能说明常见集合的线程安全问题。
