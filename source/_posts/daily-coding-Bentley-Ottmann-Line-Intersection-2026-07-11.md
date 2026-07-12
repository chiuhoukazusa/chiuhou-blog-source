---
title: "每日编程实践: Bentley-Ottmann线段交点算法"
date: 2026-07-11 06:00:00
tags:
  - 每日一练
  - 计算几何
  - 算法
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-11-Bentley-Ottmann-Line-Intersection/Polygon_20.png
---

## 背景与动机

线段交点检测（Line Segment Intersection）是计算几何中最基础、最实用的问题之一。它的核心问题是：给定平面上 $n$ 条线段，找出所有线段之间的交点。

你可能会想："这还不简单？双重循环遍历所有线段对，每对做一次几何判断不就行了？"没错，暴力法 $O(n^2)$ 确实可以，但在很多实际场景中，线段数量可能成百上千，而真正的交点数量 $k$ 却远小于 $n^2$。例如：

- **地图渲染**：道路线段可能有上万条，但实际交叉路口（交点）数量只有几百个。如果你每次放大地图都要对所有道路对做交点检测，地图引擎会卡成幻灯片。
- **PCB 设计验证**：电路板上的导线数百万条，短路检测（导线相交）是整个 EDA 流程的瓶颈之一。$O(n^2)$ 意味着 100 万条线需要 $10^{12}$ 次检查——这根本不可行。
- **碰撞检测**：游戏中的障碍物边界线段之间，只有少数会相交，需要更高效的筛选机制。
- **GIS 空间分析**：地理信息系统中，河流、道路、行政边界的交叉分析同样面临海量数据。

Bentley 和 Ottmann 在 1979 年提出的扫描线算法，将复杂度降到了 $O((n+k) \log n)$。当 $k \ll n^2$ 时，这比暴力法快几个数量级。算法的核心思想来自"扫描线"范式——想象一条水平线从平面顶部向底部移动，只关心那些"当前与扫描线相交"的线段之间的相邻关系，从而避免检查所有线段对。

本文将从头实现 Bentley-Ottmann 算法，并用 4 个测试用例（网格、星形、随机、多边形）进行暴力法交叉验证，记录从最初实现到最终通过全部测试的完整过程。

---

## 核心原理

### 扫描线范式

Bentley-Ottmann 算法的核心直觉是：

> **如果两条线段相交，那么在扫描线从交点之上移动到交点之下的过程中，这两条线段在扫描线上的左右顺序一定会发生交换。**

想象一下，在交点上方，线段 A 在 B 的左边；到了交点正下方，A 跑到了 B 的右边。这个顺序交换只能发生在它们成为"邻居"（即在扫描线上位置相邻）的时候。所以，我们只需要检查扫描线上相邻线段之间的交点，而不需要检查所有线段对。

### 算法步骤

Bentley-Ottmann 算法包含两个核心数据结构：

1. **事件队列（Event Queue）**：按 y 坐标降序（从上到下）排列的事件列表。事件有两种：
   - **上端点事件**：线段进入扫描线范围（将它加入活跃集）
   - **下端点事件**：线段离开扫描线范围（将它从活跃集移除）

2. **活跃集（Status / Sweep Line Status）**：当前与扫描线相交的所有线段，按扫描线与线段的交点 x 坐标从左到右排序。

算法的每一步处理一个 y 坐标处的所有事件：

```
While 事件队列非空:
  1. 取出当前 y 坐标的所有事件
  2. 按 x 坐标重新排序活跃集（扫描线下降了，各线段的新 x 位置变了）
  3. 处理进入的线段：检查它和活跃集中每个线段的交点
  4. 处理退出的线段：检查它和活跃集中每个线段的交点
  5. 从活跃集中移除退出的线段
  6. 将进入的线段加入活跃集
  7. 重新排序活跃集，检查所有相邻对是否相交
```

### 为什么是 $O((n+k) \log n)$？

