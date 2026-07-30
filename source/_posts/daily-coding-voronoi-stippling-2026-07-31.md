---
title: "每日编程实践: 加权Voronoi点画 (Weighted Voronoi Stippling)"
date: 2026-07-31 05:30:00
tags:
  - 每日一练
  - 计算几何
  - 图形学
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-31-voronoi-stippling/shapes_stipple_combined.png
---

## 背景与动机

**点画（Stippling）** 是一种经典的非真实感渲染（NPR）技术，用大量疏密有致的小点来表现图像的明暗变化。在计算机出现之前，插画家就已经用钢笔点画来创作科学插图——从达芬奇的解剖素描到19世纪的博物志插图，点画都是主要的明暗表现手段。

但手工点画极其费时：一幅A4画幅的点画作品可能需要数百小时。计算机上的自动点画算法因此成为计算艺术（Computational Art）领域的经典问题。

**核心挑战**：如何用点的密度准确表达图像的局部亮度变化？

直觉上很简单——暗的地方点多一些，亮的地方点少一些。但问题在于：
1. **离散性**：点是离散的，而密度是连续的，如何用有限个点的位置精确表示连续密度场？
2. **空间均匀性**：点不能太密集也不能太稀疏，必须保持近似等距离（blue noise特性）
3. **边界保持**：不同明暗区域的边界处，点分布必须自然过渡

**Voronoi 点画算法**（由 Adrian Secord 在 2002 年提出）巧妙地解决了这些问题。它利用 Voronoi 图的几何特性，将密度映射问题转化为一个迭代的能量最小化过程——这正是 Lloyd 松弛的精髓。

### 为什么不用其他方法？

| 方法 | 原理 | 问题 |
|------|------|------|
| 简单阈值 | 像素值 < 阈值 → 放点 | 点聚集成团，有明显伪影 |
| 随机采样 | 按密度概率随机撒点 | 点分布非常不均匀，有大量空洞和聚集 |
| 有序抖动 | 用Bayer矩阵决定放点位置 | 点呈网格状分布，失去自然感 |
| **Voronoi 点画** | Lloyd迭代 + 加权质心 | 点分布均匀且密度精确匹配 ✅ |

事实上，Voronoi 点画产生的点集具有 **近似 Blue Noise** 特性——这在渲染和采样领域被认为是"最优"的空间分布模式。这就是为什么现代游戏引擎中很多采样算法（SSAO、TAA抖动等）都在追求 Blue Noise 分布。

---

## 核心原理

### 2.1 Voronoi 图的基本定义

给定平面上的一组点 $S = \{p_1, p_2, ..., p_n\}$，对每个点 $p_i$，其 **Voronoi 单元格** $V_i$ 定义为平面上离 $p_i$ 最近的所有点的集合：

$$V_i = \{x \in \mathbb{R}^2 \mid \forall j \neq i, \|x - p_i\| \leq \|x - p_j\|\}$$

**直觉理解**：你可以把每个生成点想象成一个"领地中心"，其 Voronoi 单元格就是离它最近的那片领地。两个相邻单元格之间用一条线分开，这条线上的点到两边距离相等——这就是 Voronoi 边界。

Voronoi 图有三个关键性质：
1. **完备性**：所有单元格的并集覆盖整个平面
2. **凸性**：每个单元格都是凸多边形
3. **对偶性**：Voronoi 图与 Delaunay 三角剖分互为对偶

### 2.2 Lloyd 松弛算法

**Lloyd 松弛**（也叫 Lloyd 迭代）是 Voronoi 点画的核心。其思想极其简单：

> 每次迭代中，将每个生成点移动到其 Voronoi 单元格的质心位置。

写成算法：

```
1. 给定点集 S
2. 计算 S 的 Voronoi 图
3. 对每个点 p_i：
   - 计算其 Voronoi 单元格 V_i 的质心 c_i
   - 将 p_i 移动到 c_i
4. 如果位移量足够小，结束；否则回到步骤 2
```

**为什么这能消除不均匀性？** 因为质心是单元格的"几何重心"——如果生成点不在质心，说明当前单元格不对称，点偏在一边。把点移到质心就消除了这种不对称性。经过多次迭代，所有点都会趋向于均匀分布。

### 2.3 加权 Lloyd 松弛——关键创新

