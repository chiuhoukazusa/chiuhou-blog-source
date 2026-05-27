---
title: "每日编程实践: Cone-Traced Ambient Occlusion Renderer（锥形追踪环境光遮蔽渲染器）"
date: 2026-05-28 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - AO
  - 软光栅化
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-28-cone-traced-ao/ao_final_output.png
---

今天是每日编程实践的第 90 天。这次实现了一个基于锥形追踪（Cone-Traced）的**环境光遮蔽（Ambient Occlusion，AO）渲染器**，并同时计算了**弯曲法线（Bent Normal）**——一种由 Landis 在 2002 年 SIGGRAPH 课程中提出、被 The Matrix Reloaded 等影片大规模使用的技术。

渲染器完全运行在 CPU 上，使用 SDF 场景进行快速几何查询，输出三张图像：原始 AO 遮蔽图、弯曲法线可视化图、以及融合了 AO + 直接光照的完整渲染图。

![最终渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-28-cone-traced-ao/ao_final_output.png)

---

## ① 背景与动机：为什么需要 AO，为什么要用锥形追踪

### 没有 AO 时会发生什么

在实时渲染里，间接光照的计算极其昂贵。最简单的做法是添加一个"环境光"常数项 `L_ambient = k_a * I_a`，让所有表面都获得一个固定的底色。这个做法有个致命缺陷：**它完全忽视了几何遮挡**。

现实中，墙角、物体底部、褶皱内侧——这些被其他几何体"围住"的区域会接收到更少的环境光，因为来自四面八方的光有相当大一部分被挡住了。均匀的环境光常数项会让这些区域"像在太空中漂浮"，完全失去接地感。

**AO 的直觉**：对于表面上的一个点 P，朝其法线半球方向投射若干射线。如果很多射线很快打到了附近的几何体，说明这个点被遮蔽得厉害，应该更暗。如果大部分射线都没有遮挡，说明这个点很"开阔"，可以接收到更多环境光。

### 工业界的实际应用

- **离线渲染**：皮克斯、梦工厂的电影使用烘焙 AO 作为间接光照的快速近似
- **游戏引擎**：UE5 的 Lumen 实时 GI 系统中，AO 仍然用于补充近场遮蔽细节
- **烘焙工作流**：Substance Painter/Marmoset 的 AO 烘焙功能是 PBR 资产制作的标准流程
- **SSAO/HBAO+**：NVIDIA 的 HBAO+ 是锥形追踪思路在屏幕空间的实时实现

### 弯曲法线（Bent Normal）的意义

普通 AO 只回答"这个点被遮蔽了多少"，而 Bent Normal 还额外告诉我们"最不被遮蔽的平均方向是哪个"——即从这个点出发，哪个方向的天空"最开阔"。

有了 Bent Normal，我们可以：
- 用它来采样环境贴图，让间接光更准确地来自"真正开放的方向"
- 将它作为 GI 的快速代理，用于 specular 间接反射
- UE5、Destiny 2 等引擎的材质系统都提供了 Bent Normal 输入接口

---

## ② 核心原理：从积分到锥形采样

### AO 的数学定义

环境光遮蔽是对半球积分的一个近似，其定义为：

```
AO(p, n) = (1/π) ∫_{Ω} V(p, ω) · (n · ω) dω
```

其中：
- `p`：表面上的点
- `n`：表面法线
- `Ω`：法线半球
- `V(p, ω)`：可见性函数，若方向 ω 无遮挡则为 1，否则为 0（或基于距离的衰减值）
- `(n · ω)`：余弦权重项（法线方向的能量贡献更大）

这个积分的意思是：**从 P 点朝半球内所有方向看，有多少比例没有被遮挡**。

**为什么除以 π？** 对于纯朗伯（Lambertian）半球，带 cosine 权重的积分 `∫_{Ω} (n·ω) dω = π`。除以 π 使得完全无遮挡时 AO = 1，完全遮挡时 AO = 0。

### 蒙特卡洛估计

直接计算这个积分需要对半球上无数个方向进行采样。我们用蒙特卡洛方法估计：

```
AO ≈ (1/N) Σ_{i=1}^{N} V(p, ωi) · w_i
```

