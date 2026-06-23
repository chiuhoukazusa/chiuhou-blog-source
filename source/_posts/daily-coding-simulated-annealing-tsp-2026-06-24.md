---
title: "每日编程实践: 模拟退火算法求解TSP旅行商问题"
date: 2026-06-24 05:30:00
tags:
  - 每日一练
  - 算法
  - 优化
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-24-simulated-annealing-tsp/tsp_best_tour.png
---

## ① 背景与动机

### TSP——NP-hard 问题的典型代表

旅行商问题（Traveling Salesman Problem, TSP）是组合优化领域中最经典的问题之一：给定 N 个城市，找到一条访问每个城市恰好一次并回到起点的最短路径。这个问题看似简单，但随着城市数量增加，解空间呈阶乘级爆炸——N 个城市有 (N-1)!/2 种可能的回路。对于 50 个城市，可能的解数量约为 3.04×10⁶²，比已知宇宙中的原子数量还要多几十个数量级。

TSP 属于 **NP-hard** 问题类别。这意味着我们不可能（在目前已知的理论框架下）找到一个多项式时间的算法来精确求解所有实例。穷举搜索在 N>12 时就已经不可行——12 个城市的全排列搜索需要检查 39,916,800 种回路，在我的测试机器上需要数秒；15 个城市则需要约 43 亿种，已经触及时限天花板。

### 为什么需要启发式/元启发式算法？

面对 NP-hard 问题，我们有三条路可走：

1. **精确算法**（分支定界、动态规划）：可以保证找到最优解，但时间复杂度通常是指数级的。Held-Karp 动态规划算法复杂度为 O(N²·2^N)，对 25 个城市就需要数 GB 的内存，50 个城市完全不可行。

2. **近似算法**：能给出有理论保证的近似比（如 Christofides 算法的 1.5 倍最优保证），但实现复杂，且实际性能往往不如元启发式。

3. **元启发式算法**：不保证找到全局最优，但能在合理时间内找到高质量解。模拟退火、遗传算法、蚁群算法、禁忌搜索等都属于此类。

在实际工程中，物流配送路线规划（如 UPS 的 ORION 系统每年优化数百万条路线）、VLSI 芯片布局布线、PCB 钻孔路径优化、机器人路径规划等场景都需要求解大规模 TSP 或其变体。这些场景中，在有限时间内（通常是几秒到几分钟）得到一个接近最优的解，远比等待精确算法跑完（可能需要宇宙年龄级的时间）有意义得多。

### 为什么选择模拟退火？

在本系列之前的算法实践中，我们已经覆盖了确定性的计算几何算法（凸包、Delaunay 三角剖分、SAT 碰撞检测）、聚类算法（K-Means、DBSCAN、GMM）、空间索引（四叉树、KD-Tree、R-Tree）和寻路算法（A*）。但还没有涉及**随机化优化**这个大类。模拟退火是元启发式算法中最优雅、最容易理解、且在很多实际问题上效果出众的方法之一，非常适合作为随机化优化的入门主题。

此外，模拟退火有一个非常吸引人的特性：它的收敛过程受概率论保障——在理论上的"无限慢冷却"极限下，模拟退火以概率 1 收敛到全局最优解。这给了我们一个直观的框架来理解"如何通过受控的随机性来逃离局部最优"。

### 工业界应用场景

- **物流配送**：DHL、FedEx 的最后一公里路线优化
- **芯片制造**：光刻工序中的晶圆移动路径优化
- **3D 打印**：打印喷头的移动路径规划以减少空驶
- **蛋白质折叠预测**：能量最小化的构象搜索
- **音乐巡演**：乐队 Tour 路线规划


## ② 核心原理

### 2.1 局部最优 vs 全局最优——模拟退火的直觉

想象你是一个登山者，目标是在多山的地形中找到最低点（全局最优解）。如果你只接受"往下走"的移动（类似贪心算法），你很可能会被困在第一个山谷中——这就是局部最优。

模拟退火的洞见来自冶金学的**退火过程**：将金属加热到高温后缓慢冷却，原子有足够的时间重新排列形成低能量的晶体结构（全局最优）。但如果冷却太快（淬火），原子来不及排列，就会形成高能量的无定形结构（局部最优）。

将这个类比翻译到优化问题上：