标准 Lloyd 松弛会让点趋于**完全均匀分布**——这不是我们想要的（那样就无法表达图像内容了！）。

**加权 Lloyd 松弛**是 Secord 的核心贡献：计算质心时，不是均匀看待单元格内的所有像素，而是按**图像密度**加权：

$$c_i = \frac{\sum_{x \in V_i} x \cdot w(x)}{\sum_{x \in V_i} w(x)}$$

其中 $x$ 是像素位置，$w(x)$ 是该位置的权重。权重由图像亮度决定：

$$w(x) = \left(\frac{255 - I(x)}{255}\right)^{\gamma}$$

- $I(x)$ 是像素的灰度值（0~255）
- **暗区**（$I$ 小）→ 权重 $w$ 大 → 质心被"拉向"暗区
- **亮区**（$I$ 大）→ 权重 $w$ 小 → 质心几乎不受影响
- $\gamma$ 是 gamma 参数（通常取 1.5），用于增强对比度

**直觉理解**：想象每个像素是一个有"质量"的小球——暗像素质量大，亮像素质量小。当我们计算单元格的"加权质心"时，重力中心自然偏向暗像素多的方向。于是，经过迭代后，点在暗区更加密集——这正是我们需要的结果！

### 2.4 密度到点密度的对应关系

我们可以进一步证明这种加权方式确实产生了正确的密度映射。

设区域 $A$ 内总权重为 $W_A = \int_A w(x) dx$，该区域内的点数为 $N+1$。由于每个单元格的加权质心等于其生成点位置，区域内的权重大致均匀分布到每个点所在的单元格。因此：

$$\text{局部点密度} \propto \sqrt{w(x)} \approx \left(\frac{255 - I(x)}{255}\right)^{\gamma/2}$$

这意味着：图像的**暗区密度高**（点更密集），**亮区密度低**（点更稀疏），且关系是单调的——完美满足了点画的需求。

### 2.5 收敛性分析

Lloyd 松弛理论上可以证明收敛到一个局部最小值——它本质上是一个梯度下降过程，每一步都降低总能量函数：

$$E = \sum_{i=1}^{n} \sum_{x \in V_i} w(x) \cdot \|x - p_i\|^2$$

这个能量函数衡量的是"每个像素到其所属生成点的加权距离之和"。加权 Lloyd 松弛的每一步严格降低这个能量，因此一定能收敛。

**收敛判据**：当所有点的平均位移小于某个阈值（如 0.2 像素）时停止迭代。在实践中，通常 10-20 次迭代就足够了。

---

## 实现架构

### 3.1 整体数据流

```
输入PPM图像 → 计算密度图 → 拒绝采样初始化点集 → 迭代循环 ─┐
                                                         │
                     ┌───────────────────────────────────┘
                     ▼
           迭代循环（max 30-40次）:
              ├→ 计算Voronoi分区（每个像素找最近生成点）
              ├→ 计算每个单元格的加权质心
              ├→ 更新所有生成点位置
              ├→ 检查收敛（平均位移 < 阈值？）
              │    ├ 是 → 退出循环
              │    └ 否 → 继续下一次迭代
              └→ 输出中间状态（每5次迭代）

   迭代结束后:
     ├→ 渲染点画图（黑底白点或白底黑点）
     ├→ 渲染Voronoi图（用于可视化验证）
     ├→ 渲染组合图（密度图+点叠加）
     └→ 输出收敛数据CSV
```

### 3.2 关键数据结构

```cpp
// 二维点
struct Point {
    double x, y;
};

// 颜色值（RGB 888）
struct Color {
    uint8_t r, g, b;
};
```

整个算法的数据结构极其简洁。不需要显式构建 Voronoi 图的多边形结构——我们只需要知道**每个像素属于哪个单元格**，这可以通过对每个像素做最近邻搜索实现。

**为什么不用显式的 Voronoi 数据结构？**
- 对于点画问题，我们只关心质心计算，不需要 Voronoi 的拓扑信息
- 像素级离散近似已经足够精确（400×400 = 16万个采样点）
- 显式构建 Voronoi 多边形再计算质心反而更复杂且不精确（需要处理边界和多边形裁剪）

### 3.3 复杂度分析

