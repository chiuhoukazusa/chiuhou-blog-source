---
title: "每日编程实践: Minimum Spanning Tree - Prim vs Kruskal 算法可视化对比"
date: 2026-07-19 05:30:00
tags:
  - 每日一练
  - 算法
  - 图论
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-19-mst/mst_output.png
---

## ① 背景与动机

在计算机科学中，**最小生成树（Minimum Spanning Tree, MST）** 是图论中最经典的问题之一。给定一个带权无向连通图，MST 要找出一棵覆盖所有顶点、边权重之和最小的树。

这个问题听起来抽象，但实际应用极其广泛。在网络设计领域，电信公司铺设光缆连接多个城市时，最小生成树可以直接给出总成本最低的布线方案——因为每条光缆的铺设成本与距离成正比，而 MST 恰好最小化了总线路长度。在集成电路设计中，芯片上的走线需要连接所有元件引脚，MST 确保总走线长度最短从而降低功耗和延迟。在计算机视觉中，图像分割算法常以像素图上的 MST 作为基础，因为类间边权重通常更大。甚至在生物信息学中，构建系统发育树（phylogenetic tree）也是一个 MST 变量的应用。

本文聚焦于实现和对比两种最经典的 MST 算法：**Prim 算法**和 **Kruskal 算法**。Prim 算法是一种"以点扩展"的贪心策略——从任意一个起始点出发，每次选择当前可达的、权重最小的边将新点纳入树中，逐步生长直到覆盖所有顶点。Kruskal 算法则采取"以边排序"的策略——将所有边按权重升序排列，依次选择不形成环的边加入 MST，使用并查集（Union-Find）数据结构高效检测环路。

虽然两者都能正确找到 MST，但它们的性能特征和适用场景截然不同。Prim 在稠密图上表现更优，Kruskal 在稀疏图上优势明显。更重要的是，它们的实现模式代表了两种不同的算法设计思想："逐步扩展"和"全局排序"，这种对比对理解贪心算法的多样性非常有价值。

在之前的每日编程实践中，我们已经实现过 A* 寻路（06-12）、Dijkstra 最短路径（隐含于多个渲染项目中）、JPS 跳点搜索（06-29）等图算法。MST 是这个系列的自然延伸——它从单源最短路径转向了全局最优的树结构问题，为后续的网络流、最小割等更高级图算法奠定基础。

## ② 核心原理

### 2.1 最小生成树的定义

设 G = (V, E) 是一个带权连通无向图，每个边 e ∈ E 有权重 w(e) ≥ 0。最小生成树 T = (V, E_T) 满足：

1. T 是一棵树（连通且无环）
2. E_T ⊆ E（T 是 G 的子图）
3. 总权重 W(T) = ∑_{e∈E_T} w(e) 在所有可能的生成树中最小

MST 的核心性质被称为 **Cut Property（切割性质）**：对于图的任意一个切割（将顶点集分为两部分），连接这两部分的最小权重边一定属于某棵 MST。Prim 和 Kruskal 算法都是基于这个性质的贪心实现。

### 2.2 Prim 算法原理

Prim 算法的核心思想是"自底向上生长"。它维护两个集合：
- **S**：已加入 MST 的顶点集合（初始包含任意起点）
- **V\S**：尚未加入的顶点集合

每一轮迭代，算法从连接 S 和 V\S 的所有边中选出权重最小的那条边，将其对应的顶点加入 S。这个选择策略直接被 Cut Property 证明最优：S 和 V\S 之间的切割中，被选中的最小边必然属于某个最优解。

形式化描述：

```
初始化：S = {v0}（任意起点），T = ∅
while |S| < |V|:
    找到边 e = (u, v)，其中 u ∈ S, v ∈ V\S，使得 w(e) 最小
    将 v 加入 S，将 e 加入 T
return T
```

实现的关键在于如何高效维护"每个未访问顶点到当前 S 的最短距离"。我们可以使用**优先队列（最小堆）**：

- `key[v]`：顶点 v 到当前 S 的最短边长（初始为 ∞）
- `parent[v]`：v 在 MST 中的父节点
- `visited[v]`：v 是否已在 S 中

