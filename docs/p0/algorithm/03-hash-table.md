# 哈希表

## 题型定位

哈希表用于把“查找某个值是否出现过”从 O(n) 降到均摊 O(1)。面试里它经常作为底层工具，而不是最终算法本身。只要题目出现去重、计数、配对、映射、缓存、前缀状态查找，就应该考虑哈希表。

常见形式：

- `Set`：判断是否存在、去重。
- `Map<K, V>`：记录频次、下标、最后出现位置、对象映射。
- `Map<State, Count>`：统计前缀状态出现次数。
- `LinkedHashMap`：保留插入顺序，常见于 LRU。
- `PriorityQueue + Map`：堆中延迟删除或频次统计。

## 核心模型

### 1. 查补数

两数之和是哈希表的基础模型：遍历当前值 `x`，查找 `target - x` 是否出现过。

```java
int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> index = new HashMap<>();

    for (int i = 0; i < nums.length; i++) {
        int need = target - nums[i];
        if (index.containsKey(need)) {
            return new int[]{index.get(need), i};
        }
        index.put(nums[i], i);
    }
    return new int[0];
}
```

关键是先查再放，避免同一个元素被使用两次。

### 2. 频次统计

适合异位词、TopK 高频元素、字符替换、投票前置统计。

```java
Map<Integer, Integer> freq = new HashMap<>();
for (int x : nums) {
    freq.put(x, freq.getOrDefault(x, 0) + 1);
}
```

### 3. 首次出现位置

当题目要求最长长度、最短距离、是否重复出现时，哈希表可以记录状态第一次出现的位置。

典型例子：

- 最长和为 k 的子数组。
- 连续数组中 0 和 1 数量相同。
- 字符串中最长无重复子串的右边界维护。

### 4. 状态压缩键

如果 key 是数组、列表或复合状态，不能直接依赖对象地址。需要转成稳定 key：

- 排序后的字符串：异位词分组。
- 计数数组转字符串：小写字母异位词。
- 坐标编码：`r + "," + c` 或 `r * cols + c`。
- 自定义类时实现 `equals` 和 `hashCode`。

## 常见模板

### 异位词分组

```java
List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> groups = new HashMap<>();

    for (String s : strs) {
        char[] arr = s.toCharArray();
        Arrays.sort(arr);
        String key = new String(arr);
        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
    }
    return new ArrayList<>(groups.values());
}
```

### 最长连续序列

```java
int longestConsecutive(int[] nums) {
    Set<Integer> set = new HashSet<>();
    for (int x : nums) {
        set.add(x);
    }

    int best = 0;
    for (int x : set) {
        if (!set.contains(x - 1)) {
            int cur = x;
            int len = 1;
            while (set.contains(cur + 1)) {
                cur++;
                len++;
            }
            best = Math.max(best, len);
        }
    }
    return best;
}
```

### 前缀和计数

```java
int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> count = new HashMap<>();
    count.put(0, 1);

    int sum = 0;
    int ans = 0;
    for (int x : nums) {
        sum += x;
        ans += count.getOrDefault(sum - k, 0);
        count.put(sum, count.getOrDefault(sum, 0) + 1);
    }
    return ans;
}
```

## 复杂度

- 插入、查询、删除：均摊 O(1)，最坏可能退化。
- 频次统计：时间 O(n)，空间 O(k)，`k` 为不同元素数量。
- 排序生成 key：每个字符串 O(m log m)，总复杂度取决于字符串长度。
- 哈希表空间通常是主要成本，需要关注是否可以用数组优化。

## 边界条件

- 重复元素很多或全部相同。
- 目标值可能需要同一个数两次，但不能使用同一个下标两次。
- key 是否可能溢出，例如坐标编码 `r * cols + c`。
- `Map.get` 返回 `null` 时不能直接拆箱成 `int`。
- 频次减到 0 后是否需要删除 key，取决于窗口有效性判断。
- 遍历 `Map` 时不要同时结构性修改它，除非使用迭代器。

## 典型题型

- 两数之和。
- 有效的字母异位词。
- 字母异位词分组。
- 存在重复元素。
- 最长连续序列。
- 和为 k 的子数组。
- LRU 缓存。
- 设计哈希集合、哈希映射。
- 找到字符串中所有字母异位词。
- 前 k 个高频元素。

## 面试讲解口径

哈希表题建议明确三件事：

1. key 是什么：元素值、字符、前缀和、状态编码还是对象。
2. value 是什么：次数、下标、首次位置、列表还是布尔存在性。
3. 遍历时先查询还是先更新：这决定是否允许当前元素参与匹配。

示例口径：

> 我用哈希表记录已经遍历过的数字到下标的映射。遍历到 `nums[i]` 时，检查 `target - nums[i]` 是否已经出现。如果出现，两个下标就是答案。查询放在插入之前，可以避免同一个下标被使用两次。

## 易错点

- 两数之和先放再查，误用同一个元素。
- 对数组或 `List` 直接作为 key，但内容变化后哈希值变化。
- 频次减到 0 后不删除，导致 `map.size()` 判断窗口种类数错误。
- 忽略哈希表空间复杂度。
- `containsKey` 和 `getOrDefault` 语义混用，无法区分不存在和存在但值为 0。
- 自定义 key 没有正确实现 `equals` 和 `hashCode`。

## 训练清单

- 能写出两数之和和异位词分组。
- 能说清楚 `HashMap`、`HashSet`、`LinkedHashMap` 的差异。
- 能用哈希表统计频次并结合堆解决 TopK。
- 能用前缀和加哈希表统计子数组个数。
- 能设计稳定的复合 key。
- 能解释为什么某些题用数组计数比哈希表更合适。
