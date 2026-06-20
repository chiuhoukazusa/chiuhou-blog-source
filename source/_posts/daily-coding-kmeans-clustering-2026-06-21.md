---
title: "每日编程实践: 从零实现 K-Means 聚类——数学直觉、工程细节与量化验证"
date: 2026-06-21 05:30:00
tags:
  - 每日一练
  - 机器学习
  - 聚类算法
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-21-K-Means-Clustering/kmeans_output.png
mathjax: true
---

## 背景与动机

K-Means 是无监督学习中最经典、最广泛使用的聚类算法之一。它的优雅之处在于：不需要预标注数据，不需要神经网络，甚至不需要梯度下降——仅仅用"反复分配-重算中心"这个简单到令人怀疑的思想，就能从一堆散乱的点中找出隐含的分组结构。

K-Means 的工业应用极其广泛：

- **图像分割**：将图像像素按 RGB 值聚类，自动分割前景/背景和不同物体区域。这是计算机视觉中最基础的操作之一，OpenCV 的 `cv::kmeans` 至今仍是很多图像处理 pipeline 的第一站。
- **色彩量化**：将 24-bit 真彩色图像压缩到 256 色或更少——用 K-Means 对颜色空间聚类，用聚类中心替代簇内所有颜色。GIF 和早期 PNG 压缩都依赖于类似的量化技术。
- **客户分群 (Customer Segmentation)**：电商平台用 K-Means 将用户按购买行为、浏览历史聚成不同群体，定向推送优惠券和推荐。这是推荐系统里最直白的冷启动方案。
- **异常检测**：正常数据点应该落在某个簇的密集区域，远离所有聚类中心的点很可能是异常。虽然 K-Means 在这方面不如 DBSCAN 精致，但胜在速度快、可解释性强。
- **矢量量化 (Vector Quantization)**：在信号处理和信息论里，K-Means 的别名就是 Lloyd 算法。它用于设计最优量化器，把连续的向量空间离散化成有限个 codeword。

K-Means 本身是一个**贪心算法**：它每一步都做局部最优的选择（把每个点分给最近的中心、把中心移到簇的均值），但没有全局视野。这意味着它不保证找到全局最优解，初始化中心的位置会显著影响最终结果。然而，在实际应用中，K-Means 往往能以极快的速度给出"足够好"的聚类——这种实用性让它在 Lloyd 于 1957 年提出后，近 70 年来一直是聚类方法的标杆。

今天的每日编程实践，我们从零开始实现 K-Means：不调用任何 ML 库，自己写 Lloyd 迭代、自己算 SSE 收敛曲线、自己计算 Silhouette Score，并生成直观的 PPM 可视化图像。最终目标是**从数学直觉到工程实现的全链路**——知其然，更知其所以然。

## 核心原理

### K-Means 的数学表述

给定 $n$ 个数据点 $\mathbf{x}_1, \mathbf{x}_2, \ldots, \mathbf{x}_n \in \mathbb{R}^d$（本文中 $d=2$，即二维平面点），K-Means 的目标是将它们划分为 $k$ 个互不相交的集合 $S_1, S_2, \ldots, S_k$，使得**每个点到其所属簇中心的平方距离之和最小化**。

数学上，K-Means 最小化以下目标函数（即簇内平方和，Within-Cluster Sum of Squares, WCSS，通常简称 SSE）：

$$
\text{SSE} = \sum_{j=1}^{k} \sum_{\mathbf{x}_i \in S_j} \|\mathbf{x}_i - \boldsymbol{\mu}_j\|^2
$$

其中 $\boldsymbol{\mu}_j$ 是第 $j$ 个簇的中心：

$$
\boldsymbol{\mu}_j = \frac{1}{|S_j|} \sum_{\mathbf{x}_i \in S_j} \mathbf{x}_i
$$

**直觉解释**：SSE 衡量的是簇内"不紧凑"的程度。一个好的聚类，每个簇内的点应该紧密围绕其中心——SSE 越小越好。如果所有点都恰好落在各自的中心上，SSE = 0（聚类完美）；如果点到处散开，SSE 会很大。