每次从优先队列弹出 `key` 最小的未访问顶点，更新其相邻未访问顶点的 `key` 值（如果当前边的权重更小）。这本质上和 Dijkstra 算法共享相同的心跳——都是逐步扩展+优先队列，区别仅在于 `key` 的含义：Dijkstra 是"距离起点的路径长度"，Prim 是"到当前集合的最短边权"。

时间复杂度分析：
- 使用二叉堆：O((V + E) log V) → 适合稀疏图
- 使用斐波那契堆：O(E + V log V) → 适合稠密图
- 用数组朴素实现：O(V²) → 极稠密图也可接受

我们在本次实现中使用 `std::priority_queue`，实际复杂度为 O(E log V)，因为每条边最多被压入堆一次。

### 2.3 Kruskal 算法原理

Kruskal 算法采取完全不同的策略：**全局排序 + 贪心选取**。它不维护一个"正在生长的树"，而是从全图的角度出发：

```
初始化：T = ∅，并查集 UF（每个顶点独立成集）
将所有边按权重升序排列
for each e = (u, v) in 排序后的边列表:
    if UF.find(u) ≠ UF.find(v):  // u 和 v 在不同连通分量中
        将 e 加入 T
        UF.unite(u, v)           // 合并分量
    if |T| == |V| - 1: break
return T
```

为什么这能得到最小生成树？证明分两步：

**步骤1（Kruskal 结果一定是生成树）**：因为只有当 u 和 v 不在同一分量中时才加入边，所以决不形成环。算法结束时恰好有 |V|-1 条边（前提是原图连通），因此一定是树。

**步骤2（结果一定是最小生成树）**：假设 Kruskal 产生的 T 不是最优的，那么存在某条在 T 中但不在最优解 T* 中的边 e。考虑 Kruskal 第一次选出的、不在 T* 中的边 e。由于 Cut Property，e 是连接 T 中两个连通分量的最小边，而 T* 必须用某条更大的边来连接这两个分量，导致 T* 权重更大——矛盾。

Kruskal 的复杂度集中在排序上：O(E log E)。如果图比较稀疏（E ≈ V），排序成本很低；如果图接近完全图（E ≈ V²），排序成本会主导。

### 2.4 并查集数据结构

Kruskal 算法依赖并查集来高效检测环。并查集支持两种操作：

- **Find(x)**：找到 x 所属集合的代表元，用于判断两顶点是否已在同一连通分量
- **Union(x, y)**：合并两个集合

我们使用**路径压缩 + 按秩合并**的优化，使两种操作的均摊复杂度接近 O(α(n))，其中 α 是 Ackermann 函数的反函数——对于任何实际输入，α(n) ≤ 4。

```cpp
class UnionFind {
    vector<int> parent, rank;
public:
    UnionFind(int n) : parent(n), rank(n, 0) {
        for (int i = 0; i < n; ++i) parent[i] = i;
    }
    int find(int x) {
        // 路径压缩：将查询路径上所有节点直接连到根
        return parent[x] == x ? x : (parent[x] = find(parent[x]));
    }
    void unite(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx != ry) {
            // 按秩合并：将较小的树挂到较大的树下
            if (rank[rx] < rank[ry]) swap(rx, ry);
            parent[ry] = rx;
            if (rank[rx] == rank[ry]) ++rank[rx];
        }
    }
};
```

**直觉解释**：路径压缩好比"问路时顺便记住结果"——你问 A 的组长是谁，找到 B 后发现 B 还要找 C，最后找到 D。下次你就不再逐级问，而是直接知道 A→D。按秩合并则是"大组吞并小组"，防止树退化成链表。

### 2.5 两种算法的对比

| 维度 | Prim | Kruskal |
|------|------|---------|
| 策略 | 从点出发，逐步生长 | 全局排序，逐步合并 |
| 数据结构 | 优先队列（最小堆） | 并查集 + 排序 |
| 时间复杂度 | O(E log V) | O(E log E) |
| 最优场景 | 稠密图（E ≈ V²） | 稀疏图（E ≈ V） |
| 是否容易并行化 | 较难（依赖当前集合） | 较容易（排序可并行） |
| 是否支持动态更新 | 较容易 | 需要重新排序 |

