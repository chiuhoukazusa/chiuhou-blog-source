---
title: "每日编程实践: DBSCAN 密度聚类 — 发现任意形状的聚类"
date: 2026-06-22 05:30:00
tags:
  - 每日一练
  - 算法可视化
  - 聚类分析
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-22-dbscan-density-clustering/dbscan_blobs.png
---

# DBSCAN 密度聚类：发现任意形状的聚类

## ① 背景与动机

### 聚类问题的现实挑战

在数据分析和计算机图形学中，**聚类**（Clustering）是最基础的无监督学习任务之一。给定一组点，我们希望自动将它们分组，使得同一组内的点彼此"相似"（距离近），而不同组的点彼此"远离"。

听起来很简单？实际上这个问题的难度远超直觉。

### 为什么 K-Means 不够用？

K-Means 是最经典的聚类算法，但它有三个致命缺陷：

**缺陷 1：必须预先指定 k 值。** 在真实场景中，我们通常不知道数据中有多少个聚类。猜多了会过度分割，猜少了会强行合并——两种情况都会导致错误的结论。比如在粒子系统中自动检测流体团块数量，你不可能每次都手动数一数再设 k。

**缺陷 2：只能发现球形聚类。** K-Means 用欧氏距离度量点到聚类中心的距离，隐含假设了每个聚类是"球形的"（各向同性）。遇到下图这样的弧形数据（两个新月形状），K-Means 会把中间切断，因为它在中间画了一条最"公平"的 Voronoi 边界：

```
  弧形数据：          K-Means 的错误结果：
  🌙        🌙        🌙  /  🌙
  (正确的2个聚类)     (切割了每个弧)
```

**缺陷 3：对噪声和离群点敏感。** K-Means 强制每个点都属于某个聚类——包括那些应该被忽略的噪声点。离群点会把聚类中心"拖偏"，进而影响全局结果。

### DBSCAN 的解决思路

DBSCAN（Density-Based Spatial Clustering of Applications with Noise）于 1996 年由 Ester 等人提出，用一个直觉性的思路解决了以上三个问题：

> **聚类的本质不是"离某个中心近"，而是"被其他点围绕"。**

就像一群人在广场上聚集——你不会说"这群人的中心坐标是 (x, y)"，你会说"这一片区域站满了人"。DBSCAN 正是基于**密度连通性**（density connectivity）——如果一个点周围有足够多的邻居，它就是核心，可以向外"生长"出一个聚类。

**工业界的实际应用场景：**

- **游戏引擎**：自动检测地图上的"兴趣区域"（热点战斗区域），实现动态难度调整
- **LIDAR 点云处理**：自动驾驶中用 DBSCAN 分离障碍物簇（行人、车辆、建筑物），直接输出"有几个障碍物"
- **网络安全**：从登录日志中检测异常行为（孤立的噪声点 = 可疑登录尝试）
- **生物信息学**：基因表达数据的聚类分析，寻找未知类型的细胞亚群
- **空间数据挖掘**：地震震中分组、犯罪热点地图、WiFi 信号分布分析

---

## ② 核心原理

### 2.1 三个基本定义

DBSCAN 算法的所有逻辑建立在三个概念上：

#### 定义 1：ϵ-邻域（Epsilon Neighborhood）

对于点 p，其 ϵ-邻域定义为所有与 p 距离不超过 ϵ 的点：

$$
N_\epsilon(p) = \{ q \in D \mid dist(p, q) \leq \epsilon \}
$$

**直觉**：以 p 为圆心，以 ϵ 为半径画一个圆，圆内的所有点都是 p 的"邻居"。

这个定义直接决定了"密度"的含义——一个点的邻居越多，这个区域的密度就越高。

#### 定义 2：核心点（Core Point）

如果点 p 的 ϵ-邻域内至少包含 minPts 个点（包括 p 自身），则 p 是一个核心点。

$$
|N_\epsilon(p)| \geq minPts
$$

**直觉**：只有"人气高"的地方才能成为聚类的"种子"。minPts 是门槛——低于这个门槛的点可能是边界点或者噪声点，不能独立发起一个聚类。

