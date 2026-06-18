# 二分

## 题型定位

二分用于在有序空间中快速定位答案。这里的“有序”不一定是数组升序，也可以是一个单调判断函数：当答案大于某个值后全部可行，或小于某个值后全部不可行。

常见类型：

- 有序数组查找目标值。
- 查找第一个大于等于目标的位置。
- 查找最后一个小于等于目标的位置。
- 旋转数组查找。
- 在答案范围上二分，例如最小速度、最小容量、最大甜度。
- 二维矩阵映射成一维二分。

## 核心模型

### 1. 左闭右闭查找目标

```java
int search(int[] nums, int target) {
    int left = 0;
    int right = nums.length - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) {
            return mid;
        } else if (nums[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    return -1;
}
```

### 2. 左边界 lower_bound

寻找第一个 `>= target` 的位置。

```java
int lowerBound(int[] nums, int target) {
    int left = 0;
    int right = nums.length;

    while (left < right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] >= target) {
            right = mid;
        } else {
            left = mid + 1;
        }
    }
    return left;
}
```

返回值可能等于 `nums.length`，表示不存在大于等于目标的元素。

### 3. 右边界 upper_bound

寻找第一个 `> target` 的位置，最后一个 `<= target` 的位置就是 `upperBound - 1`。

```java
int upperBound(int[] nums, int target) {
    int left = 0;
    int right = nums.length;

    while (left < right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] > target) {
            right = mid;
        } else {
            left = mid + 1;
        }
    }
    return left;
}
```

### 4. 答案二分

答案二分不是查数组，而是查最小可行值或最大可行值。

```java
int left = minAnswer;
int right = maxAnswer;
while (left < right) {
    int mid = left + (right - left) / 2;
    if (can(mid)) {
        right = mid;
    } else {
        left = mid + 1;
    }
}
return left;
```

要求 `can(x)` 具有单调性。

## 常见模板

### 搜索插入位置

```java
int searchInsert(int[] nums, int target) {
    return lowerBound(nums, target);
}
```

### 查找元素第一个和最后一个位置

```java
int[] searchRange(int[] nums, int target) {
    int l = lowerBound(nums, target);
    if (l == nums.length || nums[l] != target) {
        return new int[]{-1, -1};
    }
    int r = upperBound(nums, target) - 1;
    return new int[]{l, r};
}
```

### 旋转排序数组搜索

```java
int searchRotated(int[] nums, int target) {
    int left = 0;
    int right = nums.length - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) {
            return mid;
        }

        if (nums[left] <= nums[mid]) {
            if (nums[left] <= target && target < nums[mid]) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        } else {
            if (nums[mid] < target && target <= nums[right]) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
    }
    return -1;
}
```

### 矩阵搜索

```java
boolean searchMatrix(int[][] matrix, int target) {
    int m = matrix.length;
    int n = matrix[0].length;
    int left = 0;
    int right = m * n - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;
        int x = matrix[mid / n][mid % n];
        if (x == target) {
            return true;
        } else if (x < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    return false;
}
```

## 复杂度

- 标准二分：时间 O(log n)，空间 O(1)。
- 答案二分：时间 O(log R * checkCost)，`R` 为答案范围大小。
- 二维矩阵映射：时间 O(log(mn))。

## 边界条件

- 空数组。
- 目标值小于所有元素或大于所有元素。
- 重复元素下查左边界还是右边界。
- `mid = (left + right) / 2` 可能溢出，应写成 `left + (right - left) / 2`。
- 左闭右闭与左闭右开模板不要混用。
- 答案二分中 `left`、`right` 必须覆盖所有可能答案。
- `can(mid)` 的单调方向要确认，是可行区在右侧还是左侧。

## 典型题型

- 二分查找。
- 搜索插入位置。
- 在排序数组中查找元素的第一个和最后一个位置。
- 搜索旋转排序数组。
- 寻找旋转排序数组中的最小值。
- 搜索二维矩阵。
- 寻找峰值。
- 爱吃香蕉的珂珂。
- 在 D 天内送达包裹的能力。
- 分割数组的最大值。

## 面试讲解口径

二分题建议从搜索空间讲起：

1. 搜索空间是什么，是数组下标还是答案范围。
2. 单调性是什么，为什么可以二分。
3. 循环不变量是什么，例如答案始终在 `[left, right)` 中。
4. `mid` 不满足条件时，哪一半可以被排除。
5. 返回值是否需要检查越界和等值。

答案二分示例：

> 速度越大，完成时间只会越短，所以可行性关于速度单调。我要找最小可行速度，因此当 `can(mid)` 为真时，答案在左半边包含 `mid`；否则答案在右半边。

## 易错点

- `left < right` 模板里写了 `right = mid - 1`，导致跳过答案。
- `left <= right` 模板里写了 `right = mid`，导致死循环。
- 查边界后没有检查返回位置是否越界。
- 旋转数组中没有判断哪一半有序。
- 答案二分的上下界设错，真实答案不在范围内。
- `can` 函数没有处理整除向上取整。
- 重复元素场景下仍按无重复旋转数组模板写。

## 训练清单

- 能默写标准查找、`lowerBound`、`upperBound`。
- 能用边界模板实现搜索插入位置和搜索范围。
- 能解释左闭右开模板的返回值含义。
- 能识别答案二分，并写出 `can` 函数。
- 能处理旋转数组的有序半边判断。
- 能手推长度为 0、1、2 的数组。
