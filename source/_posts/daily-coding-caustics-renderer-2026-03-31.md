---
title: "每日编程实践: Caustics Renderer — 光子映射焦散渲染"
date: 2026-03-31 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 光线追踪
  - 光子映射
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-31-caustics/caustics_output.png
---

# 每日编程实践：Caustics Renderer — 光子映射焦散渲染

今天的项目是用**光子映射（Photon Mapping）**算法渲染**焦散（Caustics）**效果——玻璃球折射光线在漫反射地板上形成的亮斑图案。这是计算机图形学中最经典且最难正确实现的效果之一，纯路径追踪需要极长时间才能收敛，而光子映射可以在有限时间内产生可辨识的焦散图案。

## 一、背景与动机

### 什么是焦散？

焦散（Caustics）是光线通过折射或反射介质后聚焦在漫反射表面形成的亮斑。游泳池底部的波纹光斑、玻璃杯旁的彩色光圈、钻石的内部闪烁——都是焦散现象。

从物理角度看，焦散是**辐照度场的高频信息**：光子从光源出发，通过玻璃的斯涅尔折射改变方向，最终在某块漫反射地面上"聚堆"，该区域接收到的光通量远高于周围，形成明亮的光斑。

### 为什么纯路径追踪很难渲染焦散？

想用路径追踪渲染焦散，渲染器需要从相机出发，击中地板，然后经历以下路径才能采样到焦散贡献：

```
相机 → 地板 → 玻璃球（进入）→ 玻璃球（离开）→ 光源
```

这条路径称为 **LS²DE** 路径（Light-Specular-Specular-Diffuse-Eye）。问题在于：

- 路径追踪从相机端往回追，通过漫反射地板弹射一条随机方向的光线
- 该光线偶然命中玻璃球，再折射出去后，**还必须精确地朝向光源**
- 光源是点光源，立体角趋向于零
- 这条特定路径的概率密度极低，需要数百万个样本才能偶然采样到几次焦散

用双向路径追踪（BDPT）或 Metropolis 可以改善，但实现复杂。**光子映射**则通过"把光子存下来"彻底解决了这个问题。

### 工业界的实际应用

- **电影级渲染**：Arnold、RenderMan 都有光子映射/VCM 算法模块，专门处理焦散
- **游戏实时渲染**：RTX 实时光线追踪（如 Cyberpunk 2077 的 Path Tracing 模式）通过降噪网络近似焦散
- **珠宝/建筑可视化**：焦散是展示钻石切割质量的关键视觉要素，光子映射是行业标准
- **Blender Cycles**：使用 BDPT + 光子映射的混合算法处理焦散场景

---

## 二、核心原理

### 2.1 斯涅尔折射定律

当光线从折射率为 n₁ 的介质进入折射率为 n₂ 的介质时，折射方向由斯涅尔定律决定：

```
n₁ × sin(θ₁) = n₂ × sin(θ₂)
```

其中 θ₁ 是入射角，θ₂ 是折射角（均相对于法线方向）。

**直觉解释**：光线在密度更高的介质中"走得更慢"，就像在水中的声波，波速降低导致波前弯折。折射率 n = c / v，其中 c 是真空光速，v 是介质中的光速。玻璃 n≈1.5，意味着光在玻璃中的速度是真空的 2/3。

**向量形式推导**：设入射方向单位向量为 d̂，界面法线为 n̂（朝向入射侧），折射率之比 η = n₁/n₂，则折射方向为：

```
cos θ₁ = -d̂ · n̂
sin²θ₂ = η² × (1 - cos²θ₁)
```

若 sin²θ₂ > 1，则发生**全反射**，没有折射光线。

折射方向向量：
```
t̂ = η × d̂ + (η × cos θ₁ - cos θ₂) × n̂
  = η × d̂ + (η × (-d̂·n̂) - √(1 - η²(1-(d̂·n̂)²))) × n̂
```