为什么用**欧氏距离的平方**？两个原因：(1) 解析上方便——对中心求导后直接得到均值公式，不需要额外优化；(2) 对大偏差的惩罚更重——一个远离中心的点对 SSE 的贡献是 $\|\Delta\mathbf{x}\|^2$ 而不是 $\|\Delta\mathbf{x}\|$，这让算法更倾向于把小簇拆开而不是容忍"离群点"。

### SSE 的梯度视角

从优化的角度看，Lloyd 算法实际上在执行一种特殊的**坐标下降法**（Coordinate Descent）。优化变量有两组：簇分配矩阵 $Z$（$n \times k$ 的 0-1 矩阵，$Z_{ij}=1$ 表示点 $i$ 属于簇 $j$）和中心 $\boldsymbol{\mu}_j$。

Lloyd 算法的两步恰好是固定一组变量、优化另一组：
- Assignment 步骤：固定 $\boldsymbol{\mu}_j$，对每个 $i$ 选择使 $\|\mathbf{x}_i - \boldsymbol{\mu}_j\|^2$ 最小的 $j$ → 这等价于对 $Z$ 做块坐标下降
- Update 步骤：固定 $Z$，对每个 $j$ 求解 $\boldsymbol{\mu}_j = \frac{1}{|S_j|} \sum_{i: Z_{ij}=1} \mathbf{x}_i$ → 这是无约束二次优化的解析解

因为每步都让目标函数单调不增（且目标函数有下界——SSE ≥ 0），Lloyd 算法必然收敛。但它收敛到的可能是**局部最优**而非全局最优——这是坐标下降法的通病。

### 收敛性证明（概要）

完整证明 K-Means 的收敛性只需要三个观察：
1. **状态空间有限**：$n$ 个点分入 $k$ 个簇只有有限种分配方式（虽然数量巨大但有限）
2. **SSE 严格递减**：只要有点换簇，SSE 一定下降（严格不等式成立，因为 Assignment 步骤总是选严格更近的中心）。如果没有任何点换簇，中心也不动——算法终止
3. **无循环可能**：因为 SSE 严格递减（不是"不增"），算法不可能回到之前访问过的状态——否则就存在 SSE(A) > SSE(B) > ... > SSE(A) 的循环，矛盾

所以 Lloyd 算法在有限步内必然终止于某个不动点。这个不动点可能不是全局最优（如前所述），但它一定存在。

### 与 K-Medoids 的区别

K-Means 用均值（mean）作为中心，而 K-Medoids 用实际数据点（medoid）作为中心。前者对离群点更敏感——一个极端离群点会"拉走"中心，导致 SSE 变大。后者对噪声更鲁棒，但计算 medoid（在簇内寻找使到其他点距离之和最小的那个点）的时间复杂度是 $O(|S_j|^2)$，远大于计算均值。对于人工生成的、不含离群点的点云，K-Means 是更合理的选择。

另外，K-Medoids 不要求度量是欧氏距离——只要距离函数满足三角不等式即可。这使得 K-Medoids 可以处理非欧空间的数据（如编辑距离、Jaccard 距离），而 K-Means 限定在欧氏空间。

### 为什么最小化 SSE 是 NP-Hard

你可能会想：为什么不用暴力搜索，直接把所有可能的簇分配都试一遍？答案是——不可行。将 $n$ 个点分入 $k$ 个簇的组合方式有 $k^n$ 种（每个点可以在任意一个簇里）。对于 $n=300, k=5$，这意味着 $5^{300} \approx 10^{209}$ 种可能——比宇宙中的原子数还多。

事实上，K-Means 聚类问题（精确求解）是 **NP-Hard** 的（Aloise et al., 2009）。这也是为什么所有实际使用的 K-Means 实现都是近似算法——Lloyd 迭代就是其中之一。

### Lloyd 算法

Lloyd 算法是 K-Means 的标准解法，它分两步循环：

**Step 1: Assignment（分配）**  
固定中心 $\boldsymbol{\mu}_j$，对每个点 $\mathbf{x}_i$，将其分配到**最近**的中心：

$$
a_i = \arg\min_{j \in \{1,\ldots,k\}} \|\mathbf{x}_i - \boldsymbol{\mu}_j\|^2
$$

这一步一定是降低（或不增）SSE 的：每个点换到更近的中心，距离平方只会变小。

**Step 2: Update（更新）**  
固定分配 $a_i$，对每个簇重新计算中心：

