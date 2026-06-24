---
title: "每日编程实践: 视锥剔除 (Frustum Culling)"
date: 2026-06-25 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 实时渲染
  - 性能优化
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/06/06-25-frustum-culling/nocull.png
---

在日常编程实践系列中，我们已经在软光栅化、光线追踪、PBR、阴影、后处理等方向做了大量探索。但有一个基础问题我们一直没碰——渲染管线的**性能优化**。当场景中有成千上万个物体时，我们真的需要逐个绘制它们吗？显卡强大的光栅化能力不是无限的，把时间浪费在渲染根本看不见的物体上是最大的性能浪费。

这就是视锥剔除（Frustum Culling）要解决的问题。

## ① 背景与动机

想象你站在一个房间里，面前是一扇窗户。你能看到的只有窗户框架范围内的东西——墙后面的、头顶天花板以上的、脚底下地板以下的物体，你看不到。3D渲染的"窗户"就是**视锥体**（View Frustum），由相机的FOV（视场角）、近裁面（Near Plane）、远裁面（Far Plane）定义的棱台形状区域。

视锥剔除的基本思想极其朴素：**在把物体送进渲染管线之前，先问一句"它在视锥体里面吗？"**如果不在，直接跳过。这个简单的想法可以节省大量GPU工作——在开放世界游戏中，视野外的建筑物、NPC、植被占据了场景的绝大部分，但如果把它们全部送入渲染管线，每帧将浪费大量计算。

### 工业界的实际应用

在商业游戏引擎中，Frustum Culling 是渲染管线的第一道关卡：

- **Unity** 的 CullingGroup API 允许开发者注册回调，在物体进出视锥体时接收通知。其内部使用 AABB 与视锥体平面的分离轴测试。
- **Unreal Engine** 在 `FSceneRenderer::ComputeViewVisibility()` 中执行 Frustum Culling，这是整个可见性系统的基石。UE5 在此基础上叠加了 Nanite 的 Cluster Culling 和 Virtual Shadow Map 的剔除。
- **id Software** 在 DOOM (2016) 的 GDC 演讲中提到，他们的"Visibility System"从 Frustum Culling 开始，然后是 Portal Culling，最后是 Occlusion Culling，逐层过滤。

但不要被"基础"二字迷惑。Frustum Culling 的实现细节直接影响性能——如果AABB-Plane测试写错了，可能会**错误地剔除可见物体**（屏幕上出现"消失的物体"）或者**漏掉本该剔除的物体**（浪费GPU时间）。

### 为什么软光栅化更需要它

在我们的软光栅化渲染器中，每个三角形都要经过投影变换、包围盒计算、重心坐标遍历和深度测试——全部在CPU上完成。一个1000球体的场景，每个球体有6×5×5×2=300个三角形，总共300,000个三角形。如果在视野外的球体也全部参与这些计算，即使最终被Z-buffer裁剪掉，也白白消耗了CPU时间。

**核心洞察**：Frustum Culling 不是让渲染"更快"，而是让渲染"只做必要的事"。在视野外的物体，连投影变换都不应该做。

## ② 核心原理：六平面视锥体

### 2.1 视锥体的定义

透视投影的视锥体是一个截断的金字塔形（frustum），由以下6个平面围成：

1. **近平面（Near Plane）**：距离相机 z=near 处，裁剪掉相机后方和过近的物体
2. **远平面（Far Plane）**：距离相机 z=far 处，裁剪掉太远的物体
3. **左平面（Left Plane）**：视锥体左侧边界
4. **右平面（Right Plane）**：视锥体右侧边界
5. **顶平面（Top Plane）**：视锥体顶部边界
6. **底平面（Bottom Plane）**：视锥体底部边界

每个平面用方程表示：

$$
n_x \cdot x + n_y \cdot y + n_z \cdot z + d = 0
$$

其中 $(n_x, n_y, n_z)$ 是平面的**单位法向量**（通常指向视锥体内部），$d$ 是从原点到平面的**有符号距离**。

对于视锥体内部的任意点 $P$，必须同时满足所有6个平面的不等式：