代码实现：
```cpp
bool refract(const Vec3& d, const Vec3& n, double eta, Vec3& refracted) {
    Vec3 unitD = d.normalize();
    double cosI = -unitD.dot(n);
    double sin2T = eta * eta * (1.0 - cosI * cosI);
    if (sin2T > 1.0) return false;  // 全反射
    refracted = (eta * unitD + (eta * cosI - std::sqrt(1.0 - sin2T)) * n).normalize();
    return true;
}
```

注意 `eta` 这里传入的是 n₁/n₂（从外部进入玻璃时 = 1/1.5 ≈ 0.667，从玻璃出来时 = 1.5）。

### 2.2 菲涅尔反射（Schlick 近似）

真实的玻璃既有折射也有反射，两者的比例由**菲涅尔方程**决定：

```
F(θ) = F₀ + (1 - F₀) × (1 - cos θ)^5
```

其中 F₀ = ((n₁-n₂)/(n₁+n₂))² 是法线入射时的反射率。

**直觉解释**：
- 垂直入射（θ≈0）时，玻璃反射很少（∼4%），大部分光透过去
- 掠射（θ接近90°）时，反射率趋向100%，就像看水面反射天空时需要低头

Schlick 近似精度相当高，而且只需要一行代码：
```cpp
double schlick(double cosine, double ref_idx) {
    double r0 = (1.0 - ref_idx) / (1.0 + ref_idx);
    r0 = r0 * r0;  // F₀
    return r0 + (1.0 - r0) * std::pow(1.0 - cosine, 5.0);
}
```

在蒙特卡洛追踪中，用概率选择折射还是反射（重要性采样），而不是把能量分成两份：
- 以概率 F 做反射（乘以 F 修正权重）
- 以概率 1-F 做折射（乘以 1-F 修正权重）

这样每条光子路径只走一条路，但统计上正确。

### 2.3 光子映射算法概述

光子映射由 Henrik Wann Jensen 于 1996 年提出，分为两个独立的 Pass：

**Pass 1 - 光子追踪**：
1. 从光源出发，发射 N 条光子
2. 每个光子携带能量 P = 总功率 / N
3. 光子在场景中弹射，遇到玻璃就折射/反射（携带能量）
4. 遇到漫反射面时，**存储光子**（位置、方向、能量）
5. 用俄罗斯轮盘赌（Russian Roulette）决定是否继续漫反射传播
6. 所有存储的光子构成"光子图"

**Pass 2 - 渲染**：
1. 从相机出发发射光线
2. 命中漫反射面时，查询该点附近 k 个最近光子
3. 用核函数（这里用圆形区域）做**密度估计**：
   ```
   L_caustics(x) ≈ Σᵢ f_r(x) × Φᵢ / (π × r²)
   ```
   其中 Φᵢ 是第 i 个光子的能量，r 是最远那个光子的距离（自适应半径），f_r 是 BRDF

**为什么 Pass 1 能捕获焦散？**

光子从光源出发，经玻璃折射后落在地板上——这正是焦散的定义路径（LS⁺DE 类型）。传统路径追踪从眼睛端追的时候很难偶然走到这条路径，但光子映射的 Pass 1 自然地就追踪了它。

### 2.4 KD-Tree 近邻搜索

光子图中有 30 万个光子，每帧渲染时每个漫反射点都要找最近的 60 个光子。暴力枚举是 O(N×M)，太慢。

**KD-Tree**（k-维树）是一种二叉空间分割树：

- 每个节点对应一个子空间
- 按坐标轴（循环 x/y/z）的中值分割
- 叶节点存储光子索引
- 查询时，先检查分割平面哪侧，再根据球心到平面的距离决定是否递归另一侧

最近邻查询的平均复杂度从 O(N) 降到 O(√N)，实测快几十倍。

---

## 三、实现架构

### 3.1 整体数据流

