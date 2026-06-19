# 数组

## 题型定位

数组是面试算法的基础容器，核心特点是下标访问 O(1)、元素连续、长度通常固定。大多数数组题不是考“会不会遍历”，而是考能否把问题转成下标、区间、状态维护、原地覆盖或排序后的结构性处理。

常见考点包括：

- 线性扫描：一次遍历维护最大值、最小值、计数、当前位置状态。
- 原地修改：删除元素、去重、移动零、颜色分类。
- 区间处理：合并区间、插入区间、缺失区间、会议室。
- 下标映射：把值放到应该出现的位置，常用于找缺失正数、重复数。
- 排序后处理：先排序再双指针、贪心、合并。
- 矩阵变体：二维数组遍历、旋转、螺旋、岛屿类题目的前置基础。

## 核心模型

### 1. 扫描聚合

定义一个或多个变量，在遍历过程中不断更新答案。

适合：

- 最大子数组和。
- 买卖股票一次交易。
- 找峰值、找主元素。
- 统计满足条件的元素个数。

关键点：

- 明确变量含义，例如 `best` 是历史最优，`cur` 是以当前位置结尾的状态。
- 每轮循环结束后，变量必须仍然满足定义。
- 不要把“当前位置贡献”和“历史答案”混在一个变量里。

### 2. 原地覆盖

用 `write` 指针表示下一个应该写入的位置，用 `read` 指针扫描原数组。

适合：

- 删除指定元素。
- 有序数组去重。
- 移动零。
- 保留最多 k 个重复元素。

模板思想：

```java
int write = 0;
for (int read = 0; read < nums.length; read++) {
    if (shouldKeep(nums, read, write)) {
        nums[write++] = nums[read];
    }
}
return write;
```

### 3. 下标归位

当题目出现“长度为 n、元素范围 1..n、找缺失/重复”时，优先考虑把值 `x` 放到下标 `x - 1`。

```java
int i = 0;
while (i < nums.length) {
    int target = nums[i] - 1;
    if (nums[i] >= 1 && nums[i] <= nums.length && nums[i] != nums[target]) {
        swap(nums, i, target);
    } else {
        i++;
    }
}
```

这种模型通常可以做到 O(n) 时间、O(1) 额外空间。

### 4. 区间合并

区间题的核心是先排序，再维护当前合并区间。

```java
Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));
List<int[]> ans = new ArrayList<>();

for (int[] it : intervals) {
    if (ans.isEmpty() || ans.get(ans.size() - 1)[1] < it[0]) {
        ans.add(new int[]{it[0], it[1]});
    } else {
        ans.get(ans.size() - 1)[1] = Math.max(ans.get(ans.size() - 1)[1], it[1]);
    }
}
```

## 常见模板

### 有序数组去重

```java
int removeDuplicates(int[] nums) {
    if (nums.length == 0) {
        return 0;
    }

    int write = 1;
    for (int read = 1; read < nums.length; read++) {
        if (nums[read] != nums[write - 1]) {
            nums[write++] = nums[read];
        }
    }
    return write;
}
```

### 移动零

```java
void moveZeroes(int[] nums) {
    int write = 0;
    for (int x : nums) {
        if (x != 0) {
            nums[write++] = x;
        }
    }
    while (write < nums.length) {
        nums[write++] = 0;
    }
}
```

### 旋转数组

```java
void rotate(int[] nums, int k) {
    int n = nums.length;
    if (n == 0) {
        return;
    }

    k %= n;
    reverse(nums, 0, n - 1);
    reverse(nums, 0, k - 1);
    reverse(nums, k, n - 1);
}

void reverse(int[] nums, int l, int r) {
    while (l < r) {
        int t = nums[l];
        nums[l++] = nums[r];
        nums[r--] = t;
    }
}
```

### 最大子数组和

```java
int maxSubArray(int[] nums) {
    int cur = nums[0];
    int best = nums[0];

    for (int i = 1; i < nums.length; i++) {
        cur = Math.max(nums[i], cur + nums[i]);
        best = Math.max(best, cur);
    }
    return best;
}
```

## 复杂度

- 单纯扫描：时间 O(n)，空间 O(1)。
- 原地覆盖：时间 O(n)，空间 O(1)。
- 排序后处理：时间 O(n log n)，空间取决于排序实现和返回结果。
- 下标归位：每个元素最多被交换到正确位置，整体时间 O(n)，空间 O(1)。

## 边界条件

- 空数组、长度为 1 的数组。
- 全部元素相同、全部元素不同。
- 包含负数、零、最大最小整数。
- 题目是否允许修改原数组。
- 返回值是新长度、数组本身、下标还是具体元素。
- 区间边界是闭区间还是半开区间。
- 旋转数组时 `k` 可能大于 `n`，需要取模。

## 典型题型

- 删除排序数组中的重复项。
- 移动零。
- 合并两个有序数组。
- 合并区间。
- 轮转数组。
- 缺失的第一个正数。
- 多数元素。
- 最大子数组和。
- 买卖股票的最佳时机。
- 除自身以外数组的乘积。

## 面试讲解口径

讲数组题时，建议先说明变量定义，而不是直接贴代码。

可以按下面顺序表达：

1. 这道题的数据结构是数组，随机访问便宜，所以我会围绕下标维护状态。
2. 如果题目要求原地处理，我用 `write` 表示下一个写入位置，`read` 负责扫描。
3. 循环不变量是 `[0, write)` 始终保存已经处理过且应该保留的元素。
4. 每次遇到满足条件的元素就写入并推进 `write`。
5. 最终 `write` 就是新长度，前 `write` 个元素就是答案。

对于区间题：

1. 先按左端点排序。
2. 当前区间和结果最后一个区间比较。
3. 如果不重叠就追加，否则扩展右端点。
4. 排序保证只需要和最后一个合并区间比较。

## 易错点

- 原地覆盖时用 `nums[write]` 比较，却忘记 `write` 可能还没写入有效值。
- 删除元素后仍然访问旧长度范围内的内容。
- 区间合并用 `<` 还是 `<=` 没有结合题目边界定义。
- `Arrays.sort` 比较器写成 `a[0] - b[0]`，可能整数溢出。
- 最大子数组和把初始值设为 0，导致全负数返回错误。
- 旋转数组忘记处理空数组，`k % n` 触发除零。

## 训练清单

- 能独立写出有序数组去重、移动零、旋转数组。
- 能解释原地覆盖的循环不变量。
- 能用排序加一次扫描解决合并区间。
- 能区分“需要稳定保持相对顺序”和“不需要保持顺序”的做法。
- 能在全负数、空数组、重复元素场景下手推结果。
- 能把数组题和双指针、前缀和、二分、排序题型关联起来。
