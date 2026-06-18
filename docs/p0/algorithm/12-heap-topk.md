# 堆、TopK

## 题型定位

堆是一种能快速取出最大值或最小值的数据结构。Java 中 `PriorityQueue` 默认是小根堆。TopK 问题、动态中位数、多路归并、任务调度都经常使用堆。

常见场景：

- 找第 k 大或第 k 小。
- 找前 k 个高频元素。
- 合并 k 个有序链表或数组。
- 数据流中位数。
- 滑动窗口中动态极值。
- 按优先级处理任务。

## 核心模型

### 1. 小根堆维护前 k 大

堆中始终保存当前最大的 k 个元素，堆顶是这 k 个元素里最小的。

```java
PriorityQueue<Integer> heap = new PriorityQueue<>();
for (int x : nums) {
    heap.offer(x);
    if (heap.size() > k) {
        heap.poll();
    }
}
return heap.peek();
```

适合数据流或不想修改原数组的场景，时间 O(n log k)。

### 2. 大根堆维护前 k 小

Java 中用反向比较器构造大根堆。

```java
PriorityQueue<Integer> heap = new PriorityQueue<>((a, b) -> Integer.compare(b, a));
```

注意不要写 `b - a`，可能溢出。

### 3. 频次 TopK

先用哈希表统计频次，再用堆按频次筛选。

```java
Map<Integer, Integer> freq = new HashMap<>();
for (int x : nums) {
    freq.put(x, freq.getOrDefault(x, 0) + 1);
}
```

### 4. 多路归并

堆中保存每一路当前最小元素，弹出后把同一路的下一个元素放入堆。

适合：

- 合并 k 个有序链表。
- 合并 k 个有序数组。
- 找有序矩阵第 k 小元素。

## 常见模板

### 数组中第 k 个最大元素

```java
int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> heap = new PriorityQueue<>();

    for (int x : nums) {
        heap.offer(x);
        if (heap.size() > k) {
            heap.poll();
        }
    }
    return heap.peek();
}
```

### 前 k 个高频元素

```java
int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int x : nums) {
        freq.put(x, freq.getOrDefault(x, 0) + 1);
    }

    PriorityQueue<Integer> heap = new PriorityQueue<>(
        (a, b) -> Integer.compare(freq.get(a), freq.get(b))
    );

    for (int x : freq.keySet()) {
        heap.offer(x);
        if (heap.size() > k) {
            heap.poll();
        }
    }

    int[] ans = new int[k];
    for (int i = k - 1; i >= 0; i--) {
        ans[i] = heap.poll();
    }
    return ans;
}
```

### 合并 k 个有序链表

```java
ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> heap = new PriorityQueue<>(
        (a, b) -> Integer.compare(a.val, b.val)
    );

    for (ListNode node : lists) {
        if (node != null) {
            heap.offer(node);
        }
    }

    ListNode dummy = new ListNode(0);
    ListNode tail = dummy;
    while (!heap.isEmpty()) {
        ListNode node = heap.poll();
        tail.next = node;
        tail = tail.next;
        if (node.next != null) {
            heap.offer(node.next);
        }
    }
    return dummy.next;
}
```

### 数据流中位数

```java
class MedianFinder {
    private final PriorityQueue<Integer> small = new PriorityQueue<>((a, b) -> Integer.compare(b, a));
    private final PriorityQueue<Integer> large = new PriorityQueue<>();

    void addNum(int num) {
        if (small.isEmpty() || num <= small.peek()) {
            small.offer(num);
        } else {
            large.offer(num);
        }

        if (small.size() > large.size() + 1) {
            large.offer(small.poll());
        } else if (large.size() > small.size()) {
            small.offer(large.poll());
        }
    }

    double findMedian() {
        if (small.size() > large.size()) {
            return small.peek();
        }
        return ((long) small.peek() + large.peek()) / 2.0;
    }
}
```

## 复杂度

- 堆插入、删除堆顶：O(log k) 或 O(log n)。
- 查看堆顶：O(1)。
- 小根堆维护前 k 大：时间 O(n log k)，空间 O(k)。
- 排序做 TopK：时间 O(n log n)，空间取决于排序。
- 快选做第 k 大：平均 O(n)，空间 O(1)，但不适合动态数据流。
- 合并 k 个链表：总节点数 N，时间 O(N log k)，空间 O(k)。

## 边界条件

- `k == 0` 或 `k > n`，按题目约束处理。
- 堆为空时不能 `peek`。
- 比较器不要用减法。
- 频次相同时题目是否要求固定顺序。
- 返回第 k 大还是前 k 大。
- TopK 输出是否需要排序。
- 对象入堆后，如果参与比较的字段被修改，会破坏堆性质。

## 典型题型

- 数组中的第 k 个最大元素。
- 前 k 个高频元素。
- 数据流中的第 k 大元素。
- 合并 k 个升序链表。
- 查找和最小的 k 对数字。
- 数据流的中位数。
- 最接近原点的 k 个点。
- 有序矩阵中第 k 小的元素。
- 任务调度器。

## 面试讲解口径

TopK 题先说明方案取舍：

> 如果只做一次查询，可以排序，简单但 O(n log n)。如果只要第 k 大，可以用快选，平均 O(n)。如果数据流持续进入，或者要维护前 k 个元素，用大小为 k 的堆更合适，时间 O(n log k)，空间 O(k)。

堆维护前 k 大：

> 我用小根堆保存当前最大的 k 个元素。每次插入新元素，如果堆大小超过 k，就弹出最小的。遍历结束后，堆顶就是第 k 大。

## 易错点

- Java `PriorityQueue` 默认小根堆，误以为是大根堆。
- 比较器写 `b - a` 导致溢出。
- 维护前 k 大时用了大根堆，弹出了最大的元素。
- TopK 高频元素按元素值排序，而不是按频次排序。
- `findMedian` 中两个 `int` 相加溢出。
- 合并链表时没有把弹出节点的下一个节点入堆。
- 题目要求输出有序，但堆弹出的顺序和最终要求不一致。

## 训练清单

- 能写第 k 大的小根堆方案。
- 能写 TopK 高频元素。
- 能写合并 k 个有序链表。
- 能说明堆、排序、快选三种 TopK 方案差异。
- 能写数据流中位数的双堆方案。
- 能处理比较器和溢出问题。
