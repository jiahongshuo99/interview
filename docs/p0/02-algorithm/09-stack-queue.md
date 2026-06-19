# 栈、队列

## 题型定位

栈和队列都是受限线性结构。栈后进先出，适合处理最近未匹配、递归展开、单调关系；队列先进先出，适合按层扩展、滑动窗口、任务调度。

常见考点：

- 栈：括号匹配、表达式求值、路径简化、字符串消除。
- 单调栈：下一个更大元素、每日温度、柱状图最大矩形。
- 队列：BFS、层序遍历、最近请求计数。
- 双端队列：滑动窗口最大值、单调队列。
- 栈和队列互相实现。

## 核心模型

### 1. 普通栈匹配

当题目出现“最近的未匹配元素”，优先考虑栈。

```java
Deque<Character> stack = new ArrayDeque<>();
for (char c : s.toCharArray()) {
    if (isLeft(c)) {
        stack.push(c);
    } else {
        if (stack.isEmpty() || !match(stack.pop(), c)) {
            return false;
        }
    }
}
return stack.isEmpty();
```

### 2. 单调栈

单调栈维护一个单调递增或递减的下标栈。它解决“每个元素左/右第一个更大/更小元素”。

```java
int[] nextGreater(int[] nums) {
    int n = nums.length;
    int[] ans = new int[n];
    Arrays.fill(ans, -1);
    Deque<Integer> stack = new ArrayDeque<>();

    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && nums[i] > nums[stack.peek()]) {
            ans[stack.pop()] = nums[i];
        }
        stack.push(i);
    }
    return ans;
}
```

栈里存下标通常比存值更通用。

### 3. BFS 队列

队列保存下一批待处理节点。

```java
Queue<TreeNode> queue = new ArrayDeque<>();
queue.offer(root);
while (!queue.isEmpty()) {
    TreeNode node = queue.poll();
    if (node.left != null) {
        queue.offer(node.left);
    }
    if (node.right != null) {
        queue.offer(node.right);
    }
}
```

### 4. 单调队列

滑动窗口最大值维护一个下标双端队列，队首永远是当前窗口最大值下标。

```java
Deque<Integer> deque = new ArrayDeque<>();
for (int i = 0; i < nums.length; i++) {
    while (!deque.isEmpty() && deque.peekFirst() <= i - k) {
        deque.pollFirst();
    }
    while (!deque.isEmpty() && nums[deque.peekLast()] <= nums[i]) {
        deque.pollLast();
    }
    deque.offerLast(i);
}
```

## 常见模板

### 有效括号

```java
boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();

    for (char c : s.toCharArray()) {
        if (c == '(') {
            stack.push(')');
        } else if (c == '[') {
            stack.push(']');
        } else if (c == '{') {
            stack.push('}');
        } else if (stack.isEmpty() || stack.pop() != c) {
            return false;
        }
    }
    return stack.isEmpty();
}
```

### 最小栈

```java
class MinStack {
    private final Deque<Integer> data = new ArrayDeque<>();
    private final Deque<Integer> mins = new ArrayDeque<>();

    void push(int val) {
        data.push(val);
        if (mins.isEmpty() || val <= mins.peek()) {
            mins.push(val);
        }
    }

    void pop() {
        int x = data.pop();
        if (x == mins.peek()) {
            mins.pop();
        }
    }

    int top() {
        return data.peek();
    }

    int getMin() {
        return mins.peek();
    }
}
```

### 滑动窗口最大值

```java
int[] maxSlidingWindow(int[] nums, int k) {
    int n = nums.length;
    int[] ans = new int[n - k + 1];
    Deque<Integer> deque = new ArrayDeque<>();

    for (int i = 0; i < n; i++) {
        while (!deque.isEmpty() && deque.peekFirst() <= i - k) {
            deque.pollFirst();
        }
        while (!deque.isEmpty() && nums[deque.peekLast()] <= nums[i]) {
            deque.pollLast();
        }
        deque.offerLast(i);
        if (i >= k - 1) {
            ans[i - k + 1] = nums[deque.peekFirst()];
        }
    }
    return ans;
}
```

## 复杂度

- 普通栈、队列操作：均摊 O(1)。
- 单调栈：每个元素最多入栈出栈一次，时间 O(n)，空间 O(n)。
- 单调队列：每个元素最多入队出队一次，时间 O(n)，空间 O(k)。
- BFS：时间 O(V + E)，空间取决于最大层宽或队列峰值。

## 边界条件

- 空栈时不能 `pop` 或 `peek`。
- Java 推荐用 `ArrayDeque` 做栈，不建议用遗留类 `Stack`。
- `ArrayDeque` 不允许存 `null`。
- 单调栈里存值还是下标，取决于是否需要距离或位置。
- 单调队列要先移除窗口外下标，再维护单调性。
- BFS 需要标记 visited，防止图中重复入队。

## 典型题型

- 有效括号。
- 最小栈。
- 用队列实现栈。
- 用栈实现队列。
- 简化路径。
- 逆波兰表达式求值。
- 每日温度。
- 下一个更大元素。
- 柱状图中最大的矩形。
- 滑动窗口最大值。

## 面试讲解口径

栈题：

> 我用栈保存还没有匹配的左括号。遇到右括号时，它必须和最近的左括号匹配，所以弹出栈顶比较。最后栈为空说明全部匹配。

单调栈题：

> 栈中保存还没找到答案的元素下标，并保持对应值单调。当当前元素破坏单调性时，它就是栈顶元素等待的下一个更大值，于是持续弹栈并填答案。

队列题：

> 队列保证先进先出，适合 BFS 按距离或层级扩展。第一次访问到某个节点时，通常就是最短步数。

## 易错点

- 括号题没有在栈空时处理右括号。
- `MinStack` 中最小值重复时没有同步入栈，弹出后最小值错误。
- 单调栈使用值而非下标，无法计算距离。
- 单调队列忘记移除过期下标。
- BFS 图搜索没有 visited，导致死循环。
- 使用 `Stack` 类而不是 `Deque`，代码风格较旧。

## 训练清单

- 能写有效括号、最小栈。
- 能写每日温度并解释单调栈。
- 能写滑动窗口最大值并解释单调队列。
- 能用队列写树和图的 BFS。
- 能区分普通队列、双端队列、优先队列。
- 能处理空结构访问。