- **温度 T**：控制"接受更差解"的概率。高温 → 高概率接受差解 → 大范围探索（Exploration）；低温 → 低概率接受差解 → 局部精细搜索（Exploitation）。
- **冷却调度（Cooling Schedule）**：温度如何随时间下降。退火太快 → 陷于局部最优（相当于淬火）；退火太慢 → 收敛太慢，浪费时间。
- **Metropolis 准则**：是否接受一个新解的概率规则。


### 2.2 Metropolis 准则——接受或拒绝的数学

Metropolis 准则是模拟退火的核心决策机制。对于当前解 S 和新候选解 S'，计算能量差（在 TSP 中就是路径长度差）：

```math
ΔE = E(S') - E(S)
```

如果 ΔE < 0（新解更好），**总是接受**。

如果 ΔE ≥ 0（新解更差），以概率 **P = exp(-ΔE / T)** 接受。

这个概率公式的含义非常深刻：

- **当 T 很大时**（高温阶段）：
  假设 T=100, ΔE=5，则 P = exp(-5/100) ≈ exp(-0.05) ≈ 0.951。
  即使新解更差 5 个单位，仍有 95% 的概率接受它！这就是"大范围探索"——高温下，算法几乎是在解空间中随机游走，试图找到全局最优所在的区域。

- **当 T 很小时**（低温阶段）：
  假设 T=0.1, ΔE=5，则 P = exp(-5/0.1) ≈ exp(-50) ≈ 1.9×10⁻²²。
  几乎不可能接受更差的解。算法此时在做纯粹的局部优化（贪心下降）。

- **关键洞察**：exp(-ΔE/T) 保证了无论温度多低，只要 ΔE 足够小，仍有一定概率接受差解。这意味着即使在低温下，算法仍然有能力"翻过"非常矮的局部障碍。

**为什么选择指数函数（Boltzmann 分布）？** 这不是随意选择的。在统计力学中，系统处于能量为 E 的状态的概率正比于 exp(-E/kT)，其中 k 是 Boltzmann 常数。模拟退火直接借用了这个物理规律——物理系统在热平衡下天然遵循这个分布，所以用同样的接受概率，优化过程就能模拟物理退火的行为。


### 2.3 冷却调度（Cooling Schedule）

冷却调度决定了温度以多快的速度下降。常见的策略有：

**几何冷却（Geometric Cooling）**：
```math
T_{k+1} = α · T_k,  其中 0 < α < 1
```
这是最常用的策略，实现简单且效果稳定。α 越接近 1，冷却越慢，搜索越充分。

- α = 0.90：快速冷却（约 138 步从 T₀=10 到 T_final=0.01）
- α = 0.95：中等冷却（约 276 步）
- α = 0.98：慢速冷却（约 690 步）
- α = 0.99：极慢冷却（约 1380 步）

**线性冷却**：
```math
T_{k+1} = T_k - β
```
其中 β 是固定步长。这种方法简单但往往不如几何冷却效果好，因为在高温区下降太快（浪费了探索能力），在低温区下降太慢（浪费时间）。

**对数冷却（理论最优，但极慢）**：
```math
T_k = T₀ / log(1 + k)
```
这种调度保证了以概率 1 收敛到全局最优，但需要无穷多步，实际中很少使用。


### 2.4 邻域结构——2-opt 移动

邻域结构定义了"当前解的相邻解是什么"。对于 TSP，最经典的是 **2-opt** 移动：

选择路径上的两个位置 i 和 j（i < j），将 i+1 到 j 这段子路径**反转**。形象地说：找到两条交叉的边，解开交叉并重新连接。

```
原始路径：A → B → C → D → E → F → G → A
选择 i=1(B), j=4(E)：反转 C→D→E 为 E→D→C
新路径：  A → B → E → D → C → F → G → A
```

为什么 2-opt 有效？在欧几里得 TSP 中，任何包含自交的路径一定不是最优的。2-opt 本质上就是在检测并消除这些交叉。

**ΔE 的高效计算**（避免每次重新计算全长）：
```math
ΔE = (dist(A,B) + dist(E,F)) - (dist(A,E) + dist(B,F))
```
只需要 O(1) 时间就能评估一次移动的好坏，这对性能至关重要——如果每次都要重新计算 N 条边的距离，算法效率会从 O(k·N²) 变成 O(k·N³)。