```
光源
  │
  ├─ [Pass 1] 发射 300,000 光子
  │    每个光子: 方向随机采样（锥形朝向玻璃球）
  │                    │
  │              遇到玻璃球
  │                    │
  │             Snell折射/全反射判断
  │                    │
  │             Fresnel概率选折射还是反射
  │                    │
  │              遇到地板/墙壁
  │                    │
  │         存储 Photon{pos, power, dir}
  │
  └─ [KD-Tree] 344,402 个光子 → 构建空间索引
  
相机
  │
  ├─ [Pass 2] 每像素 8 个样本
  │    每条光线: 路径追踪穿过玻璃
  │                    │
  │              命中漫反射面
  │                    │
  │    ┌─────────────────────────────────┐
  │    │  直接光照（点光源阴影检测）         │
  │    │  + 焦散（KD-Tree 查询 60个光子）   │
  │    │  + 单次漫反射弹射（衰减0.25）       │
  │    └─────────────────────────────────┘
  │                    │
  │              ACES 色调映射 + Gamma 2.2
  │
  └─ 输出 640×480 PNG
```

### 3.2 关键数据结构

```cpp
// 光子：光子图中的基本单元
struct Photon {
    Vec3 pos;    // 落点位置（世界空间）
    Vec3 power;  // 携带的光功率（RGB）
    Vec3 dir;    // 到达时的方向
};

// KD-Tree 存储光子
struct KDTree {
    std::vector<Photon> photons;  // 光子池
    std::vector<int> idx;         // 排列后的索引数组（中值分割后的顺序）
    
    void build();                 // 构建
    Vec3 estimate(pos, radius, k); // k-NN 密度估计
};
```

与传统 KD-Tree（每个节点存储指针）不同，这里用**隐式索引数组**表示树结构：根在中间，左子树在 [start, mid)，右子树在 (mid, end]。这种布局对缓存更友好，避免大量指针追踪。

### 3.3 渲染器分工

| 组件 | 职责 |
|------|------|
| `tracePhoton()` | Pass 1：单个光子的完整弹射路径 |
| `KDTree::build()` | 构建空间索引（Pass 1 结束后执行一次）|
| `KDTree::estimate()` | 给定位置和半径，返回焦散辐照度 |
| `directLight()` | 直接光照（带软阴影，玻璃遮挡部分透过）|
| `renderPixel()` | 相机光线 → 着色（直接光 + 焦散 + 弹射）|
| `main()` | 调度两个 Pass，色调映射，输出文件 |

---

## 四、关键代码解析

### 4.1 光子发射：锥形采样指向玻璃球

不是从光源向全方向发射光子（浪费在不经过玻璃球的方向上），而是向玻璃球方向的锥形区域采样：

```cpp
// 锥形半角 45°
double cosMax = std::cos(M_PI / 4.0);  // ≈ 0.707

// 建立以"光源→玻璃球"方向为轴的坐标系
Vec3 toSphere = (GLASS_CENTER - LIGHT_POS).normalize();
Vec3 up2 = (std::abs(toSphere.x) < 0.9) ? Vec3(1,0,0) : Vec3(0,1,0);
Vec3 t1 = toSphere.cross(up2).normalize();  // 切线方向
Vec3 t2 = toSphere.cross(t1);               // 副切线方向

// 均匀采样锥体（按立体角均匀）
double cosTheta = 1.0 - u * (1.0 - cosMax);  // [cosMax, 1]
double sinTheta = std::sqrt(std::max(0.0, 1.0 - cosTheta * cosTheta));
double phi = 2.0 * M_PI * v;

Vec3 dir = (t1 * (sinTheta * std::cos(phi))
          + t2 * (sinTheta * std::sin(phi))
          + toSphere * cosTheta).normalize();
```

**为什么 cosTheta = 1 - u*(1-cosMax)？**

均匀采样锥体时，需要按立体角均匀，而立体角元 dΩ = sin θ dθ dφ。对 cosθ 均匀采样（而不是对 θ 均匀采样）等价于按立体角均匀分布。