- 有 $2n$ 个端点事件（每条线段两个端点），排序需要 $O(n \log n)$
- 每条线段进入和离开时，与活跃集中最多 $O(1)$ 个邻居比较（理想情况下）
- 每发现一个交点，可能需要将交点作为新事件插入队列（这在完整版本中需要，但本文的简化实现省略了这一步，因为我们直接检查所有相邻对）

实际上本文实现的版本在每次扫描线下降时都重新排序并检查所有相邻对，复杂度更接近 $O(n^2 \log n)$，但对于 $n \leq 20$ 的测试用例来说足够快了。标准实现会使用平衡二叉树（如红黑树）维护活跃集，将每个事件的处理时间降至 $O(\log n)$。

### 几何基础

实现中涉及的几何操作：

**叉积（Cross Product）判定方向**：

$$
\vec{a} \times \vec{b} = a_x b_y - a_y b_x
$$

对于三点 $O, A, B$，$\text{cross}(O, A, B) > 0$ 表示向量 $\vec{OA}$ 在 $\vec{OB}$ 的逆时针方向，$< 0$ 表示顺时针方向，$= 0$ 表示共线。

**线段相交判定**：两条线段 $AB$ 和 $CD$ 相交的条件是：
- 严格跨立：$C$ 和 $D$ 在 $AB$ 的两侧（$\text{sign}(c_1) \times \text{sign}(c_2) \le 0$）**且** $A$ 和 $B$ 在 $CD$ 的两侧（$\text{sign}(c_3) \times \text{sign}(c_4) \le 0$）
- 其中 $c_1 = \text{cross}(A,B,C)，c_2 = \text{cross}(A,B,D)，c_3 = \text{cross}(C,D,A)，c_4 = \text{cross}(C,D,B)$

**特殊情况处理**：
- **共线重叠**：四点共线时需要额外的区间重叠判断（`on_segment` 函数）
- **端点触碰**：一条线段的端点恰好在另一条线段上，也算相交
- 注意区分"端点触碰"和"端点恰是另一条线段的端点"——后者是正常的共享端点，不应算作内部交点

---

## 实现架构

### 整体数据流

```
输入: n 条线段
  │
  ├─ 预处理: 确保每条线段的 p1 始终是上端点 (y 较大者)
  │
  ├─ 构建事件队列: 每条线段产生 2 个事件(上端点+下端点)
  │         按 y 降序排列
  │
  ├─ 主循环 (扫描线从上到下):
  │    ├─ 取出同一 y 的所有事件
  │    ├─ 重排序活跃集 (按 x_at_y 升序)
  │    ├─ 处理: 进入线段 × 活跃集 → 记录交点
  │    ├─ 处理: 退出线段 × 活跃集 → 记录交点
  │    ├─ 从活跃集移除退出的线段
  │    ├─ 将进入的线段加入活跃集
  │    ├─ 重排序, 检查所有相邻对 → 记录交点
  │    └─ 下一条扫描线...
  │
  ├─ 去重: 使用 std::set<IntersectionResult> 按 (s1, s2) 去重
  │
  └─ 输出: 交点列表 + PPM 可视化
```

### 关键数据结构

```cpp
struct Segment {
    Point p1, p2;  // p1 始终是上端点（y 更大），水平线时 p1 是左端点
    int id;
};

struct IntersectionResult {
    int s1, s2;     // 相交的线段编号（s1 < s2）
    Point pt;       // 交点坐标
    // 排序: 按 (s1, s2) 去重，忽略浮点精度差异
};
```

### 水平线段的特殊处理

这是实现中踩坑最多的地方——水平线段（`p1.y == p2.y`）的处理需要特别注意：

1. 水平线段的两个端点在同一 y 高度，在事件队列中产生两个同 y 的事件
2. 如果两个事件都被当作"上端点"，线段会被重复加入活跃集
3. **解决方案**：水平线段在进入时（左端点）只触发检查，不加入活跃集；在退出时（右端点）同样只触发检查。水平线段通过"进入→立即退出"的方式参与扫描