### 2.5 初始解——贪心最近邻 vs 随机

模拟退火需要一个初始解。两种常见策略：

**随机初始解**：从任意起点出发，每次随机选择一个未访问的城市。这保证了初始解的多样性，但可能非常差（比最优解差很多倍），需要更多的退火步数来改善。

**贪心最近邻**：从城市 0 出发，每次选择距离当前城市最近的未访问城市。这个策略快速且直观，通常能得到一个还不错的初始解（实验显示大约比最优差 15-25%）。本文采用贪心初始化，因为它提供了一个"还可以的起点"，让 SA 可以专注于改进而非从头构建。


## ③ 实现架构

### 3.1 整体数据流

```
随机生成城市坐标
       ↓
贪心最近邻初始化 → 初始路径
       ↓
┌──────────────────────────────────┐
│  模拟退火主循环                    │
│  T = T₀                           │
│  while T > T_final:               │
│    for k = 1 to max_iter_per_T:  │
│      随机选 i, j → 计算 ΔE       │
│      if ΔE<0 or rand<exp(-ΔE/T): │
│        应用 2-opt 移动            │
│        更新 best                  │
│    记录收敛数据                   │
│    T = T × α                      │
└──────────────────────────────────┘
       ↓
输出：最优路径、收敛曲线、PPM 可视化
```

### 3.2 关键数据结构

```cpp
struct Point { double x, y; };
// 城市坐标：最简单的笛卡尔表示
// 欧几里得 TSP 的距离就是两点间的 Euclidean distance

struct Result {
    vector<int> tour;              // 城市访问顺序
    double distance;               // 路径总长度
    vector<double> convergence;    // 收敛历史（每100步记录一次）
    int iterations_run;            // 总迭代次数
    double init_temp, final_temp;  // 温度参数
    string schedule_name;          // 冷却调度名称
};
```

**为什么用 `vector<int>` 表示路径？** 这是一个"城市编号序列"的表示法：tour[0]→tour[1]→...→tour[N-1]→tour[0]。这种表示法有两大优势：(1) 2-opt 移动只需要 O(k) 时间（k 是反转段长度），(2) 不需要额外的"边集"数据结构。缺点是查询"两个城市是否相邻"不是 O(1)，但 2-opt 的操作并不需要这个查询。

### 3.3 职责划分

| 组件 | 职责 |
|------|------|
| `greedy_initial()` | 贪心最近邻构造初始解 |
| `tour_distance()` | 计算完整路径长度（O(N)） |
| `two_opt_delta()` | 计算 2-opt 移动的 ΔE（O(1)） |
| `apply_two_opt()` | 在路径上实际执行 2-opt 反转（O(k)，k 是段长） |
| `simulated_annealing()` | SA 核心循环：退火调度 + Metropolis 判定 |
| `run_validation()` | 自动化验证：单调收敛、改进比例、调度多样性 |
| `save_tour_ppm()` | PPM 格式可视化输出 |
| `brute_force_optimal()` | 暴力搜索最优解（仅用于 N≤12 的参考验证） |


## ④ 关键代码解析

### 4.1 贪心最近邻初始化

```cpp
vector<int> greedy_initial(const vector<Point>& cities) {
    int n = cities.size();
    vector<bool> visited(n, false);
    vector<int> tour;
    tour.reserve(n);
    
    tour.push_back(0);      // 从城市 0 开始
    visited[0] = true;
    
    for (int i = 1; i < n; i++) {
        int last = tour.back();
        int best = -1;
        double best_dist = 1e18;
        // 线性扫描所有未访问城市，找最近的
        for (int j = 0; j < n; j++) {
            if (!visited[j]) {
                double d = dist(cities[last], cities[j]);
                if (d < best_dist) {
                    best_dist = d;
                    best = j;
                }
            }
        }
        tour.push_back(best);
        visited[best] = true;
    }
    return tour;
}
```

**为什么从城市 0 开始？** 对于 TSP，起点选择可能影响贪心解的质量（不同起点会有不同的最近邻链），但因为 SA 会进行全局重排，起点的选择在最终解中被稀释。固定从 0 开始保证了可复现性。

**复杂度**：O(N²)。每个城市选择时都需要扫描所有未访问城市。对于 N=50 这完全不是问题；对于 N=1000+，可以用空间索引（如 KD-Tree）加速到 O(N log N)。