$$
n_i \cdot P + d_i \geq 0, \quad \forall i \in \{0,1,2,3,4,5\}
$$

这意味着点在每个平面的"内部"一侧。如果点在任一平面的外侧（$n_i \cdot P + d_i < 0$），这个点就在视锥体之外。

### 2.2 Gribb/Hartmann 平面提取法

你可能会想：有了相机位置、朝向、FOV，手动构建6个平面并不难。但实际工程中有一个更优雅的方法——直接从 **Model-View-Projection 矩阵** 中提取平面。这个技巧由 Gil Gribb 和 Klaus Hartmann 在 2001 年的《Fast Extraction of Viewing Frustum Planes from the World-View-Projection Matrix》中提出。

核心思想：MVP矩阵将一个世界空间点 $P = (x, y, z, w=1)$ 变换到裁剪空间 $P' = (x', y', z', w')$。在裁剪空间中，视锥体被简化为一个轴对齐的立方体：

$$
-w' \leq x' \leq w' \quad \text{(左右平面)}
$$
$$
-w' \leq y' \leq w' \quad \text{(底顶平面)}
$$
$$
-w' \leq z' \leq w' \quad \text{(近远平面)}
$$

设 MVP 矩阵为 $M$，其行向量为 $R_0, R_1, R_2, R_3$。对于左平面条件 $x' \geq -w'$：

$$
R_0 \cdot P \geq -(R_3 \cdot P)
$$
$$
(R_0 + R_3) \cdot P \geq 0
$$

所以左平面的系数就是 $R_0 + R_3$（逐分量相加）。类似地：

- **左平面**：$R_3 + R_0$（即 $R_0 \cdot P \geq -w'$ 推出 $x' + w' \geq 0$）
- **右平面**：$R_3 - R_0$（即 $x' \leq w'$ 推出 $w' - x' \geq 0$）
- **底平面**：$R_3 + R_1$
- **顶平面**：$R_3 - R_1$
- **近平面**：$R_3 + R_2$
- **远平面**：$R_3 - R_2$

> **直觉解释**：MVP矩阵的行向量本身就编码了几何约束。$R_0$（矩阵第0行）负责左右裁剪，$R_1$ 负责上下裁剪，$R_2$ 负责前后裁剪。通过将 Row3（w分量）加减对应的行向量，我们"恢复"出每个平面在世界空间中的方程。

**关键实现细节**：提取出的平面法向量需要归一化！不归一化的话，$d$ 值代表的是缩放后的有符号距离，AABB-Plane 测试中会出现错误的剔除判断。我的初版代码在这个细节上栽了跟头——平面都提取对了，但没有除以法向量的长度，导致所有球体都被错误剔除（剔除率100%）。

### 2.3 P-Vertex 方法：AABB 与平面的快速测试

现在我们有了6个平面，需要对每个物体的 AABB（轴对齐包围盒）做测试。最朴素的方法：检查 AABB 的8个顶点是否全部在某个平面的外侧。但每个平面检查8个顶点太慢了。

**P-Vertex（最远顶点法）**是此优化的关键：对于给定的平面法向量 $n$，AABB 中**离平面最远的顶点**是确定的——它就是选择 AABB 在法向量方向上最远的那个角：

$$
P_{vertex} = \left(
\begin{cases}
x_{max} & \text{if } n_x > 0 \\
x_{min} & \text{if } n_x \leq 0
\end{cases},
\quad
\begin{cases}
y_{max} & \text{if } n_y > 0 \\
y_{min} & \text{if } n_y \leq 0
\end{cases},
\quad
\begin{cases}
z_{max} & \text{if } n_z > 0 \\
z_{min} & \text{if } n_z \leq 0
\end{cases}
\right)
$$

如果这个"最远的顶点"都在平面外侧（$n \cdot P_{vertex} + d < 0$），那么 AABB 的**所有顶点**必然也都在外侧——这意味着整个物体在视锥体外，可以被剔除。

