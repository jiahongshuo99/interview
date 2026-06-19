# 链表

## 题型定位

链表题考的是指针操作和边界处理。链表不支持 O(1) 随机访问，但插入、删除节点只需要改指针。面试中常用链表题观察候选人是否能稳定处理空指针、头节点变化、环、反转和合并。

常见考点：

- 虚拟头节点。
- 快慢指针。
- 链表反转。
- 合并有序链表。
- 删除节点。
- 环检测。
- 找中点、找倒数第 k 个节点。
- 分组反转。

## 基础结构

```java
class ListNode {
    int val;
    ListNode next;

    ListNode(int val) {
        this.val = val;
    }
}
```

## 核心模型

### 1. 虚拟头节点

当头节点可能被删除、替换或结果链表需要逐步拼接时，用 dummy 简化边界。

```java
ListNode dummy = new ListNode(0);
dummy.next = head;
// 操作 dummy.next
return dummy.next;
```

### 2. 反转链表

维护 `prev` 和 `cur`，每次暂存下一个节点，再改变 `cur.next`。

```java
ListNode reverseList(ListNode head) {
    ListNode prev = null;
    ListNode cur = head;

    while (cur != null) {
        ListNode next = cur.next;
        cur.next = prev;
        prev = cur;
        cur = next;
    }
    return prev;
}
```

### 3. 快慢指针

快指针每次走两步，慢指针每次走一步。

适合：

- 判断是否有环。
- 找环入口。
- 找链表中点。
- 删除倒数第 n 个节点。

### 4. 合并链表

有序链表合并通常使用 dummy 和 tail。

```java
ListNode mergeTwoLists(ListNode a, ListNode b) {
    ListNode dummy = new ListNode(0);
    ListNode tail = dummy;

    while (a != null && b != null) {
        if (a.val <= b.val) {
            tail.next = a;
            a = a.next;
        } else {
            tail.next = b;
            b = b.next;
        }
        tail = tail.next;
    }
    tail.next = a != null ? a : b;
    return dummy.next;
}
```

## 常见模板

### 删除倒数第 n 个节点

```java
ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;

    ListNode fast = dummy;
    ListNode slow = dummy;
    for (int i = 0; i < n; i++) {
        fast = fast.next;
    }

    while (fast.next != null) {
        fast = fast.next;
        slow = slow.next;
    }

    slow.next = slow.next.next;
    return dummy.next;
}
```

### 判断环

```java
boolean hasCycle(ListNode head) {
    ListNode slow = head;
    ListNode fast = head;

    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            return true;
        }
    }
    return false;
}
```

### 找环入口

```java
ListNode detectCycle(ListNode head) {
    ListNode slow = head;
    ListNode fast = head;

    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            ListNode p = head;
            while (p != slow) {
                p = p.next;
                slow = slow.next;
            }
            return p;
        }
    }
    return null;
}
```

### 两两交换节点

```java
ListNode swapPairs(ListNode head) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode prev = dummy;

    while (prev.next != null && prev.next.next != null) {
        ListNode a = prev.next;
        ListNode b = a.next;

        a.next = b.next;
        b.next = a;
        prev.next = b;

        prev = a;
    }
    return dummy.next;
}
```

## 复杂度

- 单次遍历：时间 O(n)，空间 O(1)。
- 反转链表：时间 O(n)，空间 O(1)。
- 合并两个链表：时间 O(m + n)，空间 O(1)。
- 递归反转：时间 O(n)，空间 O(n) 调用栈。
- 合并 k 个链表：使用堆时间 O(N log k)，空间 O(k)。

## 边界条件

- 空链表。
- 单节点链表。
- 删除的是头节点。
- `n` 等于链表长度。
- 偶数长度和奇数长度的中点定义。
- 链表有环时不能普通遍历。
- 反转时必须先保存 `next`，否则后续链表丢失。

## 典型题型

- 反转链表。
- 合并两个有序链表。
- 环形链表。
- 环形链表 II。
- 删除链表倒数第 n 个节点。
- 两两交换链表中的节点。
- K 个一组翻转链表。
- 回文链表。
- 相交链表。
- 排序链表。

## 面试讲解口径

链表题建议边画图边讲：

1. 我用 dummy 节点统一处理头节点变化。
2. 指针修改前先保存后继节点，避免链表断开后找不到剩余部分。
3. 对反转链表，`prev` 表示已经反转好的前缀，`cur` 表示待处理的第一个节点。
4. 对快慢指针，先说明两个指针的间距或速度关系，再说明停止条件。

删除倒数第 n 个节点可以这样讲：

> 我让 `fast` 先走 n 步，然后 `fast` 和 `slow` 一起走。当 `fast` 到达尾部时，`slow.next` 正好是要删除的节点。dummy 可以处理删除头节点的情况。

## 易错点

- 删除头节点时没有 dummy，逻辑分支复杂且容易漏。
- 反转链表忘记保存 `next`。
- 快慢指针循环条件写成 `fast.next.next` 前没有判空。
- 找中点时没有明确偶数长度取左中点还是右中点。
- 合并链表后忘记接上剩余部分。
- 判断链表相交时比较节点引用，而不是节点值。
- 分组反转中每组尾部连接错误。

## 训练清单

- 能熟练写反转链表。
- 能用 dummy 写删除节点和合并链表。
- 能写环检测和环入口。
- 能写删除倒数第 n 个节点。
- 能画图解释两两交换。
- 能挑战 K 个一组翻转链表，重点练边界连接。