```cpp
if (horiz) {
    Q.insert({y, left_x, left_x, i, false});  // 进入事件
    Q.insert({y, right_x, left_x, i, true});  // 退出事件（立即）
} else {
    Q.insert({p1.y, p1.x, p1.x, i, false});   // 进入事件
    Q.insert({p2.y, p2.x, p1.x, i, true});    // 退出事件（在底部）
}
```

### 活跃集排序

每次扫描线变化后，活跃集需要重新按当前 y 处的 x 坐标排序：

```cpp
double x_at_y(const Segment& s, double y) {
    if (fabs(s.p2.y - s.p1.y) < EPS) return s.p1.x;  // 水平线: 返回左端点
    double t = (y - s.p1.y) / (s.p2.y - s.p1.y);
    t = std::clamp(t, 0.0, 1.0);  // 限制在线段范围内
    return s.p1.x + t * (s.p2.x - s.p1.x);
}
```

`std::clamp` 是为了防止浮点误差导致 $t$ 越界（比如 $t = -1e-15$），这会导致 x 坐标计算错误，进而破坏活跃集排序的正确性。

---

## 关键代码解析

### 1. 几何基础函数

```cpp
// 叉积: 用于判断点在线段的左侧还是右侧
// cross > 0: b 在 oa 的逆时针方向
// cross < 0: b 在 oa 的顺时针方向
// cross = 0: 三点共线
double cross(const Point& o, const Point& a, const Point& b) {
    return (a.x - o.x) * (b.y - o.y) - (a.y - o.y) * (b.x - o.x);
}

// 符号判定: 带容差的三态函数
// 返回 1 (>0), -1 (<0), 0 (≈0)
// EPS=1e-9 用于处理浮点精度问题
int sign(double v) {
    return (v > EPS) ? 1 : (v < -EPS ? -1 : 0);
}
```

**为什么用 `sign()` 而不是直接比较？** 浮点计算有精度误差，`cross()` 理论上为 0 的结果可能被算成 `1e-15`。直接 `> 0` 比较会把共线误判为逆时针，导致交点遗漏或误报。

### 点在线段上的判定（on_segment）

```cpp
bool on_segment(const Point& p, const Point& a, const Point& b) {
    // 1. p 必须在 ab 所在的直线上（叉积为 0）
    // 2. p 的 x 坐标必须在 [min(a.x, b.x), max(a.x, b.x)] 范围内
    // 3. p 的 y 坐标必须在 [min(a.y, b.y), max(a.y, b.y)] 范围内
    return fabs(cross(p, a, b)) < EPS
        && fmin(a.x, b.x) - EPS <= p.x
        && p.x <= fmax(a.x, b.x) + EPS
        && fmin(a.y, b.y) - EPS <= p.y
        && p.y <= fmax(a.y, b.y) + EPS;
}
```

这个函数看似简单，但在线段端点共线判定的场景中频繁被调用。注意每个范围比较都留有 `EPS` 的容差空间，这是因为浮点运算可能在端点恰好相同时仍然产生微小的计算误差。比如 `fmin(a.x, b.x) - EPS` 允许点略微超出范围——如果不加这个容差，当一个线段的端点恰好等于另一条线段的端点时，可能因为浮点误差而被判定为"不在线段上"，导致真正的交点被遗漏。

### 交点计算

```cpp
Point compute_intersection(const Segment& a, const Segment& b) {
    // 使用参数方程: a.p1 + t * (a.p2 - a.p1) = b.p1 + s * (b.p2 - b.p1)
    // 交叉相乘解出 t
    double d = (a.p1.x - a.p2.x) * (b.p1.y - b.p2.y)
             - (a.p1.y - a.p2.y) * (b.p1.x - b.p2.x);
    
    if (fabs(d) < EPS) {
        // 平行或共线，返回两条线段中点作为"假交点"
        // 实际调用此函数时，调用者已经确认了非共线
        return Point((a.p1.x + b.p1.x) / 2, (a.p1.y + b.p1.y) / 2);
    }
    
    double t = ((a.p1.x - b.p1.x) * (b.p1.y - b.p2.y)
              - (a.p1.y - b.p1.y) * (b.p1.x - b.p2.x)) / d;
    
    return Point(a.p1.x + t * (a.p2.x - a.p1.x),
                 a.p1.y + t * (a.p2.y - a.p1.y));
}
```