> **为什么叫 P-Vertex？** "P" 代表 Positive。我们选择 AABB 中在每个轴分量上对法向量最"配合"的角（法向量指向正方向就取最大值，指向负方向就取最小值），这个角沿法向量方向的投影距离最大，因此它最有可能在平面内侧。如果连它都在外侧，就彻底没救了——整个AABB都在外面。

### 2.4 球体测试（更简单也更保守）

如果物体用球体包围盒（Bounding Sphere）表示，测试更简单：

对于球心 $C$ 和半径 $r$，球体在平面外侧当且仅当：

$$
n \cdot C + d < -r
$$

这比 AABB 测试更快（一次点积 vs. 三次条件分支），但球体贴合物体的能力比 AABB 差（球体通常包含更多空区域），可能导致**更保守的剔除**（本该剔除的物体被认为"可能可见"）。

## ③ 实现架构

### 3.1 整体渲染管线

本文实现的是一个软光栅化渲染器（不使用OpenGL/DirectX），管线如下：

```
场景输入 (1000球体)
    │
    ▼
┌──────────────────┐
│  Mat4::lookAt()  │  ← 构建View矩阵（世界→相机空间）
└──────────────────┘
    │
    ▼
┌──────────────────┐
│  Mat4::persp()   │  ← 构建Projection矩阵（相机→裁剪空间）
└──────────────────┘
    │
    ▼
┌──────────────────┐
│  mulM(view,proj) │  ← 列主序乘法: VP = View × Proj
└──────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│  Frustum::fromVP(VP)             │
│  提取6平面 (Gribb/Hartmann)      │
│  归一化法向量                     │
└──────────────────────────────────┘
    │
    ▼ (对每个球体)
┌──────────────────────────────────────┐
│  cullAABB(vmin, vmax)                │
│  P-vertex方法，判断AABB是否在视锥外  │
│  → YES: 跳过此球体                   │
│  → NO:  继续渲染                     │
└──────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────┐
│  球面细分 → 三角形光栅化             │
│  立方体投影→球面 → Z-Buffer + Lambert│
└──────────────────────────────────────┘
    │
    ▼
   PPM / PNG 输出
```

### 3.2 矩阵约定：列主序的陷阱

这是整个项目中最容易出错的环节。我们的矩阵使用 OpenGL 风格的**列主序**（column-major）存储：

```cpp
struct Mat4 {
    float m[16];
    float& operator()(int col, int row) { return m[(col << 2) + row]; }
    // m[0]=第0列第0行, m[1]=第0列第1行, m[2]=第0列第2行, m[3]=第0列第3行
    // m[4]=第1列第0行, ...
};
```

在这个约定下，矩阵乘法 `C = A * B` 必须交换乘法顺序：

```cpp
// 错误: vp = mulM(proj, view);  // 这等价于 V * P（顺序反了!）
// 正确:
Mat4 mulM(const Mat4& a, const Mat4& b) {
    Mat4 r;
    for (int i = 0; i < 4; i++)        // col
        for (int j = 0; j < 4; j++)    // row
            for (int k = 0; k < 4; k++)
                r(i, j) += a(i, k) * b(k, j);
    return r;
}
vp = mulM(view, proj);  // 注意: view在前，不是proj
```

**为什么？** 在列主序矩阵中，`a(i,k)` 访问的是第k行的第i列元素（即 `A[k][i]`，行列交换）。`a(i,k) * b(k,j)` 实际计算的是 $\sum_k A_{ki} B_{jk} = (B A)_{ji}$。所以 `mulM(view, proj)` 实际产生的是 `proj * view` 的转置（以列主序存储），等价于我们需要的 VP 矩阵。

如果直接写 `mulM(proj, view)`，会得到 `view * proj`，视锥体平面提取将完全错误——这就是我最初的bug。

向量-矩阵乘法也同理：

```cpp
Vec3 mulV(const Mat4& m, const Vec3& v, float w, float& ow) {
    float x = m(0,0)*v.x + m(1,0)*v.y + m(2,0)*v.z + m(3,0)*w;
    float y = m(0,1)*v.x + m(1,1)*v.y + m(2,1)*v.z + m(3,1)*w;
    float z = m(0,2)*v.x + m(1,2)*v.y + m(2,2)*v.z + m(3,2)*w;
    ow      = m(0,3)*v.x + m(1,3)*v.y + m(2,3)*v.z + m(3,3)*w;
    return {x, y, z};
}
```

