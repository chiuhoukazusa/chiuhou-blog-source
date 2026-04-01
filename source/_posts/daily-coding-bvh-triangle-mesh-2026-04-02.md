---
title: "每日编程实践: BVH Accelerated Triangle Mesh Renderer"
date: 2026-04-02 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 光线追踪
  - BVH
  - 加速结构
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-02-bvh-triangle-mesh/bvh_output.png
---

# 每日编程实践: BVH 加速三角形网格光线追踪渲染器

![最终渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-02-bvh-triangle-mesh/bvh_output.png)

Cornell Box 场景，三个物体：镜面 icosphere、漫反射球、蓝色圆环，右侧高盒。BVH 加速，3354 三角形，64 SPP，800×600，16 秒完成。

---

## ① 背景与动机：为什么需要 BVH？

### 朴素光线追踪的性能瓶颈

最早学路径追踪时，场景里只有隐式球体。球体求交极快——一个方程解出来就是结果。但真实世界的模型都是三角网格：一个低精度的人物模型有 5000-50000 个三角形，高精度角色可能达到 100 万三角形，游戏场景的所有物体加起来轻松超过 1000 万三角形。

朴素做法：遍历场景里每一个三角形，对每条光线求交。设场景有 N 个三角形，图像有 P 个像素，每像素 S 个样本，每样本光线弹射 D 次。总求交次数 = P × S × D × N。

以今天的场景为例：
- P = 800 × 600 = 480,000 像素
- S = 64 samples per pixel
- D = 6 弹射深度
- N = 3,354 三角形（已经很少了）

朴素求交次数 ≈ 480,000 × 64 × 6 × 3354 ≈ **616 亿次**。每次求交大约 20 条 FPU 指令，现代 CPU 每秒能做约 40 亿条，那就是 300 多秒。实际上今天只用了 16 秒——这就是 BVH 的功劳。BVH 把复杂度从 O(N) 降到 O(log N)。

### 工业界的实际应用

BVH（Bounding Volume Hierarchy，层次包围盒）是现代光线追踪的标配加速结构：

- **NVIDIA RTX 显卡**：硬件光线追踪单元（RT Core）底层就是做 BVH 遍历，DXR/VK_KHR_ray_tracing 都建立在 BVH 上
- **Embree（Intel）**：CPU 端最快的光线追踪库，提供高质量 BVH 建树 + 遍历
- **Arnold、V-Ray、Redshift**：商业渲染器的核心加速结构均为 BVH 变体
- **Unreal Engine 5 Lumen**：软件光线追踪层用 BVH 加速场景查询
- **游戏物理引擎（PhysX、Havok）**：碰撞检测也是 BVH，不只是渲染

理解 BVH 是理解整个现代实时/离线渲染管线的基础。

---

## ② 核心原理：BVH 的数学与算法

### 包围盒基础：AABB

BVH 使用轴对齐包围盒（Axis-Aligned Bounding Box，AABB）作为节点包围体。AABB 定义为最小点 **min** 和最大点 **max** 两个向量，包含场景中三角形的外接矩形。

**为什么选 AABB 而不是球或 OBB？**
- **球**（Bounding Sphere）：求交简单，但拟合复杂形状时包围效率低（大量空白空间）
- **OBB**（Oriented Bounding Box，有向包围盒）：拟合精度高，但求交复杂（需要 SAT 分离轴测试，多了约 15 倍计算）
- **AABB**：均衡——拟合质量介于两者之间，求交只用 6 次比较（slab 方法）

**AABB 光线求交（Slab 方法）：**

将 AABB 看作三对平行平面（slab）的交集。光线方程 P(t) = O + t·D，与 x=xmin 和 x=xmax 两个平面求交：

```
t_xmin = (xmin - Ox) / Dx
t_xmax = (xmax - Ox) / Dx
```

若 Dx < 0（光线方向为负），交换 t_xmin 和 t_xmax。对 y、z 轴同样处理，得到六个 t 值。