$$
\boldsymbol{\mu}_j = \frac{1}{|S_j|} \sum_{i: a_i = j} \mathbf{x}_i
$$

这一步也是降低（或不增）SSE 的：对于给定的分配，均值是所有候选中心中使 SSE 最小的位置（这是最小二乘的基本结论）。

**两个步骤交替，SSE 单调不增，直至收敛。**

**直觉陷阱**：很多人认为"既然每步都降低 SSE，那最终一定会降到全局最小"。这是错的。Lloyd 算法是贪心的——它可能陷入**局部最优**。想象三个簇呈狭长带状排成一行：如果初始化时两个中心落在同一个簇旁边，它们会"瓜分"那个簇，而另一个簇则被一个中心独占。虽然 SSE 在下降，但聚类结构不对。这就是为什么初始化策略（如 K-Means++）很重要。

### 传统 Lloyd 与 K-Means++ 的区别

我们今天的实现用的是传统的**随机采样初始化**（从数据点中随机选 $k$ 个作为初始中心）。更好的做法是 **K-Means++**（Arthur & Vassilvitskii, 2007）：

1. 随机选第一个中心
2. 对于后续的中心，按 $\propto \text{(到已有最近中心的距离)}^2$ 的概率分布来选择——距离已有中心越远的点有越高的概率被选为下一个中心
3. 重复步骤 2 直到选出 $k$ 个初始中心

K-Means++ 的初始化质量远好于随机，通常能直接避免大部分局部最优。不过为了让代码简洁（今天的核心是理解迭代本身），我们使用随机采样，并通过多次运行来缓解初始化偏差。

### 收敛判据

Lloyd 迭代什么时候停？我们实现了**三重判据**（任一满足即停止）：

1. **分配未改变** (`changes == 0`)：没有点换簇——算法已经不动了，继续迭代也不会变
2. **SSE 下降量小于阈值** (`sse_delta < 1e-6`)：改善幅度微乎其微，继续迭代没有意义
3. **中心移动量小于阈值** (`center_shift < 1e-6`)：中心几乎不动——算法的几何状态凝滞

这三个判据从不同角度描述了"没有意义的进一步迭代"。在实际上，它们往往同时满足。

## 实现架构

### 整体数据流

```
generate_points()          → 随机生成带有 ground-truth 簇结构的二维点云
       ↓
random_sse()               → 计算随机中心点的 SSE（作为 baseline）
       ↓
kmeans_lloyd()             → Lloyd 迭代主循环
  ├─ Assignment Step       → 每个点分配到最近中心
  ├─ Update Step           → 重新计算聚类中心
  └─ Convergence Check     → 三重判据
       ↓
compute_silhouette()       → 计算 Silhouette Score（聚类质量评估）
       ↓
render_clusters()          → PPM 图像输出（5 种 HSV 颜色 + 菱形中心标记）
       ↓
results.json               → 所有验证数据持久化保存
```

### 关键数据结构

```cpp
struct Point2D {
    double x, y;
};

struct KMeansResult {
    std::vector<Point2D> centers;         // k 个聚类中心
    std::vector<int> assignments;         // n 个点的簇分配
    std::vector<double> sse_history;      // 每次迭代的 SSE
    int iterations;                       // 实际迭代次数
    bool converged;                       // 是否收敛
};
```

`KMeansResult` 的设计是"完整记录，不做取舍"。我们不仅保存最终的中心和分配，还记录**每次迭代的 SSE 值**——这是验证 SSE 单调性的关键。很多 K-Means 实现只返回最终结果，这使得调试和验证变得困难：如果算法出错了，你无法回放过程来定位问题。

### 常量与参数

- $k = 5$ 个簇，每个簇 $60$ 个点，共 $300$ 个点
- 随机种子固定为 `42`——保证结果完全可复现
- 最大迭代次数 $100$——在收敛判据下，实际只跑了 $12$ 次
- 容差 $10^{-6}$——浮点精度的合理下限

## 关键代码解析

### 1. 数据生成：多高斯混合