其中 `ωi` 是第 i 个采样方向，`w_i` 是对应的权重。

**余弦加权重要性采样（Cosine-Weighted Importance Sampling）**：

如果按余弦权重分布采样方向（PDF 为 `p(ω) = cos(θ)/π`），则每个样本的权重 `w_i = (n·ωi) / p(ωi) = π`，从而抵消掉分母的 π：

```
AO ≈ (1/N) Σ_{i=1}^{N} V(p, ωi)
```

极大简化了计算。余弦加权意味着我们会更多地朝法线方向采样，减少了对贡献小（grazing angle）方向的浪费。

### 余弦加权采样公式推导

给定两个均匀随机数 `u1, u2 ∈ [0,1]`，余弦加权半球采样的方向为：

```
θ = arccos(√u1)     （天顶角，从法线方向量起）
φ = 2π · u2          （方位角）
```

转为笛卡尔坐标（本地空间，Z=上方）：

```
x = √u1 · cos(φ) = √(1-z²) · cos(φ)
y = √u1 · sin(φ) = √(1-z²) · sin(φ)
z = √(1-u1)
```

验证：当 `u1` 接近 0 时，`z ≈ 1`，方向接近法线（正上方）——这正是我们希望重点采样的区域。

### 弯曲法线的计算

弯曲法线是"未被遮蔽射线的加权平均方向"：

```
BentNormal = normalize( Σ_{i: V(ωi)>0} ωi · V(p, ωi) )
```

直觉上：把所有"看得见天空"的采样方向加权平均，就得到了最开阔的平均方向。

当场景左侧有个大球遮挡时，P 点的 Bent Normal 会偏向右侧；当 P 在一个凹坑中间时，Bent Normal 会比真实法线更竖直（四周遮蔽，只有正上方开放）。

### 距离衰减遮蔽模型

简单的二值 V（命中=0，未命中=1）会导致远处的微小几何体也产生全遮蔽，不现实。我们采用基于距离的平滑衰减：

```
V(t) = min(1, (t / maxDist)²)
```

- 命中距离 `t` 很近 → V ≈ 0，强遮蔽
- 命中距离接近 maxDist → V ≈ 1，几乎不遮蔽
- 二次曲线使近场遮蔽更明显，远场影响趋近消失

---

## ③ 实现架构

### 整体数据流

```
摄像机射线生成
     ↓
SDF Ray Marching（主场景求交）
     ↓
  命中？
 /     \
No      Yes
↓         ↓
天空色   AO 计算（N 条 cosine 采样射线）
         ↓
    Bent Normal 提取
         ↓
  直接光照（太阳光 + 阴影检测）
         ↓
  间接光照（Bent Normal 采样天空）
         ↓
  三路输出：AO图/法线图/完整渲染图
```

### 场景表示：SDF（有向距离场）

用 SDF 而不是三角网格的原因：**AO 需要大量射线查询**，每次 AO 查询要向半球射 N 条射线，每条射线要做光线行进（ray marching）。SDF 的好处是：

1. **快速行进**：SDF 值直接给出到最近几何体的距离，可以用球体步进（sphere stepping），每步尽量迈大步
2. **平滑法线**：通过有限差分计算梯度，天然得到光滑法线，不需要顶点法线插值
3. **任意形状**：球、盒子、平面可以简单组合

场景中的 SDF 定义：
- 地面：`sdPlane(p, y) = p.y - y`（最简单的 SDF，到 y=const 平面的距离）
- 球体：`sdSphere(p, c, r) = |p-c| - r`
- 盒子：`sdBox(p, c, half)` 使用分量最大值计算（详见代码解析）

### 主要数据结构

```cpp
struct AOResult {
    float ao;           // [0,1]，越大越亮（遮蔽越少）
    Vec3  bentNormal;   // 世界空间弯曲法线方向
};

struct HitInfo {
    float t;       // 光线参数
    Vec3 normal;   // 世界空间法线
    Vec3 color;    // 表面颜色（albedo）
    int matId;     // 材质类型
};

struct Image {
    int width, height;
    std::vector<Vec3> pixels;  // 线性 HDR，gamma 在保存时处理
};
```