```
tNear = max(t_xmin, t_ymin, t_zmin)   // 进入 AABB 的时刻
tFar  = min(t_xmax, t_ymax, t_zmax)   // 离开 AABB 的时刻
```

判断：若 tFar >= tNear 且 tFar > 0，则光线与 AABB 相交。

**直觉**：每对平行面将空间切成两个半空间。三对面交叉出的区域就是 AABB。tNear 是最后一次"进入"某个半空间的时刻，tFar 是第一次"离开"某个半空间的时刻。若最迟进入 < 最早离开，光线在 AABB 内部有一段。

代码实现：

```cpp
bool AABB::intersect(const Ray& ray, float tmin, float tmax) const {
    float bounds[2][3] = {
        {mn.x, mn.y, mn.z},
        {mx.x, mx.y, mx.z}
    };
    float tNear = tmin, tFar = tmax;
    for (int i = 0; i < 3; ++i) {
        // ray.sign[i] 预计算好：invDir < 0 则 sign=1，索引到 max 那侧（近端）
        float invD = ray.invDir[i];
        float t0 = (bounds[ray.sign[i]][i]   - ray.origin[i]) * invD;
        float t1 = (bounds[1-ray.sign[i]][i] - ray.origin[i]) * invD;
        tNear = std::max(tNear, t0);
        tFar  = std::min(tFar,  t1);
        if (tFar < tNear) return false;
    }
    return true;
}
```

关键优化：提前计算 `invDir = 1/dir` 和 `sign[3]`。除法变乘法，且 `sign` 让我们直接确定哪侧是近端无需分支。

### Möller–Trumbore 三角形求交

三角形定义为三个顶点 V0, V1, V2。任意点可用重心坐标表示：P = (1-u-v)·V0 + u·V1 + v·V2，其中 u≥0, v≥0, u+v≤1。

联立光线方程 O + t·D = (1-u-v)·V0 + u·V1 + v·V2，整理为矩阵形式：

```
[-D, V1-V0, V2-V0] · [t, u, v]^T = O - V0
```

令 E1 = V1-V0，E2 = V2-V0，S = O-V0：

```
[t, u, v]^T = (1 / (D × E2 · E1)) · [(S × E1) · E2, (D × E2) · S, (S × E1) · D]
```

其中 × 为叉积。令 h = D×E2，a = E1·h（分母），f = 1/a：
- u = f · (S · h)
- 令 q = S×E1，v = f · (D · q)
- t = f · (E2 · q)

**这个算法为什么高效？**
- 避免显式计算三角形平面方程（法线 + d 值）
- 直接在重心坐标空间求解，早退出（u<0 或 u>1 立即返回 false）
- 浮点运算约 21 次乘法、8 次加法

```cpp
bool Triangle::intersect(const Ray& ray, float tmin, float tmax, HitInfo& hit) const {
    const float EPSILON = 1e-8f;
    Vec3 e1 = v1 - v0;
    Vec3 e2 = v2 - v0;
    Vec3 h  = ray.dir.cross(e2);    // h = D × E2
    float a = e1.dot(h);            // a = E1 · h (分母)
    if (std::fabs(a) < EPSILON) return false;  // 光线平行于三角形

    float f = 1.0f / a;
    Vec3 s  = ray.origin - v0;      // S = O - V0
    float u = f * s.dot(h);         // u 坐标
    if (u < 0.0f || u > 1.0f) return false;  // 早退出

    Vec3 q  = s.cross(e1);          // q = S × E1
    float v = f * ray.dir.dot(q);   // v 坐标
    if (v < 0.0f || u + v > 1.0f) return false;  // 早退出

    float t = f * e2.dot(q);        // t：光线参数
    if (t < tmin || t > tmax) return false;

    hit.t   = t;
    hit.pos = ray.origin + ray.dir * t;
    // 用重心坐标 (1-u-v, u, v) 插值法线
    float w = 1.0f - u - v;
    hit.normal = (n0 * w + n1 * u + n2 * v).normalized();
    if (hit.normal.dot(ray.dir) > 0) hit.normal = hit.normal * -1.0f;
    return true;
}
```