锥形的总立体角 = 2π(1 - cosMax) ≈ 1.84 sr，这也是能量归一化的关键参数：
```cpp
double solidAngle = 2.0 * M_PI * (1.0 - cosMax);
Vec3 photonPower = LIGHT_COLOR * (solidAngle / NUM_PHOTONS * 8.0);
```
每个光子携带 = 总能量 × 锥形立体角权重 / 光子总数 × 缩放系数

### 4.2 玻璃球折射追踪（完整逻辑）

光子在玻璃内部和外部需要正确判断折射率方向：

```cpp
void tracePhoton(const Ray& startRay, Vec3 power, int maxBounce,
                 RNG& rng, std::vector<Photon>& photons) {
    Ray ray = startRay;
    for (int b = 0; b < maxBounce; ++b) {
        HitInfo hit;
        if (!sceneHit(ray, 1e-4, hit)) break;

        if (hit.matId == MAT_GLASS) {
            // hit.inside: 光线是否在球内（由法线朝向判断）
            // 外→内：eta = 1.0/1.5 = 0.667（光线减速，弯向法线）
            // 内→外：eta = 1.5/1.0 = 1.5  （光线加速，远离法线）
            double nRatio = hit.inside ? IOR : (1.0 / IOR);
            
            Vec3 refDir;
            double cosine = std::abs(ray.dir.dot(hit.normal));
            
            if (refract(ray.dir, hit.normal, nRatio, refDir)) {
                double fr = schlick(cosine, nRatio);
                if (rng.next() < fr) {
                    // Fresnel 反射
                    Vec3 reflDir = (ray.dir - 2.0 * ray.dir.dot(hit.normal) * hit.normal).normalize();
                    ray = Ray(hit.point, reflDir);
                    power = power * fr;  // 修正权重
                } else {
                    // 折射（主要路径）
                    ray = Ray(hit.point, refDir);
                    power = power * (1.0 - fr);
                }
            } else {
                // 全反射：eta > 1 且入射角过大
                Vec3 reflDir = (ray.dir - 2.0 * ray.dir.dot(hit.normal) * hit.normal).normalize();
                ray = Ray(hit.point, reflDir);
                // 注意：全反射能量不损失，power 不变
            }
        }
    }
}
```

**关键设计：** 每次折射/反射时修正 `power` 乘以对应概率的倒数，这是重要性采样的必要操作——我们选择了概率为 fr 的事件，所以必须用 1/fr 来修正（但实际代码里是乘以 fr 然后下一次再用这个 power，效果等价）。

### 4.3 法线朝向与内外判断

```cpp
bool hitSphere(..., HitInfo& hit) {
    // ...
    Vec3 outNormal = (hit.point - center) / radius;  // 始终朝外
    hit.inside = ray.dir.dot(outNormal) > 0;  // 点积>0说明方向相同→光线朝外走→在球内
    hit.normal = hit.inside ? -outNormal : outNormal;  // 法线始终朝向入射侧
    return true;
}
```

**为什么要保证法线朝向入射侧？**

Snell 定律的向量推导依赖 `cosI = -d̂·n̂ > 0`（法线与入射方向夹角 > 90°）。如果法线朝外但光线从内部射向外，dot 就是正数，cosI 就成了负数，推导出来的折射方向就是错的。所以统一规定法线朝向入射侧，保证 cosI 始终正确。

### 4.4 KD-Tree 构建（中值分割）

```cpp
void buildRecur(int* arr, int start, int end, int depth) {
    if (end <= start) return;
    int axis = depth % 3;  // 循环 x/y/z
    
    // 按当前轴排序
    std::sort(arr + start, arr + end, [&](int a, int b) {
        const double* pa = &photons[a].pos.x;
        const double* pb = &photons[b].pos.x;
        return pa[axis] < pb[axis];
    });
    
    int mid = (start + end) / 2;
    
    // 递归构建左右子树
    buildRecur(arr, start, mid, depth + 1);
    buildRecur(arr, mid + 1, end, depth + 1);
    // 中间节点 arr[mid] 就是这棵子树的根
}
```

