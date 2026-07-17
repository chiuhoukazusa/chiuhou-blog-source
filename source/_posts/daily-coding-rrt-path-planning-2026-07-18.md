---
title: "每日编程实践: RRT 路径规划"
date: 2026-07-18 06:10:00
tags:
  - 每日一练
  - 算法
  - C++
  - 路径规划
  - 机器人
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-18-rrt-path-planning/rrt_output.png
---

## 背景与动机

想象一下：你有一个机器人需要从房间的一端移动到另一端，但房间里有桌子、椅子和柜子。机器人的任务是找到一条可行路径——不要求是最优的，但必须存在，而且要快。

这就是**运动规划**（Motion Planning）的核心问题。在传统算法中，我们可能会想到 A\* 或 Dijkstra，但这些算法要求整个搜索空间被离散化为网格。对于高维连续空间（比如六自由度机械臂），网格的大小会呈指数爆炸——这就是经典的"维度灾难"（Curse of Dimensionality）。

**RRT（Rapidly-exploring Random Tree，快速探索随机树）**由 Steven M. LaValle 在 1998 年提出，是解决连续空间运动规划问题的一个里程碑。它不需要离散化空间，不需要建立完整的连通图，只需要在连续空间中**采样**——它通过随机采样将搜索自然地"拉扯"向空间的未知区域。

RRT 在工业界的应用非常广泛：

- **自动驾驶**：用于轨迹规划（尤其是低速场景下的非完整约束规划），百度 Apollo、Waymo 都使用 RRT 变体
- **机器人机械臂**：MoveIt! 框架（ROS 默认的运动规划库）的默认规划器就基于 RRT-Connect
- **无人机导航**：在 3D 空间中绕过建筑、电线等障碍物
- **游戏 AI**：用于 NPC 的动态寻路，尤其是地形不规则的开放世界游戏
- **蛋白质折叠模拟**：在生物信息学中探索分子构象空间

为什么不用 A\* 直接做？因为 A\* 在连续空间中需要将世界离散化为网格，而 RRT 天然不需要。对于 3D 空间中的 6-DOF 机械臂，A\* 需要的状态数量是 O(k^n)（k 是每维离散化精度，n 是自由度），而 RRT 的复杂度只与空间的"困难程度"相关，与维度关系不大。

今天我们就来实现一个 2D 的 RRT 路径规划器，包含：圆形和矩形障碍物、碰撞检测、路径平滑和量化验证。

---

## 核心原理

### 基本思想

RRT 的核心思想非常简单，用一句话概括：**从起点不断往随机方向扩展一棵树，直到有一根树枝碰到目标区域**。

具体而言，每一步：

1. 在自由空间中随机采一个点（偶尔直接采目标点，这叫 goal bias）
2. 在已有的树中找到离这个采样点最近的节点
3. 从最近节点向采样点方向"延伸"一小步（step size）
4. 如果延伸过程中没有碰到障碍物，就把新点加入树，成为最近节点的子节点
5. 如果新点离目标足够近，连接目标点并回溯构造路径

这个算法的精妙之处在于：由于采样是均匀随机的，**未探索的大面积区域被选中的概率更高**，因此树会天然向空间的"空白区域"生长——这就是"Rapidly-exploring"的来源。

### 数学直觉

为什么 RRT 会快速探索大空间？这背后的数学直觉是**Voronoi 偏置**：

考虑每个已有节点对应的 Voronoi 区域（空间中距离该节点最近的点集合）。当你在整个空间中随机采样时，某个节点的 Voronoi 区域越大，采样点落入该区域的概率就越大。而树边缘处节点的 Voronoi 区域通常比树内部节点的更大（内部节点的 Voronoi 区域被其他节点挤占），因此新节点更大概率被"拉"向未探索区域。

换句话说：**RRT 不需要显式地"决定"探索方向——随机采样自然地偏向空间中的大空白区域**。