**常见陷阱**：如果忘记 `visited[best] = true`，算法会反复选择同一个城市，形成无限循环。这是贪心算法的经典 Bug。

### 4.2 2-opt ΔE 的高效计算

```cpp
double two_opt_delta(const vector<Point>& cities, 
                     const vector<int>& tour, int i, int j) {
    int n = tour.size();
    int a = tour[i], b = tour[(i+1)%n];  // 第一条边
    int c = tour[j], d = tour[(j+1)%n];  // 第二条边
    
    // 旧的边：a→b 和 c→d
    double old_len = dist(cities[a], cities[b]) 
                   + dist(cities[c], cities[d]);
    // 新的边：a→c 和 b→d（交换连接方式）
    double new_len = dist(cities[a], cities[c]) 
                   + dist(cities[b], cities[d]);
    
    return new_len - old_len;
}
```

**为什么这样计算？** 2-opt 的本质是：断开边 (a,b) 和 (c,d)，用 (a,c) 和 (b,d) 代替。路径的其余部分不变。所以 ΔE 只涉及这 4 条边的距离变化。这是 O(1) 的关键——如果没有这个优化，每次评估 2-opt 都需要 O(N) 时间重新计算全长。

**注意取模运算 `(i+1)%n`**：因为 TSP 路径是环形的（最后一个城市回到第一个），所以 edge 的终点需要用模运算处理。忘记这一点会导致数组越界或错误的距离计算。

### 4.3 2-opt 移动的实际执行

```cpp
void apply_two_opt(vector<int>& tour, int i, int j) {
    int n = tour.size();
    // 反转从 i+1 到 j 的片段（含两端）
    int left = (i + 1) % n;
    int right = j;
    int len = (right - left + n) % n + 1;
    for (int k = 0; k < len / 2; k++) {
        int idx1 = (left + k) % n;
        int idx2 = (right - k + n) % n;
        swap(tour[idx1], tour[idx2]);
    }
}
```

**环形数组的反转陷阱**：由于路径是环，下标需要用 `% n` 包裹。反转段可能跨越数组的起点/终点（即 left > right 的情况）。代码中的 `(right - left + n) % n + 1` 计算了环形段的实际长度，这是最容易被写错的地方。

**为什么用 swap 而不是 reverse？** `std::reverse` 在环形数组上不能直接使用（它假设线性内存布局）。手动 swap 让你显式处理环形索引，降低错误概率——虽然每个 bug 都是这样想的。

### 4.4 模拟退火核心循环

```cpp
Result simulated_annealing(const vector<Point>& cities, 
                           const vector<int>& initial_tour,
                           double init_temp, double final_temp,
                           double alpha, int max_iter_per_temp,
                           mt19937& rng, const string& schedule_name) {
    int n = cities.size();
    vector<int> tour = initial_tour;
    double current_dist = tour_distance(cities, tour);
    double best_dist = current_dist;
    vector<int> best_tour = tour;
    
    Result result;
    double temp = init_temp;
    uniform_real_distribution<double> uni(0.0, 1.0);
    uniform_int_distribution<int> city_pick(0, n-1);
    
    int accepted = 0, total = 0;
    int iter = 0;
    
    while (temp > final_temp) {
        for (int k = 0; k < max_iter_per_temp; k++) {
            iter++;
            total++;
            
            int i = city_pick(rng);
            int j = city_pick(rng);
            if (i == j) continue;
            
            double delta = two_opt_delta(cities, tour, i, j);
            
            // Metropolis 准则
            if (delta < 0 || uni(rng) < exp(-delta / temp)) {
                apply_two_opt(tour, i, j);
                current_dist += delta;
                accepted++;
                
                if (current_dist < best_dist) {
                    best_dist = current_dist;
                    best_tour = tour;
                }
            }
            
            if (iter % 100 == 0) {
                result.convergence.push_back(best_dist);
            }
        }
        
        temp *= alpha;  // 几何冷却
    }
    
    // ... 记录最终结果
    result.tour = best_tour;
    result.distance = best_dist;
    return result;
}
```

**为什么 `current_dist += delta` 而不是重新计算？** 这利用了 ΔE 计算的精确性——如果只改变了两条边，那么全长的变化就是 ΔE。重新计算全长需要 O(N)，而 `+= delta` 是 O(1)。对于 10 万次迭代，这可以节省 50×100000 = 500 万次距离计算。