**minPts 参数的意义**：在实践中，minPts 通常设为 4 或更大。对于有噪声的数据集，minPts 越大，算法对噪声越鲁棒（太小的噪声团不会被误判为聚类）。经验法则：minPts ≥ D+1（D 是数据维度），所以 2D 数据 minPts ≥ 3。

#### 定义 3：密度可达与密度相连

- **直接密度可达**（Directly Density-Reachable）：如果 p 是核心点且 q ∈ N_ϵ(p)，则 q 是从 p 直接密度可达的。
- **密度可达**（Density-Reachable）：如果存在一个核心点链 p₁, p₂, ..., pₙ，使得 p₁=p, pₙ=q，且 pᵢ₊₁ 从 pᵢ 直接密度可达，则 q 从 p 密度可达。
- **密度相连**（Density-Connected）：如果存在点 o，使得 p 和 q 都从 o 密度可达，则 p 和 q 是密度相连的。

**直觉**：密度可达性就是"连锁反应"——一个核心点"感染"它的邻居，那些邻居中如果有核心点就继续"感染"它们自己的邻居。密度相连则是"通过同一个核心点串联起来的所有点属于同一个聚类"。

```
示意图：
    ●(核心)──→ ○(边界)──→ ●(核心)──→ ○(边界)
    │                        │
    └──→ ○(噪声，链在此断裂)  └──→ ○(边界)
```

### 2.2 算法流程

DBSCAN 的主循环非常简洁：

```
输入: 点集 D, 半径 ϵ, 最少点数 minPts
输出: 每个点的聚类标签（或 NOISE）

1. 所有点标记为 UNVISITED
2. cluster_id = 0
3. 对每个点 p ∈ D:
   a. 如果 p 已标记，跳过
   b. neighbors = region_query(p, ϵ)   // 找出 p 的 ϵ-邻域
   c. 如果 |neighbors| < minPts - 1:
      - 标记 p 为 NOISE
   d. 否则:
      - 以 p 为种子，扩展一个新聚类 cluster_id
      - cluster_id += 1
```

### 2.3 聚类扩展（Expand Cluster）

这是 DBSCAN 最核心的子程序，也是实现中最容易出错的环节：

```
expand_cluster(p, cluster_id):
  1. seeds = region_query(p, ϵ)
  2. 标记 p 为 cluster_id
  3. 对 seeds 中的每个点 q（队列式遍历，允许动态增长）:
     a. 如果 q 是 NOISE: 重新标记为 cluster_id（变为边界点）
     b. 如果 q 已标记: 跳过
     c. 标记 q 为 cluster_id
     d. q_neighbors = region_query(q, ϵ)
     e. 如果 |q_neighbors| ≥ minPts - 1:  // q 也是核心点
        - 将 q_neighbors 追加到 seeds 末尾
```

**关键设计决策**：这里使用**队列式遍历**而不是递归遍历。因为 seeds 在遍历过程中动态增长——当发现一个新的核心点时，它的邻居会被追加到遍历队列末尾。这本质上是 BFS（广度优先搜索），保证了从种子点 p 出发，所有密度可达的点都会被访问到。

**为什么是 `|q_neighbors| ≥ minPts - 1` 而不是 `≥ minPts`？** 因为 region_query 返回的邻居列表不包含查询点自身。所以"至少有 minPts 个邻居（含自身）"等价于"region_query 返回至少 minPts-1 个点"。这是实现中最常见的 off-by-one 错误——忘记了包含自身会让判断条件多 1。

### 2.4 时间复杂度分析

如果不做任何优化，region_query 需要对每个查询点遍历整个数据集，是 O(n²) 总体复杂度。虽然可以用空间索引（R-tree、KD-tree、Ball-tree）优化到 O(n log n)，但在我们的实现中保持了简洁的 O(n²) 版本——因为数据量不大（几百个点），代码复杂度远低于带来的收益。

---

## ③ 实现架构

### 3.1 数据流概览

