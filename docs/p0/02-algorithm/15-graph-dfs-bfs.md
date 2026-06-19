# 图 DFS / BFS

## 题型定位

图题处理的是节点和边的关系。面试基础图题通常围绕 DFS、BFS、连通性、最短步数、拓扑顺序和网格图展开。图可以显式给出邻接表，也可以隐式存在于矩阵、字符串状态或单词转换中。

常见考点：

- 图的表示：邻接表、邻接矩阵、边列表。
- DFS 遍历连通块。
- BFS 求无权图最短路。
- 网格图四方向搜索。
- visited 防止重复访问。
- 拓扑排序判断有向无环图。
- 并查集可作为连通性替代方案，但基础图题先掌握 DFS/BFS。

## 核心模型

### 1. 邻接表

```java
List<Integer>[] graph = new ArrayList[n];
for (int i = 0; i < n; i++) {
    graph[i] = new ArrayList<>();
}
for (int[] edge : edges) {
    int a = edge[0];
    int b = edge[1];
    graph[a].add(b);
    graph[b].add(a);
}
```

有向图只加 `a -> b`，无向图两边都加。

### 2. DFS

DFS 适合遍历连通块、判断是否可达、做路径搜索。

```java
void dfs(int u, List<Integer>[] graph, boolean[] visited) {
    visited[u] = true;
    for (int v : graph[u]) {
        if (!visited[v]) {
            dfs(v, graph, visited);
        }
    }
}
```

### 3. BFS

BFS 适合无权图最短路径或最少步数。

```java
int bfs(int start, int target, List<Integer>[] graph) {
    Queue<Integer> queue = new ArrayDeque<>();
    boolean[] visited = new boolean[graph.length];
    queue.offer(start);
    visited[start] = true;

    int step = 0;
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int u = queue.poll();
            if (u == target) {
                return step;
            }
            for (int v : graph[u]) {
                if (!visited[v]) {
                    visited[v] = true;
                    queue.offer(v);
                }
            }
        }
        step++;
    }
    return -1;
}
```

### 4. 网格图

矩阵中的每个格子是节点，上下左右是边。

```java
int[] dr = {1, -1, 0, 0};
int[] dc = {0, 0, 1, -1};

for (int d = 0; d < 4; d++) {
    int nr = r + dr[d];
    int nc = c + dc[d];
    if (nr < 0 || nr >= m || nc < 0 || nc >= n) {
        continue;
    }
}
```

## 常见模板

### 岛屿数量 DFS

```java
int numIslands(char[][] grid) {
    int m = grid.length;
    int n = grid[0].length;
    int ans = 0;

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == '1') {
                ans++;
                dfs(grid, i, j);
            }
        }
    }
    return ans;
}

void dfs(char[][] grid, int r, int c) {
    if (r < 0 || r >= grid.length || c < 0 || c >= grid[0].length) {
        return;
    }
    if (grid[r][c] != '1') {
        return;
    }

    grid[r][c] = '0';
    dfs(grid, r + 1, c);
    dfs(grid, r - 1, c);
    dfs(grid, r, c + 1);
    dfs(grid, r, c - 1);
}
```

### 腐烂的橘子 BFS

```java
int orangesRotting(int[][] grid) {
    int m = grid.length;
    int n = grid[0].length;
    Queue<int[]> queue = new ArrayDeque<>();
    int fresh = 0;

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == 2) {
                queue.offer(new int[]{i, j});
            } else if (grid[i][j] == 1) {
                fresh++;
            }
        }
    }

    int minutes = 0;
    int[] dr = {1, -1, 0, 0};
    int[] dc = {0, 0, 1, -1};

    while (!queue.isEmpty() && fresh > 0) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] cur = queue.poll();
            for (int d = 0; d < 4; d++) {
                int nr = cur[0] + dr[d];
                int nc = cur[1] + dc[d];
                if (nr < 0 || nr >= m || nc < 0 || nc >= n || grid[nr][nc] != 1) {
                    continue;
                }
                grid[nr][nc] = 2;
                fresh--;
                queue.offer(new int[]{nr, nc});
            }
        }
        minutes++;
    }

    return fresh == 0 ? minutes : -1;
}
```

### 课程表拓扑排序

```java
boolean canFinish(int numCourses, int[][] prerequisites) {
    List<Integer>[] graph = new ArrayList[numCourses];
    for (int i = 0; i < numCourses; i++) {
        graph[i] = new ArrayList<>();
    }

    int[] indegree = new int[numCourses];
    for (int[] p : prerequisites) {
        int course = p[0];
        int pre = p[1];
        graph[pre].add(course);
        indegree[course]++;
    }

    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < numCourses; i++) {
        if (indegree[i] == 0) {
            queue.offer(i);
        }
    }

    int visited = 0;
    while (!queue.isEmpty()) {
        int u = queue.poll();
        visited++;
        for (int v : graph[u]) {
            indegree[v]--;
            if (indegree[v] == 0) {
                queue.offer(v);
            }
        }
    }
    return visited == numCourses;
}
```

## 复杂度

- 邻接表 DFS/BFS：时间 O(V + E)，空间 O(V + E) 存图，遍历额外 O(V)。
- 邻接矩阵遍历：时间 O(V^2)，空间 O(V^2)。
- 网格图 DFS/BFS：时间 O(mn)，空间 O(mn) 最坏。
- 拓扑排序：时间 O(V + E)，空间 O(V + E)。

## 边界条件

- 图可能不连通，需要从每个未访问节点启动遍历。
- 无向图建边要加两次。
- 有向图环检测不能只用 boolean visited，DFS 需要三色标记或拓扑排序。
- BFS 中入队时就标记 visited，避免重复入队。
- 网格为空或只有一行一列。
- 修改原 grid 是否允许；不允许时使用 visited 数组。
- 递归 DFS 在大图上可能栈溢出，可改迭代。

## 典型题型

- 岛屿数量。
- 岛屿的最大面积。
- 被围绕的区域。
- 腐烂的橘子。
- 打开转盘锁。
- 单词接龙。
- 克隆图。
- 课程表。
- 省份数量。
- 判断二分图。
- 所有可能路径。

## 面试讲解口径

图题先说明图如何建模：

> 我把每个节点编号为 0 到 n-1，用邻接表保存每个节点能到达的节点。无向边需要双向加入，有向边只加入一个方向。

DFS：

> DFS 从一个未访问节点出发，把整个连通块全部标记。外层遍历所有节点，每遇到一个未访问节点就说明发现了一个新的连通块。

BFS：

> 无权图中 BFS 按层扩展，第一次到达目标节点时，层数就是最短步数。因此要在入队时标记 visited，避免同一节点重复入队。

拓扑排序：

> 入度为 0 的课程表示没有前置依赖，可以先学。每学完一门课，就把它指向课程的入度减一。最后如果学完数量等于课程总数，说明没有环。

## 易错点

- 无向图只加了一条边。
- 图不连通时只从 0 开始遍历。
- BFS 出队时才标记 visited，导致同一节点被多次入队。
- 网格 DFS 没有先判边界就访问数组。
- 岛屿题修改 grid 后，后续逻辑还依赖原值。
- 有向图环检测用普通 visited，无法区分正在访问和已经访问完成。
- 拓扑排序中课程和前置课程方向建反。

## 训练清单

- 能用邻接表建无向图和有向图。
- 能写 DFS 统计连通块。
- 能写 BFS 求无权最短步数。
- 能写网格四方向 DFS/BFS。
- 能写岛屿数量和腐烂橘子。
- 能写课程表拓扑排序，并说明入度含义。