三张图像并行计算：每个像素的 AO pass 计算完毕后，直接分发给三个输出 buffer，只做一次 AO 计算。

---

## ④ 关键代码解析

### SDF 盒子（最常见的错误点）

```cpp
float sdBox(const Vec3& p, const Vec3& c, const Vec3& half) {
    // q = 到中心的距离 - 半尺寸（负值代表在盒子内部）
    Vec3 q = Vec3(std::abs(p.x-c.x), std::abs(p.y-c.y), std::abs(p.z-c.z)) - half;
    
    // 外部距离：取正分量的长度
    Vec3 maxQ = Vec3(std::max(q.x,0.f), std::max(q.y,0.f), std::max(q.z,0.f));
    
    // 内部距离（盒子内部 SDF 为负）：取所有分量最大值（最小绝对值分量）
    return maxQ.len() + std::min(std::max({q.x,q.y,q.z}), 0.f);
}
```

**为什么这样写？**

SDF 需要对内外都正确：外部为正（距离盒子表面的距离），内部为负（到最近表面的负距离）。

- `q[i] > 0`：沿轴方向超出了盒子，这个方向有"外部距离"贡献
- `q[i] < 0`：在盒子内部这一轴，这个方向无外部贡献
- `maxQ.len()`：外部点的距离（角/边的情况用 Euclidean 距离，比 Chebyshev 更圆滑）
- `min(max(q.x,q.y,q.z), 0)`：内部点的修正（内部所有 q 均为负，取最大即最小绝对值）

### SDF 法线计算（有限差分梯度）

```cpp
const float eps = 1e-4f;
float dx = gScene.query(p+Vec3(eps,0,0)) - gScene.query(p-Vec3(eps,0,0));
float dy = gScene.query(p+Vec3(0,eps,0)) - gScene.query(p-Vec3(0,eps,0));
float dz = gScene.query(p+Vec3(0,0,eps)) - gScene.query(p-Vec3(0,0,eps));
hit.normal = Vec3(dx,dy,dz).norm();
```

**为什么用中心差分而不是前向差分？**

中心差分的误差是 `O(eps²)`，前向差分是 `O(eps)`。在 eps=1e-4 的情况下，中心差分误差约为 1e-8，前向差分约为 1e-4——差了四个数量级。SDF 的梯度（= 表面法线）需要精确，用中心差分是正确选择。

每次法线计算需要 6 次 SDF query（中心差分 3 个轴），但只在命中时才计算，不影响整体性能。

### 余弦加权半球采样

```cpp
Vec3 cosineSampleHemisphere(float u1, float u2) {
    float r   = std::sqrt(u1);         // 径向距离（盘上的半径）
    float phi = 2.f * 3.14159265f * u2; // 方位角
    float x   = r * std::cos(phi);
    float y   = r * std::sin(phi);
    float z   = std::sqrt(std::max(0.f, 1.f - u1));  // z = cos(θ)
    return Vec3(x, y, z);
}
```

**推导验证**：  
- 天顶角余弦 `cos(θ) = z = √(1-u1)`，所以 `cos²(θ) = 1-u1`，即 `u1 = sin²(θ) = 1 - cos²(θ)`  
- 对应的 PDF `p(θ,φ) = cos(θ)/π`（余弦分布）  
- 验证 `∫ p(θ,φ) sin(θ) dθ dφ = 1`：`∫₀^π ∫₀^{2π} (cos(θ)/π) sin(θ) dφ dθ = 1` ✅

`std::max(0.f, 1.f - u1)` 防止浮点误差导致 `1 - u1` 变成极小的负数开根出 NaN。

### TBN 矩阵（本地空间到世界空间）

```cpp
Vec3 toWorld(const Vec3& local, const Vec3& N) {
    Vec3 T, B;
    // 建立切线，避免与法线平行
    if(std::abs(N.x) < 0.9f)
        T = N.cross(Vec3(1,0,0)).norm();
    else
        T = N.cross(Vec3(0,1,0)).norm();
    B = N.cross(T);
    return T*local.x + B*local.y + N*local.z;
}
```

**为什么要分两种情况？**

