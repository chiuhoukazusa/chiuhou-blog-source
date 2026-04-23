---
title: "每日编程实践: Bent Normal Ambient Occlusion — 弯曲法线环境遮蔽"
date: 2026-04-24 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - AO
  - 环境光照
  - 球谐函数
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-24-bent-normal-ao/bent_normal_ao_output.png
---

# Bent Normal Ambient Occlusion — 弯曲法线环境遮蔽

## ① 背景与动机：普通 AO 的盲区

### 问题的根源

在实时渲染中，**Ambient Occlusion（环境遮蔽）** 是模拟间接光照最廉价的近似手段——通过计算一个表面点被周围几何体遮挡的程度，得到一个 0~1 的标量，直接乘到环境光上。

但这有一个根本性的缺陷：

> 普通 AO 只告诉你"这个点被遮蔽了多少"，却不告诉你"哪个方向还有光进来"。

设想一个角落里的点，它的北面和东面都被遮住，但西面和天空方向完全开放。普通 AO 算法会给它分配一个比较低的 AO 值（比如 0.4），然后将整个环境光均匀地乘以 0.4——包括西面那些本应亮起来的光。结果是整个点都偏暗，而且色调偏向错误的方向。

**在游戏引擎里，这个问题尤为明显：**

- **角色放在墙角**：脸部该被天空光打亮，却被 AO 压暗了
- **洞穴内壁**：内壁朝向洞口方向的部分该有颜色渐变，普通 AO 会让整个内壁均匀变暗
- **布料折叠处**：折叠褶皱内侧和外侧的受光方向完全不同，普通 AO 无法区分

### Bent Normal 的解法

**Bent Normal（弯曲法线）** 是解决这个问题的关键。它不是几何法线，而是一个"统计方向"：

> 在法线半球上采样若干方向，把所有**没有被遮蔽的方向**的加权均值，就是 Bent Normal。

它指向的是"光线最容易从这个方向进来"的方向。有了 Bent Normal，我们可以：

1. **用 Bent Normal 采样环境光**（SH、Cubemap、Probe），代替原始几何法线——这样遮蔽了的方向不会被过多采样，开放方向会被正确加权
2. **用采样率（未遮蔽射线数/总射线数）作为 AO 值**——这比传统基于距离的 AO 估计更准确

### 工业界应用

Bent Normal AO 在以下场景被广泛使用：

- **Unreal Engine 5**：Lumen 全局光照的辅助 Pass 之一，用于快速近似间接光照方向
- **The Order: 1886**（Ready At Dawn）：预计算 Bent Normal 贴图，结合球谐光探针实现精确的间接光照
- **Naughty Dog（《最后生还者》系列）**：实时 Bent Normal + 光探针系统是其角色光照管线的核心
- **光照贴图烘焙工具**：Substance Designer、Marmoset Toolbag、xNormal 都支持烘焙 Bent Normal 贴图，用于离线材质制作

---

## ② 核心原理：从 AO 到 Bent Normal

### 2.1 传统 AO 的数学定义

对于表面点 $\mathbf{p}$，法线为 $\hat{\mathbf{n}}$，AO 值定义为：

$$AO(\mathbf{p}) = \frac{1}{\pi} \int_{\Omega^+} V(\mathbf{p}, \boldsymbol{\omega}) \cos\theta \, d\boldsymbol{\omega}$$

其中：
- $\Omega^+$ 是法线半球
- $V(\mathbf{p}, \boldsymbol{\omega})$ 是可见性函数：沿方向 $\boldsymbol{\omega}$ 在半径 $r$ 内未被遮挡则为 1，否则为 0
- $\cos\theta = \boldsymbol{\omega} \cdot \hat{\mathbf{n}}$ 是余弦权重

**直觉解释**：这个积分计算了半球上所有方向的"加权开放程度"——余弦项越大（方向越接近法线），它的权重越高。结果是 0（全被遮蔽）到 1（完全开放）之间的值。

用蒙特卡洛估计（余弦加权采样）：

$$AO \approx \frac{1}{N} \sum_{i=1}^{N} V(\mathbf{p}, \boldsymbol{\omega}_i)$$

**注意**：余弦加权采样 PDF 为 $p(\boldsymbol{\omega}) = \cos\theta / \pi$，代入蒙特卡洛估计器后，$\cos\theta$ 和 PDF 恰好约掉，分子分母的 $\pi$ 也约掉，结果就是上面这个简洁形式——只需统计未遮蔽比例即可。