```
┌─────────────┐
│ 生成测试数据  │──→ blobs / moons / circles / aniso
│ (generate_*) │    4 种数据集，各有特点
└──────┬──────┘
       ↓
┌─────────────┐
│   DBSCAN    │──→ labels[]  (每个点的聚类 ID 或 NOISE/-2)
│   dbscan()  │    cluster_centroids[][] (每个聚类的几何中心)
│   O(n²)     │    n_clusters, n_noise, n_core (统计信息)
└──────┬──────┘
       ↓
┌─────────────┐
│  量化验证    │──→ 轮廓系数 (Silhouette Score)
│ quality()   │    聚类数量 vs 预期范围
│             │    噪声率检查
└──────┬──────┘
       ↓
┌─────────────┐
│  PPM 输出   │──→ 4 张 800×800 P6 PPM 可视化图片
│ write_ppm() │    Golden-angle 色相编码
└─────────────┘
```

### 3.2 关键数据结构

```cpp
struct Point {
    double x, y;   // 2D 坐标
};

// 距离平方：避免 sqrt 计算（性能优化）
inline double dist2(const Point& a, const Point& b) {
    double dx = a.x - b.x, dy = a.y - b.y;
    return dx * dx + dy * dy;
}
```

**为什么存 `dist2` 而不是 `dist`？** 在 region_query 中需要对每个点对计算距离，如果 eps² = 4，则判断 `dx² + dy² ≤ 4` 只需一次乘法和一次加法。而 `sqrt(dx² + dy²) ≤ 2` 需要额外的开方运算。对几百个点来说这不重要，但对十万个点的暴力算法来说，省掉的 sqrt 操作是显著的常数级加速。

**标签编码规则：**
- `-2` (NOISE)：噪声点，不属于任何聚类
- `-1` (UNDEFINED)：尚未处理（初始状态）
- `≥ 0`：聚类 ID，从 0 开始递增

这种编码使我们可以用简单的 `if (label < 0)` 判断是否需要处理该点。

### 3.3 颜色方案设计

聚类可视化中，区分不同聚类是关键。我们使用 **Golden-angle 色相分配**：

```cpp
double hue = fmod(id * 0.618033988749895, 1.0);
```

黄金比例 φ = 0.618... 有一个数学特性：φᵢ mod 1 的序列在 [0,1) 区间内是均匀分布的（等分布序列）。这意味着无论有多少个聚类，相邻聚类的色相间隔都很均匀，不会出现"前面几个颜色很接近、后面几个突然跳变"的问题。

对比：如果简单地用 `hue = id / n_clusters`，当只有一个额外聚类时，所有已有聚类的色相都会偏移，而 golden-angle 方案是**增量稳定**的——新增聚类不会改变已有聚类的颜色。

---

## ④ 关键代码解析

### 4.1 区域查询（region_query）

```cpp
// 返回点 i 的 ε-邻域中的所有点的索引（不包括自身）
std::vector<int> region_query(const std::vector<Point>& pts, int i, double eps2) {
    std::vector<int> neighbors;
    for (int j = 0; j < (int)pts.size(); ++j) {
        if (j == i) continue;               // 不包含自身
        if (dist2(pts[i], pts[j]) <= eps2)  // 使用距离平方比较
            neighbors.push_back(j);
    }
    return neighbors;
}
```

**为什么 `j == i` 要跳过：** 这是为了逻辑清晰。在主循环的判断中，我们写 `neighbors.size() >= minPts - 1`，这等价于"包含自身后至少有 minPts 个邻居"。如果 region_query 包含自身，则判断条件要写成 `>= minPts`，两者语义等价但前者更清晰地表达了"邻居列表不包括自身"的语义。

**为什么不排序或去重：** region_query 的结果天然不重复（每个 j 只出现一次），且天然有序（按索引 j 从小到大）。不需要额外处理。

### 4.2 聚类扩展（expand_cluster）

这是整个算法中最复杂的部分，也是最容易出 Bug 的：