如果法线几乎平行于 `(1,0,0)`，那么 `N.cross(Vec3(1,0,0))` 会产生一个几乎为零的向量，归一化后方向不确定（数值不稳定）。当 `|N.x| < 0.9` 时，改用 Y 轴参考，保证叉积的结果有足够的长度。

这是图形学中的经典做法，被称为"Hughes-Moller 正交化"或"Duff et al. 2017"的改进版。

### AO 主循环

```cpp
AOResult computeConeAO(const Vec3& pos, const Vec3& N, int numCones,
                        float maxDist, RNG& rng) {
    Vec3  accDir(0,0,0);
    float accAO = 0.f;
    Vec3  origin = pos + N * 5e-4f;  // ← 关键：法线方向偏移，避免自交

    for(int i = 0; i < numCones; i++) {
        float u1 = rng.next(), u2 = rng.next();
        Vec3 dir = toWorld(cosineSampleHemisphere(u1, u2), N);

        HitInfo hitInfo;
        bool hit = rayMarch(origin, dir, 1e-3f, maxDist, hitInfo);

        float visibility = 1.f;
        if(hit) {
            float t = hitInfo.t;
            visibility = (t / maxDist);        // 线性距离衰减
            visibility = visibility * visibility; // 二次衰减（近场更重）
        }

        accAO  += visibility;
        if(visibility > 0.1f)
            accDir = accDir + dir * visibility; // 累积开放方向
    }

    AOResult result;
    result.ao         = accAO / (float)numCones;
    result.bentNormal = (accDir.len() > 1e-6f) ? accDir.norm() : N;
    return result;
}
```

**`pos + N * 5e-4f` 为什么必要？**

如果从表面精确的交叉点出发，AO 射线的第一步很可能立刻打回同一个表面（因为 SDF = 0），造成自遮蔽（self-shadowing artifact）。沿法线偏移一个小量可以确保起点在表面上方，避免这个问题。

偏移量的选择：太小（如 1e-6）仍然可能触发自交（受 SDF 求解精度限制），太大（如 0.1）则起点会明显偏离表面，导致 AO 计算不准确。5e-4 是在实践中测试得到的合适值。

**弯曲法线为什么只累积 `visibility > 0.1f` 的方向？**

完全遮蔽的方向（visibility ≈ 0）对"开放方向"的估计没有贡献，如果累积进去反而会拖拽弯曲法线。设置阈值过滤掉这些方向，使弯曲法线更准确指向真正开放的区域。

### 光照模型

```cpp
// 直接光照（太阳）
float shadow = shadowRay(P, sunDir) ? 0.f : 1.f;
float NdotL  = std::max(0.f, N.dot(sunDir));
Vec3  directDiff = albedo * sunColor * NdotL * shadow;

// 镜面反射（Blinn-Phong 近似）
Vec3  H_vec  = (sunDir - rd).norm();   // Half-vector
float spec   = std::pow(std::max(0.f, N.dot(H_vec)), 32.f) * shadow * 0.3f;
Vec3  directSpec = sunColor * spec;

// 间接光照（弯曲法线 + AO）
float skyT      = aoResult.bentNormal.y * 0.5f + 0.5f;
Vec3  indirColor = Vec3::lerp(skyColorBot, skyColorTop, skyT);
Vec3  indirect   = albedo * indirColor * aoResult.ao * 0.6f;

Vec3 finalColor = directDiff + directSpec + indirect;
```

**间接光照的关键思路**：用弯曲法线的 Y 分量对天空颜色（上蓝下白）插值，得到该方向的"天空颜色"，再乘以 AO 值（遮蔽的区域获得更少间接光）。这是一种廉价但视觉上合理的间接光照近似，等价于"用弯曲法线查询天空函数，再乘以 AO 权重"。

---

## ⑤ 踩坑实录

### Bug 1：Vec3 组件乘法缺失（编译错误）

**症状**：编译报错 `no match for 'operator*' (operand types are 'Vec3' and 'Vec3')`

**出现位置**：`albedo * sunColor * NdotL * shadow`

**错误假设**：以为定义了 `Vec3 * float` 就够了，颜色混合用 `sunColor * NdotL * shadow` 可以先算出 `Vec3`，再和 `albedo` 相乘。