### 2.2 Bent Normal 的定义

Bent Normal 不是单一的标量，而是一个方向向量：

$$\hat{\mathbf{n}}_\text{bent}(\mathbf{p}) = \text{normalize}\!\left( \int_{\Omega^+} V(\mathbf{p}, \boldsymbol{\omega}) \cdot \boldsymbol{\omega} \, d\boldsymbol{\omega} \right)$$

**直觉解释**：对每个未遮蔽方向 $\boldsymbol{\omega}$，把它当作一个向量累加起来，最后归一化。结果是所有"能看到天空"的方向的平均方向。

在有几何遮挡的地方，这个方向会偏离几何法线，朝向更多开放空间的方向。

### 2.3 Bent Normal 的用途：改进环境光采样

标准的 Lambertian 间接光照：

$$L_\text{indirect}(\mathbf{p}) = \int_{\Omega^+} L_\text{env}(\boldsymbol{\omega}) \cos\theta \, d\boldsymbol{\omega}$$

用球谐函数（Spherical Harmonics, SH）近似环境光时，通常用几何法线 $\hat{\mathbf{n}}$ 查询：

$$L_\text{indirect} \approx \text{SH\_eval}(\hat{\mathbf{n}})$$

但在有遮蔽的地方，这样采样会包含从被遮挡方向来的光。改用 Bent Normal：

$$L_\text{indirect} \approx \text{SH\_eval}(\hat{\mathbf{n}}_\text{bent}) \cdot AO$$

**两层修正**：
- `SH_eval(bent_normal)` 的方向改进：朝向开放方向采样，不再从遮挡方向取光
- `* AO` 的量级修正：乘以遮蔽比例，防止完全遮蔽的角落仍然被环境光照亮

### 2.4 球谐函数环境光（L2 SH）

L2 球谐函数用 9 个系数表示环境辐照度（漫反射环境光）：

$$E(\hat{\mathbf{n}}) = \sum_{l=0}^{2} \sum_{m=-l}^{l} L_{lm} \cdot Y_{lm}(\hat{\mathbf{n}})$$

其中 $Y_{lm}$ 是实数球谐基函数。对于 RGB 颜色，每个系数 $L_{lm}$ 是一个 3 分量向量。

L2 SH（9 个系数）能捕获低频的环境光变化，对漫反射材质的近似误差极小（频率比漫反射 BRDF 本身还低），是游戏引擎中常用的表示方式。

**展开后的计算公式**（Ramamoorthi & Hanrahan 2001 的推导结果）：

```
E(n) = c0 * L00
     + c1*ny * L1-1 + c1*nz * L10 + c1*nx * L11
     + c2*nx*ny * L2-2
     + c2*ny*nz * L2-1
     + c3*(3nz^2-1) * L20
     + c2*nx*nz * L21
     + c4*(nx^2-ny^2) * L22
```

其中 $c_0 = 0.282095$，$c_1 = 0.488603$，$c_2 = 1.092548$，$c_3 = 0.315392$，$c_4 = 0.546274$。

### 2.5 低差异序列：Hammersley Point Set

为了得到质量更好的半球采样，使用 Hammersley 低差异序列代替伪随机数：

$$u_i = \frac{i}{N}, \quad v_i = \Phi_2(i)$$

其中 $\Phi_2(i)$ 是以 2 为基的 Van der Corput 序列（通过位逆转计算）。

**为什么不用随机数？** 随机数的采样点可能聚集，产生方差。低差异序列保证采样点在 $[0,1]^2$ 上均匀分布，减少方差，相当于 32 个样本比随机采样效果更好。

**余弦加权半球采样**（从 $(u,v) \in [0,1]^2$ 映射到半球方向）：

$$\theta = \arccos(\sqrt{1-v}), \quad \phi = 2\pi u$$

$$\boldsymbol{\omega} = (\sin\theta\cos\phi,\ \cos\theta,\ \sin\theta\sin\phi)$$

然后通过切线空间变换将这个方向转到法线 $\hat{\mathbf{n}}$ 所在坐标系。

---

## ③ 实现架构：单 Pass 软光栅化渲染器

### 数据流