### 与同类算法的比较

| 算法 | 离散化 | 高维适应性 | 路径最优 | 速度 |
|------|--------|-----------|---------|------|
| A*/Dijkstra | 需要 | 差（指数） | ✅ 最优 | 小空间快 |
| PRM（概率路线图） | 不需要 | 好 | ✅ 渐近最优 | 预处理慢 |
| RRT（基础版） | 不需要 | 最好 | ❌ 非最优 | 单次查询快 |
| RRT\*（优化版） | 不需要 | 好 | ✅ 渐近最优 | 比 RRT 慢 |
| RRT-Connect | 不需要 | 最好 | ❌ 次优 | 极快 |

**RRT 不是概率完备（probabilistically complete）的吗？** 确切地说，RRT 是**概率完备**的：如果存在可行路径，当采样次数趋于无穷时，RRT 找到路径的概率趋于 1。但 RRT 找到的路径通常不是最优的——路径长度可能远大于理论最优解（这就是"zigzag"问题）。

**RRT\*** 通过"重连"（rewire）机制来解决最优性问题：新节点加入后，会在其邻域内检查是否可以通过其他节点以更短的代价到达，从而持续优化树的结构。但这今天不在本文讨论范围内。

### 关键参数

理解 RRT 的关键参数：

- **Step Size（步长）**：每次从最近节点向采样点延伸的距离。太小会导致探索缓慢（树需要更多节点才能到达目标），太大会跳过狭窄通道。典型设置：物理空间的 2%-5%
- **Goal Bias（目标偏差）**：以某概率直接采样目标点而不是随机点。这加快了向目标的收敛速度，但过高会导致树卡在障碍物附近出不来。典型设置：5%-10%
- **Goal Radius（目标半径）**：新节点离目标多远时认为"到达"。小于 step size 即可
- **Collision Margin（碰撞边距）**：在障碍物周围留出的安全距离，考虑到物理尺寸和控制误差

---

## 实现架构

### 整体数据流

```
Main Entry
    ├── 定义 World（边界、起点、终点、障碍物）
    ├── RRT 主循环
    │   ├── Sample: 随机采样（含 goal bias）
    │   ├── Nearest: 在 O(n) 中搜索最近节点
    │   ├── Steer: 向采样点延伸一步
    │   ├── CollisionCheck: 线段碰撞检测
    │   └── AddNode: 加入树
    ├── Path Extraction: 从目标回溯到起点
    ├── Path Smoothing: Shortcut Pruning 优化路径
    ├── Verification: 5 项量化检查
    └── Visualization: PPM 渲染
```

### 关键数据结构

```cpp
struct Vec2 {
    double x, y;
    // 向量运算：加法、减法、数乘、点积、长度、归一化、距离
};

struct CircleObstacle {
    Vec2 center;
    double radius;
};

struct RectObstacle {
    Vec2 min, max;  // AABB（轴对齐包围盒）
};

struct RRTNode {
    Vec2 pos;       // 节点在 2D 空间中的位置
    int parent;     // 父节点在 tree 数组中的索引（根节点为 -1）
};
```

**为什么用 `parent` 索引而不是指针？** 因为树存储在 `std::vector<RRTNode>` 中。使用索引避免了指针的悬垂风险，且 path extraction（回溯）只需要沿着 parent 索引链逆向走即可。

**为什么 Obstacle 只有 Circle 和 Rect？** 这是为了简化碰撞检测的数学实现。这两种基本形状的组合已经可以近似大多数现实障碍物。工业级实现通常使用更复杂的几何表示（如凸多边形、SDF 符号距离场），但原理是相同的。

### 职责划分