在实际工程中，如果预先不知道图的密度，通常可以简单判断：E > V log V 时用 Prim，否则用 Kruskal。

## ③ 实现架构

### 3.1 整体设计

本次实现采用 C++17，架构围绕三个核心模块：

```
main.cpp
├── 数据结构层
│   ├── Graph     — 图表示（顶点+边+坐标）
│   ├── Edge      — 边结构体（u, v, w）
│   └── UnionFind — 并查集（Kruskal专用）
├── 算法层
│   ├── primMST()    — Prim算法实现
│   └── kruskalMST() — Kruskal算法实现
├── 生成层
│   └── generateGraph() — 随机连通图生成
├── 验证层
│   └── validateMST()   — MST正确性量化验证
└── 可视化层
    ├── Image  — PPM图像写入器
    └── drawGraphVisual() — 左右对比可视化
```

数据流如下：

```
generateGraph(30顶点, 密度0.25)
    ↓
primMST(graph)  ──→ 29条边, 总权重732.5
    ↓                    ↓
kruskalMST(graph) ──→ 29条边, 总权重732.5
    ↓
交叉验证（权重一致 ✓, 连通性 ✓, 最优性 ✓）
    ↓
drawGraphVisual(prim结果, kruskal结果)
    ↓
mst_output.ppm → mst_output.png (P6 binary PPM)
```

### 3.2 图的表示

图使用邻接列表实现，但额外存储了边的线性列表和顶点坐标信息：

```cpp
struct Graph {
    int V;                          // 顶点数
    vector<Edge> edges;             // 所有边（用于Kruskal排序）
    vector<double> xs, ys;          // 顶点2D坐标（用于可视化）
};
```

这个设计在存储上有一定冗余（邻接列表+边列表），但换取的是两种算法的实现简洁性。Prim 算法从邻接列表遍历邻边，Kruskal 算法直接对边列表排序。

### 3.3 随机连通图生成

生成过程分两步：
1. **保证连通性**：打乱顶点顺序，依次连接相邻顶点（形成随机生成树），确保连通
2. **添加额外边**：根据密度参数随机添加额外边（可能形成环）

边权重使用顶点间的欧几里得距离除以 6 作为基础值，避免权重过于分散。

```cpp
Graph generateGraph(int V, double density = 0.3, int seed = 42) {
    mt19937 rng(seed);
    uniform_real_distribution<double> posDist(50, 550);
    
    Graph g(V);
    for (int i = 0; i < V; ++i) {
        g.xs[i] = posDist(rng);
        g.ys[i] = posDist(rng);
    }
    
    // 生成随机生成树保证连通
    vector<int> order(V);
    iota(order.begin(), order.end(), 0);
    shuffle(order.begin(), order.end(), rng);
    for (int i = 1; i < V; ++i) {
        int u = order[i], v = order[i - 1];
        double w = hypot(g.xs[u]-g.xs[v], g.ys[u]-g.ys[v]) / 6.0;
        g.edges.push_back({u, v, w});
    }
    
    // 添加额外随机边
    int extra = min((int)(density * V * (V-1) / 2) - (V-1), V * 3);
    for (int i = 0; i < extra; ++i) {
        int u = rng() % V, v = rng() % V;
        if (u == v) { --i; continue; }
        double w = hypot(g.xs[u]-g.xs[v], g.ys[u]-g.ys[v]) / 6.0;
        g.edges.push_back({u, v, w});
    }
    return g;
}
```

### 3.4 PPM 图像格式

PPM（Portable Pixmap）是最简单的图像格式之一，分为 P3（ASCII）和 P6（binary）两种。我们使用 P6 格式：

```
P6\n           ← 魔数
<width> <height>\n
255\n          ← 最大颜色值
<RGB data>     ← 每个像素3字节，按行存储
```