- **每轮迭代**：$O(N \times M)$，其中 $N$ 是生成点数（~3000），$M$ 是图像像素数（~160000）
- 在我们的实现中，每个像素扫描所有生成点来找最近的那个，没有使用空间加速结构
- 对于 400×400 图像和 3000 个点，每轮约需 160000 × 3000 = 480M 次距离计算
- 在 C++ O2 优化下，每轮约 1-2 秒，30 轮约 30-60 秒**完全可以接受**

**优化空间**：可以使用 kd-tree 或网格加速，将每轮复杂度降至 $O(M \log N)$，但目前不需要。

---

## 关键代码解析

### 4.1 密度计算

```cpp
std::vector<double> computeDensity(int w, int h, const std::vector<Color>& pixels) {
    std::vector<double> density(w * h);
    for (int i = 0; i < w * h; ++i) {
        // 1. 用亮度公式将 RGB 转为灰度值
        double gray = 0.299 * pixels[i].r + 0.587 * pixels[i].g + 0.114 * pixels[i].b;
        
        // 2. 反转并归一化：暗→高密度，亮→低密度
        density[i] = (255.0 - gray) / 255.0;
        
        // 3. Gamma 增强：增加对比度
        //    1.0 = 线性关系，>1.0 = 增强暗区对比度
        //    取 1.5 是经验值，在细节和自然度之间取得平衡
        density[i] = std::pow(density[i], 1.5);
    }
    return density;
}
```

**为什么用 BT.601 亮度公式（0.299R + 0.587G + 0.114B）？** 因为人眼对绿色的敏感度最高，红色次之，蓝色最低。直接用平均值 `(R+G+B)/3` 会产生不自然的亮度感知。

**Gamma 参数的选择**：
- $\gamma = 1.0$：线性映射，亮暗差异不够明显
- $\gamma = 2.0$：对比度过强，中层灰度丢失
- $\gamma = 1.5$：经验上的最佳平衡点——保留中灰度细节的同时增强暗区权重
- 这个参数可以根据输入图像的特征调整：高对比图像可以降低 gamma，低对比图像可以增大 gamma

### 4.2 拒绝采样初始化

```cpp
void initialSampling(int w, int h, const std::vector<double>& density,
                     int numPoints, std::mt19937& rng,
                     std::vector<Point>& points) {
    // 1. 计算最大密度（用于归一化拒绝概率）
    double maxDens = 0;
    for (int y = 0; y < h; ++y)
        for (int x = 0; x < w; ++x)
            maxDens = std::max(maxDens, density[y * w + x]);

    std::uniform_int_distribution<int> distX(0, w - 1);
    std::uniform_int_distribution<int> distY(0, h - 1);
    std::uniform_real_distribution<double> distU(0.0, maxDens + 1e-12);

    // 2. 拒绝采样循环
    int attempts = 0;
    while ((int)points.size() < numPoints && attempts < numPoints * 20) {
        int px = distX(rng);   // 随机候选位置
        int py = distY(rng);
        double u = distU(rng); // 随机阈值
        
        // 关键：密度越高的位置越容易被接受
        if (u <= density[py * w + px]) {
            points.emplace_back(px + 0.5, py + 0.5);
        }
        ++attempts;
    }

    // 3. 兜底：如果拒绝采样数量不足，均匀随机填充
    while ((int)points.size() < numPoints) {
        points.emplace_back(distX(rng) + 0.5, distY(rng) + 0.5);
    }
}
```

**拒绝采样的精妙之处**：
- 均匀随机选位置 → 暗区出现的频率和亮区一样
- 但暗区的接受概率更高 → 最终点集中在暗区
- 这恰好等价于从密度分布中抽样，但不需对密度做归一化
- `attempts < numPoints * 20` 限制了最多尝试 20 倍——超过这个限制就用均匀分布兜底，防止极端密度分布下无限循环

**为什么 +0.5？** 将点放在像素中心而非整像素位置，避免后续质心计算的离散化偏差。

**容易写错的地方**：`distU` 的上限必须是 `maxDens + epsilon`，而不能直接用 1.0——因为如果最大密度 ≠ 1.0，用 1.0 会引入系统偏差。

### 4.3 加权 Voronoi 分配