| 模块 | 职责 |
|------|------|
| `Vec2` | 2D 向量数学，所有几何计算的基础 |
| 碰撞检测系统 | `pointInCircle/pointInRect`（点检测）、`segmentCollidesCircle/segmentCollidesRect`（线段检测）|
| `rrt()` | 主算法循环（采样、最近邻搜索、steer、碰撞检查） |
| `smoothPath()` | 后处理：通过 shortcut pruning 优化路径 |
| `writePPM()` | PPM 格式的可视化输出（树、原始路径、平滑路径、障碍物）|
| `main()` | 场景配置、运行、验证、渲染 |

---

## 关键代码解析

### 1. 碰撞检测：线段 vs 圆

这是整个规划器的核心——如果碰撞检测不准确，会导致路径穿过障碍物或拒绝有效路径。

```cpp
bool segmentCollidesCircle(const Vec2& a, const Vec2& b, 
                           const CircleObstacle& c, double margin = 0.0) {
    Vec2 d = b - a;                      // 线段方向向量
    Vec2 f = a - c.center;               // 圆心到线段起点的向量
    double r = c.radius + margin;        // 有效碰撞半径

    // 参数化线段: P(t) = a + t*d, t ∈ [0,1]
    // 求 min ||P(t) - center||² = ||a + t*d - center||²
    //                         = ||f + t*d||²
    //                         = (f + t*d)·(f + t*d)
    //                         = t²(d·d) + 2t(f·d) + (f·f)
    double A = d.dot(d);                 // 二次项系数
    double B = 2.0 * f.dot(d);           // 一次项系数
    double C = f.dot(f) - r * r;         // 常数项（包含半径）

    double discriminant = B*B - 4*A*C;
    if (discriminant < 0) return false;  // 无交点，不相交

    double sqrtD = std::sqrt(discriminant);
    double t1 = (-B - sqrtD) / (2*A);    // 距离线段起点最近的交点
    double t2 = (-B + sqrtD) / (2*A);    // 距离线段起点最远的交点

    // 仅当任意交点在 [0,1] 范围内，或线段完全在圆内
    return (t1 >= 0 && t1 <= 1) 
        || (t2 >= 0 && t2 <= 1) 
        || (t1 < 0 && t2 > 1);
}
```

**为什么要用二次方程的判别式？** 线段与圆的碰撞本质上是求直线与圆的交点（二次方程），然后检查交点是否落在 [0,1] 参数范围内。直接计算比迭代采样更精确、更高效。

**最后一个条件 `(t1 < 0 && t2 > 1)` 是什么意思？** 这覆盖了线段完全在圆内的情况——t1 在起点"之前"（<0），t2 在终点"之后"（>1）。如果缺少这个条件，当整个线段都在障碍物圆内时，碰撞检测会错误地返回"无碰撞"。

### 2. 碰撞检测：线段 vs 矩形

```cpp
bool segmentCollidesRect(const Vec2& a, const Vec2& b, 
                         const RectObstacle& r, double margin = 0.0) {
    // 先扩展矩形（margin）
    RectObstacle expanded = {
        Vec2(r.min.x - margin, r.min.y - margin),
        Vec2(r.max.x + margin, r.max.y + margin)
    };

    // 快速检查：有端点在里面？
    if (pointInRect(a, expanded)) return true;
    if (pointInRect(b, expanded)) return true;

    // 检查与四条边的交点
    // 水平边：y = expanded.min.y 和 y = expanded.max.y
    for (double y : {expanded.min.y, expanded.max.y}) {
        if (std::abs(a.y - b.y) > 1e-9) {          // 非水平线段
            double t = (y - a.y) / (b.y - a.y);     // 解 a.y + t*(b.y-a.y) = y
            if (t >= 0 && t <= 1) {                 // 交点在参数范围内
                double x = a.x + t * (b.x - a.x);
                if (x >= expanded.min.x && x <= expanded.max.x) return true;
            }
        }
    }
    // 垂直边同理
    for (double xv : {expanded.min.x, expanded.max.x}) {
        // ... 与水平边对称，交换 x/y 角色
    }
    return false;
}
```