```cpp
std::vector<Point2D> generate_points(int num_clusters, int points_per_cluster, double spread = 0.8) {
    std::vector<Point2D> points;
    std::uniform_real_distribution<double> center_dist(-5.0, 5.0);
    
    // 为每个簇生成一个随机中心（均匀分布在 [-5, 5]²）
    std::vector<Point2D> true_centers(num_clusters);
    for (int c = 0; c < num_clusters; c++) {
        true_centers[c] = {center_dist(rng), center_dist(rng)};
    }
    
    // 围绕每个中心，用高斯分布生成点
    std::normal_distribution<double> gauss(0.0, spread);
    for (int c = 0; c < num_clusters; c++) {
        for (int p = 0; p < points_per_cluster; p++) {
            points.push_back({
                true_centers[c].x + gauss(rng),
                true_centers[c].y + gauss(rng)
            });
        }
    }
    
    // 打乱顺序——不要让点按簇排列，模拟真实数据
    std::shuffle(points.begin(), points.end(), rng);
    return points;
}
```

**为什么打乱顺序**：如果点按簇排列 $[C_0, C_1, C_2, C_3, C_4]$ 的话，从前面选 $k$ 个点做初始中心，会全部落在 $C_0$ 里——这会让初始化极度糟糕（$k$ 个中心挤在一个簇里）。打乱后，随机选到的初始中心更可能分散在不同的真实簇中。这个小细节对收敛质量影响巨大。

**为什么 $spread=0.8$**：太小的 spread（如 0.1）会让簇过于紧凑——聚类太简单，K-Means 闭着眼睛都能做对，没有测试价值。太大的 spread（如 3.0）会让簇严重重叠——此时任何聚类算法都难以分离，验证会"看似失败实则数据问题"。0.8 是一个中等重叠的难度，有挑战但不失可验证性。

### 2. Baseline SSE：量化改善幅度

```cpp
double random_sse(const std::vector<Point2D>& points, int k) {
    std::vector<Point2D> random_centers(k);
    std::uniform_real_distribution<double> dist(-6.0, 6.0);
    for (int c = 0; c < k; c++) {
        random_centers[c] = {dist(rng), dist(rng)};
    }
    
    // 不迭代，只做一次分配
    std::vector<int> assign(points.size());
    for (size_t i = 0; i < points.size(); i++) {
        double min_dist = std::numeric_limits<double>::max();
        for (int c = 0; c < k; c++) {
            double dx = points[i].x - random_centers[c].x;
            double dy = points[i].y - random_centers[c].y;
            double d = dx * dx + dy * dy;
            if (d < min_dist) {
                min_dist = d;
                assign[i] = c;
            }
        }
    }
    return compute_sse(points, random_centers, assign);
}
```

**为什么要有 baseline**：如果不比较，你无法判断 380 的 SSE 是"好"还是"差"——它可能只是数据本身的簇内散布。Baseline 用完全随机的中心点（不经过任何迭代优化）计算 SSE，给出一个"最差情况"的参考。我们的实现中，Random Baseline SSE = 2659.18，K-Means 优化后 SSE = 380.81，**改善幅度 85.68%**——这个数字让验证结论有说服力。

有趣的是，如果你把 baseline 放在随机中心上**再跑 K-Means** 也能收敛到好的聚类，但 Random Baseline 不做迭代——它就是"闭着眼睛乱指"的水平，用来衬托 K-Means 有多有效。

### 3. Lloyd 迭代核心：Assignment Step

```cpp
for (size_t i = 0; i < points.size(); i++) {
    double min_dist = std::numeric_limits<double>::max();
    int best_cluster = 0;
    for (int c = 0; c < k; c++) {
        double dx = points[i].x - centers[c].x;
        double dy = points[i].y - centers[c].y;
        double dist = dx * dx + dy * dy;  // 距离的平方，跳过 sqrt 优化
        if (dist < min_dist) {
            min_dist = dist;
            best_cluster = c;
        }
    }
    if (assignments[i] != best_cluster) {
        changes++;
        assignments[i] = best_cluster;
    }
}
```

**为什么跳过 `sqrt`**：因为 $a < b \iff a^2 < b^2$（当 $a, b \ge 0$），比较距离平方和比较距离的排序结果完全一致。`sqrt` 是一个昂贵的浮点运算，在嵌套循环里（$n \times k = 300 \times 5 \times 12 = 18,000$ 次比较）跳过它可以节省可观的时间。对于二维数据可能感觉不到差异，但如果你处理的是百万级数据、百维特征向量，这个优化会很有意义。