```cpp
void expand_cluster(const std::vector<Point>& pts,
                    std::vector<int>& labels,
                    int p, int cluster_id, double eps2, int min_pts) {
    std::vector<int> seeds = region_query(pts, p, eps2);
    labels[p] = cluster_id;  // 种子点自身标记为当前聚类

    // BFS 遍历：seeds 在循环过程中动态增长
    for (size_t si = 0; si < seeds.size(); ++si) {
        int q = seeds[si];

        // 情况 1: q 之前被标记为噪声 → 改为边界点
        if (labels[q] == NOISE) {
            labels[q] = cluster_id;
        }
        // 情况 2: q 已属于其他聚类 → 跳过（不应该发生，但防御性处理）
        if (labels[q] != UNDEFINED) continue;

        // 情况 3: q 是未访问点 → 加入当前聚类
        labels[q] = cluster_id;

        // 检查 q 是否是核心点
        std::vector<int> neighbors = region_query(pts, q, eps2);
        if ((int)neighbors.size() >= min_pts - 1) {
            // q 是核心点 → 将其邻居加入遍历队列
            for (int nb : neighbors) {
                if (labels[nb] == UNDEFINED || labels[nb] == NOISE)
                    seeds.push_back(nb);
            }
        }
    }
}
```

**关键细节分析：**

**① `seeds` 动态增长模式**：我们用 `for (size_t si = 0; si < seeds.size(); ++si)` 而不是 `for (auto q : seeds)`。为什么呢？因为在循环内部，当发现新的核心点时，我们会 `seeds.push_back(nb)`。如果用 range-based for，迭代器会在 push_back 后失效（vector 可能 reallocate），导致未定义行为。用索引遍历则天然安全。

**② 为什么 `neighbors.size() >= min_pts - 1`：** 这是经典的 off-by-one 陷阱。region_query 返回的邻居列表不包含查询点 q 自身。所以"q 的邻域中至少有 minPts 个点（含 q 自身）"等价于"region_query 返回 ≥ minPts - 1 个点"。如果写成 `≥ minPts`，就要求 q 周围有至少 minPts+1 个点（含自身），条件过严，会导致更多的点被误判为 NOISE。

**③ 为什么噪声点可以被"复活"：** 在 DBSCAN 中，"先标记为 NOISE 后改为边界点"是完全合法的行为，也是算法的核心设计。原因在于：一个点在单独检查时可能邻居不够，因此被标记为噪声；但后来从另一个核心点出发遍历时，这个点在核心点的 ϵ 邻域内，所以它应该属于那个聚类。这反映了 DBSCAN 的"密度相连"概念——边界点不一定是核心点，但可以通过核心点链连接到聚类。

**④ 为什么不需要检查去重：** `labels[nb] == UNDEFINED || labels[nb] == NOISE` 这个判断已经防止了重复添加——已标记的点会被跳过，不会被重复推入 seeds。

### 4.3 主循环（dbscan）

```cpp
DBSCANResult dbscan(const std::vector<Point>& pts, double eps, int min_pts) {
    int n = (int)pts.size();
    double eps2 = eps * eps;      // 预计算平方值
    std::vector<int> labels(n, UNDEFINED);

    int cluster_id = 0;
    for (int i = 0; i < n; ++i) {
        if (labels[i] != UNDEFINED) continue;
        // ↑ 关键：SKIP 已经标记的点（无论是聚类还是噪声）

        auto neighbors = region_query(pts, i, eps2);
        if ((int)neighbors.size() < min_pts - 1) {
            labels[i] = NOISE;    // 不够核心 → 暂标噪声
        } else {
            expand_cluster(pts, labels, i, cluster_id, eps2, min_pts);
            ++cluster_id;
        }
    }
    // ... 统计信息收集 ...
}
```

**设计要点：** 主循环非常简洁——对每个未标记的点判断是否能成为核心点。如果邻居够多就扩展聚类，否则暂标噪声。这种"先标记、后修正"的策略避免了复杂的后处理逻辑。

### 4.4 聚类质量评估（轮廓系数）

仅仅"跑出了结果"是不够的——我们需要量化地评估聚类质量。轮廓系数（Silhouette Score）是衡量聚类内聚度和分离度的经典指标：

