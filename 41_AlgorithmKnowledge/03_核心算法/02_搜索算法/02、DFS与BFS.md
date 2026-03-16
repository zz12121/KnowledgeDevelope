# DFS 与 BFS - 图和树的核心搜索策略

## 🎯 两种搜索策略对比

| 维度 | DFS（深度优先） | BFS（广度优先） |
|------|----------------|----------------|
| 数据结构 | 栈（递归调用栈） | 队列 |
| 访问顺序 | 一路走到底，回溯 | 按层扩展 |
| 空间复杂度 | O(h)，h为路径深度 | O(w)，w为最宽层节点数 |
| 最短路径 | ❌（无权图不保证） | ✅（无权图保证最短） |
| 连通性检测 | ✅ | ✅ |
| 实现方式 | 递归或显式栈 | 队列 |

---

## 🔍 深度优先搜索（DFS）

### 1.1 基本框架

```java
// 二叉树 DFS（递归）
void dfs(TreeNode node) {
    if (node == null) return;  // 终止条件

    // 前序位置：处理当前节点
    process(node);

    dfs(node.left);   // 递归左子树
    dfs(node.right);  // 递归右子树

    // 后序位置：回溯时处理
}

// 图 DFS（递归，需要 visited 数组防止重复）
void dfs(List<List<Integer>> graph, int node, boolean[] visited) {
    visited[node] = true;
    process(node);

    for (int neighbor : graph.get(node)) {
        if (!visited[neighbor]) {
            dfs(graph, neighbor, visited);
        }
    }
}
```

### 1.2 迭代版 DFS（显式栈）

```java
void dfsIterative(List<List<Integer>> graph, int start, int n) {
    boolean[] visited = new boolean[n];
    Deque<Integer> stack = new ArrayDeque<>();
    stack.push(start);

    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (visited[node]) continue;
        visited[node] = true;
        process(node);

        for (int neighbor : graph.get(node)) {
            if (!visited[neighbor]) stack.push(neighbor);
        }
    }
}
```

### 1.3 DFS 应用：路径问题

```java
// 二叉树所有根到叶的路径（LeetCode 257）
public List<String> binaryTreePaths(TreeNode root) {
    List<String> result = new ArrayList<>();
    dfsPath(root, "", result);
    return result;
}

private void dfsPath(TreeNode node, String path, List<String> result) {
    if (node == null) return;
    path += node.val;
    if (node.left == null && node.right == null) {
        result.add(path);  // 到达叶节点，记录路径
        return;
    }
    dfsPath(node.left, path + "->", result);
    dfsPath(node.right, path + "->", result);
}
```

```java
// 矩阵中的路径（DFS + 回溯）
// 岛屿数量（LeetCode 200）
public int numIslands(char[][] grid) {
    int count = 0;
    for (int i = 0; i < grid.length; i++) {
        for (int j = 0; j < grid[0].length; j++) {
            if (grid[i][j] == '1') {
                dfs(grid, i, j);
                count++;
            }
        }
    }
    return count;
}

private void dfs(char[][] grid, int i, int j) {
    int m = grid.length, n = grid[0].length;
    if (i < 0 || i >= m || j < 0 || j >= n || grid[i][j] != '1') return;
    grid[i][j] = '0';  // 标记为已访问
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    for (int[] d : dirs) dfs(grid, i + d[0], j + d[1]);
}
```

### 1.4 DFS 应用：连通分量数量

```java
public int countComponents(int n, int[][] edges) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < n; i++) graph.add(new ArrayList<>());
    for (int[] e : edges) {
        graph.get(e[0]).add(e[1]);
        graph.get(e[1]).add(e[0]);
    }

    boolean[] visited = new boolean[n];
    int count = 0;
    for (int i = 0; i < n; i++) {
        if (!visited[i]) {
            dfs(graph, i, visited);
            count++;
        }
    }
    return count;
}
```

---

## 🌊 广度优先搜索（BFS）

### 2.1 基本框架

```java
// 图 BFS
void bfs(List<List<Integer>> graph, int start, int n) {
    boolean[] visited = new boolean[n];
    Queue<Integer> queue = new LinkedList<>();

    visited[start] = true;
    queue.offer(start);

    while (!queue.isEmpty()) {
        int node = queue.poll();
        process(node);

        for (int neighbor : graph.get(node)) {
            if (!visited[neighbor]) {
                visited[neighbor] = true;
                queue.offer(neighbor);
            }
        }
    }
}

// 按层 BFS（记录层数 = 距离）
int bfsLevel(List<List<Integer>> graph, int start, int end, int n) {
    boolean[] visited = new boolean[n];
    Queue<Integer> queue = new LinkedList<>();
    visited[start] = true;
    queue.offer(start);
    int level = 0;

    while (!queue.isEmpty()) {
        int size = queue.size();  // 当前层的节点数
        for (int i = 0; i < size; i++) {
            int node = queue.poll();
            if (node == end) return level;
            for (int neighbor : graph.get(node)) {
                if (!visited[neighbor]) {
                    visited[neighbor] = true;
                    queue.offer(neighbor);
                }
            }
        }
        level++;
    }
    return -1;  // 不可达
}
```