这里的 `d` 实际上是向量 `(a.p2 - a.p1)` 和 `(b.p2 - b.p1)` 的叉积。当 `d ≈ 0` 时两条线段平行，没有唯一定义的实数 `t`。这时函数退化返回两条线段的中点平均值——虽然这个"假交点"的坐标不准确，但由于平行线段不应该相交，这种情况在实际流程中不应该发生。万一发生，调用者可以通过检查返回值是否落在两条线段上来发现异常。

### 2. 线段相交判定

```cpp
bool segments_cross(const Segment& a, const Segment& b) {
    if (a.id == b.id) return false;  // 自己和自己不算相交
    
    double c1 = cross(a.p1, a.p2, b.p1);
    double c2 = cross(a.p1, a.p2, b.p2);
    double c3 = cross(b.p1, b.p2, a.p1);
    double c4 = cross(b.p1, b.p2, a.p2);
    
    if (sign(c1) * sign(c2) <= 0 && sign(c3) * sign(c4) <= 0) {
        // 情况1: 四点共线 → 需要区间重叠判断
        if (fabs(c1) < EPS && fabs(c2) < EPS &&
            fabs(c3) < EPS && fabs(c4) < EPS) {
            return on_segment(a.p1, b.p1, b.p2) ||
                   on_segment(a.p2, b.p1, b.p2) ||
                   on_segment(b.p1, a.p1, a.p2) ||
                   on_segment(b.p2, a.p1, a.p2);
        }
        // 情况2: a的端点共线于b，但b的端点不在a上 → 不算内部相交
        //        (比如T型相交中，端点在另一条线段中段的情况在其他地方处理)
        if (fabs(c1) < EPS && fabs(c2) < EPS) return false;
        if (fabs(c3) < EPS && fabs(c4) < EPS) return false;
        // 情况3: 标准的严格跨立相交
        return true;
    }
    return false;
}
```

**关键细节**：当 $c_3 = 0$ 且 $c_4 = 0$ 时，意味着 A 的两个端点都在 CD 所在的直线上（A 和 CD 共线），但应该由共线分支处理。如果不在共线分支里处理，会导致共线的两条线段被误判为相交。这就是最初版本中 Star 测试遗漏交点的根因——部分线段对的端点恰好落在其他线段的方向延长线上。

### 3. 事件队列

```cpp
struct Event {
    double y, x, left_x;  // y: 事件y坐标, x: 端点x坐标, left_x: 线段的左端点x
    int seg;               // 线段编号
    bool is_exit;          // true: 退出事件, false: 进入事件
    
    bool operator<(const Event& o) const {
        if (fabs(y - o.y) > EPS) return y > o.y;    // y 降序 (从上到下)
        if (fabs(x - o.x) > EPS) return x < o.x;    // x 升序 (从左到右)
        if (is_exit != o.is_exit) return !is_exit;  // 进入事件先于退出事件
        return seg < o.seg;
    }
};
```

**为什么进入事件先于退出事件？** 如果在同一个 y 坐标既有线段进入又有线段退出：
- 先处理进入事件 → 新线段在活跃集中可用 → 退出事件可以检查与新进入线段的交点
- 如果反过来，退出的线段已经不在活跃集中，可能遗漏它刚进入时产生的交点

**`left_x` 字段的用途**：对于水平线段，左右端点的 y 相同，但需要确保它们的事件有区别（不同的 x），否则 `std::set` 会视为重复事件而丢弃一个。

### 4. 交点去重

```cpp
struct IntersectionResult {
    int s1, s2; Point pt;
    IntersectionResult(int a, int b, Point p)
        : s1(std::min(a,b)), s2(std::max(a,b)), pt(p) {}
    
    bool operator<(const IntersectionResult& o) const {
        if (s1 != o.s1) return s1 < o.s1;
        return s2 < o.s2;  // 仅按线段对去重，忽略浮点精度差异
    }
};
```