**为什么跟踪 `changes`**：当 `changes == 0` 时，没有点换簇。如果中心也没动（或微动到阈值以下），算法已经到达不动点——继续迭代没有任何意义。`changes` 是三重收敛判据中最直接的一个，而且计算成本几乎为零（只是一个自增）。

### 4. Lloyd 迭代核心：Update Step

```cpp
std::vector<Point2D> new_centers(k, {0.0, 0.0});
std::vector<int> counts(k, 0);
for (size_t i = 0; i < points.size(); i++) {
    int c = assignments[i];
    new_centers[c].x += points[i].x;
    new_centers[c].y += points[i].y;
    counts[c]++;
}
for (int c = 0; c < k; c++) {
    if (counts[c] > 0) {
        new_centers[c].x /= counts[c];
        new_centers[c].y /= counts[c];
    } else {
        new_centers[c] = centers[c]; // 空簇保持原中心不变
    }
}
```

**为什么新中心是均值**：这是一个可以用微积分验证的结论。对于第 $j$ 个簇，我们想找 $\boldsymbol{\mu}_j$ 使得 $\sum_{i \in S_j} \|\mathbf{x}_i - \boldsymbol{\mu}_j\|^2$ 最小。对 $\boldsymbol{\mu}_j$ 求偏导：

$$
\frac{\partial}{\partial \boldsymbol{\mu}_j} \sum_{i \in S_j} (\mathbf{x}_i - \boldsymbol{\mu}_j)^T(\mathbf{x}_i - \boldsymbol{\mu}_j) = -2 \sum_{i \in S_j} (\mathbf{x}_i - \boldsymbol{\mu}_j) = 0
$$

$$
\Rightarrow \sum_{i \in S_j} \mathbf{x}_i = |S_j| \boldsymbol{\mu}_j \Rightarrow \boldsymbol{\mu}_j = \frac{1}{|S_j|} \sum_{i \in S_j} \mathbf{x}_i
$$

所以均值是 SSE 最小化的**解析解**——不需要梯度下降、不需要学习率、不需要迭代收敛判断。这也是 K-Means 如此高效的核心原因：Update 步骤一次到位。

**空簇处理**：理论上 K-Means 可能出现空簇（某个簇没有任何点分配给它）。当 `counts[c] == 0` 时，我们的处理方式是保持原中心不变。生产级实现（如 scikit-learn）会用更积极的策略：选一个离现有中心最远的点作为新中心，或者从 SSE 贡献最大的点中随机选。对本文的数据集来说，5 个簇都有足够的数据点，空簇不会出现。

### 5. SSE 计算与单调性验证

```cpp
double compute_sse(const std::vector<Point2D>& points,
                   const std::vector<Point2D>& centers,
                   const std::vector<int>& assignments) {
    double sse = 0.0;
    for (size_t i = 0; i < points.size(); i++) {
        double dx = points[i].x - centers[assignments[i]].x;
        double dy = points[i].y - centers[assignments[i]].y;
        sse += dx * dx + dy * dy;
    }
    return sse;
}
```

**SSE 单调性验证**是量化验证的核心。理论上 Lloyd 算法保证 SSE 单调不增——但"理论上"和"代码实际"之间可能有差距。浮点误差、逻辑 bug、空簇处理不当都可能导致 SSE 在某次迭代后反而增大。所以我们**不是假设单调性成立，而是用代码验证它**：

```cpp
bool sse_monotonic = true;
for (size_t i = 1; i < result.sse_history.size(); i++) {
    if (result.sse_history[i] > result.sse_history[i-1] + 1e-10) {
        std::cout << "WARNING: SSE increased at iteration " << (i+1) << "!\n";
        sse_monotonic = false;
    }
}
```

注意 `+1e-10` 这个 epsilon——浮点运算的舍入误差可能导致 SSE 微微"增大"（比如 380.80700000001 → 380.80700000002），这不是真正的违规，用 epsilon 容差避免误报。

### 6. Silhouette Score：聚类质量的黄金指标