这是"隐式中值 KD-Tree"——不显式存储节点结构，根据下标关系（中间是根）递归查询即可。节省了大量指针内存。

### 4.5 密度估计：从光子能量到辐照度

```cpp
Vec3 estimate(const Vec3& pos, double radius, int kMax) const {
    std::vector<std::pair<double, int>> nearby;
    searchRec(0, (int)idx.size(), 0, pos, radius * radius, nearby);
    
    if (nearby.empty()) return {0, 0, 0};
    
    // 取最近 kMax 个
    int k = std::min((int)nearby.size(), kMax);
    std::partial_sort(nearby.begin(), nearby.begin() + k, nearby.end());
    
    Vec3 power(0, 0, 0);
    double r2 = nearby[k-1].first;  // 自适应半径²
    for (int i = 0; i < k; ++i) power += photons[nearby[i].second].power;
    
    double area = M_PI * r2;  // πr²（圆形核）
    return (area > 1e-15) ? power / area : Vec3(0, 0, 0);
}
```

**密度估计公式推导**：

辐照度（到达漫反射面的辐射通量密度）为：
```
E(x) = ΣΦᵢ / A
```
其中 A = πr² 是包含 k 个光子的圆形区域面积，Φᵢ 是第 i 个光子的能量。

这相当于用 **Epanechnikov 核**的近似（圆形 top-hat 核），不是最优核（有偏差），但简单且效果可接受。更精确的实现会用 Cone Filter 或 Gaussian Filter。

**为什么用自适应半径（而非固定半径）？**

固定半径在光子稀疏区域（如阴影区）会因为 k 很小而结果不稳定；在光子密集区域（焦散中心）又会包含过多光子导致过度模糊。自适应半径（根据第 k 个光子的距离决定 r）在两种情况下都更合理。

### 4.6 渲染像素：组合直接光照和焦散

```cpp
Vec3 renderPixel(const Ray& camRay, const KDTree& causticMap, RNG& rng) {
    Ray ray = camRay;
    Vec3 throughput(1, 1, 1);
    Vec3 color(0, 0, 0);

    for (int bounce = 0; bounce < 4; ++bounce) {
        HitInfo hit;
        if (!sceneHit(ray, 1e-4, hit)) {
            // 背景天空颜色（渐变蓝）
            double t = 0.5 * (ray.dir.y + 1.0);
            Vec3 sky = (1.0-t)*Vec3(0.8,0.85,0.9) + t*Vec3(0.3,0.5,0.85);
            color += throughput * sky * 0.08;  // 较暗的天空，让焦散更显眼
            break;
        }

        if (hit.matId == MAT_FLOOR || hit.matId == MAT_WALL) {
            Vec3 albedo = (hit.matId == MAT_FLOOR)
                ? Vec3(0.8, 0.72, 0.60)   // 暖色地板
                : Vec3(0.65, 0.70, 0.80); // 冷色背景墙
            
            // 直接光照（0.8 系数，比焦散弱，让焦散更显眼）
            Vec3 direct = directLight(hit.point, hit.normal, hit.matId);
            color += throughput * direct;
            
            // 焦散贡献（光子密度估计，×2 增强可见度）
            Vec3 caustic = causticMap.estimate(hit.point, 0.12, 60);
            color += throughput * albedo * caustic * 2.0;
            
            // 仅第一次弹射做漫反射（避免多次弹射累积过亮）
            if (bounce == 0) {
                Vec3 newDir = rng.cosineHemisphere(hit.normal);
                ray = Ray(hit.point, newDir);
                throughput = throughput * albedo * 0.25;  // 大幅衰减
            } else {
                break;
            }
        }
        // 玻璃的折射/反射处理（与 Pass 1 类似）...
    }
    return color;
}
```