### 3.3 场景生成

1000个球体排列在 $10 \times 10 \times 10$ 的网格中，间距2.5单位：

```cpp
std::vector<SphereInst> mkScn() {
    std::vector<SphereInst> objs;
    std::mt19937 rng(42);  // 固定种子，保证可复现
    float sp = 2.5f, half = (10 - 1) * sp * 0.5f;
    for (int ix = 0; ix < 10; ix++)
        for (int iy = 0; iy < 10; iy++)
            for (int iz = 0; iz < 10; iz++) {
                Vec3 c(ix*sp - half + jitter,
                       iy*sp - half + jitter,
                       iz*sp - half + jitter);
                float r = 0.4f + randf() * 0.3f;
                // ...
                objs.emplace_back(c, r, color);
            }
    return objs;
}
```

每个球体加少量随机 jitter (±0.15) 避免完美的网格排列，让场景更自然。颜色随机生成，但强制总亮度 ≥0.6，避免过暗的球体在屏幕上不可见。

## ④ 关键代码解析

### 4.1 平面提取

```cpp
struct Frustum {
    Vec3 n[6];  // 归一化法向量（指向视锥体内部）
    float d[6];  // 有符号距离

    static Frustum fromVP(const Mat4& vp) {
        Frustum f;
        // 左平面: Row3 + Row0
        f.n[0] = {vp(0,3)+vp(0,0), vp(1,3)+vp(1,0), vp(2,3)+vp(2,0)};
        f.d[0] =  vp(3,3)+vp(3,0);
        // 右平面: Row3 - Row0
        f.n[1] = {vp(0,3)-vp(0,0), vp(1,3)-vp(1,0), vp(2,3)-vp(2,0)};
        f.d[1] =  vp(3,3)-vp(3,0);
        // 底平面: Row3 + Row1
        f.n[2] = {vp(0,3)+vp(0,1), vp(1,3)+vp(1,1), vp(2,3)+vp(2,1)};
        f.d[2] =  vp(3,3)+vp(3,1);
        // 顶平面: Row3 - Row1
        f.n[3] = {vp(0,3)-vp(0,1), vp(1,3)-vp(1,1), vp(2,3)-vp(2,1)};
        f.d[3] =  vp(3,3)-vp(3,1);
        // 近平面: Row3 + Row2
        f.n[4] = {vp(0,3)+vp(0,2), vp(1,3)+vp(1,2), vp(2,3)+vp(2,2)};
        f.d[4] =  vp(3,3)+vp(3,2);
        // 远平面: Row3 - Row2
        f.n[5] = {vp(0,3)-vp(0,2), vp(1,3)-vp(1,2), vp(2,3)-vp(2,2)};
        f.d[5] =  vp(3,3)-vp(3,2);

        // 归一化（关键！不归一化会导致错误剔除）
        for (int i = 0; i < 6; i++) {
            float len = f.n[i].len();
            if (len > 1e-10f) {
                f.n[i] = f.n[i] * (1.0f / len);
                f.d[i] /= len;
            }
        }
        return f;
    }
};
```

> **为什么需要归一化？** MVP矩阵的行向量本身不是单位向量——它们包含了投影变换的缩放因子。直接从矩阵中提取的平面系数，其法向量长度可能远大于1或小于1。不归一化时，`n·P + d` 的数值会被缩放，导致本来在平面内侧0.001距离的点被误判为外侧。

> **容易写错的地方**：每个平面的符号（Row3 + RowX vs Row3 - RowX）。左/底/近用加号（代表 `x'/w' >= -1`, `y'/w' >= -1`, `z'/w' >= -1`），右/顶/远用减号。如果某个平面符号写反了，会导致该方向的物体被错误剔除或漏检。

### 4.2 AABB 剔除测试