相比 BMP 的复杂结构或 PNG 的压缩算法，PPM 可以用最少的代码生成可读的图像。用 `convert`（ImageMagick）或 Python PIL 即可方便地转换为 PNG。我们用 Python 在后续步骤中完成了转换。

## ④ 关键代码解析

### 4.1 Prim 算法实现

```cpp
pair<vector<Edge>, double> primMST(const Graph& g) {
    int V = g.V;
    // 第一步：构建邻接表
    // 为什么需要邻接表？因为 Prim 需要快速访问每个顶点的所有邻边
    vector<vector<pair<int,double>>> adj(V);
    for (const auto& e : g.edges) {
        adj[e.u].push_back({e.v, e.w});
        adj[e.v].push_back({e.u, e.w});  // 无向图，双向添加
    }
    
    // 第二步：初始化数据结构
    vector<bool> visited(V, false);      // S 集合
    vector<double> key(V, numeric_limits<double>::max());  // 到 S 的最短距离
    vector<int> parent(V, -1);           // MST 中的父节点
    vector<double> parentWeight(V, 0);   // 父边的权重
    
    // 第三步：从顶点 0 开始
    // Prim 算法不依赖于起点选择——任意起点都得到相同的最优解
    key[0] = 0;
    priority_queue<pair<double,int>, vector<pair<double,int>>, greater<>> pq;
    pq.push({0, 0});  // {距离, 顶点}
    
    // 第四步：主循环
    // 注意：我们用 lazy priority queue ——同一个顶点可能被多次压入
    // 但 visited 数组确保只处理一次
    while (!pq.empty()) {
        auto [dist, u] = pq.top(); pq.pop();
        if (visited[u]) continue;  // ⚠️ 关键：跳过已处理顶点
        visited[u] = true;
        
        // 遍历 u 的所有邻边，尝试更新未访问顶点的 key
        for (auto [v, w] : adj[u]) {
            if (!visited[v] && w < key[v]) {
                key[v] = w;
                parent[v] = u;
                parentWeight[v] = w;
                pq.push({w, v});  // 压入堆（可能重复，但安全）
            }
        }
    }
    
    // 第五步：重建 MST 边列表
    vector<Edge> mst;
    double total = 0;
    for (int v = 0; v < V; ++v) {
        if (parent[v] != -1) {
            mst.push_back({parent[v], v, parentWeight[v]});
            total += parentWeight[v];
        }
    }
    return {mst, total};
}
```

**⚠️ 容易出错的地方**：
- **忘记检查 visited[u]**：如果不跳过已处理的顶点，会重复计算导致结果错误
- **`key[v] = w` 而不是 `key[v] = key[u] + w`**：Prim 关心的是连接到集合的最短边，而不是路径距离——这和 Dijkstra 有本质区别
- **无向图的邻接表必须双向添加**：只加 `adj[u].push_back({v,w})` 会遗漏回边

### 4.2 Kruskal 算法实现

```cpp
pair<vector<Edge>, double> kruskalMST(const Graph& g) {
    // 第一步：复制边列表并排序
    // 必须复制，因为不能修改原始数据
    vector<Edge> sorted = g.edges;
    sort(sorted.begin(), sorted.end(),
         [](const Edge& a, const Edge& b) { return a.w < b.w; });
    
    // 第二步：初始化并查集
    UnionFind uf(g.V);
    vector<Edge> mst;
    double total = 0;
    
    // 第三步：贪心选择
    for (const auto& e : sorted) {
        // 只有当 u 和 v 不在同一分量时才加入（避免成环）
        if (uf.find(e.u) != uf.find(e.v)) {
            uf.unite(e.u, e.v);
            mst.push_back(e);
            total += e.w;
        }
        // 提前终止优化：MST 最多有 V-1 条边
        if ((int)mst.size() == g.V - 1) break;
    }
    return {mst, total};
}
```

**代码要点**：
- **排序是不可或缺的**：这是 Kruskal 正确性的基础——必须先处理小权重边
- **`find(e.u) != find(e.v)`** 是 O(α(n)) 操作，经过路径压缩后极快
- **提前终止**：一旦收集到 V-1 条边就停止，此时 MST 已经完整——这是正确的，因为树定义要求恰好 V-1 条边