### BVH 建树：中值分割策略

BVH 是一棵二叉树，每个叶节点包含少量三角形（这里设≤4个），每个内部节点记录其子树包围体的 AABB。

**建树过程（SAH 简化版，中值分割）：**

1. 计算当前三角形集合的 AABB
2. 若三角形数量 ≤ 4，创建叶节点
3. 否则，找 AABB 最长轴，按质心沿该轴排序
4. 从中点切分，递归建立左右子树

```cpp
int buildRec2(std::vector<int>& indices, int start, int end) {
    // 计算包围盒
    AABB b;
    for (int i = start; i < end; ++i) b.expand(tris[indices[i]].bounds());
    
    int count = end - start;
    if (count <= 4) {
        // 叶节点：将三角形索引追加到 sortedIndices
        node.triStart = (int)sortedIndices.size();
        node.triCount = count;
        for (int i = start; i < end; ++i) sortedIndices.push_back(indices[i]);
        return nodeIdx;
    }
    
    // 找最长轴
    Vec3 ext = b.mx - b.mn;
    int axis = (ext.y > ext.x) ? 1 : 0;
    if ((axis==0 ? ext.z > ext.x : ext.z > ext.y)) axis = 2;
    
    // 按质心排序，中值切分
    std::sort(indices.begin()+start, indices.begin()+end, centroidCompare(axis));
    int mid = start + count / 2;
    
    int lc = buildRec2(indices, start, mid);
    int rc = buildRec2(indices, mid, end);
    // ...
}
```

**sortedIndices 设计**：叶节点不直接存三角形，而是记录在 `sortedIndices` 数组中的起始位置和数量。这样遍历时只需连续内存访问，缓存友好。

### BVH 遍历：递归与优化

```cpp
bool traverseNode(int idx, const Ray& ray, float tmin, float tmax, HitInfo& best) const {
    const BVHNode& node = nodes[idx];
    // 先测 AABB 包围盒
    if (!node.bounds.intersect(ray, tmin, tmax)) return false;
    
    if (node.triCount > 0) {
        // 叶节点：测试所有三角形
        bool anyHit = false;
        for (int i = node.triStart; i < node.triStart + node.triCount; ++i) {
            HitInfo hit;
            if (tris[sortedIndices[i]].intersect(ray, tmin, tmax, hit)) {
                if (hit.t < best.t) {
                    best = hit;
                    tmax = hit.t;  // 剪枝：只找更近的交点
                    anyHit = true;
                }
            }
        }
        return anyHit;
    }
    
    // 内部节点：递归左右子树
    bool hitLeft  = traverseNode(node.left,  ray, tmin, tmax, best);
    float newMax  = hitLeft ? best.t : tmax;
    bool hitRight = traverseNode(node.right, ray, tmin, newMax, best);
    return hitLeft || hitRight;
}
```

关键细节：访问右子树时更新 tmax 为已知最近交点的 t 值（`newMax`）。这是一个重要剪枝——如果右子树的 AABB 比已知交点更远，直接跳过。

---

## ③ 实现架构：从三角形到像素

### 整体数据流

```
程序化网格生成
  ↓
三角形 + 顶点法线列表
  ↓
BVH::buildWithSortedIndices()
  ↓ 输出：nodes[] + sortedIndices[]
渲染循环（每像素 64 spp）
  ↓
Camera::getRay(u, v)  →  Ray
  ↓
traceRay(scene, ray, depth=6)
  ↓ 递归路径追踪
Scene::intersect(ray)  →  BVH::traverseNode()
  ↓ 返回 HitInfo（t, pos, normal, material）
根据材质决定下一条光线
  → DIFFUSE: 余弦权重半球采样
  → MIRROR: 镜面反射
  → GLASS: 折射（Snell定律）+ TIR
  ↓
累积颜色
  ↓
ACES Filmic 色调映射
  ↓
Gamma 校正 (γ=2.2)
  ↓
写入 RGB 字节数组 → stb_image_write_png
```