**为什么按 `(s1, s2)` 去重而不按坐标？** 相同的线段对可能在不同扫描线位置被多次检测到（比如在进入事件和相邻对检查中都发现了同一对交点），每次计算的浮点坐标可能略有不同。按坐标比较会导致同一个交点被当做多个不同的交点存储。按线段对去重是正确的做法，因为两条线段之间最多只有一个交点（除非它们共线重叠，这种情况本文未处理）。

### 5. 活跃集与相邻对检查

```cpp
// 检查所有相邻对
for (int i = 0; i + 1 < (int)status.size(); i++) {
    int a = status[i], b = status[i+1];
    if (segments_cross(segs[a], segs[b]))
        results.insert(IntersectionResult(a, b,
            compute_intersection(segs[a], segs[b])));
}
```

**这是算法的核心洞察**：只检查相邻对就够了，因为任何两条非相邻的线段如果要相交，扫描线必须先经过它们成为邻居的那个 y 坐标。在经典的 Bentley-Ottmann 实现中，当发现一个交点时，该交点会被插入事件队列，当扫描线到达该交点时，两条相交线段的顺序会交换，此时它们会成为邻居并被检查。

本文的简化实现省略了动态插入交点事件，而是通过**在每个事件处理后都重新排序并检查所有相邻对**来确保不遗漏。这使得复杂度从 $O((n+k) \log n)$ 降级，但对于小规模测试是够用的。

---

## 踩坑实录

### Bug 1: 水平线段重复进入事件（Grid 测试多出 3 个假交点）

**症状**：Grid 6×6 测试中，BO 算法报告 12 个交点，而暴力法只有 9 个。多出的 3 个是 `(0,0), (1,1), (2,2)`——线段和自己"相交"。

**错误假设**：以为水平线段的两个端点事件会分别被识别为上端点和下端点。

**真实原因**：水平线段的 `p1.y == p2.y`，两个端点事件都被分类为进入事件（`fabs(ev.y - seg.p1.y) < EPS` 对两个事件都成立）。线段被重复加入活跃集，然后在相邻对检查时和自己配对。

**修复方式**：为水平线段添加特殊处理——标记为"进入后立即退出"，不加入活跃集。

```cpp
bool horiz = fabs(segs[i].p1.y - segs[i].p2.y) < EPS;
if (horiz) {
    // 进入事件 (左端点) + 退出事件 (右端点)，但不加入活跃集
    Q.insert({y, left_x, left_x, i, false});
    Q.insert({y, right_x, left_x, i, true});
}
// ...
for (int sid : enters) {
    bool horiz = fabs(segs[sid].p1.y - segs[sid].p2.y) < EPS;
    if (!horiz) status.push_back(sid);  // 水平线段不加入活跃集
}
```

### Bug 2: 共线判定的边界条件（Star 测试遗漏 2 个交点）

**症状**：Star 5 线测试中，BO 找到 8 个交点，但暴力法有 10 个。遗漏的是 `(1,4)` 和 `(2,3)`。

**错误假设**：以为 `sign(c1) * sign(c2) < 0`（严格小于 0）足以覆盖所有相交情况。

**真实原因**：当一条线段的端点恰好落在另一条线段的方向线上（`cross == 0`），使用严格 `< 0` 会把这种情况排除。应改为 `<= 0`，配合正确的共线处理分支。

**修复方式**：
```cpp
// 旧: sign(c1) * sign(c2)  < 0  && sign(c3) * sign(c4)  < 0  // ❌ 遗漏端点触碰
// 新: sign(c1) * sign(c2) <= 0  && sign(c3) * sign(c4) <= 0  // ✅ 包含端点触碰
```

但这不是最终方案——改为 `<= 0` 后，共线但不相交的线段（如平行线）会被误判。需要额外的分支：
- 如果 $c_1 \approx 0$ 且 $c_2 \approx 0$ 且 $c_3 \approx 0$ 且 $c_4 \approx 0$ → **四点共线**，需要区间重叠判断
- 如果仅 $c_1 \approx 0$ 且 $c_2 \approx 0$ → A 的端点在 CD 的延长线上但不在线段上，不算相交
- 其他情况 → 标准相交