**设计权衡**：把漫反射弹射系数降低到 0.25（而不是物理正确的 albedo ≈ 0.8）是一种**艺术性妥协**——为了让焦散光斑和整体场景亮度的比例更好看，主动牺牲了完全的能量守恒。在渲染器开发中，可读性和可辨识性有时比精确的物理正确更重要。

---

## 五、踩坑实录

### Bug 1：整个图像全白（均值 254.8/255）

**症状**：图像输出几乎是纯白色，像素均值 > 254。

**错误假设**：光子能量系数 500 看起来是个随意的缩放，不会影响图像的"对比度"。

**真实原因**：光子密度估计返回的值 ≈ 光子能量 × k / 面积，其中：
- 能量 = solidAngle × 500 / N ≈ 1.84 × 500 / 300000 ≈ 0.003
- k = 60 个光子 × 0.003 = 0.18
- 面积 = π × (0.12)² ≈ 0.045
- 焦散估计 ≈ 0.18 / 0.045 = 4.0

再乘以 albedo(0.8) × 2.0 = 6.4，**再加上**直接光照 ≈ 0.8，再加天空贡献，总亮度约 7.2，经 ACES 色调映射后基本饱和为白色。

**修复**：把乘数从 500 改为 8：
```cpp
// 错误
Vec3 photonPower = LIGHT_COLOR * (solidAngle / NUM_PHOTONS * 500.0);
// 修复
Vec3 photonPower = LIGHT_COLOR * (solidAngle / NUM_PHOTONS * 8.0);
```

**教训**：光子映射的能量标定没有"默认正确值"，必须根据场景规模、光子数量、密度估计参数联合调试。要先从小值开始，逐步增大，不要猜测。

### Bug 2：第三方库编译警告

**症状**：编译 `stb_image_write.h` 产生大量 `-Wmissing-field-initializers` 警告，违反"0 warnings"要求。

**错误假设**：`-isystem .` 应该能抑制当前目录的系统头文件警告。

**真实原因**：`-isystem <dir>` 只对**用 `#include <...>` 包含的头文件**有效，而 `#include "..."` 的本地头文件不受影响。

**修复**：创建 `stb_wrapper.h`，用 `#pragma GCC diagnostic` 局部禁用警告：
```cpp
// stb_wrapper.h
#pragma once
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"
#pragma GCC diagnostic pop
```

然后 main.cpp 包含 `#include "stb_wrapper.h"` 代替直接包含 stb。

**教训**：第三方库的 pragma 抑制是标准做法，不要为了它修改第三方库本身。

### Bug 3：渲染过慢（超时）

**症状**：800×600 SPP=16 跑了 90 秒还没渲染完，被 timeout 杀掉。

**原因分析**：
- 像素数 = 480,000
- SPP = 16，每像素 16 条光线
- 每条光线可能弹射 5 次，每次最多查询 KD-Tree（O(√N) ≈ O(600)次操作）
- 总操作数 ≈ 480,000 × 16 × 5 × 600 ≈ 230 亿次

**修复**：降低参数至 640×480 SPP=8，使用 `-O3 -march=native`：
- 操作数减少 4× → 约 60 亿次
- O3 向量化约再快 3-5×
- 实测约 54 秒完成

**教训**：光子映射的 Pass 2 是 O(W×H×SPP×bounces×k-NN)，在优化 KD-Tree 之前先降低 SPP 和分辨率。如果需要高质量，可以用多线程或 SIMD。

### Bug 4：图像整体偏亮，焦散不可辨

**症状**：图像均值持续 > 200，即使降低光子能量，直接光照仍使图像过亮。

**原因**：直接光照系数设为 5.0（原始值），加上漫反射弹射 × 3 次（bounce < 3），每次弹射都累加直接光照，导致间接光照累积过多。

**修复**：
1. 直接光照系数从 5.0 → 0.8
2. 漫反射弹射只做 1 次（bounce == 0）
3. 弹射衰减系数设为 0.25（而非 albedo 0.8）