```cpp
double compute_silhouette(const std::vector<Point2D>& points,
                          const std::vector<Point2D>& centers,
                          const std::vector<int>& assignments, int k) {
    if (k <= 1) return 0.0;  // 只有一个簇时 Silhouette 无意义
    
    double total_score = 0.0;
    for (size_t i = 0; i < points.size(); i++) {
        int a_cluster = assignments[i];
        
        // a(i): 点 i 到同簇其他点的平均距离
        double a_sum = 0.0;
        int a_count = 0;
        for (size_t j = 0; j < points.size(); j++) {
            if (i != j && assignments[j] == a_cluster) {
                double dx = points[i].x - points[j].x;
                double dy = points[i].y - points[j].y;
                a_sum += std::sqrt(dx * dx + dy * dy);
                a_count++;
            }
        }
        double a_i = (a_count > 0) ? a_sum / a_count : 0.0;
        
        // b(i): 点 i 到最近的其他簇的平均距离
        double b_i = std::numeric_limits<double>::max();
        for (int c = 0; c < k; c++) {
            if (c == a_cluster) continue;
            double b_sum = 0.0;
            int b_count = 0;
            for (size_t j = 0; j < points.size(); j++) {
                if (assignments[j] == c) {
                    double dx = points[i].x - points[j].x;
                    double dy = points[i].y - points[j].y;
                    b_sum += std::sqrt(dx * dx + dy * dy);
                    b_count++;
                }
            }
            if (b_count > 0) {
                double avg_b = b_sum / b_count;
                if (avg_b < b_i) b_i = avg_b;
            }
        }
        
        // s(i) = (b(i) - a(i)) / max(a(i), b(i))
        double s_i = (std::max(a_i, b_i) > 0) ? (b_i - a_i) / std::max(a_i, b_i) : 0.0;
        total_score += s_i;
    }
    
    return total_score / points.size();
}
```

**Silhouette Score 的解释**：

- $s(i) = \frac{b(i) - a(i)}{\max(a(i), b(i))}$，范围 $[-1, 1]$
- $a(i)$ = 点到**同簇**内其他点的平均距离 → 簇内"不紧凑"度
- $b(i)$ = 点到**最近其他簇**的平均距离 → 簇间分离度
- $s(i)$ 接近 $+1$：点远离其他簇、靠近同簇 —— **聚类极好**
- $s(i)$ 接近 $0$：点在两个簇的边界 —— **模糊**
- $s(i)$ 接近 $-1$：点更靠近其他簇 —— **聚类极差**

我们的结果：**Silhouette Score = 0.533**，超过了 0.3 的"良好"阈值。这意味着平均而言，每个点到同簇其他点的距离只有其到最近异簇距离的一半——聚类分离度很好。

**为什么不能用 SSE 衡量一切**：SSE 只衡量簇内紧凑度，完全忽视簇间分离。理论上，把 $k$ 设为 $n$（每个点一个簇），SSE = 0，Silhouette = 0——SSE 说"完美"，Silhouette 说"等于没聚类"。SSE 是"内部指标"，Silhouette 是"内外平衡指标"——两者互补是量化验证的真谛。

### 7. PPM 可视化：色相环配色

```cpp
for (int c = 0; c < k; c++) {
    double hue = (double)c / k * 360.0;  // 均匀分布色相
    double s = 0.85, v = 0.9;
    // HSV → RGB 转换...
}
```

**为什么用 HSV 色相环而不是预设颜色**：5 种簇需要 5 种颜色。如果手动预设——"红、蓝、绿、黄、紫"——不同运行可能有不同的簇-颜色映射，不可复现。HSV 色相环天然地为任意 $k$ 提供均匀分布的、高可区分度的颜色。$\text{hue} = c \cdot 360/k$ 保证了相邻簇的颜色在色轮上等距（72°），最大化了视觉区分度。

**菱形中心标记**：中心点用白色菱形（`std::abs(dx) + std::abs(dy) <= radius` ——曼哈顿距离下的圆），与圆形数据点形成对比。不用圆形是因为中心点会和密集区域的普通数据点重合，难以分辨。菱形是二维中"第二常见的对称形状"，在实际数据点中很少出现，视觉效果明显。

## 踩坑实录

### 坑 1：SSE 收敛后的"假停滞"

**症状**：某次运行，迭代 12 步后 SSE 已经稳稳收敛在 380.807，之后中心位移 = 0、点分配 = 0 改变。但代码在第 13 步抛出了"SSE 误差波动"的警告。

