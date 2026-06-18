# 双指针

## 题型定位

双指针用于把嵌套枚举降维，常见于数组、字符串、链表。它不是一种数据结构，而是一种利用单调性、顺序性或相对位置关系的扫描方式。

常见指针形态：

- 左右指针：从两端向中间收缩。
- 快慢指针：同向移动，速度不同或语义不同。
- 读写指针：一个扫描，一个写入。
- 多路指针：合并多个有序序列。
- 分区指针：把数组划分为小于、等于、大于某值的区域。

## 核心模型

### 1. 左右收缩

适合有序数组、回文、容器盛水等问题。

```java
int l = 0;
int r = nums.length - 1;
while (l < r) {
    if (shouldMoveLeft(nums, l, r)) {
        l++;
    } else {
        r--;
    }
}
```

关键是能解释为什么移动某一侧不会漏掉答案，通常依赖有序性或短板效应。

### 2. 快慢指针

适合链表环、删除倒数第 k 个节点、数组去重。

```java
ListNode slow = head;
ListNode fast = head;
while (fast != null && fast.next != null) {
    slow = slow.next;
    fast = fast.next.next;
}
```

### 3. 读写指针

适合原地修改。`read` 扫描所有元素，`write` 表示下一个有效位置。

```java
int write = 0;
for (int read = 0; read < nums.length; read++) {
    if (keep(nums[read])) {
        nums[write++] = nums[read];
    }
}
```

### 4. 三指针分区

荷兰国旗问题维护三个区域：

- `[0, lt)` 小于目标。
- `[lt, i)` 等于目标。
- `(gt, n - 1]` 大于目标。

```java
void sortColors(int[] nums) {
    int lt = 0;
    int i = 0;
    int gt = nums.length - 1;

    while (i <= gt) {
        if (nums[i] == 0) {
            swap(nums, lt++, i++);
        } else if (nums[i] == 2) {
            swap(nums, i, gt--);
        } else {
            i++;
        }
    }
}
```

注意和右侧交换后，`i` 不能立刻前进，因为换过来的元素还没检查。

## 常见模板

### 有序数组两数之和

```java
int[] twoSumSorted(int[] nums, int target) {
    int l = 0;
    int r = nums.length - 1;

    while (l < r) {
        int sum = nums[l] + nums[r];
        if (sum == target) {
            return new int[]{l, r};
        } else if (sum < target) {
            l++;
        } else {
            r--;
        }
    }
    return new int[0];
}
```

### 判断子序列

```java
boolean isSubsequence(String s, String t) {
    int i = 0;
    int j = 0;

    while (i < s.length() && j < t.length()) {
        if (s.charAt(i) == t.charAt(j)) {
            i++;
        }
        j++;
    }
    return i == s.length();
}
```

### 盛最多水的容器

```java
int maxArea(int[] height) {
    int l = 0;
    int r = height.length - 1;
    int ans = 0;

    while (l < r) {
        ans = Math.max(ans, Math.min(height[l], height[r]) * (r - l));
        if (height[l] < height[r]) {
            l++;
        } else {
            r--;
        }
    }
    return ans;
}
```

## 复杂度

- 大多数双指针题时间 O(n)，因为每个指针单调移动。
- 空间通常 O(1)。
- 如果外层固定一个点、内层双指针，例如三数之和，时间 O(n^2)。
- 如果需要排序后双指针，总时间通常 O(n log n) 或 O(n^2)，取决于枚举层数。

## 边界条件

- 数组长度小于 2。
- 左右指针循环条件是 `l < r` 还是 `l <= r`。
- 有序数组是升序还是降序。
- 是否允许复用同一个元素。
- 原地覆盖是否需要保持相对顺序。
- 链表快指针是否可能为空。
- 三数之和需要跳过重复值，避免重复答案。

## 典型题型

- 有序数组两数之和。
- 三数之和。
- 盛最多水的容器。
- 接雨水。
- 验证回文串。
- 判断子序列。
- 删除排序数组中的重复项。
- 移动零。
- 环形链表。
- 删除链表倒数第 n 个节点。

## 面试讲解口径

双指针题的关键不是说“我用两个指针”，而是说明移动规则为什么正确。

例如有序数组两数之和：

1. 数组有序，当前和小于目标时，右指针左移只会让和更小，所以只能左指针右移。
2. 当前和大于目标时，左指针右移只会让和更大，所以只能右指针左移。
3. 每次移动都排除一批不可能答案，且不会漏解。

例如盛水容器：

1. 面积由宽度和较短高度决定。
2. 宽度每次必然变小。
3. 如果移动较高的一侧，短板不变或更低，不可能得到更大面积。
4. 所以每次移动较短的一侧。

## 易错点

- 没有有序性却强行使用左右双指针。
- 去重时只跳过一侧，导致重复答案。
- `while (l <= r)` 导致同一个元素被使用两次。
- 三数之和固定点后忘记跳过相同固定值。
- 快慢指针访问 `fast.next.next` 前没有判空。
- 分区交换后错误推进指针。

## 训练清单

- 能写出有序两数之和、三数之和。
- 能解释盛水容器移动短板的正确性。
- 能写出读写指针类原地覆盖题。
- 能熟练处理重复答案去重。
- 能区分快慢指针在数组和链表中的用法。
- 能在白板上画出指针移动过程。
