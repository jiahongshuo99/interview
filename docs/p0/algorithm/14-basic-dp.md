# 动态规划基础

## 题型定位

动态规划用于解决有重叠子问题和最优子结构的问题。面试中的基础 DP 重点不是记公式，而是能定义状态、写出转移、确定初始化和遍历顺序。

适合特征：

- 求最值、方案数、可行性。
- 当前选择会影响后续。
- 暴力递归存在重复计算。
- 问题可以拆成规模更小的同类子问题。

常见类型：

- 一维线性 DP。
- 二维路径 DP。
- 背包 DP。
- 子序列 DP。
- 区间 DP 的入门题。

## 核心模型

### 1. DP 五步

1. 定义状态：`dp[i]` 或 `dp[i][j]` 表示什么。
2. 写转移：状态如何由更小状态推来。
3. 初始化：最小子问题答案。
4. 遍历顺序：保证依赖状态已经计算。
5. 返回答案：返回哪个状态，是否需要取最大值。

### 2. 自顶向下记忆化

适合先写递归，再加缓存。

```java
int dfs(int i) {
    if (i <= 1) {
        return i;
    }
    if (memo[i] != -1) {
        return memo[i];
    }
    memo[i] = dfs(i - 1) + dfs(i - 2);
    return memo[i];
}
```

### 3. 自底向上递推

适合面试最终代码，空间和性能更稳定。

```java
int fib(int n) {
    if (n <= 1) {
        return n;
    }
    int a = 0;
    int b = 1;
    for (int i = 2; i <= n; i++) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

### 4. 状态压缩

如果当前状态只依赖上一行或前几个状态，可以把二维压成一维，或把数组压成常量变量。

## 常见模板

### 爬楼梯

```java
int climbStairs(int n) {
    if (n <= 2) {
        return n;
    }

    int a = 1;
    int b = 2;
    for (int i = 3; i <= n; i++) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

### 打家劫舍

```java
int rob(int[] nums) {
    int prev2 = 0;
    int prev1 = 0;

    for (int x : nums) {
        int cur = Math.max(prev1, prev2 + x);
        prev2 = prev1;
        prev1 = cur;
    }
    return prev1;
}
```

`prev1` 表示处理到前一个房子的最大收益，`prev2` 表示处理到前两个房子的最大收益。

### 不同路径

```java
int uniquePaths(int m, int n) {
    int[] dp = new int[n];
    Arrays.fill(dp, 1);

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[j] += dp[j - 1];
        }
    }
    return dp[n - 1];
}
```

### 0-1 背包

```java
int knapsack(int[] weights, int[] values, int capacity) {
    int[] dp = new int[capacity + 1];

    for (int i = 0; i < weights.length; i++) {
        for (int c = capacity; c >= weights[i]; c--) {
            dp[c] = Math.max(dp[c], dp[c - weights[i]] + values[i]);
        }
    }
    return dp[capacity];
}
```

0-1 背包容量倒序遍历，防止同一物品被重复使用。

### 最长递增子序列 O(n^2)

```java
int lengthOfLIS(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1);
    int ans = 1;

    for (int i = 0; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
        ans = Math.max(ans, dp[i]);
    }
    return ans;
}
```

## 复杂度

- 一维线性 DP：通常 O(n)，空间 O(n) 或 O(1)。
- 二维 DP：通常 O(mn)，空间 O(mn)，可压缩到 O(n)。
- 背包 DP：O(n * capacity)。
- 子序列 DP：常见 O(n^2)，部分可用二分优化到 O(n log n)。
- 记忆化递归：时间等于状态数乘每个状态转移成本，空间包含缓存和递归栈。

## 边界条件

- `n == 0`、空数组。
- 初始化是否覆盖第一行、第一列。
- 返回 `dp[n]` 还是 `dp[n - 1]`。
- 求最大值时初始值不能错误设为 0，特别是允许负数时。
- 方案数可能溢出 `int`，题目可能要求取模。
- 背包容量遍历方向：0-1 背包倒序，完全背包正序。
- 子序列和子串不同，子串要求连续。

## 典型题型

- 爬楼梯。
- 打家劫舍。
- 最大子数组和。
- 不同路径。
- 最小路径和。
- 零钱兑换。
- 分割等和子集。
- 最长递增子序列。
- 最长公共子序列。
- 编辑距离。
- 单词拆分。

## 面试讲解口径

DP 题必须先讲状态定义：

> 我定义 `dp[i]` 表示处理到前 `i` 个元素时的最优结果。这样当前状态只依赖更小规模的状态，可以按 i 从小到大递推。

背包题：

> `dp[c]` 表示容量不超过 c 时的最大价值。每个物品只能选一次，所以容量要倒序遍历，避免当前物品在同一轮被重复使用。

子序列题：

> `dp[i]` 表示以 `nums[i]` 结尾的最长递增子序列长度。由于必须以 i 结尾，所以只从前面比它小的元素转移。

## 易错点

- 没有定义清楚状态，直接套公式。
- 遍历顺序不满足依赖关系。
- 0-1 背包容量正序遍历，导致同一物品重复使用。
- 二维 DP 压缩后一维遍历方向错误。
- 子序列题把答案固定为 `dp[n - 1]`，但真实答案可能在任意位置。
- 初始化漏掉空串、空数组状态。
- 记忆化递归用 0 表示未计算，但 0 可能是合法答案。

## 训练清单

- 每道 DP 题先写出状态定义和转移方程。
- 能写爬楼梯、打家劫舍、不同路径。
- 能写 0-1 背包，并说明倒序原因。
- 能写最长递增子序列 O(n^2)。
- 能区分子数组、子串、子序列。
- 能尝试把二维 DP 压缩为一维，并说明遍历方向。