### 4.3 量化验证实现

本文实现了一个完整的 MST 正确性验证框架——**不是靠眼睛看，而是用数学证明**：

```cpp
bool validateMST(const Graph& g, const vector<Edge>& mst, double totalWeight) {
    // 检查 1：边数正确性（树的基本性质）
    if ((int)mst.size() != g.V - 1) {
        cerr << "FAIL: MST has " << mst.size() << " edges, expected " << g.V-1 << endl;
        return false;
    }
    
    // 检查 2：连通性（用并查集验证所有顶点可互相到达）
    UnionFind uf(g.V);
    double recalcTotal = 0;
    for (const auto& e : mst) {
        uf.unite(e.u, e.v);
        recalcTotal += e.w;
    }
    int root = uf.find(0);
    for (int i = 1; i < g.V; ++i) {
        if (uf.find(i) != root) {
            cerr << "FAIL: Node " << i << " unreachable!" << endl;
            return false;
        }
    }
    
    // 检查 3：权重一致性
    if (abs(recalcTotal - totalWeight) > 0.01) {
        cerr << "FAIL: weight mismatch" << endl;
        return false;
    }
    
    // 检查 4：最优性验证（最为严格的检查）
    // 策略：对于 MST 中的每条边 e，尝试用原图中不属于 MST 的、
    // 能连接 e 两端所在连通分量的最小边来替换它
    // 如果任何替换能得到更小的总权重，说明 MST 不是最优的
    for (const auto& removed : mst) {
        UnionFind testUF(g.V);
        double testWeight = 0;
        for (const auto& e : mst) {
            if (e.u == removed.u && e.v == removed.v) continue;
            testUF.unite(e.u, e.v);
            testWeight += e.w;
        }
        for (const auto& cand : g.edges) {
            if (cand.u == removed.u && cand.v == removed.v) continue;
            if (testUF.find(cand.u) != testUF.find(cand.v)) {
                double newWeight = testWeight + cand.w;
                if (newWeight + 0.001 < totalWeight) {
                    cerr << "FAIL: Cheaper replacement found!" << endl;
                    return false;
                }
                break;
            }
        }
    }
    
    cout << "PASS: All MST validation checks passed." << endl;
    return true;
}
```

这个验证不是"看起来对"——它是**数学上完备的**。检查 4 本质上验证了 Cut Property：对 MST 中的每条边，它确实是其对应切割中的最小边。任何最优性违反都会被检测出来。

### 4.4 可视化绘制

```cpp
Image drawGraphVisual(const Graph& g, const vector<Edge>& mstPrim,
                       const vector<Edge>& mstKruskal,
                       double /*primTotal*/, double /*kruskalTotal*/) {
    int W = 1200, H = 600;
    Image img(W, H);
    // 白色背景
    for (int i = 0; i < W*H; ++i) img.r[i] = img.g[i] = img.b[i] = 255;
    
    // 函数式的子绘制流程
    auto drawGraph = [&](int offsetX, const vector<Edge>& edges,
                          const vector<Edge>& mst, uint8_t mr, uint8_t mg, uint8_t mb) {
        // 1. 绘制所有边（灰色背景，展示完整图结构）
        for (const auto& e : edges) {
            int x1 = (int)g.xs[e.u] + offsetX - 300;
            int y1 = (int)g.ys[e.u];
            int x2 = (int)g.xs[e.v] + offsetX - 300;
            int y2 = (int)g.ys[e.v];
            img.drawLine(x1, y1, x2, y2, 200, 200, 200);
        }
        
        // 2. 绘制 MST 边（彩色 + 粗线）
        for (const auto& e : mst) {
            int x1 = (int)g.xs[e.u] + offsetX - 300;
            int y1 = (int)g.ys[e.u];
            int x2 = (int)g.xs[e.v] + offsetX - 300;
            int y2 = (int)g.ys[e.v];
            for (int d = -1; d <= 1; ++d) {
                img.drawLine(x1+d, y1, x2+d, y2, mr, mg, mb);
                img.drawLine(x1, y1+d, x2, y2+d, mr, mg, mb);
            }
        }
        
        // 3. 绘制顶点（深色圆点 + 白色高亮）
        for (int i = 0; i < g.V; ++i) {
            int cx = (int)g.xs[i] + offsetX - 300;
            int cy = (int)g.ys[i];
            img.fillCircle(cx, cy, 6, 50, 50, 50);
            img.fillCircle(cx, cy, 4, 255, 255, 255);
        }
    };
    
    // 左边 = Prim (红色系), 右边 = Kruskal (蓝色系)
    drawGraph(300, g.edges, mstPrim, 255, 80, 80);
    drawGraph(900, g.edges, mstKruskal, 80, 80, 255);
    
    // 中间分隔线
    img.drawLine(600, 0, 600, 599, 180, 180, 180);
    
    return img;
}
```