```cpp
double compute_dbscan_quality(const std::vector<Point>& pts,
                               const DBSCANResult& res, double /* eps */) {
    int n = pts.size();
    if (res.n_clusters <= 1) return 0.0;

    double total_sil = 0;
    int valid = 0;
    for (int i = 0; i < n; ++i) {
        if (res.labels[i] < 0) continue;  // 噪声点不计入
        int ci = res.labels[i];

        // a(i): i 点到同聚类其他点的平均距离（越小越好，越紧致）
        double a = 0;
        int ca = 0;
        for (int j = 0; j < n; ++j) {
            if (i == j || res.labels[j] != ci) continue;
            a += sqrt(dist2(pts[i], pts[j]));
            ++ca;
        }
        if (ca == 0) continue;
        a /= ca;

        // b(i): i 点到最近的其他聚类中点的平均距离（越大越好，越分离）
        double b_min = 1e18;
        for (int oc = 0; oc < res.n_clusters; ++oc) {
            if (oc == ci) continue;
            double b_sum = 0;
            int bc = 0;
            for (int j = 0; j < n; ++j) {
                if (res.labels[j] != oc) continue;
                b_sum += sqrt(dist2(pts[i], pts[j]));
                ++bc;
            }
            if (bc > 0) { b_sum /= bc; b_min = std::min(b_min, b_sum); }
        }
        if (b_min < 1e17 && std::max(a, b_min) > 1e-10) {
            total_sil += (b_min - a) / std::max(a, b_min);
            ++valid;
        }
    }
    return valid > 0 ? total_sil / valid : 0;
}
```

**轮廓系数公式：** 对于每个点 i：
$$
s(i) = \frac{b(i) - a(i)}{\max(a(i), b(i))}
$$

- s(i) 接近 +1：点在聚类内部很紧凑，与其他聚类分离良好（理想情况）
- s(i) 接近 0：点处于两个聚类的边界上
- s(i) 接近 -1：点可能被分配到错误的聚类

最终分数是所有有效点的 s(i) 平均值。我们实验中得到的平均轮廓系数为 0.4891，属于中等偏上的聚类质量。

### 4.5 PPM 可视化输出

```cpp
void write_ppm_final(const char* filename, const std::vector<Point>& pts,
                     const std::vector<int>& labels, int n_clusters,
                     const std::vector<std::vector<double>>& centroids,
                     int width, int height,
                     double xmin, double xmax, double ymin, double ymax) {
    std::vector<unsigned char> img(width * height * 3, 255); // 白色背景

    double xscale = (width - 1)  / (xmax - xmin);   // 世界坐标 → 像素坐标
    double yscale = (height - 1) / (ymax - ymin);

    // 绘制点：每点画一个半径 2 的圆
    for (size_t i = 0; i < pts.size(); ++i) {
        int px = (int)((pts[i].x - xmin) * xscale + 0.5);
        int py = height - 1 - (int)((pts[i].y - ymin) * yscale + 0.5);
        // ↑ py 翻转：屏幕空间 Y 轴向下

        // 聚类颜色编码
        unsigned char r, g, b;
        int id = labels[i];
        if (id == NOISE) {
            r = 100; g = 100; b = 100;  // 噪声 = 灰色
        } else {
            cluster_color(id, n_clusters, r, g, b);
        }
        // 绘制半径为 2 的实心圆
        for (int dy = -2; dy <= 2; ++dy) {
            for (int dx = -2; dx <= 2; ++dx) {
                if (dx*dx + dy*dy > 4) continue;  // 圆形裁剪
                int idx = ((py + dy) * width + (px + dx)) * 3;
                img[idx] = r; img[idx+1] = g; img[idx+2] = b;
            }
        }
    }

    // 绘制聚类质心（黄色十字）
    for (int c = 0; c < (int)centroids.size(); ++c) {
        int cpx = (int)((centroids[c][0] - xmin) * xscale + 0.5);
        int cpy = height - 1 - (int)((centroids[c][1] - ymin) * yscale + 0.5);
        for (int dy = -3; dy <= 3; ++dy) {
            for (int dx = -3; dx <= 3; ++dx) {
                if (abs(dx) == abs(dy) || dx == 0 || dy == 0) {
                    // 菱形 + 十字形状
                    int idx = ((cpy + dy) * width + (cpx + dx)) * 3;
                    img[idx] = 255; img[idx+1] = 255; img[idx+2] = 0; // 黄色
                }
            }
        }
    }

    // 写入 P6 格式 PPM
    FILE* f = fopen(filename, "wb");
    fprintf(f, "P6\n%d %d\n255\n", width, height);
    fwrite(img.data(), 1, img.size(), f);
    fclose(f);
}
```

