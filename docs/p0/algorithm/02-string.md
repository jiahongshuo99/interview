# 字符串

## 题型定位

字符串题本质上是字符数组、计数、匹配和状态机问题。Java 中 `String` 不可变，频繁拼接应使用 `StringBuilder`；如果需要原地修改，通常先转成 `char[]`。

面试中的字符串题常考：

- 字符计数：异位词、字符替换、最长不重复子串。
- 双指针：回文判断、反转字符串、压缩字符串。
- 滑动窗口：最小覆盖子串、排列包含。
- 子串匹配：暴力、KMP、滚动哈希。
- 解析模拟：字符串转整数、版本号比较、表达式简化。
- 字符串构造：按规则生成、去重、栈式消除。

## 核心模型

### 1. 字符计数

当字符集固定，例如小写字母，可以用数组替代哈希表。

```java
int[] cnt = new int[26];
for (char c : s.toCharArray()) {
    cnt[c - 'a']++;
}
```

如果字符范围不固定，使用 `Map<Character, Integer>`。

### 2. 双指针回文

```java
boolean isPalindrome(String s) {
    int l = 0;
    int r = s.length() - 1;

    while (l < r) {
        while (l < r && !Character.isLetterOrDigit(s.charAt(l))) {
            l++;
        }
        while (l < r && !Character.isLetterOrDigit(s.charAt(r))) {
            r--;
        }

        char a = Character.toLowerCase(s.charAt(l));
        char b = Character.toLowerCase(s.charAt(r));
        if (a != b) {
            return false;
        }
        l++;
        r--;
    }
    return true;
}
```

### 3. 子串匹配

普通匹配可以从每个起点尝试，复杂度 O(nm)。如果面试官要求优化，KMP 的核心是用 `lps` 数组记录模式串每个位置之前的最长相等前后缀。

```java
int[] buildLps(String p) {
    int[] lps = new int[p.length()];
    int len = 0;

    for (int i = 1; i < p.length(); i++) {
        while (len > 0 && p.charAt(i) != p.charAt(len)) {
            len = lps[len - 1];
        }
        if (p.charAt(i) == p.charAt(len)) {
            len++;
        }
        lps[i] = len;
    }
    return lps;
}
```

### 4. 解析状态机

字符串转整数、版本号比较、路径简化等题目，不要急着写分支，先定义状态：

- 当前读到的位置。
- 当前 token 的含义。
- 是否已经遇到符号、小数点、指数、分隔符。
- 溢出时应该截断还是报错。

## 常见模板

### 反转字符数组

```java
void reverseString(char[] s) {
    int l = 0;
    int r = s.length - 1;
    while (l < r) {
        char t = s[l];
        s[l++] = s[r];
        s[r--] = t;
    }
}
```

### 判断异位词

```java
boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) {
        return false;
    }

    int[] cnt = new int[26];
    for (int i = 0; i < s.length(); i++) {
        cnt[s.charAt(i) - 'a']++;
        cnt[t.charAt(i) - 'a']--;
    }

    for (int x : cnt) {
        if (x != 0) {
            return false;
        }
    }
    return true;
}
```

### 字符串压缩

```java
int compress(char[] chars) {
    int write = 0;
    int i = 0;

    while (i < chars.length) {
        char c = chars[i];
        int j = i;
        while (j < chars.length && chars[j] == c) {
            j++;
        }

        chars[write++] = c;
        int count = j - i;
        if (count > 1) {
            for (char d : String.valueOf(count).toCharArray()) {
                chars[write++] = d;
            }
        }
        i = j;
    }
    return write;
}
```

## 复杂度

- 字符计数：时间 O(n)，空间 O(字符集大小)。
- 回文双指针：时间 O(n)，空间 O(1)。
- `StringBuilder` 构造：时间 O(n)，空间 O(n)。
- KMP：构造 `lps` O(m)，匹配 O(n)，总空间 O(m)。
- 排序判断异位词：时间 O(n log n)，空间取决于排序实现。

## 边界条件

- 空字符串、长度为 1。
- 大小写是否敏感。
- 是否只包含小写字母，还是包含 ASCII、Unicode、空格、标点。
- 字符串中是否包含前导零、正负号、小数点。
- Java `char` 是 UTF-16 code unit，面试普通题通常按 `char` 处理即可；如果题目强调 Unicode 码点，需要用 `codePointAt`。
- 子串题要区分子串连续、子序列不连续。
- 返回空串、`-1`、`false` 还是原串，要严格按题目定义。

## 典型题型

- 有效回文。
- 反转字符串。
- 字母异位词分组。
- 字符串中的第一个唯一字符。
- 最长公共前缀。
- 实现 `strStr`。
- 字符串转换整数。
- 比较版本号。
- 简化路径。
- 解码字符串。

## 面试讲解口径

字符串题讲解时，优先说明字符集和操作成本。

可以这样说：

1. 如果题目限定小写字母，我使用长度 26 的数组做计数，常数更低。
2. 如果要频繁构造结果，Java 中不能反复用 `+` 拼接，应该用 `StringBuilder`。
3. 如果是回文或反转类问题，使用左右指针，每轮收缩边界。
4. 如果是子串匹配，先给出暴力方案，再根据复杂度要求说明 KMP 或滑动窗口。
5. 如果是解析题，先列出状态和非法输入，再写分支，避免遗漏边界。

## 易错点

- 把子序列当成子串处理。
- 用 `String` 反复拼接导致 O(n^2)。
- 忘记判断长度不同的异位词。
- `Character.toLowerCase` 和非法字符跳过顺序写反。
- 字符计数数组下标越界，题目实际包含大写或其他字符。
- KMP 回退时写成 `len--`，导致退化或错误；应回到 `lps[len - 1]`。
- 解析整数时先乘 10 再判断溢出，已经越界。

## 训练清单

- 能写出回文判断、反转字符串、异位词判断。
- 能说明 `String`、`char[]`、`StringBuilder` 的适用场景。
- 能独立实现字符串压缩和路径简化。
- 能区分滑动窗口字符串题的“窗口内计数”和“窗口有效性”。
- 能讲清楚 KMP 的 `lps` 数组含义，不要求死记所有细节。
- 能针对大小写、空格、标点、空串手推结果。