**为什么检查线段与矩形边的交点就够了？** 矩形是凸集，AABB 的四条边构成了矩形的边界。如果一条线段与矩形碰撞（且两端点都不在内部），那么它必然穿过至少一条边——这是凸集分离定理的直接推论。因此检查四条边的交点可以完整判定碰撞。

**注意浮点精度**：`std::abs(a.y - b.y) > 1e-9` 防止除以零（水平线段与水平边的"交点"）。在几何计算中，忽略这种边界情况会导致 NaN 传播和静默失败。

### 3. RRT 主循环

```cpp
RRTResult rrt(const Vec2& start, const Vec2& goal,
              const std::vector<CircleObstacle>& circles,
              const std::vector<RectObstacle>& rects,
              double xmin, double xmax, double ymin, double ymax,
              double stepSize = 0.5,    // 每次延伸步长
              double goalBias = 0.05,   // 5% 概率直接采样目标
              double goalRadius = 0.5,  // 距目标多近即为到达
              int maxIterations = 5000) {

    std::mt19937 rng(42);  // 固定种子保证可复现
    std::uniform_real_distribution<double> distX(xmin, xmax);
    std::uniform_real_distribution<double> distY(ymin, ymax);
    std::uniform_real_distribution<double> dist01(0.0, 1.0);

    RRTResult result;
    result.tree.push_back(RRTNode(start, -1));  // 根节点

    for (int iter = 0; iter < maxIterations; iter++) {

        // --- Sample: 采样（含 goal bias）---
        Vec2 sample;
        if (dist01(rng) < goalBias) {
            sample = goal;   // 以 goalBias 概率直接采样目标
        } else {
            sample = Vec2(distX(rng), distY(rng));  // 均匀随机采样
        }

        // --- Nearest: O(n) 最近邻搜索 ---
        int nearest = 0;
        double bestDist = sample.distToSq(result.tree[0].pos);
        for (size_t i = 1; i < result.tree.size(); i++) {
            double d = sample.distToSq(result.tree[i].pos);
            if (d < bestDist) {
                bestDist = d;
                nearest = (int)i;
            }
        }

        // --- Steer: 向采样点延伸 ---
        Vec2 nearPos = result.tree[nearest].pos;
        Vec2 dir = sample - nearPos;
        double dist = dir.len();
        if (dist < 1e-9) continue;  // 采样点与节点重合，跳过

        Vec2 newPos;
        if (dist > stepSize) {
            // 往采样点方向走 stepSize
            newPos = nearPos + dir.normalized() * stepSize;
        } else {
            // 采样点在本步长内，直接到达
            newPos = sample;
        }

        // --- Collision Check ---
        if (!isPointCollisionFree(newPos, circles, rects)) continue;
        if (!isCollisionFree(nearPos, newPos, circles, rects)) continue;

        // --- Add Node ---
        int newNodeIdx = (int)result.tree.size();
        result.tree.push_back(RRTNode(newPos, nearest));

        // --- Goal Check ---
        if (newPos.distTo(goal) < goalRadius) {
            if (isCollisionFree(newPos, goal, circles, rects)) {
                result.tree.push_back(RRTNode(goal, newNodeIdx));
                result.success = true;
                result.iterations = iter + 1;

                // 回溯提取路径
                int idx = (int)result.tree.size() - 1;
                while (idx >= 0) {
                    result.path.push_back(idx);
                    idx = result.tree[idx].parent;
                }
                std::reverse(result.path.begin(), result.path.end());
                return result;
            }
        }
    }

    result.iterations = maxIterations;
    return result;  // 未能找到路径
}
```

**为什么使用 `distToSq`（距离平方）而不是 `distTo`？** 因为 `sqrt()` 是昂贵的操作，我们在最近邻搜索中只需要比较距离大小，不需要实际距离值。使用平方距离避免了数百次平方根运算，对大规模 RRT 树（成千上万节点）的加速效果显著。