```cpp
bool cull(const Vec3& vmin, const Vec3& vmax) const {
    for (int i = 0; i < 6; i++) {
        // P-Vertex: 找到AABB中沿法向量方向最远的角
        Vec3 pv(
            n[i].x > 0 ? vmax.x : vmin.x,
            n[i].y > 0 ? vmax.y : vmin.y,
            n[i].z > 0 ? vmax.z : vmin.z
        );
        // 如果最远顶点都在平面外侧，整个AABB在视锥体外
        if (n[i].dot(pv) + d[i] < 0)
            return true;  // 被剔除
    }
    return false;  // 可能可见（可能在内部）
}
```

> **为什么这个测试是保守的？** P-Vertex 方法找到的是 AABB 中对当前平面"最安全"的顶点（离平面内侧最远）。如果连这个顶点都在平面外侧，那么 AABB 所有顶点都在外侧——这是确定性的剔除。**但反过来不成立**：如果 P-Vertex 在平面内侧，AABB **可能**部分在外侧（还需要其他平面继续检查），也可能完全在内部。所以这个测试只会保守地剔除物体，**永远不会错误地剔除可见物体**。

### 4.3 球体渲染

球体使用**立方体到球面投影**的方法：6个面各做 sub×sub 的均匀网格，每个网格点沿径向投影到球面上，然后拆成2个三角形：

```cpp
Vec3 c2s(float u, float v, int face, float rad, const Vec3& cen) {
    float s = (u - 0.5f) * 2, t = (v - 0.5f) * 2;
    Vec3 p;
    switch (face) {
        case 0: p = { 1, -t, -s}; break;  // +X面
        case 1: p = {-1, -t,  s}; break;  // -X面
        case 2: p = { s,  1,  t}; break;  // +Y面
        case 3: p = { s, -1, -t}; break;  // -Y面
        case 4: p = { s, -t,  1}; break;  // +Z面
        case 5: p = {-s, -t, -1}; break;  // -Z面
    }
    return p.norm() * rad + cen;
    // 关键：normalize() 将立方体面上的点投影到球面上
}
```

每个球体 = 6面 × sub×sub 四边形 × 2三角形。sub=5 时每个球体 300 个三角形。1000球体理论上有 300,000 个三角形，但只有视野内的球体三角形才会进入包围盒检查。

### 4.4 每三角形渲染

```cpp
void tri(const Vec3& p0, const Vec3& p1, const Vec3& p2,
         const Vec3& col, const Vec3& n) {
    float d0, d1, d2;
    Vec3 s0 = projP(p0, d0), s1 = projP(p1, d1), s2 = projP(p2, d2);
    // 包围盒检查
    int bx0 = std::max(0, int(std::floor(std::min({s0.x, s1.x, s2.x}))));
    int bx1 = std::min(W-1, int(std::ceil(std::max({s0.x, s1.x, s2.x}))));
    int by0 = std::max(0, int(std::floor(std::min({s0.y, s1.y, s2.y}))));
    int by1 = std::min(H-1, int(std::ceil(std::max({s0.y, s1.y, s2.y}))));
    if (bx0 > bx1 || by0 > by1) return;  // 屏幕外，跳过
    // Lambert光照
    float diff = std::max(0.f, n.norm().dot(lightDir)) * 0.8f + 0.2f;
    // 重心坐标遍历 + Z-Buffer
    float area = (s1.x-s0.x)*(s2.y-s0.y) - (s1.y-s0.y)*(s2.x-s0.x);
    if (std::abs(area) < 0.1f) return;
    float iA = 1.f / area;
    for (int y = by0; y <= by1; y++)
        for (int x = bx0; x <= bx1; x++) {
            float px = x + 0.5f, py = y + 0.5f;
            float w0 = ((s1.x-s2.x)*(py-s2.y) - (s1.y-s2.y)*(px-s2.x)) * iA;
            float w1 = ((s2.x-s0.x)*(py-s0.y) - (s2.y-s0.y)*(px-s0.x)) * iA;
            float w2 = 1 - w0 - w1;
            if (w0 >= -1e-3f && w1 >= -1e-3f && w2 >= -1e-3f) {
                float z = w0*s0.z + w1*s1.z + w2*s2.z;
                int idx = y*W + x;
                if (z < zb[idx]) { zb[idx] = z; sp(x, y, cr, cg, cb); }
            }
        }
}
```