```
场景定义（SDF）
       ↓
主射线 Ray March（找交点）
       ↓
G-Buffer：position, normal, albedo, object_id
       ↓
Bent Normal AO Pass（对每个像素做半球采样）
    ├── 32条采样射线 × Hammersley序列 → 余弦加权半球方向
    ├── 每条射线: AO Ray March（短距离，<1.2m）
    ├── 统计未遮蔽射线 → AO值
    └── 累加未遮蔽方向 → Bent Normal
       ↓
光照着色
    ├── 直接光照: NdotL * soft_shadow
    ├── 间接光照: albedo * SH_eval(bent_normal) * AO
    └── final = direct + indirect
       ↓
Tone Mapping（Reinhard）+ Gamma（2.2）
       ↓
PPM → PNG（ImageMagick）
```

### 关键数据结构

**Vec3**：三维向量，支持加减乘除、点积、叉积、归一化。不使用 SIMD，保持代码简洁。

**Hit**：射线与场景的交点信息，包含 SDF 值和物体 ID。物体 ID 用于查找材质（albedo）。

**SH9**：9 个 Vec3 系数，对应 L0、L1x/y/z、L2 的五个项。每个系数是 RGB 颜色向量。

**BentNormalResult**：Bent Normal 方向 + AO 标量，是 AO Pass 的输出。

### 渲染参数

- 分辨率：640×480
- AO 采样数：32（使用 Hammersley 序列，相当于随机采样 64 次的质量）
- AO 半径：1.2（世界空间单位，超出此距离视为未遮蔽）
- SDF 步进迭代：主射线 256 步，AO 射线按自适应步长（最大 AO_RADIUS）

---

## ④ 关键代码解析

### 4.1 Bent Normal AO 核心计算

这是整个渲染器最核心的部分——`computeBentNormal` 函数：

```cpp
struct BentNormalResult {
    Vec3  bentNormal;   // 弯曲法线（未遮蔽方向的加权平均）
    float ao;           // 0=完全遮蔽, 1=完全未遮蔽
};

BentNormalResult computeBentNormal(Vec3 pos, Vec3 normal,
                                    int numSamples, float aoRadius)
{
    const float STEP_MIN = 0.005f;  // 最小步长，防止无限循环
    const float EPS = 0.002f;       // 起点偏移量，防止自交

    Vec3  bentDir{0,0,0};
    int   unoccluded = 0;

    for(int i = 0; i < numSamples; ++i){
        // Hammersley 序列：u 是均匀线性序列，v 是 van der Corput 序列
        float u = float(i) / float(numSamples);
        float v = vanDerCorput((unsigned)i + 1u);

        // 将 (u,v) 映射到余弦加权半球方向
        Vec3 dir = hemisphereDir(u, v, normal);

        // 从偏移点开始 ray march，避免采样到自身表面
        Vec3 rayPos = pos + normal * EPS;
        float t = STEP_MIN;
        bool occluded = false;

        while(t < aoRadius){
            Vec3 p = rayPos + dir * t;
            float d = sceneSDF(p).d;  // 到最近场景物体的距离
            if(d < 0.001f){           // 碰到几何体 → 遮蔽
                occluded = true;
                break;
            }
            t += std::max(d, STEP_MIN);  // 球形步进，d 保证安全步长
        }

        if(!occluded){
            bentDir += dir;  // 累加未遮蔽方向
            ++unoccluded;
        }
    }

    // AO = 未遮蔽比例
    float ao = float(unoccluded) / float(numSamples);
    // Bent Normal = 未遮蔽方向的均值方向（若全被遮蔽，退化为几何法线）
    Vec3 bent = (unoccluded > 0) ? bentDir.normalize() : normal;

    return {bent, ao};
}
```

**关键设计决策：**

1. **起点偏移 `EPS = 0.002f`**：没有这个偏移，射线会立刻"碰到"自身所在的表面（SDF 值约为 0），误判为遮蔽。偏移量要足够大以跳过表面，但不能太大以至于穿过薄墙。

2. **最小步长 `STEP_MIN = 0.005f`**：当 SDF 值接近 0 但未真正相交时，如果步长为 0，射线会原地踏步。最小步长保证循环推进。

3. **退化处理 `unoccluded > 0`**：如果 32 条射线全部被遮蔽（深陷角落），bentDir 是零向量，归一化会产生 NaN。此时退化为几何法线（方向无意义，但 AO=0 保证了间接光为 0，结果正确）。