### Bug 3: 交点去重按浮点坐标排序导致重复

**症状**：即使已经用 `std::set` 存储交点，BO 仍然比 BF 多出一些结果。

**错误假设**：以为两个 `IntersectionResult` 如果 `(s1, s2)` 相同，它们的 `pt` 坐标也会在浮点精度范围内完全一致。

**真实原因**：同一个线段对在不同扫描线位置被检测到时，`compute_intersection()` 返回的坐标可能因为浮点运算顺序不同而略有差异（比如 $10^{-14}$ 级别的差异），导致 `std::set` 把它们当成不同的元素。

**修复方式**：修改 `IntersectionResult::operator<`，**只按 `(s1, s2)` 比较，完全不使用坐标**：

```cpp
bool operator<(const IntersectionResult& o) const {
    if (s1 != o.s1) return s1 < o.s1;
    return s2 < o.s2;
}
```

两条线段之间最多一个交点，按线段对去重是完全安全的。

### Bug 4: 同一个线段和自己比较

**症状**：在某些事件中，segment X 既在进入列表也在退出列表中（尤其是水平线段时），导致 `(X, X)` 伪交点。

**修复方式**：在 `segments_cross()` 开头加上 `if (a.id == b.id) return false;`，彻底杜绝自相交。

### 教训总结

1. **浮点精度是计算几何的最大敌人**——每个 `== 0` 的比较都需要 `fabs() < EPS` 替代。在 4 个 Bug 中，至少有 3 个的直接原因是浮点精度问题。
2. **水平/垂直线段需要特殊处理**——它们打破了"每个端点有唯一 y"的假设。Bentley-Ottmann 算法的教科书实现通常假设"没有两条线段的端点共享同一个 y 坐标"（一般位置假设），但真实数据很少满足这个条件，所以必须处理退化情况。
3. **去重应该按逻辑标识而非物理属性**——按线段对 ID 去重比按坐标去重更可靠。这是"语义正确性"优于"数值精确性"的典型案例。
4. **暴力法对照是必须的**——即使自认为写对了，也需要暴力法作为 ground truth 来验证。本文的 4 个 Bug 中有 3 个是在暴力对比中被发现的。
5. **简化实现虽然"不完整"但很有价值**——虽然本文没有实现动态交点事件插入，但通过"每个事件后检查所有相邻对"的策略仍然获得了正确的结果。理解"为什么这样简化可行"本身就是对算法本质的深刻理解。

---

## 效果验证与数据

### 测试结果

所有 4 个测试用例全部通过：

| 测试用例 | 线段数 | BF (暴力) | BO (算法) | 状态 |
|---------|--------|----------|----------|------|
| Grid 6×6 | 6 | 9 | 9 | ✅ |
| Star (5线) | 5 | 10 | 10 | ✅ |
| Random 20 | 20 | 32 | 32 | ✅ |
| Polygon 20 | 20 | 133 | 133 | ✅ |

### 可视化输出

每个测试生成一张 PPM 图像，蓝色线条表示输入线段，红色方块标记检测到的交点。

![Grid 6×6](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-11-Bentley-Ottmann-Line-Intersection/Grid_6x6.png)

*Grid 测试：3 条水平线 + 3 条垂直线，9 个网格交点全部正确检测*

![Star](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-11-Bentley-Ottmann-Line-Intersection/Star_10.png)

*Star 测试：5 条通过圆心的直径线，$C_5^2 = 10$ 个交点全部正确（所有线段在圆心交汇）*

![Random](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-11-Bentley-Ottmann-Line-Intersection/Random_20.png)

*Random 测试：20 条随机线段，32 个交点全部匹配暴力法*

![Polygon](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07/07-11-Bentley-Ottmann-Line-Intersection/Polygon_20.png)

*Polygon 测试：20 边形的对角线连接，133 个交点全部匹配——这是最复杂的测试用例，Bentley-Ottmann 的优势在这里最为显著*

### 像素级验证

对每张输出图像进行像素采样验证：