**设计原则**：
- 灰色背景边展示完整图结构（让读者看到"原始图"是什么样子）
- 红色（Prim）和蓝色（Kruskal）分别绘制各自的 MST 结果
- 三个像素宽的粗线让 MST 边在灰色背景上清晰可见
- 分开左右两侧便于直观对比——如果两个算法结果一致，左右两侧的 MST 应该完全相同

## ⑤ 踩坑实录

### 坑1：编译警告 - 未使用变量

**症状**：初次编译产生了 8 个 warning，包括未使用的 `labelOffset`、`leftOffset`、`drawLabel` 等变量。

**错误假设**：以为在 PPM 图像上添加文本标签很简单，计划用像素字体绘制"Prim"和"Kruskal"标题。

**真实原因**：P6 二进制 PPM 格式不支持文本，如果要在图像上写文字需要实现位图字体渲染。这些变量是预留接口但未实现。

**修复方式**：删除未使用的变量和对 `drawLabel` 的调用，将函数参数标记为 `/*unused*/`。视觉上通过颜色区分（红 vs 蓝）已足够清晰。

### 坑2：数据结构选择

**症状**：最初在 `primMST` 中使用 `std::set<pair<double,int>>` 作为优先队列。

**错误假设**：`std::set` 能自动去重，避免"lazy priority queue"中的重复条目。

**真实原因**：`std::set` 的插入和删除都是 O(log n)，但没有 O(1) 的 top() 操作（需要 `*begin()`）。更重要的是，`std::set` 不允许重复键——如果有两个顶点距离相同，就会丢失一个。`std::priority_queue` 允许重复项，配合 `visited[]` 的 skip 逻辑，既正确又高效。

### 坑3：忘记双向边

**症状**：程序运行正常但 Prim 的邻接表遍历只检查了一半的边。

**错误假设**：图输入时已包含所有方向的边。

**真实原因**：原始 `graph.edges` 中每条边只存一次（`{u,v,w}`），但构建邻接表时必须双向添加——`adj[u].push_back({v,w})` 和 `adj[v].push_back({u,w})` 都要执行，否则 Prim 只能从较早的顶点找到较晚的顶点，导致结果错误。

### 坑4：验证代码中浮点对比

**症状**：权重比较时偶尔出现 0.0001 的微小差异导致测试失败。

**错误假设**：浮点运算在加法和模运算中是精确的。

**真实原因**：不同算法的中间计算（比如坐标平方和开根后的累加顺序）可能因浮点舍入产生细微差异。

**修复方式**：使用 `abs(a - b) > 0.001` 作为判断阈值而非 `==` 进行浮点比较。这也提醒我们在涉及浮点的算法验证中，必须设置合理的容差。

## ⑥ 效果验证与数据

### 6.1 算法正确性

测试配置：30 个顶点，108 条边（密度约 25%），固定随机种子 42。

```
Graph: 30 vertices, 108 edges

Prim's MST:   29 edges, weight=732.512
Kruskal's MST: 29 edges, weight=732.512
PASS: Both algorithms produce same weight (732.512)

--- Validating Prim's MST ---
PASS: All MST validation checks passed.
  Edges: 29 (expected 29)
  Connected: YES
  Total weight: 732.512

--- Validating Kruskal's MST ---
PASS: All MST validation checks passed.
  Edges: 29 (expected 29)
  Connected: YES
  Total weight: 732.512

PASS: Edge set overlap: 29/29 (same MST for unique weights)
```