### 4.2 余弦加权半球采样

```cpp
// 将单位正方形 (u,v) ∈ [0,1]² 映射到法线 N 的余弦加权半球
Vec3 hemisphereDir(float u, float v, const Vec3& N){
    // 余弦加权：theta ~ arccos(sqrt(1-v))，phi = 2π*u
    float phi      = 2.0f * M_PI * u;
    float cosTheta = std::sqrt(1.0f - v);  // 余弦加权的正弦θ分布
    float sinTheta = std::sqrt(v);          // sinθ = sqrt(v)

    // 在 "N 朝上" 的局部坐标系中
    float sx = sinTheta * std::cos(phi);
    float sy = cosTheta;                    // y = 上方向 = N 方向
    float sz = sinTheta * std::sin(phi);

    // 构造切线空间变换矩阵（TBN 的逆，即 T|B|N 的列向量）
    // 选择 up 向量时，如果 N 接近 y 轴，改用 x 轴避免退化
    Vec3 up        = std::abs(N.y) < 0.999f ? Vec3{0,1,0} : Vec3{1,0,0};
    Vec3 tangent   = up.cross(N).normalize();   // T = up × N
    Vec3 bitangent = N.cross(tangent).normalize(); // B = N × T

    // 变换：local (sx,sy,sz) → world
    return tangent*sx + N*sy + bitangent*sz;
}
```

**推导说明：**

余弦加权采样的 PDF 为 $p(\theta, \phi) = \cos\theta\sin\theta / \pi$。要从均匀分布 $(u,v)$ 得到这个分布，用逆 CDF 方法：

- $\phi = 2\pi u$（均匀方位角）
- $\cos\theta = \sqrt{1-v}$，即 $v = \sin^2\theta$

这样采样到的方向在接近法线方向（$\theta \approx 0$）的密度更高，与 Lambert 漫反射余弦项匹配，减少采样方差。

### 4.3 Van der Corput 低差异序列

```cpp
float vanDerCorput(unsigned bits){
    // 将整数的二进制位逆转，然后除以 2^32
    bits = (bits << 16u) | (bits >> 16u);
    bits = ((bits & 0x55555555u) << 1u) | ((bits & 0xAAAAAAAAu) >> 1u);
    bits = ((bits & 0x33333333u) << 2u) | ((bits & 0xCCCCCCCCu) >> 2u);
    bits = ((bits & 0x0F0F0F0Fu) << 4u) | ((bits & 0xF0F0F0F0u) >> 4u);
    bits = ((bits & 0x00FF00FFu) << 8u) | ((bits & 0xFF00FF00u) >> 8u);
    return float(bits) * 2.3283064365386963e-10f;  // / (2^32)
}
```

**原理**：将整数 $i$ 的二进制表示 $b_1 b_2 b_3 \ldots b_k$ 逆转为 $0.b_k \ldots b_3 b_2 b_1$（二进制小数）。这保证了序列在 $[0,1]$ 上的均匀分布，相邻点不会聚集。

例如：$i=1 \to 0.5$，$i=2 \to 0.25$，$i=3 \to 0.75$，$i=4 \to 0.125$……每次新的点都填补已有点之间最大的空隙。

5位示例：1→16, 2→8, 3→24, 4→4（二进制逆转后转十进制）

### 4.4 SH 环境光评估

```cpp
// L2 球谐辐照度评估
// 基于 Ramamoorthi & Hanrahan (2001) 的推导
Vec3 evalSH(const SH9& sh, Vec3 n){
    float x=n.x, y=n.y, z=n.z;

    // SH 基函数的归一化常数
    const float c0 = 0.282095f;   // Y(0,0) = 1/(2*sqrt(pi))
    const float c1 = 0.488603f;   // sqrt(3/(4*pi))
    const float c2 = 1.092548f;   // sqrt(15/(4*pi))
    const float c3 = 0.315392f;   // sqrt(5/(4*pi)) * 0.5
    const float c4 = 0.546274f;   // sqrt(15/(16*pi))

    Vec3 result = sh.c[0] * c0;              // L00
    result += sh.c[1] * (c1 * y);            // L1-1 (∝ y)
    result += sh.c[2] * (c1 * z);            // L10  (∝ z)
    result += sh.c[3] * (c1 * x);            // L11  (∝ x)
    result += sh.c[4] * (c2 * x*y);          // L2-2 (∝ xy)
    result += sh.c[5] * (c2 * y*z);          // L2-1 (∝ yz)
    result += sh.c[6] * (c3 * (3*z*z - 1));  // L20  (∝ 3z²-1)
    result += sh.c[7] * (c2 * x*z);          // L21  (∝ xz)
    result += sh.c[8] * (c4 * (x*x - y*y));  // L22  (∝ x²-y²)
    return result;
}
```