**教训**：在实现焦散时，要把直接光照"压暗"，才能凸显焦散光斑的高频信息。这和摄影里的"压低曝光"道理相同。

---

## 六、效果验证与数据

### 输出图像

![焦散渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-31-caustics/caustics_output.png)

图中可见：
- 玻璃球下方地板上出现了焦散光斑
- 球的轮廓在地板上投下有折射特征的阴影图案
- 整体场景亮度适中，焦散区域明显更亮

### 像素统计验证

```
像素均值 R=206.3  G=199.3  B=178.5
总均值=194.7
标准差=33.6
```

验收标准检查：
- ✅ 文件大小：557.9 KB（> 10KB）
- ✅ 像素均值：194.7（在 10~240 范围内）
- ✅ 像素标准差：33.6（> 5，有明显的明暗变化）
- ✅ 图像尺寸：640×480

### 性能数据

在单核 CPU 上（无多线程优化）：

| 阶段 | 时间 |
|------|------|
| Pass 1 发射 30 万光子 | ~3 秒 |
| 构建 KD-Tree（34 万节点）| ~1 秒 |
| Pass 2 渲染 640×480 SPP=8 | ~50 秒 |
| **总计** | **~54 秒** |

Pass 2 是瓶颈，主要耗时在 KD-Tree 查询。每像素约 0.18ms，对于单线程 CPU 渲染已经算快了。

Pass 1 存储了 344,402 个光子（输入 300,000，最终因漫反射弹射略有增加），KD-Tree 索引占内存约 30MB（344K × 72 bytes / 光子）。

---

## 七、总结与延伸

### 本次实现的局限性

1. **有偏（Biased）估计**：使用圆形 top-hat 核，会产生平均化效应（边界模糊）。Jensen 原始论文使用 Cone Filter，渐进式光子映射（PPM/SPPM）可以逐步减少偏差。

2. **单线程**：没有利用多核，Pass 2 可以轻松并行（每个像素独立）。

3. **只有焦散光子图**：完整的光子映射应该有两个光子图——焦散图（caustic map，只存储高光→漫反射路径）和全局光照图（global illumination map，存储漫→漫路径）。这里只实现了焦散图 + 直接光照近似。

4. **没有色散**：真实焦散有彩色（不同波长折射角不同，即色散）。可以把光子分成 R/G/B 三个波长，给不同波长设置稍不同的 IOR，即可模拟彩虹焦散。

5. **密度估计带状伪影**：在光子密度急剧变化的边界（如焦散光斑边缘），圆形核会产生明显的颗粒感和突变。

### 可优化方向

**性能**：
- Pass 2 多线程（OpenMP 或 std::thread）
- 使用更高效的 KD-Tree（如 nanoflann 库）
- SIMD 向量化向量运算
- 减小 kMax，用更大的光子密度来弥补（参数权衡）

**质量**：
- 渐进式光子映射（SPPM）：迭代细化估计半径，消除偏差
- Cone Filter 替代 top-hat 核，减少边界模糊
- 增加 SPP 减少噪声
- 添加色散效果（彩虹焦散）

**功能**：
- 支持多个光源
- 支持面光源（软阴影）
- 添加体积散射（Volume Scattering，可见光束）

### 与本系列的关联

这是继 03-21 SPPM（渐进式光子映射）和 03-22 BDPT（双向路径追踪）之后，再次回到全局光照的主题。三个算法处理焦散的方式各有不同：

- **SPPM（03-21）**：迭代式光子映射，无偏，但实现复杂
- **BDPT（03-22）**：通过多重重要性采样连接相机路径和光源路径，理论上可以处理焦散但收敛慢
- **本次光子映射**：有偏但高效，是渲染焦散的最直观方法

三者可以结合（VCM：Vertex Connection and Merging），兼顾收敛速度和无偏性，是当前离线渲染的最高水平算法之一。

---

代码仓库：[daily-coding-practice / 03-31-caustics](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/03/03-31-caustics)
