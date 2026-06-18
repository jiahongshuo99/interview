# 滑动窗口

## 题型定位

滑动窗口是双指针的特化，专门处理连续子数组或连续子串。窗口通常表示 `[left, right]` 或 `[left, right)` 这一段连续区间，算法在移动右边界扩张窗口的同时，按条件移动左边界收缩窗口。

适合特征：

- 题目要求连续子数组、连续子串。
- 需要最长、最短、个数、是否存在。
- 窗口内状态可以增量维护。
- 条件具有一定单调性，例如窗口变大后和增加、不同字符数增加、频次变化可追踪。

不适合：

- 子序列问题，因为不要求连续。
- 任意组合问题，因为没有连续窗口边界。
- 条件不具备可维护性，移动边界无法快速更新状态。

## 核心模型

### 1. 固定长度窗口

窗口长度固定为 `k`，每次加入右端元素，移出左端元素。

```java
int sum = 0;
for (int i = 0; i < nums.length; i++) {
    sum += nums[i];
    if (i >= k) {
        sum -= nums[i - k];
    }
    if (i >= k - 1) {
        // 当前窗口是 [i - k + 1, i]
    }
}
```

### 2. 可变窗口求最长

右指针不断扩张；一旦窗口非法，左指针收缩直到重新合法；每次合法后更新最大长度。

```java
int left = 0;
int ans = 0;
for (int right = 0; right < n; right++) {
    add(right);
    while (!valid()) {
        remove(left);
        left++;
    }
    ans = Math.max(ans, right - left + 1);
}
```

### 3. 可变窗口求最短

右指针扩张直到窗口满足条件；只要满足，就尝试移动左指针缩短，并更新答案。

```java
int left = 0;
int ans = Integer.MAX_VALUE;
for (int right = 0; right < n; right++) {
    add(right);
    while (valid()) {
        ans = Math.min(ans, right - left + 1);
        remove(left);
        left++;
    }
}
```

### 4. 计数类窗口

字符串窗口常用 `int[]` 或 `Map` 维护频次，再维护一个额外变量表示满足了多少种字符。

## 常见模板

### 最长无重复子串

```java
int lengthOfLongestSubstring(String s) {
    int[] cnt = new int[128];
    int left = 0;
    int ans = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        cnt[c]++;

        while (cnt[c] > 1) {
            cnt[s.charAt(left)]--;
            left++;
        }
        ans = Math.max(ans, right - left + 1);
    }
    return ans;
}
```

### 长度最小的子数组

```java
int minSubArrayLen(int target, int[] nums) {
    int left = 0;
    int sum = 0;
    int ans = Integer.MAX_VALUE;

    for (int right = 0; right < nums.length; right++) {
        sum += nums[right];
        while (sum >= target) {
            ans = Math.min(ans, right - left + 1);
            sum -= nums[left++];
        }
    }
    return ans == Integer.MAX_VALUE ? 0 : ans;
}
```

这个模板依赖 `nums` 全为正数；如果有负数，窗口和不再单调，应该考虑前缀和、单调队列或哈希表。

### 最小覆盖子串

```java
String minWindow(String s, String t) {
    int[] need = new int[128];
    int required = 0;
    for (char c : t.toCharArray()) {
        if (need[c] == 0) {
            required++;
        }
        need[c]++;
    }

    int[] window = new int[128];
    int formed = 0;
    int left = 0;
    int bestLen = Integer.MAX_VALUE;
    int bestLeft = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        window[c]++;
        if (need[c] > 0 && window[c] == need[c]) {
            formed++;
        }

        while (formed == required) {
            if (right - left + 1 < bestLen) {
                bestLen = right - left + 1;
                bestLeft = left;
            }

            char d = s.charAt(left++);
            if (need[d] > 0 && window[d] == need[d]) {
                formed--;
            }
            window[d]--;
        }
    }
    return bestLen == Integer.MAX_VALUE ? "" : s.substring(bestLeft, bestLeft + bestLen);
}
```

## 复杂度

- 时间通常 O(n)，因为每个元素最多进窗口一次、出窗口一次。
- 空间 O(字符集大小) 或 O(不同元素数量)。
- 固定窗口空间可以是 O(1)，如果只维护和或最大值。
- 如果窗口内需要动态最大最小值，可能需要单调队列，时间仍可 O(n)。

## 边界条件

- `k == 0`、`k > n`。
- 空字符串、空数组。
- 目标串比源串长。
- 字符集大小是否固定。
- 输入是否包含负数，决定窗口和是否单调。
- 更新答案的位置：扩张后更新，还是收缩前更新。
- 窗口定义是闭区间还是半开区间，长度计算要统一。

## 典型题型

- 长度为 k 的子数组最大平均值。
- 无重复字符的最长子串。
- 最小覆盖子串。
- 字符串的排列。
- 找到字符串中所有字母异位词。
- 替换后的最长重复字符。
- 长度最小的子数组。
- 最大连续 1 的个数 III。
- 至多包含 k 个不同字符的最长子串。

## 面试讲解口径

滑动窗口的讲解要围绕窗口状态：

1. 我维护一个窗口 `[left, right]` 表示当前考虑的连续区间。
2. 右指针每次加入一个新元素，更新窗口状态。
3. 当窗口不满足条件时，移动左指针删除元素，直到窗口恢复合法。
4. 对最长问题，在窗口合法后更新答案。
5. 对最短问题，在窗口满足条件时更新答案，并尽量收缩。

如果题目含负数，要主动说明：

> 这道题不能直接用窗口和模板，因为负数会破坏“右扩和增加、左缩和减少”的单调性。

## 易错点

- 把子序列题误判为滑动窗口。
- 可变窗口中忘记在左指针移动时同步删除状态。
- 最短窗口更新答案的位置写错。
- 字符频次减到 0 后没有维护有效种类数。
- 数组含负数仍然使用正数窗口和模板。
- `right - left + 1` 和半开区间长度混用。
- 用 `Map.size()` 代表满足条件种类数，但没有删除 0 频次 key。

## 训练清单

- 能写固定长度窗口最大和或平均值。
- 能写最长无重复子串。
- 能写长度最小的子数组，并说明正数条件。
- 能写最小覆盖子串，讲清 `need`、`window`、`formed`。
- 能判断题目是否具有滑动窗口单调性。
- 能把窗口题和前缀和题区分开。