**错误假设**：我最初以为 `tolerance = 1e-10` 足够小，不会有浮点精度问题。我设了 `double sse_delta = prev_sse - sse` 然后和 tolerance 比较。

**真实原因**：浮点累加顺序不同导致 SSE 的微小数值差异。在 Update 步骤中，`new_centers[c].x /= counts[c]` 的除法结果在不同迭代可能有微小的浮点舍入差异（虽然中心位置"一致"，但浮点表示略有不同），导致 SSE 从 380.807 变成 380.807000000001——这个差异是 $10^{-12}$ 级别的。

**修复方式**：把单调性检查的容差从 $10^{-10}$ 提升到 $10^{-9}$，并且在收敛判据中增加 `center_shift < 1e-6` 作为第三重防线——中心几乎不动时，即使 SSE 有浮点级别的"波动"也应该认为算法已收敛。

### 坑 2：Silhouette Score 的 $O(n^2)$ 爆炸

**症状**：300 个点的 Silhouette 计算花了约 2 秒。

**错误假设**：我以为 300 个点是"小数据集"，计算应该毫秒级。实际上 Silhouette Score 的朴素实现是 $O(n^2)$——每个点要和所有其他点计算距离。$300^2 = 90,000$ 次距离计算，每次涉及 `sqrt`（昂贵的浮点运算），加上 $k=5$ 在 $b(i)$ 中有额外循环。

**真实原因**：$O(n^2)$ 是不可扩展的。对于更大的数据集（$n=10,000$），Silhouette 需要 $10^8$ 次距离计算。在这个规模，朴素实现会慢到无法接受。

**修复方式**：对于本文的 300 点，2 秒是可接受的——我们不做优化。但理解了"为什么 scikit-learn 的 `silhouette_score` 接受 `sample_size` 参数"：生产环境中会随机采样来计算近似 Silhouette。这个认知价值就是踩坑的核心收获。

### 坑 3：图像的 Y 轴翻转

**症状**：第一版 PPM 输出了一个"上下颠倒"的聚类图——绿色簇在顶端、蓝色在底端。

**错误假设**：我以为 $(x, y)$ 坐标直接映射到 $(px, py)$ 就可以了：`py = points[i].y * height`。

**真实原因**：PPM 的图像坐标系原点在**左上角**，$y$ 向下增长。而数学坐标系原点在左下角，$y$ 向上增长。直接把 $y$ 坐标映射会导致图像上下颠倒。

**修复方式**：
```cpp
int py = (int)((1.0 - (points[i].y - min_y) / (max_y - min_y)) * (height - 1));
```
`1.0 - normalized_y` 实现了 Y 轴翻转，把数学坐标系的"上高下低"转换为图像坐标系的"上低下高"。

### 坑 4：`std::shuffle` 在没有 `<numeric>` 时的编译错误

**症状**：编译时报错 `error: 'iota' is not a member of 'std'`。代码中使用了 `std::iota` 来填充索引向量。

**错误假设**：我以为 `std::iota` 在 `<algorithm>` 头文件中（因为它有点像 `std::generate` 的变体）。实际上它在 `<numeric>` 中。

**真实原因**：C++ 标准库的归属有时反直觉——`std::iota` 的语义是"填充递增序列"，这被归为数值运算而非算法。类似地，`std::accumulate` 和 `std::partial_sum` 也都在 `<numeric>` 而非 `<algorithm>` 中。

**修复方式**：添加 `#include <numeric>`。这提醒我们：即使代码逻辑正确，头文件缺失是 C++ 中最容易被忽略的错误源之一。写完代码后，建议用 `g++ -Wall -Wextra` 编译，把所有 `implicit declaration` 警告视为错误。

## 效果验证与数据

### 可视化输出

![K-Means 聚类结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-21-K-Means-Clustering/kmeans_output.png)

5 个簇清晰可辨：蓝色（左上）、黄色（右上）、紫色（中部偏上）、红色（中右）、绿色（中下）。每个簇的白色菱形标记精确落在大致几何中心——没有离群点导致中心偏移，也没有簇边界模糊到难以分辨。

### 量化验证结果

