# 回溯基础

## 题型定位

回溯用于枚举所有可能解，本质是深度优先搜索加撤销选择。面试中的回溯题通常不是追求极致复杂度，而是考能否清晰定义路径、选择列表、结束条件和剪枝。

适合特征：

- 要求返回所有方案。
- 组合、排列、子集。
- 棋盘搜索。
- 字符串分割。
- 需要尝试多个选择，不行就退回。

核心关键词：

- 路径：已经做出的选择。
- 选择列表：当前还能选什么。
- 结束条件：什么时候把路径加入答案。
- 撤销选择：递归返回后恢复现场。
- 剪枝：提前排除不可能产生答案的分支。

## 核心模型

### 1. 通用模板

```java
void backtrack(状态 state, List<选择> path) {
    if (满足结束条件) {
        ans.add(new ArrayList<>(path));
        return;
    }

    for (选择 choice : 当前可选列表) {
        if (不合法) {
            continue;
        }

        做选择;
        backtrack(新状态, path);
        撤销选择;
    }
}
```

### 2. 子集

每个元素都有选或不选，也可以用起点下标枚举下一位。

```java
List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> ans = new ArrayList<>();
    backtrackSubsets(nums, 0, new ArrayList<>(), ans);
    return ans;
}

void backtrackSubsets(int[] nums, int start, List<Integer> path, List<List<Integer>> ans) {
    ans.add(new ArrayList<>(path));

    for (int i = start; i < nums.length; i++) {
        path.add(nums[i]);
        backtrackSubsets(nums, i + 1, path, ans);
        path.remove(path.size() - 1);
    }
}
```

### 3. 组合

组合不关心顺序，用 `start` 防止重复选择前面的元素。

```java
void dfs(int start) {
    for (int i = start; i <= n; i++) {
        path.add(i);
        dfs(i + 1);
        path.remove(path.size() - 1);
    }
}
```

### 4. 排列

排列关心顺序，需要 `used` 标记某个元素是否已经在路径中。

```java
List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> ans = new ArrayList<>();
    boolean[] used = new boolean[nums.length];
    backtrackPermute(nums, used, new ArrayList<>(), ans);
    return ans;
}

void backtrackPermute(int[] nums, boolean[] used, List<Integer> path, List<List<Integer>> ans) {
    if (path.size() == nums.length) {
        ans.add(new ArrayList<>(path));
        return;
    }

    for (int i = 0; i < nums.length; i++) {
        if (used[i]) {
            continue;
        }
        used[i] = true;
        path.add(nums[i]);
        backtrackPermute(nums, used, path, ans);
        path.remove(path.size() - 1);
        used[i] = false;
    }
}
```

## 常见模板

### 组合总和

```java
List<List<Integer>> combinationSum(int[] candidates, int target) {
    Arrays.sort(candidates);
    List<List<Integer>> ans = new ArrayList<>();
    dfs(candidates, target, 0, new ArrayList<>(), ans);
    return ans;
}

void dfs(int[] candidates, int remain, int start, List<Integer> path, List<List<Integer>> ans) {
    if (remain == 0) {
        ans.add(new ArrayList<>(path));
        return;
    }

    for (int i = start; i < candidates.length; i++) {
        if (candidates[i] > remain) {
            break;
        }
        path.add(candidates[i]);
        dfs(candidates, remain - candidates[i], i, path, ans);
        path.remove(path.size() - 1);
    }
}
```

同一个数字可以重复使用，所以递归传 `i`；如果每个数字只能使用一次，传 `i + 1`。

### 有重复元素的子集

```java
List<List<Integer>> subsetsWithDup(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> ans = new ArrayList<>();
    dfsSubsetsDup(nums, 0, new ArrayList<>(), ans);
    return ans;
}

void dfsSubsetsDup(int[] nums, int start, List<Integer> path, List<List<Integer>> ans) {
    ans.add(new ArrayList<>(path));

    for (int i = start; i < nums.length; i++) {
        if (i > start && nums[i] == nums[i - 1]) {
            continue;
        }
        path.add(nums[i]);
        dfsSubsetsDup(nums, i + 1, path, ans);
        path.remove(path.size() - 1);
    }
}
```

### 单词搜索

```java
boolean exist(char[][] board, String word) {
    int m = board.length;
    int n = board[0].length;
    boolean[][] used = new boolean[m][n];

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (dfs(board, word, 0, i, j, used)) {
                return true;
            }
        }
    }
    return false;
}

boolean dfs(char[][] board, String word, int idx, int r, int c, boolean[][] used) {
    if (idx == word.length()) {
        return true;
    }
    if (r < 0 || r >= board.length || c < 0 || c >= board[0].length) {
        return false;
    }
    if (used[r][c] || board[r][c] != word.charAt(idx)) {
        return false;
    }

    used[r][c] = true;
    boolean ok = dfs(board, word, idx + 1, r + 1, c, used)
        || dfs(board, word, idx + 1, r - 1, c, used)
        || dfs(board, word, idx + 1, r, c + 1, used)
        || dfs(board, word, idx + 1, r, c - 1, used);
    used[r][c] = false;
    return ok;
}
```

## 复杂度

- 子集：O(2^n * n)，结果数量为 2^n。
- 排列：O(n! * n)，结果数量为 n!。
- 组合：复杂度取决于结果数量和剪枝。
- 棋盘搜索：大致 O(mn * 4^L)，`L` 为单词长度，可通过 visited 降低回头分支。
- 回溯题的复杂度通常至少和输出规模同阶。

## 边界条件

- 空数组、空字符串。
- 结果是否允许空集。
- 元素是否有重复。
- 每个元素能否重复使用。
- 组合是否要求固定长度。
- 棋盘搜索是否允许重复使用同一格。
- 结果加入答案时必须拷贝路径，而不是直接加入引用。

## 典型题型

- 子集。
- 子集 II。
- 全排列。
- 全排列 II。
- 组合。
- 组合总和。
- 电话号码的字母组合。
- 括号生成。
- 单词搜索。
- 分割回文串。
- N 皇后。

## 面试讲解口径

回溯题建议按四要素讲：

1. 路径是什么：当前已经选出的数字或字符。
2. 选择列表是什么：从哪个下标开始选，或者哪些元素还没使用。
3. 结束条件是什么：路径长度满足、目标和为 0、走到字符串末尾。
4. 如何撤销：递归返回后删除最后一个元素、恢复 `used` 标记。

去重题要补充：

> 我先排序，让相同元素相邻。在同一层枚举时，如果当前元素和前一个相同，并且前一个已经被跳过，那么当前也跳过，避免生成重复分支。

## 易错点

- 把 `path` 直接加入答案，没有拷贝。
- 撤销选择遗漏，污染后续分支。
- 组合题递归仍从 0 开始，产生重复排列。
- 排列题没有 `used`，同一个元素被重复使用。
- 有重复元素时没有排序，去重条件失效。
- `i > 0` 和 `i > start` 混用，错误跳过合法分支。
- 棋盘搜索成功返回前没有恢复 visited。

## 训练清单

- 能写子集、组合、排列三个基础模板。
- 能说明组合和排列的区别。
- 能处理重复元素去重。
- 能写组合总和，并区分元素可重复和不可重复。
- 能写单词搜索，正确恢复 visited。
- 能解释回溯复杂度与结果规模的关系。