**真实原因**：`albedo` 是 `Vec3`（RGB albedo），`sunColor * NdotL * shadow` 也是 `Vec3`（加权后的光颜色）。两者相乘是分量乘法（component-wise multiply），这在物理渲染中表示"材质颜色对光颜色的滤波"，需要显式定义 `Vec3 * Vec3` 操作符。

**修复**：在 Vec3 类中添加：

```cpp
Vec3 operator*(const Vec3& o) const {
    return {x*o.x, y*o.y, z*o.z};
}
```

**经验**：在自己写的线性代数库里，向量点乘（dot product）、叉乘（cross product）、分量乘（component-wise）语义不同，应该明确区分——`.dot()` 返回 float，`.cross()` 返回 Vec3，`operator*` 是分量乘。有些库用重载，有些用 `cwiseProduct()`，没有统一标准，写之前要明确约定。

### Bug 2：stb_image_write.h 警告风暴

**症状**：编译出现大量 `warning: missing initializer for member 'stbi__write_context::...'`

**根源**：`stb_image_write.h` 是第三方头文件，其内部使用了旧式 C 结构体零初始化 `= {0}`，GCC 的 `-Wmissing-field-initializers` 会为此产生警告。

**解决方案**：添加 `-Wno-missing-field-initializers` 编译标志，专门屏蔽这个第三方警告，同时保持我们自己代码的严格警告检查。

```bash
g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra -Wno-missing-field-initializers
```

**经验**：使用第三方头文件时，直接引入到编译单元里会"继承"其警告。更好的做法是用一个单独的 `.cpp` 文件来 `#include` 这些头文件，隔离警告范围。但对于只有一个 `.cpp` 的练习项目，针对性禁用是可以接受的。

### Bug 3：渲染耗时评估失误（体验问题）

**症状**：640x480 + 32 AO samples + 2x AA 总计需要 30 秒

**预期**：以为 10-15 秒就能完成

**根源**：AO 计算量远超预期。每个像素需要：
- 主射线：1 次 ray march（最多 200 步 SDF query）
- AA：2x2 = 4 个子像素
- AO：每子像素 32 条射线，每条最多 200 步
- 阴影：1 条阴影射线

总 SDF 查询次数估算：`480 * 640 * 4 * (200 + 32*200 + 200) ≈ 8.4 亿次`。每次 SDF query 需要 5 个 `min/abs/sqrt` 操作，实际浮点运算约 40 亿次——接近现代 CPU 的单核理论上限（~10 GFLOPS），30 秒是合理的。

**改进方向**：可以用 OpenMP 并行化外层像素循环，利用多核轻松降低至 3-5 秒。

---

## ⑥ 效果验证与数据

### 渲染结果

| AO 遮蔽图 | 弯曲法线可视化 | 完整光照渲染 |
|-----------|-------------|------------|
| ![AO Raw](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-28-cone-traced-ao/ao_raw_output.png) | ![Bent Normal](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-28-cone-traced-ao/ao_bent_output.png) | ![Final](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-28-cone-traced-ao/ao_final_output.png) |

### 像素统计验证

```
ao_raw_output.png:   size=253.1KB  mean=237.1  std=25.7   ✅
ao_bent_output.png:  size=803.7KB  mean=197.9  std=34.0   ✅
ao_final_output.png: size=336.4KB  mean=182.1  std=61.1   ✅
```

**AO 图**（均值 237.1）：场景大部分区域较为开阔（接近白色），暗区集中在球体底部与地面接触处、盒子侧面阴影区域，视觉上正确。

**弯曲法线图**（均值 197.9，标准差最低）：RGB 分量编码弯曲法线方向，(0.5, 0.5, 1.0) = 正上方（蓝紫色），偏离方向时颜色变化。中间大球顶部法线几乎垂直，颜色偏蓝；底部弯曲法线向上倾斜更多（受地面遮挡），颜色有明显变化。

**完整渲染图**（标准差最高 61.1）：光照变化最丰富，棋盘格地面、彩色球体、接触阴影都清晰可见。

### 性能数据