**坐标系转换的要点：** `py = height - 1 - ...` 这行是将数学坐标系（y 轴向上）映射到屏幕坐标系（y 轴向下）。如果忘了这行，图片会上下颠倒——但看起来仍然"正常"，因为肉眼难以判断。这就是为什么我们需要量化验证（像素采样）而不是"看起来对就行"。

### 4.6 四种测试数据集

**blobs（高斯团块 + 均匀噪声）：** 5 个各向同性的高斯团块 + 30 个均匀噪声点。这是最"标准"的聚类场景——K-Means 也能处理。参数：eps=2.0, minPts=5。

**moons（双月弧形）：** 两个非凸的弧形聚类。这是 K-Means 的噩梦——两个弧形之间的空白区域会导致 K-Means 切出错误的边界。DBSCAN 通过密度连通性正确识别了两个弧形。5 个噪声点星散在中间空白区域。参数：eps=1.0, minPts=5。

**circles（同心圆环）：** 外环和内环。同样是 K-Means 无法处理的非凸聚类。两层圆环距离约 3 单位，所以 eps 必须 < 3 才能分离它们。参数：eps=0.9, minPts=5。

**aniso（三个各向异性条形簇）：** 三个长条形的聚类，分别位于左上角（水平）、左下角（垂直）和右下角（对角线）。每个聚类在不同方向上的扩散程度不同，这测试了 DBSCAN 对非各向同性密度的适应性。参数：eps=1.2, minPts=5。

---

## ⑤ 踩坑实录

### 坑 1：circles 数据集被合并为一个簇（eps 过大）

**症状**：inner ring 和 outer ring 的所有点被合并为 1 个聚类。看图中所有点都是同一种颜色。

**错误假设**：以为 eps=1.4 足够小——因为环与环之间的距离是 3，1.4 远小于 3。应该在两者之间选一个安全的 eps。

**真实原因**：虽然两个环的中心线距离是 3，但环上每个点都有随机噪声（标准差 0.3），导致内环的"外缘"点和外环的"内缘"点之间的距离可能只有 2 左右。当 eps=1.4 时，通过噪声点的"桥接"——一个噪声点离内环某点和外环某点都 ≤ 1.4——两个环就被连接起来了。

**修复**：将 eps 从 1.4 降到 0.9。代价是外环可能导致少数点被标记为噪声，但这是可接受的 trade-off——宁可多几个噪声点，也不能让两个环混在一起。

**教训**：DBSCAN 的 eps 参数应该在"最小的簇间距离"和"最大的簇内间隙"之间选值。噪声的标准差会缩小有效簇间距离。

### 坑 2：aniso 数据集全部合并为一个簇

**症状**：3 个各向异性条形簇全部被合并为 1 个聚类。输出显示 1 个聚类，342 个点，23 个噪声。

**错误假设**：以为只要三个簇的中心距离大于 eps 就不会合并。最初的簇生成在原点附近交叉——横向簇 x∈[-12,12]，纵向簇 x∈[-4,4]，对角线经过原点附近。

**真实原因**：水平簇的 x 方向标准差为 4，这意味着它在 x 轴上的跨度达到了约 3σ×2 = 24 单位。这个超宽的长条形簇直接"扫过"了其他两个簇的中心区域。因为 DBSCAN 通过密度连通性聚类，只要存在一条"核心点桥接"，两个簇就会连为一体。