**直觉解释**：这 9 个基函数对应环境光的"频率分量"——L0 是全局平均亮度（各向同性），L1 是主方向分量（类似漫反射主光源方向），L2 是更复杂的方向依赖变化（来自两侧的光差异等）。用 Bent Normal 查询这个"频谱"，比用几何法线查询更准确，因为 Bent Normal 更准确地描述了该点实际能接收光的方向。

### 4.5 软阴影

```cpp
float softShadow(Vec3 ro, Vec3 rd, float tmin, float tmax, float k){
    float result = 1.0f;
    float t = tmin;
    for(int i = 0; i < 64; ++i){
        float h = sceneSDF(ro + rd*t).d;
        if(h < 0.001f) return 0.0f;  // 硬阴影（完全遮蔽）
        // Iq 的软阴影公式：遮蔽比例 ∝ SDF值/距离
        result = std::min(result, k * h / t);
        t += std::max(h, 0.005f);
        if(t > tmax) break;
    }
    return saturate(result);
}
```

**Inigo Quilez 的软阴影技巧**：当射线在遮挡体附近通过时，`h/t` 的比值反映了射线与几何体的"近距离程度"。`k` 控制软阴影的宽度（k 越小阴影越软）。这不是物理上正确的半影计算，但视觉上非常自然。

### 4.6 SDF 场景组合

```cpp
// 带十字凹槽的地面：基础平面 + 两个长方体的差集
float sdGroovedPlane(Vec3 p){
    float plane  = p.y + 0.5f;       // y = -0.5 平面
    float groove1 = sdBox(p, Vec3{0, -0.5f, 0}, Vec3{0.3f, 0.3f, 2.0f}); // X 方向槽
    float groove2 = sdBox(p, Vec3{0, -0.5f, 0}, Vec3{2.0f, 0.3f, 0.3f}); // Z 方向槽
    // 差集：max(A, -B) = A 减去 B
    return std::max(plane, -std::min(groove1, groove2));
}

// 场景合并：取最小值 = SDF 的并集
Hit sceneSDF(Vec3 p){
    float ground = sdGroovedPlane(p);
    float sphere = sdSphere(p, Vec3{0, 0.6f, 0}, 0.6f);
    float cyl    = sdCylinder(p, Vec3{-1.5f, -0.1f, 0}, 0.3f, 0.8f);
    float box    = sdBox(p, Vec3{1.5f, 0.05f, 0}, Vec3{0.4f, 0.55f, 0.4f});

    Hit h = {ground, 0};                        // 初始：地面
    if(sphere < h.d){ h.d = sphere; h.id = 1; } // 球体更近
    if(cyl    < h.d){ h.d = cyl;    h.id = 2; } // 圆柱更近
    if(box    < h.d){ h.d = box;    h.id = 3; } // 方块更近
    return h;
}
```

**SDF 布尔运算**：
- 并集：`min(A, B)`（返回更近的物体）
- 差集：`max(A, -B)`（A 减去 B 的区域）
- 交集：`max(A, B)`（只保留两者重叠的区域）

---

## ⑤ 踩坑实录

### Bug 1：AO 射线起点自交

**症状**：整个场景 AO 值接近 0（几乎全黑），每个点都显示被遮蔽。

**错误假设**：以为 SDF ray marching 从表面点出发，第一步就会远离表面。

**真实原因**：表面点位于 SDF = 0 的等值面上。如果从这个点直接出发，第一次 `sceneSDF(rayPos + dir * 0)` 返回的值约为 0，满足 `d < 0.001f`，立刻被判为遮蔽。

**修复方式**：在起点处加上法线方向的微小偏移 `pos + normal * 0.002f`，确保射线起点在表面稍微外侧。偏移量 0.002 是经过测试的值：太小（<0.001）会漏检自交，太大（>0.01）会导致薄壁物体附近的 AO 射线从内部穿出，产生错误的"未遮蔽"结果。