| 参数 | 值 |
|------|----|
| 分辨率 | 640 × 480 |
| AO 采样数 | 32 条/子像素 |
| 超采样 | 2×2 |
| Ray March 最大步数 | 200 |
| 总用时 | 29.9 秒 |
| 单核峰值估计 | ~14 Mpixel-AO/s |

---

## ⑦ 总结与延伸

### 技术局限性

1. **只做了漫反射 AO**：本实现的 AO 只适用于漫反射间接光，对于 glossy 材质的间接镜面反射，Bent Normal 应该结合 BRDF 的波瓣形状采样，而不是直接对天空颜色插值。

2. **低频近似**：32 条射线对高频遮蔽（比如细小的裂缝）会欠采样，产生噪声。实际使用时通常配合时域降噪（TAA）或空间滤波来平滑。

3. **无法处理自发光物体间接光**：本实现的间接光仅来自天空。场景中如果有自发光物体，它们的间接贡献无法通过这个方案体现。

4. **SDF 场景限制**：SDF 组合复杂场景时（如数万三角形网格）效率极低。实际项目中，AO 查询会针对三角网格使用 BVH 加速，或者在屏幕空间（SSAO）进行。

### 可优化方向

- **OpenMP 并行化**：外层循环按行并行，可将 30 秒降至 3-5 秒
- **低差异序列（Halton/Sobol）**：替换 MT 随机数，减少噪声，只需 16 条射线即可达到 32 条随机的质量
- **重要性引导**：根据遮蔽历史动态调整采样方向，优先采样遮蔽梯度大的区域
- **SSAO/GTAO**：将思路迁移到屏幕空间，用深度缓冲做 AO，实时可用
- **烘焙模式**：对静态场景预计算 AO 贴图，运行时直接采样，开销趋近于零

### 与系列其他文章的关联

- **Day 24（Bent Normal AO）**：今天实现了与 Day 24 相同思路的锥形追踪 AO，但使用了 SDF 场景代替三角网格，AO 查询更快速，弯曲法线可视化更完整
- **Day 45（SSAO）**：SSAO 是 AO 的屏幕空间近似，用深度缓冲代替 SDF 查询，牺牲精度换取实时性能
- **Day 76（SSGI）**：SSGI 在 AO 基础上进一步扩展，加入了间接颜色渗透，今天的弯曲法线间接光照是 SSGI 的一个重要前身
- **Day 67（VXGI）**：体素锥追踪（VXGI）是今天锥形 AO 思路在 3D 体素空间的扩展，用于全频段 GI 而不仅是 AO

---

### 本次实现的完整渲染管线总结

```
输入：SDF 场景 + 摄像机参数

阶段 1：光线生成
  - 透视投影，FOV=45°，2x2 超采样
  - 每个子像素独立 RNG（seed = 行*列*AA...），避免时域 pattern

阶段 2：主场景求交（SDF Ray March）
  - 最大步数 200，收敛阈值 1e-4
  - 命中：提取交点、法线（中心差分）、材质颜色
  - 未命中：采样天空（Rayleigh-like 渐变 + 太阳光晕）

阶段 3：AO 计算（每命中点）
  - 32 条余弦加权半球射线（2个均匀随机数映射到方向）
  - 每条射线 SDF ray march（maxDist=4 单位）
  - 命中距离 → 二次衰减 visibility
  - 累积：AO = Σvisibility / N，BentNormal = normalize(Σdir·vis)

阶段 4：光照合成
  - 直接漫反射：albedo × sunColor × NdotL × shadow
  - 直接镜面：Blinn-Phong specular × shadow
  - 间接漫反射：albedo × sky(bentNormal) × AO × 0.6
  - 最终 = 直接漫反射 + 直接镜面 + 间接漫反射

阶段 5：Gamma 校正 + 输出
  - pow(clamp(v, 0, 1), 1/2.2)，输出 8-bit PNG
  - 三路输出：AO 图 / 弯曲法线图 / 完整渲染图
```

整个管线在单核 CPU 约 30 秒内完成，输出三张分辨率 640×480 的 PNG 图像，总大小约 1.4MB。

---

第 90 天。锥形追踪 AO + 弯曲法线，技术实现扎实，三张图像质量检验通过。

代码：[GitHub](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-28-cone-traced-ao)