**为什么碰撞检测要分两步**：`isPointCollisionFree` 检查新点是否在障碍物内，`isCollisionFree` 检查从最近节点到新点的整个线段是否无碰撞。两步缺一不可——前者防止树节点"嵌入"障碍物，后者防止路径"穿过"障碍物。

**Goal bias 为什么是 5%？** 这是一个经验值。太高的 goal bias（如 50%）会让树过度倾向于目标，在复杂障碍物环境中可能困住无法前进；太低的 goal bias（如 1%）会让树大量浪费时间在随机探索上，收敛速度极慢。5%-10% 是一个普遍的平衡点。

### 4. 路径平滑：Shortcut Pruning

RRT 找到的原始路径通常有大量 zigzag（锯齿状），因为每次延伸的方向是随机的。路径平滑能够显著改善路径质量：

```cpp
std::vector<Vec2> smoothPath(const std::vector<Vec2>& path,
                              const std::vector<CircleObstacle>& circles,
                              const std::vector<RectObstacle>& rects,
                              int maxAttempts = 100) {
    if (path.size() <= 2) return path;

    std::mt19937 rng(123);
    std::vector<Vec2> smoothed = path;

    for (int attempt = 0; attempt < maxAttempts; attempt++) {
        if (smoothed.size() <= 2) break;

        int n = (int)smoothed.size();
        // 随机选两个非相邻节点
        std::uniform_int_distribution<int> dist(0, n-1);
        int i = dist(rng);
        int j = dist(rng);
        if (i > j) std::swap(i, j);
        if (j - i <= 1) continue;  // 必须非相邻，否则没有中间点可删

        // 如果 i 到 j 直接连接无碰撞，删除中间所有节点
        if (isCollisionFree(smoothed[i], smoothed[j], circles, rects)) {
            smoothed.erase(smoothed.begin() + i + 1, smoothed.begin() + j);
        }
    }

    return smoothed;
}
```

**核心思想**：原始路径的一系列有序节点中，如果某两个非相邻节点（i 和 j，j > i+1）之间可以直接用一条线段连接（无碰撞），那么 i 到 j 之间的所有中间节点都可以被删掉，路径直接跳过它们。

**为什么使用随机选择而不是贪心（尝试所有配对）？** 贪心方法的复杂度是 O(n² × 碰撞检测)，当路径有几百个节点时变得很慢。随机采样在给定足够尝试次数（如 100-200 次）下，实际效果非常接近贪心方法，但速度快得多。

**实际效果**：在我们的 7 圆 2 矩形场景中，原始路径 47 个节点 → 平滑后仅 6 个节点，路径长度从 22.61 缩短到 19.75（减少了 12.7%）。

### 5. PPM 可视化

```cpp
void writePPM(const std::string& filename, int w, int h, ...) {
    // 白色背景
    std::vector<unsigned char> img(w * h * 3, 255);

    // 坐标变换：世界坐标 → 像素坐标
    auto worldToPixel = [&](const Vec2& p) -> std::pair<int,int> {
        int px = (int)((p.x - xmin) / (xmax - xmin) * w);
        int py = (int)((1.0 - (p.y - ymin) / (ymax - ymin)) * h);
        return {px, py};
    };

    // 1. 绘制障碍物（深灰 100,100,120）
    // 2. 绘制 RRT 搜索树（浅蓝 200,200,220）
    // 3. 绘制原始路径（蓝色 30,50,200）
    // 4. 绘制平滑路径（红色 255,30,30）——覆盖在蓝色之上
    // 5. 绘制起点（绿色 0,200,0）和终点（橙色 255,100,0）

    // PPM P6 格式：二进制 RGB
    std::ofstream out(filename, std::ios::binary);
    out << "P6\n" << w << " " << h << "\n255\n";
    out.write((const char*)img.data(), img.size());
}
```