> **注意**：`zb` 清零发生在左右半分别渲染时独立执行。如果不分开清零，左半的深度值会影响右半的绘制（因为同一个像素被两个半区共享）。

## ⑤ 踩坑实录：从100%剔除到47.4%

### Bug 1: 矩阵乘法顺序错误（最隐蔽的bug）

**症状**：所有1000个球体在剔除模式下都被判定为"在视锥体外"，剔除率100%。右半图像完全空白。

**错误假设**：我最初认为 `vp = mulM(proj, view)` 就是标准GL中的 `Projection * View`。

**真实原因**：在列主序约定下，`mulM(A, B)` 内部的 `a(i,k) * b(k,j)` 实际计算的是 `A[k][i] * B[j][k]`（行列颠倒），最终结果是 `(A^T * B^T)^T = B * A`。所以 `mulM(proj, view)` 得到的是 `View * Proj` 而不是 `Proj * View`。

**修复方式**：交换乘法顺序为 `vp = mulM(view, proj)`。这个修复让剔除率从100%降到47.4%。

**教训**：自研矩阵库时，必须在一开始就明确约定（行主序 vs. 列主序），并统一所有操作（乘法、向量变换、转置等）基于同一约定。不能用"看着像GL"就是对的——必须用具体的测试用例验证。

### Bug 2: 平面法向量未归一化

**症状**：归一化前的平面提取中，有些平面的 d 值非常大（如 Near 平面的 d=12.17），有些非常小。同一物体在不同平面测试中的距离数值在不同量级。

**错误假设**：我以为 MVP 矩阵的行向量本身已经是单位长度，提取出的平面直接可用。

**真实原因**：投影矩阵的缩放因子（如 `1/tan(fovY/2)`）使得矩阵行向量不是单位向量。例如，投影矩阵的 Row0 长度为 `1/(aspect*tan(30°)) ≈ 1.299`。这导致法向量的长度 > 1，从而 `n·P + d` 的值被放大。

**修复方式**：提取平面后立即执行归一化——对每个平面，法向量和 d 都除以法向量的长度。修复后，同一场景的剔除率从错误值变为正确的47.4%。

**教训**：Gribb/Hartmann 论文中明确说了"提取的平面需要归一化"。永远不要跳过论文中的"细节"——它们往往是关键。

### Bug 3: Z-Buffer 共享导致左右半渲染干扰

**症状**：在双图对比渲染中（左半无剔除、右半有剔除），右半的图像比左半更亮，与预期相反（剔除后应有更少像素=更暗）。

**错误假设**：我以为在同一个 `renderCompare()` 中，左右半使用相同的 `zb` 数组没问题，因为它们是不同的像素区域。

**真实原因**：虽然左右半占用不同的x坐标范围，但 Z-Buffer 是一个跨全图的数组。左半写入的深度值保留在右半的"深度比较"中，导致右半的正常像素被错误的深度值阻挡。

**修复方式**：在左半渲染完成后、右半渲染开始前，执行 `std::fill(zb.begin(), zb.end(), 1e9f)` 清除 Z-Buffer。

**教训**：即使目标像素区域不重叠，全局资源（如 Z-Buffer）的共享仍然需要边界保护。更好的做法是每个pass使用独立的 Z-Buffer。

## ⑥ 效果验证与数据

### 6.1 渲染输出对比

我们生成了两张图来直观展示剔除效果：

| 模式 | 球体数 | 三角形数 | 说明 |
|------|--------|----------|------|
| 无剔除（nocull） | 1000 | 146,136 | 所有球体都参与渲染 |
| 有剔除（cull） | 526 | 145,720 | 剔除474个球体（47.4%） |

剔除模式下，视野内的526个球体产生了145,720个有效三角形（每个约277个，因为部分三角形被屏幕裁剪）。无剔除模式虽然遍历了全部1000个球体，但视野外的球体三角形被屏幕包围盒检查（`bx0 > bx1 || by0 > by1`）跳过，所以三角形总数几乎相同。