```cpp
// 错误：
Vec3 rayPos = pos;  // SDF=0，第一步就误判为遮蔽

// 正确：
Vec3 rayPos = pos + normal * EPS;  // EPS = 0.002f
```

### Bug 2：Bent Normal 归一化时 NaN

**症状**：深陷角落的像素显示纯黑，但不是预期的黑色（AO=0），而是 NaN 传播导致的完全混乱颜色。

**错误假设**：以为归一化零向量会返回零向量，不会有问题。

**真实原因**：当所有 32 条采样射线都被遮蔽（`unoccluded = 0`），`bentDir` 是零向量 `{0,0,0}`。调用 `bentDir.normalize()` 时，`length() = 0`，除以 0 产生 NaN。NaN 传入 `evalSH` 后扩散到所有颜色通道。

**修复方式**：
```cpp
// 错误：
Vec3 bent = bentDir.normalize();

// 正确：
Vec3 bent = (unoccluded > 0) ? bentDir.normalize() : normal;
// 全遮蔽时退化为几何法线，间接光 = SH_eval(n) * 0 = 0，结果正确
```

### Bug 3：AO 射线最小步长缺失导致死循环

**症状**：渲染某些像素时卡死（无限循环），程序不响应。

**真实原因**：当射线恰好平行于某个平面且距离极近（SDF 值约 0.0001）时，如果步长等于 SDF 值，每步只移动 0.0001，需要 12000 步才能走完 AO_RADIUS=1.2 的距离，相当于卡死。

**修复方式**：加入最小步长 `STEP_MIN = 0.005f`：
```cpp
t += std::max(d, STEP_MIN);
```
这样即使 SDF 值极小，每步也至少前进 0.005，最多 240 步即可走完 1.2 的距离。

### Bug 4：坐标系切线空间退化

**症状**：法线接近 `(0,1,0)` 的点（水平面）的 Bent Normal 出现异常，指向奇怪方向。

**真实原因**：构建切线空间时，用 `up = {0,1,0}` 与 `N` 叉积求切线向量 `T = up × N`。当 `N ≈ (0,1,0)` 时，`up × N ≈ (0,0,0)`，归一化后得到 NaN。

**修复方式**：
```cpp
// 当 N 接近 y 轴时，换用 x 轴作为 up 向量
Vec3 up = std::abs(N.y) < 0.999f ? Vec3{0,1,0} : Vec3{1,0,0};
```

### Bug 5：SH 系数评估返回负值

**症状**：自定义 SH 系数的某些方向返回负的辐照度值（比如 `SH_eval({0,-1,0})` 为负数），叠加到颜色后出现负颜色，gamma 校正后产生 NaN。

**真实原因**：手动配置的 SH 系数（蓝天 + 暖阳）在向下方向（地下方向）自然返回负值，这在物理上是"能量低于 L0 平均"的含义，但渲染时如果直接使用会引发问题。

**修复方式**：对 SH 评估结果分量做 `max(*, 0)` 截断：
```cpp
shIrr.x = std::max(shIrr.x, 0.f);
shIrr.y = std::max(shIrr.y, 0.f);
shIrr.z = std::max(shIrr.z, 0.f);
```

---

## ⑥ 效果验证与数据

### 量化验证结果

```
图像尺寸: (640, 480)
像素均值: 157.4  标准差: 30.9

Bent Normal 区域均值: 164.9  标准差: 15.0
AO 区域均值: 184.0  标准差: 5.7
  AO区域 R=184.0  G=184.0  B=184.0  (R=G=B 确认灰度正确)
完整渲染区域均值: 156.4

主场景区域均值: 153.7  标准差: 32.6
```

**分析：**

- **均值 157.4（范围 10~240）**：图像亮度适中，非全黑/全白
- **标准差 30.9（>5）**：图像有丰富内容变化
- **AO 区域 R=G=B=184**：确认灰度通道正确（非彩色杂质），AO 均值 184/255 ≈ 0.72，表示场景平均约 72% 开放——合理（有遮蔽但不是深洞）
- **Bent Normal 标准差 15.0**：Bent Normal 方向有明显空间变化，反映了场景几何的遮蔽方向差异

### 性能数据