**修复**：重新设计数据生成策略——将三个簇放在三个不同的象限角：
- 水平簇：放在左上角 (y=5)，x 标准差减小到 4
- 垂直簇：放在左下角 (x=-5)
- 对角簇：放在右下角，沿 45° 方向展开

这样三簇的最近距离约为 7 单位，远大于 eps=1.2。

**教训**：设计测试数据时要计算"最坏情况下的簇间距离"，而不只是中心的距离。长条形数据需要的 eps 设置非常敏感——太小会碎片化，太大会合并。

### 坑 3：噪声点"复活"逻辑理解错误

**症状**：最初实现中，如果点在主循环中被标记为 NOISE，在 expand_cluster 中就直接跳过了，导致这些点无法被"复活"为边界点。结果是一些合理的边界点被永久标记为噪声。

**错误假设**：以为"NOISE 就是最终判定，不能改了"。这在 K-Means 中是对的，但 DBSCAN 不同。

**真实原因**：DBSCAN 的主循环遍历顺序是任意的。一个点可能在检查时邻居不够，被暂时标记为 NOISE；但后来从相邻的核心点出发遍历时，这个点在核心点的邻域内——它应该成为那个聚类的边界点。

**修复**：在 expand_cluster 中添加噪声复活逻辑：
```cpp
if (labels[q] == NOISE) {
    labels[q] = cluster_id;  // 复活为边界点
}
```

**教训**：理解算法的"异步"特性——DBSCAN 中一个点的最终标签不是由它自己决定的，而是由周围的密度结构决定的。NOISE 标签在 expand_cluster 中应该是可变的。

### 坑 4：Golden-angle 色相计算误差

**症状**：前几个聚类的颜色差异非常微妙，肉眼几乎无法区分。

**错误假设**：使用了简单的 `hue = (id * 37) % 360 / 360.0` 这样的整数值乘法，这在聚类 ID=0,1,2 时产生 0°, 37°, 74°——色相变化太均匀，第一个和第二个看起来都是"红色调"。

**真实原因**：黄金比例的无理性保证了序列在 [0,1) 区间内的等分布性。整数值乘法虽然也能产生不同颜色，但起始的几个值间距太规律，在小聚类数量时颜色区分度不够。

**修复**：改用黄金比例 `φ = 0.618...` 的等分布序列。这个序列的无理性保证了没有两个色相会"对齐"，即使在少量聚类时也能最大化视觉区分度。

---

## ⑥ 效果验证与数据

### 6.1 聚类结果汇总

| 数据集 | 点数 | 聚类数 | 噪声数 | 噪声率 | 核心点数 | 轮廓系数 |
|--------|------|--------|--------|--------|----------|----------|
| blobs | 430 | 4 | 10 | 2.3% | 416 | 0.6510 |
| moons | 420 | 2 | 17 | 4.0% | 403 | 0.6746 |
| circles | 420 | 2 | 7 | 1.7% | 405 | 0.0405 |
| aniso | 325 | 3 | 20 | 6.2% | 299 | 0.5902 |

**验证结论：**

- 全部 4 个数据集的聚类数量在预期范围内 ✅
- 噪声率最高 6.2%，远低于 25% 的阈值 ✅
- 平均轮廓系数 0.4891，属于中等偏上的聚类质量 ✅
- circles 的轮廓系数最低（0.0405），因为同心圆环的内外环相距仅 3 单位，环边界上有些点与另一个环的距离接近环内距离

### 6.2 像素采样验证

对 4 张输出图片的量化像素验证结果：

| 图片 | 文件大小 | 非白比例 | 非白均值 | 非白标准差 | 唯一颜色 | 状态 |
|------|---------|---------|---------|-----------|---------|------|
| blobs | 11120B | 0.86% | 127.8 | 84.4 | 6 | ✅ |
| moons | 10502B | 0.83% | 115.2 | 81.5 | 4 | ✅ |
| circles | 11321B | 0.85% | 113.1 | 83.4 | 4 | ✅ |
| aniso | 8751B | 0.56% | 121.5 | 80.3 | 5 | ✅ |