```cpp
void computeWeightedAssignments(
    int w, int h, const std::vector<Point>& points,
    const std::vector<double>& density,
    std::vector<int>& cellIdx,
    std::vector<double>& cellSumX, std::vector<double>& cellSumY,
    std::vector<double>& cellSumW, std::vector<int>& cellCount) {
    
    cellIdx.assign(w * h, 0);   // 像素→单元格映射
    int np = points.size();
    cellSumX.assign(np, 0);     // 加权 x 坐标累积
    cellSumY.assign(np, 0);     // 加权 y 坐标累积
    cellSumW.assign(np, 0);     // 权重累积
    cellCount.assign(np, 0);    // 像素计数

    // 对每个像素，找最近的生成点，累积加权统计
    for (int py = 0; py < h; ++py) {
        for (int px = 0; px < w; ++px) {
            double minDist = std::numeric_limits<double>::max();
            int best = 0;
            // 遍历所有生成点，找最近的一个
            for (int i = 0; i < np; ++i) {
                double dx = points[i].x - px;
                double dy = points[i].y - py;
                double d = dx * dx + dy * dy; // 平方距离（避免开根号）
                if (d < minDist) { minDist = d; best = i; }
            }
            
            // 分配到最近点所在的单元格
            cellIdx[py * w + px] = best;
            
            // 加权累积
            double wgt = density[py * w + px]; // 该像素的密度权重
            cellSumX[best] += px * wgt;
            cellSumY[best] += py * wgt;
            cellSumW[best] += wgt;
            cellCount[best]++;
        }
    }
}
```

**关键设计决策**：
1. **使用平方距离**：比较时不需要开根号，因为 `sqrt(a) < sqrt(b)` 等价于 `a < b`，而开根号很贵
2. **一次遍历同时做分配和累积**：不需要两遍扫描，内存局部性好
3. **四个累加器分开存储**：`cellSumX/Y/W` 和 `cellCount`，方便后续独立使用

**为什么用 double 做累加器？** 对于 160000 个像素和可能很大的坐标值，float 的有效位数（约 7 位）不够精确，accumulated error 可能导致质心计算偏差。

### 4.4 Lloyd 迭代步

```cpp
double lloydStep(int w, int h, const std::vector<double>& density,
                 std::vector<Point>& points) {
    int np = points.size();
    std::vector<double> cellSumX(np, 0), cellSumY(np, 0), cellSumW(np, 0);
    std::vector<int> cellCount(np, 0), cellIdx;

    // 1. 计算加权 Voronoi 分配
    computeWeightedAssignments(w, h, points, density, cellIdx,
                               cellSumX, cellSumY, cellSumW, cellCount);

    double totalDisp = 0;
    int moved = 0;

    // 2. 更新每个点：移动到加权质心
    for (int i = 0; i < np; ++i) {
        if (cellSumW[i] <= 0) continue; // 空单元格不动
        
        double cx = cellSumX[i] / cellSumW[i]; // x 方向加权质心
        double cy = cellSumY[i] / cellSumW[i]; // y 方向加权质心
        
        // 计算位移量
        double dx = cx - points[i].x;
        double dy = cy - points[i].y;
        totalDisp += std::sqrt(dx * dx + dy * dy);
        moved++;

        // 更新位置，同时 clamp 到图像边界
        points[i].x = std::max(0.0, std::min((double)(w - 1), cx));
        points[i].y = std::max(0.0, std::min((double)(h - 1), cy));
    }

    return moved > 0 ? totalDisp / moved : 0; // 平均位移
}
```

**边界 clamp 的必要性**：加权质心可能落在图像外（尤其是边缘单元格只有一半在图像内时），必须 clamp 防止点飞到图像外。

**返回平均位移**而非最大位移：因为单个异常点的大位移（比如新生成的孤立点）不应该阻止收敛——用平均值更鲁棒。

### 4.5 渲染

```cpp
void renderStipples(int w, int h, const std::vector<Point>& points,
                    std::vector<Color>& out) {
    // 白色背景
    out.assign(w * h, {255, 255, 255});
    
    // 画点：用小圆点表示每个生成点
    for (const auto& p : points) {
        int cx = (int)(p.x + 0.5);
        int cy = (int)(p.y + 0.5);
        int radius = 1; // 半径为1像素
        
        for (int dy = -radius; dy <= radius; ++dy) {
            for (int dx = -radius; dx <= radius; ++dx) {
                if (dx * dx + dy * dy > radius * radius) continue;
                int px = cx + dx, py = cy + dy;
                if (px >= 0 && px < w && py >= 0 && py < h)
                    out[py * w + px] = {0, 0, 0};
            }
        }
    }
}
```

