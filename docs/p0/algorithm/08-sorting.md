# 排序

## 题型定位

排序是很多题目的预处理步骤。面试中不一定要求手写排序算法，但必须知道排序后能带来什么结构性：相同元素相邻、大小关系单调、区间按起点有序、可以使用双指针或贪心。

常见考点：

- 排序后去重。
- 排序后双指针。
- 排序后合并区间。
- 自定义比较器。
- TopK 与排序、堆、快选的取舍。
- 归并排序思想统计逆序关系。
- 快排分区思想。

## 核心模型

### 1. 排序后相邻比较

适合判断重复、最长连续前的简化、会议冲突。

```java
Arrays.sort(nums);
for (int i = 1; i < nums.length; i++) {
    if (nums[i] == nums[i - 1]) {
        return true;
    }
}
return false;
```

### 2. 排序后双指针

适合两数之和、三数之和、最接近三数之和。

排序后，左指针右移会增大和，右指针左移会减小和。

### 3. 自定义排序

Java 中对象数组或二维数组使用比较器。

```java
Arrays.sort(intervals, (a, b) -> {
    if (a[0] != b[0]) {
        return Integer.compare(a[0], b[0]);
    }
    return Integer.compare(a[1], b[1]);
});
```

不要写 `a[0] - b[0]`，可能溢出。

### 4. 分区思想

快排的核心是选 pivot，把数组分成小于、等于、大于三块。即使不手写完整快排，很多原地分类题会用这个思想。

## 常见模板

### 合并区间

```java
int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));
    List<int[]> ans = new ArrayList<>();

    for (int[] cur : intervals) {
        if (ans.isEmpty() || ans.get(ans.size() - 1)[1] < cur[0]) {
            ans.add(new int[]{cur[0], cur[1]});
        } else {
            int[] last = ans.get(ans.size() - 1);
            last[1] = Math.max(last[1], cur[1]);
        }
    }
    return ans.toArray(new int[ans.size()][]);
}
```

### 三数之和

```java
List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> ans = new ArrayList<>();

    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i - 1]) {
            continue;
        }

        int l = i + 1;
        int r = nums.length - 1;
        while (l < r) {
            int sum = nums[i] + nums[l] + nums[r];
            if (sum == 0) {
                ans.add(Arrays.asList(nums[i], nums[l], nums[r]));
                l++;
                r--;
                while (l < r && nums[l] == nums[l - 1]) {
                    l++;
                }
                while (l < r && nums[r] == nums[r + 1]) {
                    r--;
                }
            } else if (sum < 0) {
                l++;
            } else {
                r--;
            }
        }
    }
    return ans;
}
```

### 快选 TopK

```java
int findKthLargest(int[] nums, int k) {
    int target = nums.length - k;
    int left = 0;
    int right = nums.length - 1;

    while (left <= right) {
        int p = partition(nums, left, right);
        if (p == target) {
            return nums[p];
        } else if (p < target) {
            left = p + 1;
        } else {
            right = p - 1;
        }
    }
    return -1;
}

int partition(int[] nums, int left, int right) {
    int pivot = nums[right];
    int i = left;
    for (int j = left; j < right; j++) {
        if (nums[j] <= pivot) {
            swap(nums, i++, j);
        }
    }
    swap(nums, i, right);
    return i;
}
```

实际面试中快选最好随机 pivot，避免最坏 O(n^2)。

## 复杂度

- Java `Arrays.sort(int[])`：双轴快排，平均 O(n log n)，基本 O(log n) 栈空间。
- Java `Arrays.sort(Object[])`：TimSort，稳定排序，O(n log n)。
- 合并区间：排序 O(n log n)，扫描 O(n)。
- 三数之和：排序 O(n log n)，双指针 O(n^2)。
- 快选：平均 O(n)，最坏 O(n^2)。

## 边界条件

- 空数组、长度不足。
- 比较器溢出。
- 是否要求稳定排序。
- 排序是否允许修改原数组。
- 重复元素去重。
- 区间端点相等时是否合并。
- 快选中的第 k 大和第 k 小下标换算。

## 典型题型

- 合并区间。
- 插入区间。
- 三数之和。
- 最接近的三数之和。
- 数组中的第 k 个最大元素。
- 按频率排序。
- 最大数。
- 根据身高重建队列。
- 会议室。
- 颜色分类。

## 面试讲解口径

排序题要说明排序后的收益：

> 我先排序，不是为了得到有序结果本身，而是为了让相同元素相邻、大小关系单调。这样后续可以用一次扫描、双指针或贪心处理。

对于三数之和：

1. 先排序。
2. 固定第一个数。
3. 剩余部分用左右指针找两数之和。
4. 固定值和左右指针都要跳过重复值，保证结果不重复。

对于 TopK：

1. 如果只要第 k 大，快选平均 O(n)。
2. 如果要动态维护前 k 个，使用大小为 k 的小根堆。
3. 如果数据量小或实现优先，直接排序最简单。

## 易错点

- 比较器使用减法导致溢出。
- 三数之和忘记跳过重复固定值。
- 找第 k 大时目标下标写错，应该是 `n - k`。
- 快选没有随机化 pivot，在有序数组上退化。
- 合并区间时边界相接是否合并没有按题意处理。
- 排序修改了原数组，但题目要求保留原顺序。

## 训练清单

- 能写合并区间和三数之和。
- 能说明 Java 基本类型排序与对象排序的稳定性差异。
- 能选择排序、堆、快选解决 TopK。
- 能写安全比较器。
- 能解释排序后双指针的单调性。
- 能处理重复元素去重。