**P6 格式的优点**：PPM P6（二进制格式）是最简单的图像格式之一——没有压缩，不需要任何库依赖，十几行代码就能实现。但缺点是文件体积较大（1400×1200×3 = 5MB），因此我们同时生成 PNG（31KB）作为发布格式。

**为什么覆盖顺序重要？** 先画树（浅蓝），再画原始路径（蓝），最后画平滑路径（红）。这样平滑后的最终路径会覆盖在原始路径之上，视觉上有"优化改进"的对比效果。

---

## 踩坑实录

### 坑 1：margin 参数的双重作用

**症状**：将 `margin` 从 0.1 改为 0 后，路径穿过了圆形障碍物的边缘。

**错误假设**：我以为 `isPointCollisionFree` 中已经检查了点是否在障碍物内，线段碰撞检测可以不需要 margin。

**真实原因**：RRT 的碰撞检测包含两部分——点检测和线段检测。`isPointCollisionFree` 确实会拒绝障碍物内的点，但步长为 0.5 的线段延伸可能**刚好擦过**障碍物的边缘（线段端点在障碍物外，但中间部分与障碍物的最近距离小于 step size 造成的离散化误差）。在没有 margin 时，二次方程的判别式可能刚好在临界值附近，浮点误差导致碰撞被误判为不碰撞。

**修复方式**：始终保留 margin（0.1），相当于在障碍物周围画一个"缓冲区"。在路径平滑时可以用 margin=0 以获得更逼近障碍物的路径。

### 坑 2：Goal Bias 过高导致死循环

**症状**：在障碍物密集的场景中，goalBias 设为 0.3 时，RRT 在 5000 次迭代后仍无法到达目标。

**错误假设**：更高的 goal bias 能更快到达目标，越多越好。

**真实原因**：当 goal bias 过高时（如 30%），树的大部分节点都在尝试向目标延伸。但目标可能被障碍物遮挡，导致这些延伸反复失败（碰撞检测不通过）。也就是说，大量的迭代次数被浪费在无效的"朝目标冲"上，而树的探索性增长（覆盖整个空间）不足。树在障碍物前"卡住了"。

**修复方式**：将 goalBias 降回 0.05。这保证了 95% 的迭代用于均匀探索（覆盖全空间），只有 5% 用于朝向目标。在 387 次迭代后就成功到达目标。

### 坑 3：路径回溯的索引顺序

**症状**：提取的路径顺序是"从目标到起点"的，后续平滑和渲染代码出错。

**错误假设**：我以为回溯自然就是从起点到目标。

**真实原因**：树的 `parent` 字段是从子节点指向父节点的。从目标叶子节点开始沿着 parent 链往上走，自然到达起点（parent=-1 的根节点）。因此路径数组的顺序是 `[goal, ..., start]`。

**修复方式**：在 path extraction 后加一行 `std::reverse(result.path.begin(), result.path.end())`，保证路径顺序始终是 `start → goal`。这个 bug 非常经典——几乎所有第一次实现 RRT 的人都会犯这个错误。

### 坑 4：线段完全在圆内的情况

**症状**：碰撞检测有时会漏报——整条线段完全在圆形障碍物内，但 `segmentCollidesCircle` 返回 false。

**错误假设**：如果二次方程的判别式 ≥ 0，则一定存在 t ∈ [0, 1]。

**真实原因**：当线段完全在圆内时，二次方程的两个根 t1 和 t2 满足 `t1 < 0 && t2 > 1`。这意味着从线段参数 t=0 到 t=1，整条线段都在圆内。如果只检查 `(t >= 0 && t <= 1)`，就会被漏报。

**修复方式**：添加第三个条件 `(t1 < 0 && t2 > 1)`。这个修复虽然是"一行代码"，但其背后的几何直觉是：二次方程的根代表了直线（无限延伸）与圆的参数范围，而我们需要的是线段（有限）与圆的覆盖关系。

---

## 效果验证与数据

### 量化验证（5 项全部通过）

