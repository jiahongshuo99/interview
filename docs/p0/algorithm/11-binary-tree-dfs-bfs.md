# 二叉树 DFS / BFS

## 题型定位

二叉树题主要考递归定义、遍历顺序、层序遍历和树形状态传递。DFS 关注从节点向子树递归，BFS 关注按层处理。面试中二叉树是递归能力的高频检验点。

常见考点：

- 前序、中序、后序遍历。
- 层序遍历。
- 树的最大深度、最小深度。
- 路径和。
- 最近公共祖先。
- 对称、相同、翻转。
- 二叉搜索树性质。
- 树形动态规划的基础形态。

## 基础结构

```java
class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;

    TreeNode(int val) {
        this.val = val;
    }
}
```

## 核心模型

### 1. DFS 递归

递归三问：

1. 当前函数处理什么子问题。
2. 终止条件是什么。
3. 当前节点如何使用左右子树结果。

```java
int maxDepth(TreeNode root) {
    if (root == null) {
        return 0;
    }
    return Math.max(maxDepth(root.left), maxDepth(root.right)) + 1;
}
```

### 2. 前序、中序、后序

```java
void preorder(TreeNode root, List<Integer> ans) {
    if (root == null) {
        return;
    }
    ans.add(root.val);
    preorder(root.left, ans);
    preorder(root.right, ans);
}

void inorder(TreeNode root, List<Integer> ans) {
    if (root == null) {
        return;
    }
    inorder(root.left, ans);
    ans.add(root.val);
    inorder(root.right, ans);
}

void postorder(TreeNode root, List<Integer> ans) {
    if (root == null) {
        return;
    }
    postorder(root.left, ans);
    postorder(root.right, ans);
    ans.add(root.val);
}
```

### 3. BFS 层序遍历

```java
List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> ans = new ArrayList<>();
    if (root == null) {
        return ans;
    }

    Queue<TreeNode> queue = new ArrayDeque<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        int size = queue.size();
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null) {
                queue.offer(node.left);
            }
            if (node.right != null) {
                queue.offer(node.right);
            }
        }
        ans.add(level);
    }
    return ans;
}
```

### 4. 自顶向下与自底向上

自顶向下：父节点把路径状态传给子节点，例如路径和。

自底向上：子节点返回结果给父节点，例如深度、是否平衡。

## 常见模板

### 路径和

```java
boolean hasPathSum(TreeNode root, int targetSum) {
    if (root == null) {
        return false;
    }
    if (root.left == null && root.right == null) {
        return targetSum == root.val;
    }
    return hasPathSum(root.left, targetSum - root.val)
        || hasPathSum(root.right, targetSum - root.val);
}
```

### 判断平衡二叉树

```java
boolean isBalanced(TreeNode root) {
    return height(root) != -1;
}

int height(TreeNode root) {
    if (root == null) {
        return 0;
    }

    int left = height(root.left);
    if (left == -1) {
        return -1;
    }
    int right = height(root.right);
    if (right == -1) {
        return -1;
    }
    if (Math.abs(left - right) > 1) {
        return -1;
    }
    return Math.max(left, right) + 1;
}
```

### 最近公共祖先

```java
TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) {
        return root;
    }

    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    if (left != null && right != null) {
        return root;
    }
    return left != null ? left : right;
}
```

### 验证二叉搜索树

```java
boolean isValidBST(TreeNode root) {
    return check(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

boolean check(TreeNode root, long low, long high) {
    if (root == null) {
        return true;
    }
    if (root.val <= low || root.val >= high) {
        return false;
    }
    return check(root.left, low, root.val) && check(root.right, root.val, high);
}
```

## 复杂度

- DFS 遍历：时间 O(n)，空间 O(h)，`h` 为树高。
- BFS 遍历：时间 O(n)，空间 O(w)，`w` 为最大层宽。
- 平衡树高度 O(log n)，退化链表高度 O(n)。
- 大多数二叉树题每个节点处理一次，时间 O(n)。

## 边界条件

- 空树。
- 单节点树。
- 退化成链表的树。
- 节点值可能是 `Integer.MIN_VALUE` 或 `Integer.MAX_VALUE`。
- 判断 BST 不能只比较当前节点和直接孩子，要传上下界。
- 最小深度遇到空子树时不能简单取 0。
- 递归深度过大可能栈溢出，但面试通常接受递归解。

## 典型题型

- 二叉树的最大深度。
- 二叉树的层序遍历。
- 翻转二叉树。
- 对称二叉树。
- 路径总和。
- 二叉树的直径。
- 平衡二叉树。
- 二叉树的最近公共祖先。
- 验证二叉搜索树。
- 二叉搜索树中第 k 小的元素。

## 面试讲解口径

DFS 题：

> 我把每个节点看成一棵子树的根。函数返回当前子树的答案，空节点作为递归边界。当前节点只需要合并左右子树的结果。

BFS 题：

> 我用队列保存当前层待处理节点。每轮先记录队列大小，这个大小就是当前层节点数，处理完这批节点后再进入下一层。

BST 题：

> 验证 BST 不能只看父子关系，因为右子树中的所有节点都必须大于根，左子树中的所有节点都必须小于根。所以递归时传递上下界。

## 易错点

- 层序遍历没有先保存 `size`，导致一层和下一层混在一起。
- 最小深度把空子树当成深度 0 直接取最小值。
- 验证 BST 只比较直接左右孩子。
- 路径和忘记必须到叶子节点。
- 最近公共祖先用节点值比较，遇到重复值会错；应比较节点引用。
- 递归返回值语义不清，导致全局变量和返回值混用。

## 训练清单

- 能写三种 DFS 遍历和层序遍历。
- 能用递归求最大深度、判断平衡。
- 能写路径和和路径收集。
- 能写最近公共祖先。
- 能写验证 BST，并说明上下界。
- 能区分自顶向下传状态和自底向上返回结果。