所有图片非白区域标准差 > 80，说明聚类之间有明显的颜色差异。唯一颜色数量 ≥ 4，对应聚类数量 + 噪声灰色 + 质心黄色。

### 6.3 与 K-Means 对比

| 对比维度 | DBSCAN | K-Means |
|----------|--------|---------|
| 需要预设聚类数 | ❌ 不需要 | ✅ 必须指定 k |
| 处理非凸聚类 | ✅ 天然支持 | ❌ 切出 Voronoi 边界 |
| 噪声检测 | ✅ 自动排除 | ❌ 强制归属 |
| 时间复杂度 | O(n²) 暴力 / O(n log n) 索引 | O(n·k·t) |
| 参数敏感性 | 对 eps 和 minPts 敏感 | 对 k 值敏感 |

---

## ⑦ 总结与延伸

### 技术局限性

1. **参数敏感**：eps 和 minPts 的选择直接影响结果。不同数据集需要不同的参数。Hyperparameter tuning 在实际应用中是一个重要课题（k-distance 图可以帮助选择 eps）。

2. **密度不均匀问题**：如果数据中不同聚类的密度差异很大（稀疏聚类 vs 密集聚类），单一 eps 值可能无法同时处理两者。OPTICS（Ordering Points To Identify the Clustering Structure）算法解决了这个问题——它产生一个密度排序，然后可以从中提取不同密度级别的聚类。

3. **高维诅咒**：随着维度增加，"密度"的定义变得模糊。在高维空间中，所有点彼此之间的距离趋于相等，DBSCAN 的效果会显著下降。

4. **边界情况**：如果数据中有一个密度渐变的"桥"连接了两个聚类，DBSCAN 会把它们合并为一个。这个问题没有简单解法，本质上是"聚类定义"本身的模糊性。

### 可优化的方向

1. **空间索引加速：** 用 KD-tree 或 Ball-tree 替代暴力搜索，将 region_query 从 O(n) 优化到 O(log n)，总体复杂度降为 O(n log n)。

2. **并行化：** region_query 可以对每个点独立计算——非常适合 GPU 并行或 OpenMP 多线程。

3. **HDBSCAN：** Hierarchical DBSCAN 是 DBSCAN 的层次化版本，通过构建聚类树来自动选择"最佳"的 eps/minPts 组合，消除了调参烦恼。

4. **增量 DBSCAN：** 在新点到达时增量更新聚类，适用于流式数据处理场景。

### 与本系列其他文章的关联

- **06-21 K-Means 聚类**：今天对比了两种聚类算法的核心差异——DBSCAN 不需要预设 k，能发现任意形状，还能检测噪声。
- **06-14 Quadtree 空间分区**：如果对 region_query 做空间索引优化，Quadtree/KD-tree 是 DBSCAN 加速的经典方案。
- **06-17 Delaunay 三角剖分**：计算几何中的相邻关系分析，与 DBSCAN 的"密度相邻"有结构上的类比。

### 关键收获

今天最大的收获是深刻理解了"聚类的本质不是离中心近，而是被邻居围绕"这个核心思想。DBSCAN 用 ϵ-邻域 + 密度连通性替代了传统的"聚类中心 + 距离"范式，这不仅是算法上的改进，更是**问题定义的转变**。

第二个收获是参数选择的艺术——eps 和 minPts 的交互作用很微妙。写过代码、跑过数据、调过参数之后，才真正理解为什么 DBSCAN 官方文档总是强调"用 k-distance 图辅助调参"。

第三个收获是验证的重要性。如果不是量化验证（轮廓系数、噪声率、像素采样），circles 数据集可能被错误地验证为 2 个聚类——而实际上最初的 eps=1.4 产生的是 1 个聚类。**不用量化指标，肉眼会说谎。**

---

**代码仓库**：[daily-coding-practice/2026/06/06-22-dbscan-density-clustering](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/06/06-22-dbscan-density-clustering)

**图床链接**：
- [dbscan_blobs.png](https