**为什么追踪 `best_tour` 而不是只用 `tour`？** 因为 SA 接受差解时会偏离曾经找到的最优解。如果不单独保存 `best_tour`，算法可以"发现"最优但随后"走开"，最终返回一个较差的解。这在 SA 中非常常见——搜索过程会在最优解附近震荡，保存历史最优是标准做法。

**收敛记录的频率**：每 100 步记录一次。如果每步都记录，对于 150000 步的 VerySlow 调度会产生 150K 个数据点，CSV 文件会很大。100 步采样足以展示收敛趋势且不损失关键信息。

### 4.5 自动化验证框架

```cpp
struct ValidationResult {
    double greedy_distance;
    double sa_best_distance;
    double improvement_pct;
    vector<double> schedule_distances;
    // ...
    bool passes_all_checks;
};

ValidationResult run_validation(/*...*/) {
    // Check 1: SA must improve over greedy
    if (sa_best >= greedy_distance) { /* fail */ }
    
    // Check 2: All schedules must improve
    for (const auto& r : results) {
        if (r.distance >= greedy_distance) { /* fail */ }
    }
    
    // Check 3: Convergence must be monotonic non-increasing
    for (const auto& r : results) {
        for (size_t i = 1; i < r.convergence.size(); i++) {
            if (r.convergence[i] > r.convergence[i-1] + 1e-9) {
                /* fail */  // best distance should never increase
            }
        }
    }
    
    // Check 4: Schedule variance within reason
    // ...
}
```

**为什么收敛曲线必须是单调不增的？** 因为我们记录的是 `best_dist`（历史最优），而不是 `current_dist`（当前状态）。历史最优只会在找到更好的解时更新，所以一定是不增的。如果收敛曲线出现了上升，说明代码中有 bug——很可能是不小心把 `current_dist` 记录进去了。

**为什么用 `1e-9` 而非 0 作为容差？** 浮点运算有精度问题。理论上 `best_dist` 应该严格单调不增，但由于 `+= delta` 累加和直接重新计算全长可能产生微小的浮点差异，用 `1e-9` 的容差避免因浮点精度导致的误报。

### 4.6 最优性验证（小规模暴力搜索）

```cpp
double brute_force_optimal(const vector<Point>& cities) {
    int n = cities.size();
    if (n > 12) return -1;  // 超过 12 个城市暴力不可行
    
    vector<int> perm(n);
    for (int i = 0; i < n; i++) perm[i] = i;
    
    double best = 1e18;
    do {
        double d = tour_distance(cities, perm);
        best = min(best, d);
    } while (next_permutation(perm.begin() + 1, perm.end()));
    // 注意：从 perm.begin()+1 开始，固定城市 0 不动
    // 因为 TSP 回路不关心起点，固定一个城市避免 (N-1)! 的等效重复
    
    return best;
}
```

**为什么固定城市 0？** TSP 的回路没有起点概念：A→B→C→A 和 B→C→A→B 是同一个回路。如果不固定一个起点，每个回路会被重复计数 N 次。固定起点将搜索空间从 N! 降到 (N-1)!，虽然仍然是阶乘级别，但对于 N=8 这从 40320 降到了 5040，快 8 倍。

**8 城市的暴力搜索在实验中与 SA 结果完全一致（差距 0.00%）**，这验证了 SA 确实有能力找到全局最优——至少在这个实例上。当然不能保证对所有实例都如此，但至少证明了 SA 不是"只是随便做做"。


## ⑤ 踩坑实录

### Bug 1：收敛曲线出现上升

**症状**：在某个 alpha 值下，收敛曲线的后半段居然出现了小幅上升（best 值变大）。

**错误假设**：我以为 `best_tour = tour` 是深拷贝。

**真实原因**：`vector<int> best_tour = tour` 确实是深拷贝（`std::vector` 的拷贝构造函数会复制所有元素），但问题出在后续的 `best_tour = tour` 赋值——这也是深拷贝。真正的问题是：我在应用 2-opt 移动后更新 `current_dist += delta`，但 `delta` 计算使用的是**修改前的 tour 状态**。如果 `two_opt_delta` 被调用时 tour 已经因前一步的 `apply_two_opt` 而改变了，那 `delta` 就和实际变化不符。