### 2.2 BFS 应用：最短路径

```java
// 单词接龙（LeetCode 127）
// 每次改变一个字母，从 beginWord 到 endWord 的最短步数
public int ladderLength(String beginWord, String endWord, List<String> wordList) {
    Set<String> wordSet = new HashSet<>(wordList);
    if (!wordSet.contains(endWord)) return 0;

    Queue<String> queue = new LinkedList<>();
    Set<String> visited = new HashSet<>();
    queue.offer(beginWord);
    visited.add(beginWord);
    int steps = 1;

    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            String word = queue.poll();
            char[] chars = word.toCharArray();
            for (int j = 0; j < chars.length; j++) {
                char original = chars[j];
                for (char c = 'a'; c <= 'z'; c++) {
                    if (c == original) continue;
                    chars[j] = c;
                    String next = new String(chars);
                    if (next.equals(endWord)) return steps + 1;
                    if (wordSet.contains(next) && !visited.contains(next)) {
                        visited.add(next);
                        queue.offer(next);
                    }
                }
                chars[j] = original;
            }
        }
        steps++;
    }
    return 0;
}
```

### 2.3 BFS 应用：矩阵最短距离

```java
// 01矩阵（LeetCode 542）：求每个 1 到最近 0 的距离
public int[][] updateMatrix(int[][] mat) {
    int m = mat.length, n = mat[0].length;
    int[][] dist = new int[m][n];
    Queue<int[]> queue = new LinkedList<>();

    // 多源 BFS：所有 0 同时入队
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (mat[i][j] == 0) {
                queue.offer(new int[]{i, j});
            } else {
                dist[i][j] = Integer.MAX_VALUE;
            }
        }
    }

    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    while (!queue.isEmpty()) {
        int[] cell = queue.poll();
        for (int[] d : dirs) {
            int ni = cell[0] + d[0], nj = cell[1] + d[1];
            if (ni >= 0 && ni < m && nj >= 0 && nj < n
                    && dist[ni][nj] > dist[cell[0]][cell[1]] + 1) {
                dist[ni][nj] = dist[cell[0]][cell[1]] + 1;
                queue.offer(new int[]{ni, nj});
            }
        }
    }
    return dist;
}
```

### 2.4 双向 BFS（优化搜索效率）

从起点和终点同时出发，当两者相遇时找到最短路径。空间复杂度从 O(b^d) 降低到 O(b^(d/2))。

```java
// 适用：知道终点，搜索空间大
public int bidirectionalBFS(String start, String end) {
    Set<String> beginSet = new HashSet<>();
    Set<String> endSet = new HashSet<>();
    beginSet.add(start);
    endSet.add(end);
    int steps = 0;

    while (!beginSet.isEmpty() && !endSet.isEmpty()) {
        // 始终扩展较小的集合（优化）
        if (beginSet.size() > endSet.size()) {
            Set<String> temp = beginSet; beginSet = endSet; endSet = temp;
        }
        Set<String> nextSet = new HashSet<>();
        for (String word : beginSet) {
            // 对 word 进行变换，生成邻居...
            // 如果邻居在 endSet 中，说明相遇，返回 steps+1
        }
        beginSet = nextSet;
        steps++;
    }
    return -1;
}
```

---

## 📊 DFS vs BFS 选择指南

```
需要找最短路径（无权图）？
  → BFS

需要判断两点是否连通？
  → DFS 或 BFS（并查集更高效）

需要找所有路径 / 所有方案？
  → DFS + 回溯

需要遍历树的某层所有节点？
  → BFS（层序遍历）

需要探索尽可能深的路径？
  → DFS

矩阵问题（岛屿、染色、连通区域）？
  → DFS（代码更简洁）或 BFS

起点和终点都知道，求最短？
  → 双向 BFS（效率更高）
```

---

## 📝 高频面试题

| 题号 | 题目 | 方法 |
|------|------|------|
| LeetCode 200 | 岛屿数量 | DFS/BFS/并查集 |
| LeetCode 695 | 岛屿的最大面积 | DFS |
| LeetCode 102 | 二叉树层序遍历 | BFS |
| LeetCode 127 | 单词接龙 | BFS |
| LeetCode 207 | 课程表（是否有环） | DFS/BFS拓扑 |
| LeetCode 542 | 01矩阵 | 多源BFS |
| LeetCode 994 | 腐烂的橘子 | 多源BFS |
| LeetCode 130 | 被围绕的区域 | DFS从边界出发 |

---

**记住**：
- **DFS** = 递归/栈，探索到底再回溯，用于找路径/判断连通/生成方案
- **BFS** = 队列，按层扩散，用于找最短路径/最少步数
- **多源 BFS**：多个起点同时入队，解决"到最近某类节点的距离"问题