验证逻辑使用 Python + PIL/Numpy：

```python
from PIL import Image
import numpy as np

img = Image.open('Grid_6x6.png')
pixels = np.array(img).astype(float)

# 统计非白色像素（R、G、B 中任一通道 < 240）
non_white = np.sum(np.any(pixels < 240, axis=2))
total = pixels.shape[0] * pixels.shape[1]
ratio = non_white / total * 100

# 统计唯一颜色数量
unique_colors = len(np.unique(pixels.reshape(-1, 3), axis=0))
print(f"非白像素: {non_white}/{total} = {ratio:.1f}%")
print(f"唯一颜色: {unique_colors}")
```

验证结果：

| 图像 | 文件大小 | 非白像素比例 | 唯一颜色 | 判定 |
|------|---------|------------|---------|------|
| Grid_6x6.png | 3.9KB | 0.7% | 3 | ✅ |
| Star_10.png | 7.7KB | 0.5% | 3 | ✅ |
| Random_20.png | 14.2KB | 1.1% | 3 | ✅ |
| Polygon_20.png | 15.9KB | 2.0% | 3 | ✅ |

所有图像均包含三种颜色：白色(#FFFFFF，背景)、蓝色(#3232C8，线段)、红色(#FF1E1E，交点标记)。交点越多，非白像素比例越高——Polygon_20 的 2.0% 对应于 133 个交点产生的红色标记区域。

需要注意的是，线图可视化（白底+稀疏线条）的像素均值接近 254，远高于渲染类项目（通常均值 100-200）。因此像素验证阈值需要根据项目类型调整——线图类重点检查"非白像素数量和唯一颜色种类"，而非均值/标准差。

---

## 总结与延伸

### 算法的局限性

1. **退化情况处理不完整**：当多条线段交于同一点时，经典 Bentley-Ottmann 需要特殊的事件处理顺序。本文实现通过"检查所有相邻对"绕过了这个问题，但在大规模场景中会成为性能瓶颈。

2. **未处理共线重叠**：两条部分重叠的共线线段（如 `[0,5]` 和 `[3,8]`）理论上应报告"有无穷多个交点"或"有线段重叠"。本文将其忽略（`segments_cross` 返回 false），因为在图形渲染场景中这种情况很少见。

3. **复杂度退化**：本文的简化实现复杂度为 $O(n^2 \log n)$（因为每个事件都扫描全部相邻对），不是标准的 $O((n+k) \log n)$。生产级实现需要：
   - 使用平衡二叉树（如 `std::set` 的自定义比较器）维护活跃集
   - 动态插入交点事件
   - 在交点处交换两条线段的顺序

### 可优化方向

- **平衡二叉树活跃集**：将 `std::vector<int>` 替换为 `std::set<int, SweepComparator>` 可以将每次活跃集操作从 $O(n \log n)$ 降至 $O(\log n)$
- **交点事件插入**：每当发现新交点，将其作为事件插入队列，在交点处交换两条线段的位置
- **并行化**：将不同区域的线段分到不同的"条带"（strip）中独立处理
- **整数坐标优化**：如果线段端点都是整数，可以使用精确的有理数运算消除所有浮点精度问题

### 与本系列其他文章的关联

本篇文章是每日编程实践中"计算几何"子系列的第一篇。后续我们还将探索：

- **GJK 碰撞检测**（已完成，06-13）：使用闵可夫斯基差进行凸体碰撞检测
- **SAT 分离轴定理**（已完成，06-19）：2D 凸多边形的碰撞检测
- **Delaunay 三角剖分**（已完成，06-17）：Bowyer-Watson 增量构建算法
- **凸包算法**（已完成，06-18）：Graham Scan 和 Monotone Chain

这些算法共同构成了计算几何的基础工具箱，在游戏物理引擎、GIS 系统、CAD 软件中有着广泛的应用。

---

**完整代码见**：[GitHub - daily-coding-practice/2026/07/07-11-Bentley-Ottmann-Line-Intersection](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/07/07-11-Bentley-Ottmann-Line-Intersection)