这个结果验证了**Frustum Culling 的核心价值不在于减少绘制的三角形数，而在于减少CPU端的遍历开销**——每个视野外球体省去了6面×25四边形×2三角形的投影变换+包围盒计算。

### 6.2 量化像素验证

```
[无剔除] 平均亮度: 37.54, 非背景: 99.8%
[有剔除] 平均亮度: 36.91, 非背景: 99.5%
像素差异>30: 211,806 (44.1%)
```

两张图的像素差异44.1%（差异>30的像素）。这个差异合理——视野外球体在无剔除图中贡献了一些边缘像素，而在剔除图中被完全跳过。差异集中在场景边缘区域。

差异率不是100%，说明两张图在场景中心区域（视野内的球体）是一致的，验证了剔除的正确性——只剔除了视野外的球体，没有错误剔除可见物体。

### 6.3 性能分析

虽然这个软光栅化渲染器不追求实时帧率，但我们可以统计被节省的计算量：

- 每个被剔除的球体节省：300次三角形投影变换 + 300次包围盒检查 + 约75,000次重心坐标计算（假设三角形平均覆盖2500像素）
- 474个被剔除球体 × 300 = **142,200次三角形处理被跳过**
- CPU耗时估算：每个球体的处理约2-5毫秒（纯CPU软光栅），节省约 474×3ms ≈ **1.4秒**

在FOV=45°时，剔除率上升到61.9%（619/1000），进一步验证了"窄视角 = 更多不可见物体 = 更大剔除收益"的直觉。

## ⑦ 总结与延伸

### 技术局限性

1. **AABB的保守性**：对于长条状的物体（如走廊、树木），AABB可能包含大量空区域。AABB测试可能认为"部分在视锥体内"（因为包围盒很大），但实际物体可能完全不可见。这是AABB固有的保守性——它永远不会错误剔除，但可能会漏剔。

2. **无遮挡剔除**：Frustum Culling 只检查物体是否在视锥体内，不检查物体是否被前方物体遮挡（Occlusion Culling）。在室内场景中，墙后的物体通过了Frustum Culling但仍不可见。

3. **动态场景开销**：如果物体运动频繁，AABB需要每帧重新计算。对于大量运动物体，AABB更新本身可能成为瓶颈。

### 可优化方向

1. **层次化剔除（Hierarchical Culling）**：使用 BVH（Bounding Volume Hierarchy）或 Octree，先测试大区域（如整个房间的AABB），如果区域在视锥体外，整个区域内的物体全部跳过。这是现代引擎的标准做法。

2. **遮挡剔除集成**：叠加 Hardware Occlusion Queries 或 Software Rasterization Occlusion，进一步减少绘制调用。

3. **LOD 联动**：与LOD（Level of Detail）结合——被部分剔除的远处物体可以用更粗糙的模型渲染。

4. **视锥体剔除的GPU化**：通过 Compute Shader 在GPU上并行执行剔除测试（如 UE5 的 GPU Scene 中的 Instance Culling），减少 CPU-GPU 同步开销。

### 本系列关联

视锥剔除是性能优化的第一课。在本系列中，它与以下主题形成互补：
- **BVH加速结构**（04-02）：层次化剔除的完美搭档
- **CSM级联阴影**（05-08）：阴影贴图也需要做视锥体分割
- **遮挡剔除相关主题**（未来方向）：从Frustum Culling到完整的可见性系统

---

### 关键技术要点速查

| 要点 | 实现方式 | 常见错误 |
|------|---------|--------|
| 平面提取 | Gribb/Hartmann: Row3 ± RowX | 矩阵乘法顺序错误 |
| 归一化 | 法向量长度归一化 | 忘记归一化导致全部误剔除 |
| AABB测试 | P-Vertex法（保守剔除） | 使用N-Vertex导致漏剔 |
| 矩阵约定 | 列主序，mulM(view, proj) | 写成mulM(proj, view) |

**完整代码见**：https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/06-25