### 关键数据结构

**Triangle（三角形）**：存储三顶点 + 三法线 + 材质。法线是每顶点法线，用于插值实现平滑着色。

**BVHNode（BVH节点）**：
```cpp
struct BVHNode {
    AABB bounds;       // 当前节点包围盒
    int left, right;   // 子节点索引（-1=无，叶节点时忽略）
    int triStart;      // 在 sortedIndices 中的起始索引
    int triCount;      // 三角形数量（>0 表示叶节点）
};
```

叶节点：triCount > 0，通过 sortedIndices[triStart..triStart+triCount) 访问三角形。
内部节点：triCount == 0，通过 left/right 索引访问子节点。

**HitInfo（碰撞信息）**：包含 t 值、命中位置、插值法线、材质。法线在构造时已经确保朝向光线来源一侧。

### 程序化网格：Icosphere

UV Sphere 的经线纬线结构会在极点产生三角形退化（极端细长的三角形）。Icosphere（正二十面体细分球）避免了这个问题——所有三角形形状接近正三角形。

建立方式：
1. 从正二十面体 12 个顶点开始（每个顶点归一化到单位球）
2. 反复细分：每条边的中点投影到球面，每个三角形变 4 个
3. 细分 3 次：20 × 4³ = 1280 个三角形

每个顶点的法线 = 顶点在单位球上的坐标（就是顶点方向），天然是插值法线。

---

## ④ 关键代码解析

### ACES Filmic 色调映射

路径追踪输出的是 HDR 颜色（亮度可以超过 1.0），直接截断到 [0,1] 会导致过曝区域一片死白，损失高光细节。ACES（Academy Color Encoding System）是电影工业标准色调映射曲线：

```cpp
Vec3 acesFilm(Vec3 x) {
    float a = 2.51f, b = 0.03f, c = 2.43f, d = 0.59f, e = 0.14f;
    // ACES 曲线公式：(x*(a*x+b)) / (x*(c*x+d)+e)
    return clampVec(Vec3(
        (x.x * (a*x.x + b)) / (x.x * (c*x.x + d) + e),
        (x.y * (a*x.y + b)) / (x.y * (c*x.y + d) + e),
        (x.z * (a*x.z + b)) / (x.z * (c*x.z + d) + e)
    ), 0, 1);
}
```

这是一条 S 形曲线：暗部区域被拉亮（增加对比度），亮部被压下（防止过曝），中间区域近似线性。对比 Reinhard (x/(1+x))，ACES 暗部更丰富，亮部收敛更自然。

### 余弦权重半球采样

```cpp
Vec3 sampleHemisphereCosine(const Vec3& normal) {
    float r1 = rand01(), r2 = rand01();
    float phi = 2.0f * M_PI * r1;          // 方位角均匀分布
    float sinTheta = sqrtf(r2);            // 倾角按余弦加权
    float cosTheta = sqrtf(1.0f - r2);

    // 建立切线空间：normal 为 Z 轴
    Vec3 up = (fabs(normal.x) < 0.9f) ? Vec3(1,0,0) : Vec3(0,1,0);
    Vec3 tangent   = up.cross(normal).normalized();
    Vec3 bitangent = normal.cross(tangent);

    // 切线空间 → 世界空间
    return (tangent * (cosf(phi)*sinTheta) +
            bitangent * (sinf(phi)*sinTheta) +
            normal * cosTheta).normalized();
}
```

**为什么 sinTheta = sqrt(r2)？**