```
=== Verification Results ===
  ✓ Start/goal position correct     — 起点终点位置精确匹配
  ✓ All 5 path segments collision-free — 所有路径段均无碰撞
  ✓ Path length (19.75) >= direct distance (16.40) — 合理
  ✓ Tree has sufficient nodes (282) — 树覆盖充分
  ✓ All interpolated path points collision-free — 细粒度采样验证
```

### 性能数据

| 指标 | 数值 |
|------|------|
| 总迭代次数 | 387 |
| 树节点数 | 282 |
| 原始路径节点 | 47 |
| 平滑后节点 | 6 |
| 原始路径长度 | 22.61 |
| 平滑后路径长度 | 19.75 |
| 路径缩短比例 | 12.7% |
| 起点到目标直线距离 | 16.40 |
| 路径/直线比 | 1.20 |

路径/直线比为 1.20，意味着平滑后的路径仅比理想直线长 20%——对于一个需要通过 9 个障碍物的场景来说，这是相当不错的路径质量。

### 可视化输出

![RRT Path Planning Result](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-18-rrt-path-planning/rrt_output.png)

图中可以清晰看到：
- **深灰色**：7 个圆形障碍物 + 2 个矩形障碍物
- **浅蓝色网状**：RRT 搜索树（282 个节点，覆盖了整个自由空间）
- **蓝色折线**：RRT 找到的原始路径（47 个节点，有明显 zigzag）
- **红色直线**：Shortcut Pruning 平滑后的最终路径（仅 6 个节点，几乎是最优路径）
- **绿色点**：起点 (0, 0.5)
- **橙色点**：终点 (13, 10.5)

观察树的结构可以发现，树明显向"空白区域"扩展（尤其是上方大空地），证明了 RRT 的 Voronoi 偏置效应确实有效。

---

## 总结与延伸

### 技术局限性

1. **O(n) 最近邻搜索**：本文实现使用线性扫描查找最近节点，复杂度 O(n²)。对于大规模场景（几万节点），应使用 KD-Tree 将最近邻搜索优化至 O(log n)
2. **非最优性**：基础 RRT 找到的路径不需要是最优的。虽然 shortcut pruning 有所改善，但无法保证全局最优
3. **狭窄通道困境**：RRT 在狭窄通道中表现不佳——在狭窄通道中采样的概率极低，树的扩展在瓶颈处可能"停滞"
4. **第二步路径平滑不保证碰撞安全**：如果 shortcut 的两端点跨度很大，线段可能与障碍物相交（虽然我们做了碰撞检测，但非常密集的障碍物地形中随机尝试的效率降低）

### 优化方向

1. **RRT\***：加入 rewire 机制，新节点加入后在 δ 半径邻域内检查是否可以通过其他路径以更低代价到达，实现渐近最优性
2. **RRT-Connect**：从起点和目标同时构建两棵树（dual-tree），双向扩展，速度可以提升 10-100 倍
3. **Informed RRT\***：找到可行路径后，将采样空间限制在由当前路径长度决定的椭圆内，大幅减少无效采样
4. **KD-Tree 加速**：最近邻搜索从 O(n) 优化到 O(log n)
5. **SDF（符号距离场）碰撞检测**：用距离场替代显式几何，碰撞检测变为 O(1) 的查表操作

### 本系列关联

RRT 是搜索类算法的里程碑，与本系列其他文章的关系：
- **Boid Flocking（07-12）**：集群行为与个体路径规划的互补——Boid 负责"群体智能"，RRT 负责"个体导航"
- **Octree（07-07）**：Octree 的空间划分可以加速 RRT 的碰撞检测和最近邻搜索
- 未来的 **A\* Pathfinding**（离散空间）与 RRT（连续空间）将形成完整对比

---

*完整代码见 [GitHub: daily-coding-practice/2026/07/07-18-rrt-path-planning](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/07/07-18-rrt-path-planning)*
