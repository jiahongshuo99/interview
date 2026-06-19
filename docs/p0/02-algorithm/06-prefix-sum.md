# 前缀和

## 题型定位

前缀和用于快速计算连续区间的和、计数或状态差。它把每次 O(n) 的区间求和变成 O(1)，也常与哈希表结合，统计满足条件的子数组数量。

适合特征：

- 题目关注连续子数组或矩阵子区域。
- 多次查询区间和。
- 要找和为 k、和能被 k 整除、0 和 1 数量相等。
- 数组可能包含负数，滑动窗口不再适用。

## 核心模型

### 1. 一维前缀和

定义 `pre[i]` 表示前 `i` 个元素之和，即 `nums[0..i-1]`。

```java
int n = nums.length;
int[] pre = new int[n + 1];
for (int i = 0; i < n; i++) {
    pre[i + 1] = pre[i] + nums[i];
}

// nums[l..r] 的和
int sum = pre[r + 1] - pre[l];
```

使用长度 `n + 1` 的前缀数组可以避免单独处理从 0 开始的区间。

### 2. 前缀和加哈希表

如果当前前缀和是 `sum`，希望某个历史前缀和 `old` 满足：

```text
sum - old = k
```

那么只要查 `old = sum - k` 出现过多少次。

```java
Map<Integer, Integer> count = new HashMap<>();
count.put(0, 1);

int sum = 0;
int ans = 0;
for (int x : nums) {
    sum += x;
    ans += count.getOrDefault(sum - k, 0);
    count.put(sum, count.getOrDefault(sum, 0) + 1);
}
```

### 3. 余数前缀

子数组和能被 `k` 整除，等价于两个前缀和对 `k` 取模相同。

```java
int mod = ((sum % k) + k) % k;
```

Java 中负数取模仍可能为负，需要归一化。

### 4. 二维前缀和

`pre[i][j]` 表示矩阵左上角到 `(i - 1, j - 1)` 的区域和。

```java
int[][] pre = new int[m + 1][n + 1];
for (int i = 0; i < m; i++) {
    for (int j = 0; j < n; j++) {
        pre[i + 1][j + 1] = pre[i][j + 1] + pre[i + 1][j] - pre[i][j] + matrix[i][j];
    }
}

int regionSum(int r1, int c1, int r2, int c2) {
    return pre[r2 + 1][c2 + 1] - pre[r1][c2 + 1] - pre[r2 + 1][c1] + pre[r1][c1];
}
```

## 常见模板

### 区域和检索

```java
class NumArray {
    private final int[] pre;

    NumArray(int[] nums) {
        pre = new int[nums.length + 1];
        for (int i = 0; i < nums.length; i++) {
            pre[i + 1] = pre[i] + nums[i];
        }
    }

    int sumRange(int left, int right) {
        return pre[right + 1] - pre[left];
    }
}
```

### 和为 k 的子数组个数

```java
int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    freq.put(0, 1);

    int sum = 0;
    int ans = 0;
    for (int x : nums) {
        sum += x;
        ans += freq.getOrDefault(sum - k, 0);
        freq.put(sum, freq.getOrDefault(sum, 0) + 1);
    }
    return ans;
}
```

### 连续数组中 0 和 1 数量相同

把 0 当成 -1，问题转成最长和为 0 的子数组。

```java
int findMaxLength(int[] nums) {
    Map<Integer, Integer> first = new HashMap<>();
    first.put(0, -1);

    int sum = 0;
    int ans = 0;
    for (int i = 0; i < nums.length; i++) {
        sum += nums[i] == 0 ? -1 : 1;
        if (first.containsKey(sum)) {
            ans = Math.max(ans, i - first.get(sum));
        } else {
            first.put(sum, i);
        }
    }
    return ans;
}
```

## 复杂度

- 构建一维前缀和：时间 O(n)，空间 O(n)。
- 区间和查询：时间 O(1)。
- 前缀和加哈希表：时间 O(n)，空间 O(n)。
- 二维前缀和：构建 O(mn)，查询 O(1)，空间 O(mn)。

## 边界条件

- 前缀数组长度是否为 `n + 1`。
- 查询区间是闭区间还是半开区间。
- 数组元素和可能溢出 `int`，必要时使用 `long`。
- `k == 0` 时不能做取模。
- Java 负数取模要归一化。
- 哈希表初始必须放入前缀 0，否则漏掉从下标 0 开始的子数组。
- 统计个数时记录频次，求最长长度时记录首次出现位置。

## 典型题型

- 区域和检索。
- 二维区域和检索。
- 和为 k 的子数组。
- 连续数组。
- 和可被 k 整除的子数组。
- 乘积小于 k 的子数组不能直接用前缀和，通常用滑动窗口。
- 矩阵区域和。
- 最大子矩阵和不超过 k。

## 面试讲解口径

前缀和题要先定义清楚 `pre[i]` 的含义：

> 我定义 `pre[i]` 为前 `i` 个元素的和。这样区间 `[l, r]` 的和就是 `pre[r + 1] - pre[l]`，可以统一处理从 0 开始的区间。

前缀和加哈希表可以这样讲：

> 遍历到当前位置时，当前前缀和为 `sum`。如果存在历史前缀和 `sum - k`，两者之间的子数组和就是 k。所以哈希表记录每个前缀和出现次数，每次把匹配次数累加到答案。

## 易错点

- 忘记初始化 `count.put(0, 1)`。
- 把 `pre[i]` 定义为包含 `nums[i]`，但查询公式按不包含写。
- 统计数量时只记录第一次出现，导致漏计。
- 求最长长度时记录频次，导致无法得到最远距离。
- 含负数时误用滑动窗口。
- 二维前缀和公式漏掉加回重叠区域。
- 取模题没有处理负数余数。

## 训练清单

- 能写一维前缀和并回答任意区间和。
- 能写二维前缀和查询矩形区域。
- 能用哈希表统计和为 k 的子数组。
- 能区分“统计个数”和“求最长长度”的哈希表 value。
- 能说明含负数时为什么滑动窗口可能失败。
- 能处理取模类前缀和题。