| 指标 | 数值 |
|------|------|
| 分辨率 | 640×480 |
| 每像素 AO 采样数 | 32 |
| AO 半径 | 1.2（世界空间） |
| 主射线步进（最大）| 256 次/像素 |
| AO 射线步进（最大）| ~240 次/射线 |
| 总渲染时间 | 3.9 秒（单线程 CPU） |
| 输出文件大小 | 901 KB |

### 视觉分析

底部分区可视化展示了三种信息的对比：

**左侧 1/3（Bent Normal 可视化，RGB 映射方向）**：
- 地面水平区域：偏蓝色（法线偏向 Y+ 方向，Bent Normal 也朝上，RGB → 接近 {0.5, 1.0, 0.5}）
- 凹槽边缘：颜色变化，Bent Normal 倾向于开放空间的方向
- 球体侧面：明显的方向变化，从自遮蔽到开放的过渡

**中间 1/3（AO 灰度图）**：
- 凹槽内部：明显较暗（AO 低，遮蔽严重）
- 开阔地面：较亮（AO 高，基本无遮蔽）
- 球体底部与地面接触区：阴影接触暗化

**右侧 2/3（完整渲染）**：
- 地面棋盘格有轻微的颜色倾向（暖侧/冷侧来自 SH 环境光的方向分量）
- 凹槽内部较暗但有颜色——来自 Bent Normal 修正后的 SH 采样，不是纯黑

---

## ⑦ 总结与延伸

### 技术局限性

1. **软 Bent Normal 精度有限**：每像素 32 个采样的蒙特卡洛估计仍有噪声。在实际游戏引擎中通常预计算并烘焙到纹理（Bent Normal Map），使用时直接从贴图读取，无运行时开销。

2. **AO 与 SH 的解耦近似**：本实现中 `SH_eval(bent_normal) * AO` 是一个近似——真正的 Bent Normal 理论应该用 Bent Normal 定义一个"可见性滤波的 SH 投影"，即 $\int V(\omega) L(\omega) d\omega$，而不是简单地用均值方向查询。这个近似在遮蔽变化缓慢时效果好，在复杂几何（多孔结构、镂空）时可能偏差较大。

3. **AO 半径的调参敏感性**：AO_RADIUS=1.2 适合当前场景尺度。对于不同尺度的场景，需要重新调整（太小则遮蔽效果不明显，太大则场景大遮挡物被计入）。

4. **仅处理漫反射间接光**：Bent Normal 只对 Lambertian（漫反射）BRDF 有效，镜面反射需要不同的方法（如 IBL + BRDF LUT）。

### 优化方向

1. **预计算 Bent Normal Map**：将 Bent Normal 和 AO 烘焙到纹理，运行时直接采样，去掉 32 次射线 march，大幅提升性能。

2. **多重要性采样（MIS）**：结合 AO 射线的遮蔽率分布和 SH 的能量分布做重要性采样，进一步降低方差。

3. **时序重用（TAA for AO）**：参考 04-26 的 TAA 实现，对 Bent Normal / AO 做时序累积，将有效采样数随时间提升到 512+。

4. **球谐旋转**：当光探针移动或旋转时，用 SH 旋转矩阵高效地转换系数，无需重新投影。

5. **方向性遮蔽（Directional Occlusion）**：进一步扩展，不只用 Bent Normal 的方向和 AO 标量，而是为每个方向存储一个遮蔽"函数"（方向性 AO），获得更准确的间接光照（代价是存储 9 个 AO 值而非 1 个）。

### 与本系列的关联

- 球谐函数基础见：[04-19 Ambient Light Probes & Irradiance Caching](https://chiuhoukazusa.github.io/)（L2 SH 投影和评估的完整推导）
- SDF Ray Marching 基础见：[04-01 SDF Ray Marching Renderer](https://chiuhoukazusa.github.io/)（sphere tracing 的基本框架）
- SSAO 的对比参考见：[03-25 Screen Space Ambient Occlusion](https://chiuhoukazusa.github.io/)（屏幕空间 AO 与 ray march AO 的比较）

Bent Normal AO 是 SSAO（快速但不精确）和完整路径追踪（精确但昂贵）之间的重要折中——它比 SSAO 有更好的方向感知，又比全局光照路径追踪快几个数量级，是实时渲染中间接光照质量提升的关键一步。

---

*代码仓库: [daily-coding-practice/2026/04/04-24-bent-normal-ao](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-24-bent-normal-ao)*