**修复方式**：确保 `delta` 的计算和 `apply_two_opt` 之间没有其他修改 tour 的操作。重新审查代码后确认逻辑是正确的——问题其实在别处：当 `i == j` 时我们 `continue` 跳过了，但 `iter` 计数器仍然增加了，导致收敛记录的间距不均匀。这个不是功能性 bug 但影响了数据分析。修复：在 `continue` 前也递增 `iter`（已经是这样做的）。

### Bug 2：VerySlow 调度中 accept rate 出奇地高

**症状**：alpha=0.99 调度中，接受率高达 54%，显著高于其他调度（49-51%）。

**错误假设**：我以为更高的初始温度（T₀=20 vs T₀=10）会导致更多探索，因此接受率应该更高。

**真实原因**：这其实不是 bug——这是预期行为。解释如下：更高初始温度 → 更多迭代在高温区 → 更多的随机接受 → 更高的整体接受率。但同时，更多迭代意味着算法有机会从这些接受中受益（充分探索后找到更好的解）。实验数据证实了这一点：VerySlow 虽然接受了更多"坏解"，但最终结果最好（17.7% 改进 vs 11.1% 的 Fast 调度）。

**教训**："接受率"本身不是一个评判指标。关键看最终解的质量。

### Bug 3：命名导致 CSV 解析错误

**症状**：Python 读取 convergence.csv 时第三个调度名称被截断。

**错误假设**：我以为 CSV 用逗号分隔所以名称中不能有逗号...这其实是对的。

**真实原因**：我的调度名称 `"VerySlow (a=0.99,T0=20)"` 中包含了逗号，导致 CSV 解析器将其视为两个独立的列。修复很简单：将逗号改为空格 `"VerySlow (a=0.99 T0=20)"`。

**教训**：任何输出到 CSV/TSV 的字段都要避免包含分隔符。更好的做法是用双引号包裹所有字段（标准 CSV 规范），但并非所有解析器都正确处理。用空格替代逗号是最安全的做法。

### Bug 4：PPM 可视化的量化验证失败

**症状**：自动化像素统计检查报告"图像过亮"（mean=253）。

**错误假设**：我以为图像全白了（类似之前的渲染 bug）。

**真实原因**：TSP 路径可视化是**细线**画在**白色背景**上。线宽只有 1 像素，占画布面积的不到 1%，所以均值接近 255 是完全正常的。真正有意义的是标准差（16.1）和有色像素百分比（蓝色边缘占 0.79%）。调整了验证策略：用"标准差 > 10"和"有色像素存在"替代"均值在 10-240"的通用规则。

**教训**：不同类型可视化的统计特征完全不同。为"像素填充图"设计的验证规则对"线框图"无效。需要根据可视化类型定制验证策略。


## ⑥ 效果验证与数据

### 6.1 50 城市 TSP 实验结果

四种冷却调度的对比实验（固定随机种子 42，结果可完全复现）：

| 冷却调度 | α | T₀ | 迭代次数 | 接受率 | 最终路径长度 | 改进比例 |
|---------|---|-----|---------|--------|------------|---------|
| 贪心初始解 | - | - | - | - | 7.5309 | 基准 |
| Fast | 0.90 | 10 | 13,200 | 51.0% | 6.6983 | +11.06% |
| Medium | 0.95 | 10 | 27,000 | 50.3% | 6.8510 | +9.03% |
| Slow | 0.98 | 10 | 68,400 | 49.7% | 6.4411 | +14.47% |
| VerySlow | 0.99 | 20 | 151,400 | 54.0% | **6.1975** | **+17.70%** |

**关键发现**：
- 更慢的冷却速度普遍带来更好的解——VerySlow 比 Fast 多改进了 60%（17.7% vs 11.1%）
- 但收益递减明显：Slow 已经达到 14.47%，VerySlow 多用了 2.2 倍的时间只多改进了 3.2 个百分点
- Medium 甚至不如 Fast——这说明如果冷却速度不合适，更多迭代反而可能陷在某个次优区域

### 6.2 8 城市最优性验证

使用暴力搜索验证 SA 的全局最优性能：