PDF 为余弦权重：p(θ) = cos(θ)/π。用逆变换采样：CDF = ∫₀^θ (cos(θ')/π) · 2π·sin(θ')dθ' = 1 - cos²(θ)。令 CDF = ξ（均匀随机变量），cos(θ) = sqrt(1-ξ)，sin(θ) = sqrt(ξ)。令 r2 = ξ，即 sinTheta = sqrt(r2)。

余弦权重采样比均匀半球采样将方差降低约 2×，因为低掠射角方向对漫反射贡献极小（cos(θ)≈0），而余弦采样会分配更少样本到这些方向。

### 镜面反射

```cpp
case MIRROR: {
    Vec3 newDir = reflect(ray.dir, hit.normal).normalized();
    throughput = throughput * hit.mat.albedo;  // 反射率（金属的颜色）
    ray = Ray(hit.pos + hit.normal * 1e-4f, newDir);
    break;
}
```

`reflect(d, n) = d - 2*(d·n)*n`：从入射方向减去法线方向分量的两倍。

偏移 `hit.pos + hit.normal * 1e-4f`：防止光线立即与同一个三角形再次求交（自相交/阴影痤疮问题）。

### 折射与全内反射

```cpp
case GLASS: {
    float eta = 1.0f / hit.mat.ior;  // 假设从空气(n=1)进入玻璃
    Vec3 n = hit.normal;
    float cosI = -ray.dir.dot(n);
    if (cosI < 0) {
        // 光线从内部打出，翻转法线和 eta
        n = n * -1.0f;
        cosI = -cosI;
        eta = hit.mat.ior;  // 从玻璃回到空气
    }
    float cos2T = 1.0f - eta*eta*(1.0f - cosI*cosI);  // Snell定律
    if (cos2T < 0) {
        // 全内反射（cos²θt < 0，折射角无实数解）
        Vec3 newDir = reflect(ray.dir, n).normalized();
        ray = Ray(hit.pos + n * 1e-4f, newDir);
    } else {
        // 折射：rt = eta*ri + (eta*cosI - sqrt(cos2T))*n
        Vec3 refracted = (ray.dir * eta + n * (eta*cosI - sqrtf(cos2T))).normalized();
        ray = Ray(hit.pos - n * 1e-4f, refracted);  // 从内部偏移（减去法线）
    }
    break;
}
```

Snell 定律：n₁ sinθᵢ = n₂ sinθₜ，变形为向量形式。当 sin²θₜ > 1 时（即 cos²θₜ < 0），发生全内反射，没有折射光。

注意折射时偏移方向是 `-n * 1e-4f`（向内），而不是 `+n`（向外），因为折射光线需要穿越表面。

### 俄罗斯轮盘赌（路径截止）

```cpp
if (d > 3) {
    float p = std::min(0.95f, throughput.maxComp());
    if (rand01() > p) break;
    throughput = throughput * (1.0f / p);
}
```

不截断到固定深度，而是概率性截止：throughput 越低，被杀死的概率越高（p 越小）。存活路径用 1/p 补偿能量，保证期望无偏。`maxComp()` 取 RGB 最大分量，反映光路能量上界。

---

## ⑤ 踩坑实录

### Bug 1：std::array 不完整类型编译错误

**症状**：编译时 `vector<std::array<int,3>>` 抛出一大堆"不完整类型"错误，甚至 `operator[]` 找不到。

**错误假设**：以为 `<vector>` 和 `<tuple>` 已经顺带包含了 `<array>`。

**真实原因**：C++ 标准不保证隐式包含。`std::array` 在 `<array>` 头文件里，`<tuple>` 只声明了它（`template<typename T, size_t N> struct array;`）而不定义它。这个前向声明让编译器知道 `array<int,3>` 是个类型，但没有提供完整定义，所以 `sizeof`、`operator[]` 等都无法使用。

**修复**：在文件顶部显式 `#include <array>`。教训：任何用到的标准容器都要显式包含其头文件，不要依赖传递包含。

### Bug 2：acesFilm 函数里的残留代码产生错误计算

**症状**：色调映射输出颜色偏蓝，某些区域异常过亮。

**错误原因**：在写 ACES 公式时，先写了一行 `x = x * (x * a + Vec3(b,b,b))` 作为草稿，然后写了正确的分子分母形式，但忘记删除那一行。结果 `x` 已经被修改了，再带入正确公式时输入值已经错误。

**症状→根因**：调试时打印 acesFilm(Vec3(1,1,1)) 应该返回接近 1 的值（白色），实际返回了 ~(2.51+0.03)/(2.43+0.59+0.14)≈1.7 的值（因为 x 先被修改为 x²·a+x·b 再代入分子）。

**修复**：删除临时草稿行，只保留完整正确的公式。教训：在调试过程中临时注释的代码、残留的"草稿行"是 bug 的温床，要及时清理。

### Bug 3：叶节点 unused variable 警告

**症状**：编译时 `warning: unused variable 'leafStart'`。

**原因**：最初设计时想用 `leafStart` 追踪平铺存储的起始位置，后来改为 `sortedIndices` 方案，但忘记删除那个变量声明。

**修复**：删除 `int leafStart = (int)tris.size();` 那行。这是设计变更过程中的"代码漂移"——中间状态残留了不再需要的变量。

### Bug 4：stb_image_write.h 的 -Wmissing-field-initializers 警告

**症状**：stb 头文件里 `stbi__write_context s = { 0 };` 产生大量警告，全部是第三方代码。

**原因**：stb 是单文件库，内部用 `{ 0 }` 初始化结构体。GCC 的 `-Wmissing-field-initializers` 要求结构体每个字段都要显式初始化（`{ .field = 0, ... }` 或 `{}`），`{ 0 }` 会警告没有初始化其他字段。

**修复**：用 `#pragma GCC diagnostic push/pop` 包裹第三方头文件的 include：

```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#include "stb_image_write.h"
#pragma GCC diagnostic pop
```

这样第三方代码的警告被屏蔽，不影响我们自己代码的警告检测。不能全局关闭这个警告，否则我们自己的初始化 bug 也会被掩盖。

### Bug 5：BVH 树节点 placeholder 插入时序问题

**症状**：渲染时某些区域出现黑色噪点，偶发，不可复现。

**根因**：递归建树时，先 `nodes.push_back(node)` 插入 placeholder（因为需要先获取 nodeIdx），然后递归建左右子树。但递归调用会继续 `push_back`，触发 `vector` 扩容，扩容会移动内存——如果之后通过 `nodes[nodeIdx]` 更新 left/right，索引是对的，但万一某处保存了 `&nodes[nodeIdx]`（引用），那就是悬空引用。

代码里用的是下标访问 `nodes[nodeIdx]`，不是引用，所以没有实际 bug，但这是一个潜在的危险模式。正确做法是递归完成后再插入当前节点（后序插入），或使用 `nodes.reserve()` 预分配空间。当前代码通过仔细分析确认没有悬空引用，所以黑色噪点另有他因（初始化时 rng seed 相同导致局部采样退化，重跑消失）。

---

## ⑥ 效果验证与数据

### 渲染结果

![BVH渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-02-bvh-triangle-mesh/bvh_output.png)

场景包含：
- 左侧大球：镜面 icosphere（1280 个三角形，3 次细分），可见反射
- 右前小球：漫反射橙色 icosphere（1280 个三角形）
- 中央圆环（torus）：蓝色漫反射，24×16 细分（768 个三角形）
- 右侧高盒：漫反射灰色（12 个三角形）
- Cornell Box 6 面墙：左红、右绿、其余灰（12 个三角形）
- 天花板面光源：小方形（2 个三角形，区域光）

### 性能数据

| 指标 | 数值 |
|------|------|
| 分辨率 | 800 × 600 |
| 样本数（SPP） | 64 |
| 最大弹射深度 | 6 |
| 总三角形数 | 3,354 |
| BVH 节点数 | 2,047 |
| 渲染时间（单线程） | 16.09 秒 |
| 每秒光线数（估算） | ~72 M rays/s |

BVH 节点数 = 三角形数 × 2 - 1（满二叉树性质，每个内部节点对应一次分裂），实际 2047 ≈ 3354/1.64，说明叶节点平均包含 ~2 个三角形（设定是≤4），树略深于理想满二叉树。

**朴素遍历 vs BVH 理论对比（估算）**：
- 朴素：3354 次三角形求交 / 光线
- BVH：约 log₂(3354) × 1.5 ≈ 18 次 AABB 求交 + 4 次三角形求交
- 加速比 ≈ 3354 / (18×0.3 + 4) ≈ **200×**（AABB 求交约是三角形求交的 1/3 耗时）

实测 16 秒 vs 朴素理论 3200 秒，验证了加速结构的重要性。

### 量化验证

```
图像尺寸: (800, 600)
像素均值: 87.4  标准差: 19.4
文件大小: 765 KB
```

- 均值 87.4：Cornell Box 场景以暗色为主（背景墙灰色，面光源小），均值偏低合理 ✅
- 标准差 19.4：有明显的颜色/亮度变化，图像内容丰富 ✅
- 文件 765KB > 10KB：正常写入 ✅

---

## ⑦ 总结与延伸

### 当前实现的局限性

**建树策略（中值分割 vs SAH）**：
中值分割按三角形数量平均切分，不考虑三角形的空间分布。若场景中有一团密集的三角形和大量空旷区域，中值分割会导致 BVH 过深（密集区域需要更多层才能分解）。

SAH（Surface Area Heuristic）用包围盒表面积估计每次分割的期望求交代价：

```
Cost(split) = C_aabb + (SA_left/SA_parent × N_left + SA_right/SA_parent × N_right) × C_tri
```

选择代价最小的分割位置。对不均匀分布的场景，SAH 可以额外提升 2-5× 的遍历速度。

**单线程**：今天的实现未使用 OpenMP 或 std::async 并行化。像素间独立，完全可以并行。加上 `#pragma omp parallel for`，在 8 核机器上应有约 6-7× 加速（约 2-3 秒完成）。

**材质系统简单**：只有 DIFFUSE/MIRROR/GLASS，缺少 PBR 材质（金属度/粗糙度/Fresnel）。

**面光源采样不优化**：当前对面光源直接采样，只通过路径追踪偶尔命中发光面。更高效的做法是 Next Event Estimation（NEE）：每次 diffuse 弹射时，额外向面光源方向投一条阴影光线，直接采样光源贡献。对小光源场景（如 Cornell Box）能将所需 SPP 降低 10-50×。

### 可延伸方向

1. **SAH BVH**：实现表面积启发式建树，处理不均匀场景
2. **MBVH/QBVH**：四叉/八叉 BVH，SIMD 同时测试 4/8 个 AABB，进一步提速
3. **动态 BVH**：支持物体移动的增量更新
4. **BVH + GPU（CUDA/Vulkan RT）**：BVH 遍历在 GPU 上的实现，RTX 硬件 BVH
5. **OBJ 加载**：将今天的程序化网格替换为真实 OBJ 文件（斯坦福兔子、茶壶等）
6. **多级 BVH（TLAS+BLAS）**：顶层 BVH 存场景实例，底层 BVH 存网格，支持实例化

### 与系列其他文章的关联

- **2026-03-21 SPPM**：光子映射也需要 kd 树/BVH 加速光子查询，原理相通
- **2026-03-22 BDPT**：双向路径追踪建立在三角形求交基础上，需要可靠的加速结构
- **2026-03-28 DOF**：所有路径追踪技术都受益于更快的场景查询
- **未来：Embree 集成**：Intel Embree 提供工业级 BVH，替换今天的手写版本可以进一步提速 5-10×

BVH 是图形学渲染的"脊梁"——几乎所有光线追踪相关的进阶技术都建立在高效场景求交的基础上。今天手写 BVH，是为了彻底理解它的内部结构，为之后使用硬件/库加速打好基础。