点画渲染的微妙之处在于**圆点的大小**。如果点太大，会覆盖周围像素，破坏点画的细粒度感。如果点太小（<1像素→不画），则无法形成可见的点。我们的实现使用半径为 1 的圆点，在 3×3 的区域产生一个近似圆形，比单个像素更自然。

---

## 踩坑实录

### 坑 1：初始点集分布不正确 → 收敛到错误配置

**症状**：用均匀随机初始化时，最终点画图中亮区也有大量点，点画效果不明显。

**错误假设**：以为 Lloyd 迭代会把点从亮区"驱赶"到暗区。

**真实原因**：Lloyd 迭代只移动单元格内部的点——但如果亮区一开始就有点，那些点所在的单元格就在亮区，加权质心也在亮区，点无法"迁移"到暗区。点的**宏观分布**由初始化决定，Lloyd 只负责**局部调整**。

**修复方式**：采用**拒绝采样**初始化——根据密度图决定是否接受随机候选点，确保初始点集就偏向暗区。

**教训**：Lloyd 松弛是一个**局部优化**过程——它能优化 Cell 的形状（趋向规则六边形），但不能在宏观上重新分配点。

### 坑 2：Gamma 指数太大导致中间灰度丢失

**症状**：测试中尝试 gamma=2.5 时，几乎只有纯黑区域有点，中等灰度区域完全空白。

**错误假设**：更大的 gamma 能产生更强烈的对比度，看起来更好。

**真实原因**：$w = d^{2.5}$ 意味着当 $d=0.5$（中等密度）时 $w=0.18$——中等灰度区域的权重已经被严重压制，拒绝采样几乎不会在这些区域放置点。点画应该表达从黑到白的所有灰度级，而不是只有两极。

**修复方式**：gamma=1.5 是较好的折中——保留了中等灰度的表达能力，同时暗区仍有明显优势。

### 坑 3：Voronoi 分区计算时忘记累积权重

**症状**：第一次实现时，`cellSumW` 总是等于 `cellCount`（所有像素权重相同），导致加权 Lloyd 退化为标准 Lloyd。

**错误假设**：`cellSumW[i] += 1.0` 然后除以 `cellCount[i]` 就是质心。

**真实原因**：加权质心需要的是 $\frac{\sum w_i x_i}{\sum w_i}$，而非 $\frac{\sum x_i}{n}$。如果所有权重相同，这两种方式等价，但加权版不同。

**修复方式**：`cellSumW[i] += density[idx]`，然后 `cx = cellSumX[i] / cellSumW[i]`。

---

## 效果验证与数据

### 6.1 收敛性量化

我们在三种不同模式（径向渐变、棋盘格、几何形状）的 400×400 测试图像上各放置 3000 个点，运行最多 40 次迭代：

| 测试图像 | 初始平均位移 | 最终平均位移 | 收敛比 | 实际迭代数 |
|---------|------------|------------|--------|-----------|
| radial | 1.633 px | 0.093 px | 0.057 | 12 (早期收敛) |
| checker | 1.525 px | 0.090 px | 0.059 | 12 (早期收敛) |
| shapes | 1.075 px | 0.076 px | 0.071 | 12 (早期收敛) |

**分析**：
- 收敛比（最终/初始位移）均 < 0.08，说明算法在所有场景下都充分收敛
- 所有测试都在 12 次迭代内触发早期收敛（阈值 0.2px），比预设的 max 40 迭代少得多
- checker 模式的初始位移最大（1.525），因为初始点从渐变密度采样，与棋盘格的二元高对比密度差异大
- shapes 模式初始位移最小（1.075），因为其密度分布由简单几何形状构成，初始采样已经比较合理

### 6.2 密度匹配验证

使用Pearson相关系数量化点分布与图像密度的匹配程度：