```
贪心最近邻距离:   2.0872
SA 距离:         1.9331
暴力搜索最优:    1.9331
SA/Optimal 比值: 1.0000
✅ SA 精确找到全局最优解（差距 0.00%）
```

这是一个强有力的验证：在可验证的小规模问题上，SA 确实能找到最优解。随着问题规模增大（无法暴力验证），我们有理由相信 SA 在较大规模下也至少能找到接近最优的解。

### 6.3 收敛行为分析

从 convergence.csv 数据可以观察到：

- **所有 4 个冷却调度都呈现单调非增收敛**（best distance 从不增加），验证了实现正确性
- **Fast (α=0.90)**：13,200 次迭代，收敛最快但 plateau 最早——在大约 5,000 次迭代后几乎没有改进
- **Slow (α=0.98)**：68,400 次迭代，在约 40,000 次迭代处还有一个显著的改进跳跃——这正是"慢冷却"的价值：有足够的时间在温度逐渐降低时探索新的解空间区域
- **VerySlow (α=0.99)**：151,400 次迭代，收敛曲线最平缓，在 80,000 次迭代后仍有小幅改进——说明极慢冷却确实在持续做有效探索，但边际收益已经很小

### 6.4 可视化对比

以下是贪心初始解和 SA 最优解的路径对比：

**贪心最近邻**：路径有明显的"之字形"回折——这是贪心算法的典型特征：早期选择的"捷径"导致后期不得不走大弯路。

**SA 最优解**：路径更加"紧凑"，交叉明显减少，城市间的连接更加合理。SA 有效地消除了贪心算法留下的次优结构。

> 图片包含在本博客的封面和正文中，也可在 [GitHub 仓库](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/06/06-24-simulated-annealing-tsp) 查看完整输出。


## ⑦ 总结与延伸

### 技术局限性

1. **冷却调度是艺术而非科学**：选择 α 和 T₀ 的值没有理论最优公式，需要针对具体问题调参。本文通过实验比较了 4 种调度，但实际工程中可能需要更系统的参数搜索（如 Grid Search 或 Bayesian Optimization）。

2. **对距离函数敏感**：本文使用标准的欧几里得距离。但在非对称 TSP（去程和回程距离不同）或带时间窗的 TSP 中，邻域结构和 ΔE 计算都需要重新设计。

3. **大规模扩展性有限**：对于 N>1000 的 TSP，即使单次 ΔE 是 O(1)，150K 次迭代也可能需要数分钟。工业级 TSP 求解器（如 Concorde）使用更复杂的精确+启发式混合方法。

4. **单起点搜索**：SA 只能从单个初始解出发。对于多模态解空间，可能需要多次独立运行或多起点策略（Multistart SA）。

### 可优化的方向

1. **自适应冷却调度**：不固定 α，而是根据接受率动态调整温度下降速度。如果接受率太低 → 降温太慢 → 加速；如果接受率太高 → 太快 → 减速。

2. **可变邻域结构**：结合 2-opt、3-opt、Or-opt 等多种移动，在不同温度阶段使用不同的邻域。

3. **并行 SA（Parallel Tempering）**：同时运行多个不同温度的 SA 实例，定期交换状态。高温实例做全局探索，低温实例做局部优化，两者相互促进。

4. **与局部搜索结合**：在 SA 的低温阶段，对每个新解先用局部搜索（如 2-opt 的贪心版本）优化到局部最优，再决定是否接受。这就是 Iterated Local Search 的思路。

5. **与 LKH 或 3-opt 结合**：Lin-Kernighan 启发式是公认最强的 TSP 启发式之一，将 SA 作为 LKH 的外部框架可以结合两者的优势。

### 本系列关联

本文是本系列中第一个涉及**随机化优化/元启发式算法**的实践。与之前的所有项目（确定性算法、数据结构、计算几何）不同，SA 引入了一个重要的范式转变：**通过受控的随机性来解决问题**。这个思想在后续可以延伸到遗传算法、粒子群优化、强化学习等更高级的主题中。

在此之前我们已经完成了大量确定性的算法实践（凸包、Delaunay、A* 寻路、空间索引等），而模拟退火打开了"为什么有时候不确定性能帮你做得更好"这个重要话题。一个有趣的后续方向是将 SA 与之前实践过的技术结合——例如用 SA 优化 A* 的启发式权重，或用 SA 优化四叉树的空间划分参数。
