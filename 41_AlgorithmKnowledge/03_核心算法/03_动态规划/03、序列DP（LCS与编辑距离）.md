# 最长公共子序列与编辑距离

## 序列 DP 概述

序列 DP 处理两个序列之间的关系，状态通常定义在两个序列的前缀上。

---

## 一、最长公共子序列（LCS）

### 问题描述（LeetCode 1143）

给两个字符串 text1 和 text2，返回它们的**最长公共子序列**的长度。

**子序列**：不要求连续，但要保持相对顺序。

**示例**：
```
text1 = "abcde"
text2 = "ace"
LCS = "ace"，长度为 3
```

### 状态定义

`dp[i][j]` = text1 的前 i 个字符和 text2 的前 j 个字符的最长公共子序列长度

### 状态转移

```
如果 text1[i] == text2[j]：
    dp[i][j] = dp[i-1][j-1] + 1  （当前字符匹配，在之前LCS基础上+1）
    
如果 text1[i] != text2[j]：
    dp[i][j] = max(dp[i-1][j], dp[i][j-1])  （舍去一个字符，取较大值）
```

### 代码实现

```java
public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i - 1) == text2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[m][n];
}
// 时间复杂度：O(mn)，空间复杂度：O(mn)
```

### 可视化理解

```
text1 = "ace", text2 = "abcde"

    ""  a  b  c  d  e
""   0  0  0  0  0  0
a    0  1  1  1  1  1
c    0  1  1  2  2  2
e    0  1  1  2  2  3

答案：dp[3][5] = 3
```

---

## 二、最长递增子序列（LIS）

### 问题描述（LeetCode 300）

给整数数组，找到其中最长严格递增子序列的长度。

**示例**：[10, 9, 2, 5, 3, 7, 101, 18]，LIS = [2,3,7,101]，长度 4

### 方法一：动态规划 O(n²)

**状态定义**：`dp[i]` = 以 nums[i] 结尾的最长递增子序列长度

**状态转移**：对所有 j < i，若 nums[j] < nums[i]，则可以把 nums[i] 接到 j 结尾的子序列后
```
dp[i] = max(dp[j] + 1)，其中 j < i 且 nums[j] < nums[i]
```

```java
public int lengthOfLIS(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1);  // 每个元素自身是长度为1的子序列
    int maxLen = 1;
    
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
        maxLen = Math.max(maxLen, dp[i]);
    }
    return maxLen;
}
// 时间复杂度：O(n²)，空间复杂度：O(n)
```

### 方法二：贪心 + 二分查找 O(n log n)

维护一个**单调递增**的 tails 数组，tails[i] 表示长度为 i+1 的递增子序列中，末尾元素的最小值。

```java
public int lengthOfLIS(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    
    for (int num : nums) {
        int pos = Collections.binarySearch(tails, num);
        if (pos < 0) pos = -(pos + 1);  // 找到插入位置
        
        if (pos == tails.size()) {
            tails.add(num);  // 比所有元素都大，延长序列
        } else {
            tails.set(pos, num);  // 替换，保持末尾最小
        }
    }
    return tails.size();
}
```

---

## 三、编辑距离

### 问题描述（LeetCode 72）

将字符串 word1 转换为 word2 所使用的最少操作数，操作包括：
- 插入一个字符
- 删除一个字符
- 替换一个字符

**示例**：word1 = "horse", word2 = "ros"，最少 3 步操作

### 状态定义

`dp[i][j]` = word1 前 i 个字符转换成 word2 前 j 个字符所需的最少操作数

### 状态转移

```
如果 word1[i] == word2[j]：
    dp[i][j] = dp[i-1][j-1]  （最后一个字符相同，无需操作）
    
如果 word1[i] != word2[j]：
    dp[i][j] = min(
        dp[i-1][j] + 1,    // 删除 word1[i]
        dp[i][j-1] + 1,    // 插入 word2[j]
        dp[i-1][j-1] + 1   // 替换 word1[i] 为 word2[j]
    )
```

### 边界条件

```
dp[i][0] = i  （word1 前 i 个字符转为空串：删除 i 次）
dp[0][j] = j  （空串转为 word2 前 j 个字符：插入 j 次）
```

### 代码实现

```java
public int minDistance(String word1, String word2) {
    int m = word1.length(), n = word2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    // 边界初始化
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i - 1) == word2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = Math.min(
                    Math.min(dp[i - 1][j] + 1, dp[i][j - 1] + 1),
                    dp[i - 1][j - 1] + 1
                );
            }
        }
    }
    return dp[m][n];
}
// 时间复杂度：O(mn)，空间复杂度：O(mn)
```

### 可视化理解

```
word1 = "horse", word2 = "ros"

      ""  r  o  s
""     0  1  2  3
h      1  1  2  3
o      2  2  1  2
r      3  2  2  2
s      4  3  3  2
e      5  4  4  3

最少 3 次操作
```

---

## 四、其他经典序列 DP 题

### 最长公共子串（连续）

```java
// dp[i][j] = 以 text1[i] 和 text2[j] 结尾的最长公共子串长度
public int longestCommonSubstring(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m + 1][n + 1];
    int maxLen = 0;
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i - 1) == text2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
                maxLen = Math.max(maxLen, dp[i][j]);
            }
            // 注意：不匹配时 dp[i][j] = 0（默认值），不能从别处转移
        }
    }
    return maxLen;
}
```

### 推荐练习

| 题号 | 题目 | 难度 |
|------|------|------|
| 1143 | 最长公共子序列 | Medium |
| 300 | 最长递增子序列 | Medium |
| 72 | 编辑距离 | Hard |
| 583 | 两个字符串的删除操作 | Medium |
| 115 | 不同的子序列 | Hard |
| 516 | 最长回文子序列 | Medium |