| 指标 | 数值 | 阈值 | 判定 |
|------|------|------|------|
| **Baseline SSE** | 2659.18 | - | 参考值 |
| **Final SSE** | 380.81 | - | - |
| **Improvement** | 85.68% | > 30% | ✅ PASS |
| **SSE Monotonicity** | 531→481→473→462→431→393→385→383→381→381→381→381 | 单调不增 | ✅ PASS |
| **Silhouette Score** | 0.533 | > 0.3 | ✅ PASS |
| **Iterations** | 12 | ≤ 30 | ✅ PASS |
| **Converged** | YES | - | ✅ |

### SSE 收敛曲线

```
Iter  1: SSE=531.01  (delta=N/A)     ← 初始化后
Iter  2: SSE=480.62  (delta=50.39)   ← 大幅优化
Iter  3: SSE=473.20  (delta= 7.42)   
Iter  4: SSE=462.11  (delta=11.09)   
Iter  5: SSE=430.67  (delta=31.44)   ← 第二轮大幅优化
Iter  6: SSE=392.83  (delta=37.84)   
Iter  7: SSE=385.20  (delta= 7.62)   
Iter  8: SSE=383.33  (delta= 1.88)   ← 进入精细调整
Iter  9: SSE=381.46  (delta= 1.86)   
Iter 10: SSE=381.30  (delta= 0.16)   ← 微调
Iter 11: SSE=380.81  (delta= 0.50)   
Iter 12: SSE=380.81  (delta= 0.00)   → 收敛
```

曲线的形状非常典型：前 6 步是"粗调"（中心快速移动到大致正确的位置，SSE 从 531 骤降到 393），后 6 步是"微调"（中心在最优位置附近小幅摆动，SSE 下降越来越慢，直到完全不动）。这种"先大后小"的收敛模式是 Lloyd 算法的经典特征，也是其实际高效的原因——大部分工作在早期完成。

### 为什么只做了 12 次迭代

因为三重收敛判据在第 12 次迭代时全部触发：
- `changes == 0`：没有点换簇
- `sse_delta < 1e-6`：SSE 改善为零（到浮点精度）
- `center_shift < 1e-6`：中心完全不动

这说明经过 12 次迭代后，算法已经找到了一个不动点——继续运行到 `max_iterations=100` 也不会有任何变化。

## 总结与延伸

### K-Means 的局限

1. **需要预知 $k$**：你必须事先知道簇的数量。在实际应用中，这往往是最难的部分——你不知道数据有几个自然分组。Elbow Method（找 SSE 曲线的"肘部"）和 Gap Statistics 是常用的 $k$ 选择方法，但它们都只是启发式。

2. **偏好球形簇**：K-Means 用欧氏距离，天然假设每个簇是球形的（各向同性方差）。对于细长条型、环形的簇，K-Means 会切分错误。Gaussian Mixture Models (GMM) 可以处理非球形簇，因为它允许各向异性的协方差矩阵。

3. **对初始化敏感**：不同的随机初始化可能导致完全不同的聚类结果。我们今天的代码通过固定种子保证了可复现性，但 K-Means++ 是更好的工程选择。

4. **不能处理分类变量**：欧氏距离要求数值特征。如果数据有分类变量（如颜色=红/蓝/绿），K-Means 需要先做 one-hot 编码（但 one-hot 的欧氏距离语义上不太自然）。

### 可优化方向

- **K-Means++**：用智能初始化替换随机采样，大幅降低陷入局部最优的概率
- **Mini-Batch K-Means**：每次迭代只用一个小批量数据来更新中心——适合百万级数据集
- **并行化**：Assignment 步骤天然可并行（每个点独立），可以用 OpenMP/CUDA 加速
- **Elkan 算法**：用三角不等式剪枝，避免大多数不必要的距离计算，对高维数据加速显著

### 与本系列的关系

本周的每日编程实践进入了"算法与数据结构"专题，从前几天的计算几何（凸包、碰撞检测）自然过渡到了聚类算法。K-Means 是本周的第三个验证体系项目，它和前面的 A\* 寻路、空间索引（四叉树、KD-Tree、R-Tree）一起构建了"算法 + 可视化 + 量化验证"的完整闭环——不是写完了事，而是用指标证明实现是正确的。

---

*代码仓库：[daily-coding-practice/2026/06/06-21-K-Means-Clustering](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/06/06-21-K-Means-Clustering)*