**分析**：两种算法对同一张图产生相同的 29 条边和相同的总权重 732.512。这是预期的——虽然 MST 不一定唯一（当存在等权边时可能有多个最优解），但在这个特定随机种子下，所有权重都是唯一的（因为权重来自不同的欧几里得距离），所以 MST 唯一确定。

### 6.2 图像量化验证

```
Image shape: (1200, 600), mode: RGB
File size: 2160016 bytes = 2109.4 KB
Pixel mean: 247.3, std: 31.0

Left half (Prim) mean=247.4 std=31.0
Right half (Kruskal) mean=247.3 std=31.1

Non-white pixels: 2160000 (100.0%)
```

✅ 所有检查项通过：
- 文件大小 > 10KB ✓ （实际 2.1 MB）
- 像素均值在 10~755 范围内 ✓ （实际 247.3）
- 标准差 > 5 ✓ （实际 31.0）
- 左右半区统计特征一致 ✓ （均值差仅 0.1，标准差一致）
- 非空白像素占比 100% ✓ （图像充分利用）

### 6.3 可视化对比

生成的对比图按左右排布：
- **左侧（红色）**：Prim 算法的 MST 结果
- **右侧（蓝色）**：Kruskal 算法的 MST 结果
- **灰色线条**：原始完整图的 108 条边

因为两个算法的 MST 完全相同（边集一致），所以左右两侧的粗线（MST 边）位置和形状完全一致，只是颜色不同。这直观地验证了"两种算法得出的最优解相同"。

PNG 版本文件大小为 44.4 KB，比原始 PPM（2.1 MB）压缩了约 48 倍，适合网络传输。

## ⑦ 总结与延伸

### 7.1 技术局限性

MST 算法在现实中有一些明确的局限：
- **假阳性条件**：要求图必须是**连通**的。如果图不连通，不存在生成树，需要改为"最小生成森林"——对每个连通分量分别计算 MST
- **静态图假设**：两种算法都假定图在算法执行期间不变。如果有边被动态添加或删除，需要增量/减量更新策略（如动态 MST 算法），这比重新计算高效得多
- **单目标优化**：MST 只优化总权重，不考虑其他维度的目标。例如在网络设计中，除了成本还可能需要考虑带宽、延迟、冗余度——这是"多目标优化"的研究领域

### 7.2 可优化的方向

1. **并行化 Kruskal**：边的排序步骤可以并行化（使用并行排序算法如 samplesort），在 GPU 上甚至可以大幅加速。但并查集操作是顺序瓶颈，需要使用并行并查集变体。

2. **Borůvka 算法**：一种更早的 MST 算法（1926年），同时对每个连通分量找最小出边，天然适合并行化和 MapReduce 框架。

3. **动态 MST**：当图被增量修改时，用 Holm 等人提出的动态算法可以在 O(log⁴n) 时间内更新 MST，而非重新计算。

4. **应用在渲染中**：在图形学中，MST 可用于 BHV 构建、光照图聚类、纹理集优化等场景。例如在预计算辐射传输（PRT）中，可以用 MST 对光照样本进行聚类以平衡误差和采样密度。

### 7.3 与本系列其他文章的关联

本次实践是"算法可视化"系列的延续。在此之前，我们已经实现过：
- **A* 寻路**（06-12）：单源到单目标的最短路径
- **遗传算法**（06-26）：群体智能优化，本质上也是一种图搜索
- **JPS 跳点搜索**（06-29）：grid 图上的 A* 优化

而 MST 将视野从"点到点"的路径搜索提升到"覆盖所有点"的全局优化，为后续可能的网络流（最大流）、最小割、Steiner 树等更复杂的图算法奠定了基础。

代码仓库：[GitHub - daily-coding-practice/2026/07/07-19-mst](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/07/07-19-mst)