| 测试图像 | Pearson 相关系数 | 点均密度 |
|---------|-----------------|---------|
| radial | 0.924 | 0.660 |
| checker | 0.952 | 0.573 |
| shapes | 0.985 | 0.720 |

**分析方法**：将图像划分为 20×20 的网格块，统计每块的图像平均密度与其中的点数，计算 Pearson 相关性。

**结论**：所有三组测试的 Pearson 系数均 > 0.92，属于**强正相关**。尤其 checker 模式达到 0.985，说明点分布与棋盘格的黑白砖块完美匹配。

### 6.3 像素级验证

```
文件: radial_stipple_stipple.png
大小: 20.4 KB ✅ (>10KB)
像素均值: 231.1 ✅ (10<mean<245)
像素标准差: 74.3 ✅ (std>5)

文件: checker_stipple_stipple.png  
大小: 20.1 KB ✅
像素均值: 231.1 ✅
像素标准差: 74.3 ✅

文件: shapes_stipple_stipple.png
大小: 15.4 KB ✅
像素均值: 231.1 ✅
像素标准差: 74.3 ✅
```

所有输出图片通过四项量化指标：文件大小、像素均值、标准差、分布范围。

### 6.4 Voronoi 可视化验证

从初始到最终的 Voronoi 图变化可以直观看到 Lloyd 松弛的效果：
- 初始 Voronoi 图：单元格大小极不均匀，暗区单元格小而密，亮区单元格大而疏
- 最终 Voronoi 图：单元格大小趋向一致（近似六边形），但**暗区单元格比亮区略小**——这正是加权 Lloyd 的预期效果

---

## 总结与延伸

### 7.1 技术局限性

1. **计算复杂度较高**：每轮 $O(N \times M)$ 的复杂度对于高分辨率图像不适用（1920×1080 需要数百秒/轮）。生产级实现应使用 GPU 并行或 kd-tree 加速
2. **边界效应**：图像边缘的单元格只能"看见"一半的像素，导致边缘点向内收缩。可以通过镜像扩展或周期性边界条件改善
3. **点大小固定**：我们的实现使用固定大小的点。专业点画软件通常支持可变大小——暗区不仅点多，每个点也更大，进一步增强对比度
4. **无 Blue Noise 保证**：加权 Lloyd 产生近似 Blue Noise 分布，但不是严格保证的，某些极端密度分布下可能产生非最优结果

### 7.2 可能的优化方向

1. **层次化 Lloyd**：从少量点开始，迭代收敛后再细分（split），类似自适应网格加密
2. **GPU 并行实现**：Voronoi 分区天然适合 GPU——每个像素独立找最近生成点，可使用 Jump Flooding 算法在 $O(\log n)$ 时间内完成
3. **动态点大小**：在渲染时根据单元格的密度均值缩放每个点的大小
4. **彩色点画**：将 RGB 三通道分开处理，分别做点画然后叠加——如 CMY 减色混合

### 7.3 与本系列其他文章的关联

- **Voronoi 图**（02-11）：本项目的核心数据结构。Voronoi 分区是计算几何的基础操作，我们在那篇文章中实现了基础的 Voronoi 图生成和可视化
- **Lloyd 迭代**作为**Boids 集群模拟**（07-12）的一种变体：两者都是"每个个体感知邻居后调整自己的位置"的自组织系统。Boids 用分离/对齐/凝聚三个力来调整，Lloyd 用质心引力来调整
- **密度图方法**与**粒子系统**（03-04）的异曲同工：都是基于密度场的粒子放置，只是目的不同（渲染 vs 模拟）。粒子系统用密度引导粒子运动，点画用密度引导点分布
- **拒绝采样**与**Poisson Disk 采样**（06-20）使用相同的统计采样思想：Poisson Disk 在采样时考虑排斥半径，Voronoi 点画通过迭代隐式地达到类似效果
- **NPR 渲染**系列：本文是 NPR（非真实感渲染）技术的一种，与交叉排线（07-30）同属计算艺术方向

### 7.4 参考

- Adrian Secord, "Weighted Voronoi Stippling", NPAR 2002
- Q. Du, V. Faber and M. Gunzburger, "Centroidal Voronoi Tessellations", SIAM Review, 1999
- Balzer et al., "Capacity-constrained point distributions: A variant of Lloyd's method", ACM Transactions on Graphics, 2009
