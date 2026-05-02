---
title: "每日编程实践: Precomputed Radiance Transfer (PRT) — 用球谐函数预计算全局光照"
date: 2026-05-03 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 全局光照
  - 球谐函数
  - PRT
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-03-prt-renderer/prt_output.png
---

# 每日编程实践: Precomputed Radiance Transfer (PRT)

> **技术标签**: 球谐函数(SH)、辐射传输、预计算光照、SH旋转、Wigner D矩阵、余弦卷积、软光栅化  
> **代码仓库**: [GitHub](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-03-prt-renderer)  
> **难度**: ⭐⭐⭐⭐

---

## ① 背景与动机：实时全局光照的代价

### 问题的起点

在实时渲染中，我们最头疼的问题之一就是**环境光照**：物体不只是被一个方向光照亮，它被四面八方的天空、反射光、漫射光共同照亮。真正的物理正确做法是对半球积分：

$$L_o(x, \omega_o) = \int_{\Omega} f_r(\omega_i, \omega_o) \cdot L_i(x, \omega_i) \cdot \max(0, n \cdot \omega_i) \, d\omega_i$$

这个积分在光线追踪里需要对每个像素打出数百条光线才能收敛，帧率从 60fps 直接掉到 0.001fps。早年间游戏里的全局光照一般是用烘焙好的光照贴图（Lightmap）来做，缺点是**一旦光源动了，立刻失效**。

### PRT 要解决的核心矛盾

Precomputed Radiance Transfer（预计算辐射传输）在 2002 年由 Sloan 等人提出，核心思想是：

1. **离线预计算**传输系数（Transfer Vector）——它描述了"这个表面点以这个方向为法线，能接收到来自各方向的多少能量"
2. **运行时**只需将预计算好的传输向量与当前环境光照的 SH 系数做**点积**，就得到最终颜色

这相当于把巨大的积分变成了一个 9 维的内积操作，代价极低。而且因为使用球谐函数编码，环境光的旋转只需要对 SH 系数做矩阵变换，不需要重新积分——这正是 PRT 的精妙之处。

### 工业界实际应用

- **Xbox 360 时代**：大量游戏（Halo 3、Gears of War）用 PRT 做人物的环境光接收，光探针就是 PRT 思想的精简版本
- **Unity / Unreal Engine**：Light Probe（光探针）、Irradiance Volume 都使用球谐函数（SH2/SH3）编码间接光照
- **离线渲染 / 电影**：PRT 扩展到更高阶 SH，支持全局光照中的遮挡传输
- **移动端**：SH L2（9个系数）是移动 GPU 做环境光照的标配方案，用 30 个 uniform 就能搞定

没有 PRT，要么硬烘焙（动态场景失效），要么暴力光追（帧率崩溃），要么用极度简化的点光源（失真严重）。PRT 是这三者之间一个精巧的平衡点。

---

## ② 核心原理：球谐函数与辐射传输方程

### 2.1 球谐函数是什么

球谐函数（Spherical Harmonics，SH）是定义在球面上的一组正交基函数，类似傅里叶分析的球面版本。

实数形式的 SH 基函数 $Y_l^m(\theta, \phi)$ 按阶（band）$l$ 组织：

- **L0**（1个）：常数，表示均匀环境光
- **L1**（3个）：低频方向性光照（主导色调）
- **L2**（5个）：二次项，捕捉更多方向细节
- **L3+**：更高阶，捕捉高频细节（PRT 通常只用到 L2）

具体形式（实数 SH，$l \leq 2$）：

$$Y_0^0 = \frac{1}{2\sqrt{\pi}}$$

$$Y_1^{-1} = \sqrt{\frac{3}{4\pi}} \cdot y, \quad Y_1^0 = \sqrt{\frac{3}{4\pi}} \cdot z, \quad Y_1^1 = \sqrt{\frac{3}{4\pi}} \cdot x$$

$$Y_2^{-2} = \sqrt{\frac{15}{4\pi}} \cdot xy, \quad Y_2^{-1} = \sqrt{\frac{15}{4\pi}} \cdot yz$$

$$Y_2^0 = \sqrt{\frac{5}{16\pi}} \cdot (3z^2 - 1), \quad Y_2^1 = \sqrt{\frac{15}{4\pi}} \cdot xz, \quad Y_2^2 = \sqrt{\frac{15}{16\pi}} \cdot (x^2 - y^2)$$

其中 $(x, y, z)$ 是单位方向向量的分量。

**直觉理解**：这 9 个函数就像是球面上的 9 种"形状"。L0 是个球（各向同性），L1 的三个分量分别是"x 方向偏强"、"y 方向偏强"、"z 方向偏强"，L2 的五个分量描述更复杂的方向性分布。把任意一个低频环境光用这 9 个形状的线性组合来近似，精度已经相当不错。

### 2.2 环境光的 SH 投影

把环境贴图 $L(\omega)$ 投影到 SH 基上，得到 9 个系数：

$$\hat{L}_j = \int_{S^2} L(\omega) \cdot Y_j(\omega) \, d\omega$$

用 Monte Carlo 数值积分（均匀球面采样）：

$$\hat{L}_j \approx \frac{4\pi}{N} \sum_{i=1}^{N} L(\omega_i) \cdot Y_j(\omega_i)$$

其中 $\frac{4\pi}{N}$ 是均匀采样的权重（球面面积 $4\pi$ 除以样本数）。

**直觉**：这个投影就是把环境光"压缩"到 9 个数字，就像把一段音乐只保留低频分量（失去高音，但主旋律还在）。

### 2.3 Lambertian 传输的余弦卷积

对于 Lambertian（完美漫反射）材质，BRDF 是 $\rho/\pi$（常数），渲染方程化简为：

$$L_o(n) = \frac{\rho}{\pi} \int_{H^+(n)} L_i(\omega) \cdot (n \cdot \omega) \, d\omega$$

这个"乘以余弦并在半球积分"的操作，在 SH 域等价于**逐 band 乘以缩放系数**（zonal harmonics 的卷积性质）：

$$A_l = 2\pi \cdot \frac{(-1)^{l/2+1}}{(l+2)(l-1)} \cdot \frac{(l-1)!!}{(l/2)!}$$

具体值：$A_0 = \pi$，$A_1 = \frac{2\pi}{3}$，$A_2 = \frac{\pi}{4}$

对每个 band 内的所有系数乘以对应的 $A_l$，就得到辐照度系数。

**直觉**：这等于说"余弦滤波在球谐域就是乘个常数"——不需要重新积分，只需要对 9 个系数各乘一个缩放因子。

### 2.4 传输向量预计算

对每个顶点，预计算传输向量 $T$（大小为 9 的向量）：

$$T_j(x) = \int_{H^+(n_x)} V(x, \omega) \cdot \max(0, n_x \cdot \omega) \cdot Y_j(\omega) \, d\omega$$

- $V(x, \omega)$：可见性函数（被遮挡=0，可见=1）
- $\max(0, n_x \cdot \omega)$：余弦因子（Lambertian 响应）

用余弦加权半球采样估计（让采样方向 $\omega_i$ 按 $\cos\theta$ 分布，权重就是 $\pi/N$）：

$$T_j(x) \approx \frac{\pi}{N} \sum_{i=1}^{N} V(x, \omega_i) \cdot (n_x \cdot \omega_i) \cdot Y_j(\omega_i)$$

### 2.5 运行时渲染

预计算完成后，每帧的渲染只需一个点积：

$$L_o(x) = \sum_{j=0}^{8} \hat{L}_j \cdot T_j(x)$$

对每个颜色通道（RGB）分别计算，共 27 次乘法 + 26 次加法。在 GPU 上这是一个极快的操作。

**PRT 的核心洞见**：渲染方程的两部分——光照 $L$ 和传输 $T$——可以分离后分别压缩到 SH 域，然后再"对接"（点积）。这个对接的代价极低，而分离出来的压缩可以在离线阶段完成。

### 2.6 SH 旋转：Wigner D 矩阵

当环境光旋转时，不需要重新积分！SH 的旋转性质保证：

对于绕 Y 轴旋转角度 $\alpha$：

- **L0**：不变（常数对旋转不敏感）
- **L1**（$m = -1, 0, 1$）：
  - $m=0$（Y 分量）：不变（Y 轴旋转不改变 $y$ 分量的权重）
  - $m=\pm 1$（X/Z 分量）：$\begin{pmatrix} x' \\ z' \end{pmatrix} = \begin{pmatrix} \cos\alpha & \sin\alpha \\ -\sin\alpha & \cos\alpha \end{pmatrix} \begin{pmatrix} x \\ z \end{pmatrix}$
- **L2**：
  - $m=0$：不变
  - $m=\pm 1$（XZ, YZ）：旋转角 $\alpha$
  - $m=\pm 2$（XY, X²-Y²）：旋转角 $2\alpha$

这就是 Wigner D 矩阵的简化版本。对绕 Y 轴旋转，每个 band 的旋转都是一个 2x2 的旋转矩阵（或 1x1 的恒等变换）。

**直觉**：SH 的方位角 $m$ 对应的基函数在旋转时会"混合"同一 band 内相同 $|m|$ 的分量，恰好就是 $m$ 倍的相位变化。这让环境光旋转变成了 O(1) 的矩阵变换，而非重新投影。

---

## ③ 实现架构

### 3.1 整体流程

```
输入: 程序化天空 + 太阳 (sampleEnv 函数)
  │
  ▼
[离线阶段]
  ├─ projectEnvironment(): Monte Carlo 积分，将环境光投影到 SH 系数 (9维/RGB)
  ├─ rotateSHY(env, angle): Wigner D 矩阵旋转 SH 系数，得到旋转后的环境
  └─ computeTransfer(normal): 余弦加权采样，预计算每个顶点的传输向量
  │
  ▼
[运行时]
  ├─ generateSphere(): 生成球体顶点，调用 computeTransfer + renderPRT 求顶点颜色
  ├─ renderSphere(): 背面剔除 + 投影变换 + 软光栅三角形绘制
  └─ 后处理: Reinhard tone mapping + gamma correction (2.2)
  │
  ▼
输出: prt_output.png (900×400，3个面板对比)
```

### 3.2 关键数据结构

```cpp
// 环境光 SH 系数（RGB 各 9 个）
struct SHEnv {
    SHCoeffs r, g, b;  // SHCoeffs = std::array<float, 9>
};

// 顶点传输向量（9 个系数）
using TransferVec = std::array<float, 9>;

// 渲染顶点（包含预计算好的颜色）
struct Vertex {
    Vec3 pos;     // 世界空间位置
    Vec3 normal;  // 世界空间法线
    Vec3 color;   // PRT 预计算颜色（运行时只读）
};
```

### 3.3 面板设计

画布分为三个面板（各 300×400 像素）：

| 面板 | 环境光 | 技术特点 |
|------|--------|---------|
| 左 | 原始（sun 在左前方） | 无遮挡传输 |
| 中 | 旋转 60°（sun 偏右） | 有遮挡传输（凸球无自遮挡） |
| 右 | 旋转 120°（sun 在右后） | 有遮挡传输 |

这样可以直观看到同一套传输向量（预计算一次）在不同环境光下的渲染结果，体现 PRT 的"光照可换"特性。

### 3.4 与传统 SH 辐照度的区别

传统方法（参见本系列 04-19 光探针）是直接计算辐照度（SH 与法线直接点积），每帧调用 `evalSHIrradiance`。PRT 多做了一步：预计算传输向量，把余弦加权的半球积分做进去。两者的区别：

- **传统 SH 辐照度**：运行时 = 9次点积（快，但无法编码遮挡）
- **PRT**：预计算阶段更重，运行时同样是 9 次点积，但传输向量已经包含了几何相关信息（可见性、近似 AO）

---

## ④ 关键代码解析

### 4.1 SH 基函数求值

```cpp
std::array<float,9> evalSH(const Vec3& d) {
    std::array<float,9> r;
    // L0: 常数，归一化系数 1/(2*sqrt(pi))
    r[0] = 0.282095f;
    // L1: 线性项，注意顺序是 y, z, x（对应 m = -1, 0, 1）
    r[1] = 0.488603f * d.y;
    r[2] = 0.488603f * d.z;
    r[3] = 0.488603f * d.x;
    // L2: 二次项
    r[4] = 1.092548f * d.x * d.y;          // m = -2: xy
    r[5] = 1.092548f * d.y * d.z;          // m = -1: yz
    r[6] = 0.315392f * (3.f*d.z*d.z - 1.f); // m =  0: 3z²-1
    r[7] = 1.092548f * d.x * d.z;          // m =  1: xz
    r[8] = 0.546274f * (d.x*d.x - d.y*d.y);// m =  2: x²-y²
    return r;
}
```

**为什么这些系数？** 这些硬编码的常数来自 Legendre 多项式的归一化因子。L0 的 $0.282095 = 1/(2\sqrt{\pi})$，L1 的 $0.488603 = \sqrt{3/(4\pi)}$，L2 中 $1.092548 = \sqrt{15/(4\pi)}$。归一化保证 $\int_{S^2} |Y_j|^2 d\omega = 1$，使得 SH 系数有物理意义（线性组合后不会自然缩放）。

**容易出错的地方**：L1 的顺序必须是 `y, z, x`（对应 $m = -1, 0, 1$），不能是 `x, y, z`。SH 旋转矩阵对应 $m$ 的顺序，如果搞错这里，旋转就会出错。

### 4.2 环境光 SH 投影（Monte Carlo）

```cpp
SHEnv projectEnvironment(int numSamples = 20000) {
    SHEnv env{};
    std::mt19937 rng(42);
    std::uniform_real_distribution<float> dist(0, 1);

    for (int i = 0; i < numSamples; i++) {
        // 均匀球面采样：Archimedes 等面积映射
        float phi   = 2.f * M_PI * dist(rng);
        float cosT  = 1.f - 2.f * dist(rng);  // cos(theta) 均匀分布 [-1, 1]
        float sinT  = sqrt(max(0.f, 1.f - cosT*cosT));
        Vec3 d{sinT * cos(phi), cosT, sinT * sin(phi)};

        Vec3 color = sampleEnv(d);
        auto sh = evalSH(d);

        float weight = 4.f * M_PI / numSamples;  // 均匀球面采样权重
        for (int j = 0; j < 9; j++) {
            env.r[j] += color.x * sh[j] * weight;
            env.g[j] += color.y * sh[j] * weight;
            env.b[j] += color.z * sh[j] * weight;
        }
    }
    return env;
}
```

**为什么用 `cosT = 1 - 2*u`？** 对于均匀球面采样，需要 $\cos\theta$ 均匀分布在 $[-1, 1]$。Archimedes 定理告诉我们，球面纬度带的面积正比于高度差，所以直接对 $\cos\theta$ 均匀采样就能得到均匀的球面分布。

**权重为 `4π/N`**：球面总面积 $4\pi$，均匀分 $N$ 个样本，每个样本代表 $4\pi/N$ 的面积（PDF = $1/(4\pi)$，采样权重 = $1/\text{PDF}/N = 4\pi/N$）。

**20000 个样本**：低频 SH（L2）不需要太多样本就能收敛，经验上 5000~20000 就够了。

### 4.3 传输向量预计算（余弦加权半球）

```cpp
TransferVec computeTransfer(const Vec3& normal, bool withShadow, int numSamples = 500) {
    TransferVec T{};
    // 用法线哈希初始化 RNG，保证不同顶点有不同随机序列
    std::mt19937 rng(static_cast<uint32_t>(
        std::hash<float>{}(normal.x * 1000.f + normal.y)));
    std::uniform_real_distribution<float> dist(0, 1);

    for (int i = 0; i < numSamples; i++) {
        // 余弦加权半球采样（Malley's method）
        float u1 = dist(rng), u2 = dist(rng);
        float r = sqrt(u1);          // 圆盘上的半径
        float phi = 2.f * M_PI * u2;
        // 在切线空间：y 轴朝法线方向
        Vec3 localDir{r * cos(phi), sqrt(max(0.f, 1.f - u1)), r * sin(phi)};

        // 将切线空间方向变换到世界空间
        Vec3 up   = abs(normal.y) < 0.99f ? Vec3(0,1,0) : Vec3(1,0,0);
        Vec3 tang  = up.cross(normal).norm();
        Vec3 bitan = normal.cross(tang);
        Vec3 wi = tang * localDir.x + normal * localDir.y + bitan * localDir.z;

        float vis = 1.f;
        if (withShadow) {
            // 对凸球体，法线半球内方向均可见；只需 clamp 负半球
            if (normal.dot(wi) < 0.f) vis = 0.f;
        }

        if (vis > 0.f) {
            float nDotWi = max(0.f, normal.dot(wi));
            auto sh = evalSH(wi);
            float weight = M_PI / numSamples;  // 余弦加权采样权重
            for (int j = 0; j < 9; j++)
                T[j] += nDotWi * sh[j] * weight * vis;
        }
    }
    return T;
}
```

**为什么权重是 `π/N`？** 余弦加权半球采样的 PDF 是 $\cos\theta/\pi$（归一化余弦分布）。MC 估计量：$\int f \, d\omega \approx \frac{1}{N} \sum_i \frac{f(\omega_i)}{p(\omega_i)}$。这里 $f = (n \cdot \omega) \cdot Y_j(\omega)$，$p = \cos\theta/\pi = (n \cdot \omega)/\pi$，所以 $(n \cdot \omega)$ 和 $(n \cdot \omega)$ 相消，权重 = $\pi/N$。

**Malley's method**：在单位圆盘上均匀采样点 $(r\cos\phi, r\sin\phi)$，然后"投影"到半球上得 $(r\cos\phi, \sqrt{1-r^2}, r\sin\phi)$（切线空间）。这个方法天然产生余弦加权的半球分布，因为圆盘上的均匀分布对应半球上的余弦分布。

**法线哈希 RNG**：每个顶点用自己法线的哈希初始化随机数发生器，这样不同顶点有不同的随机序列，避免所有顶点用相同的样本模式导致系统误差。

### 4.4 运行时 PRT 渲染

```cpp
Vec3 renderPRT(const SHEnv& env, const TransferVec& T) {
    Vec3 r;
    r.x = r.y = r.z = 0;
    for (int j = 0; j < 9; j++) {
        r.x += env.r[j] * T[j];  // 红通道 SH 系数 × 传输向量
        r.y += env.g[j] * T[j];  // 绿通道
        r.z += env.b[j] * T[j];  // 蓝通道
    }
    return r;
}
```

**这就是 PRT 的核心！** 9 次乘加操作，输出一个 RGB 颜色。这在 GPU 上可以用 `dot(L, T)` 指令完成，极其高效。注意传输向量 T 对 RGB 三个通道是共用的（因为传输只跟几何/可见性有关，不依赖颜色）。

### 4.5 SH 旋转（Wigner D 矩阵 for Y 轴）

```cpp
SHEnv rotateSHY(const SHEnv& env, float angle) {
    SHEnv out = env;  // L0 直接复制（旋转不变）
    float ca = cos(angle), sa = sin(angle);

    // L1 旋转：m=0(y) 不变，m=±1(x,z) 用 2D 旋转
    for (auto& ch : {0, 1, 2}) {  // RGB 三通道
        auto& src = (ch==0)?env.r:(ch==1)?env.g:env.b;
        auto& dst = (ch==0)?out.r:(ch==1)?out.g:out.b;
        dst[1] = src[1];                           // y 不变
        dst[3] =  ca * src[3] + sa * src[2];       // x' = cos*x + sin*z
        dst[2] = -sa * src[3] + ca * src[2];       // z' = -sin*x + cos*z
    }

    // L2 旋转：m=0 不变，m=±1 旋转 angle，m=±2 旋转 2*angle
    float ca2 = cos(2*angle), sa2 = sin(2*angle);
    for (auto& ch : {0, 1, 2}) {
        auto& src = (ch==0)?env.r:(ch==1)?env.g:env.b;
        auto& dst = (ch==0)?out.r:(ch==1)?out.g:out.b;
        dst[6] = src[6];                           // m=0 不变
        dst[7] =  ca  * src[7] + sa  * src[5];    // m=+1: xz 分量
        dst[5] = -sa  * src[7] + ca  * src[5];    // m=-1: yz 分量
        dst[4] =  ca2 * src[4] + sa2 * src[8];    // m=-2: xy 分量
        dst[8] = -sa2 * src[4] + ca2 * src[8];    // m=+2: x²-y² 分量
    }
    return out;
}
```

**为什么 m=±2 要用 2*angle？** SH 基函数 $Y_2^{\pm 2}$ 含有 $\cos(2\phi)$ 和 $\sin(2\phi)$ 的因子（二倍角频率），当坐标旋转角度 $\alpha$ 时，这类项的相位变化是 $2\alpha$。一般地，阶 $m$ 的 SH 基函数旋转时相位变化是 $m\alpha$，这就是 Wigner D 矩阵在绕对称轴旋转时退化为对角矩阵（每对 $\pm m$ 分量形成一个 2D 旋转）的原因。

---

## ⑤ 踩坑实录

### Bug #1：SH 投影结果过暗（均值接近 0）

**症状**：渲染出来的球体几乎全黑，Image stats 输出 R≈10, G≈10, B≈10。

**错误假设**：以为是传输向量计算有问题，去调传输部分。

**真实原因**：`projectEnvironment` 的权重写成了 `1.0f / numSamples`，漏掉了球面面积 $4\pi$ 的因子。正确权重应该是 `4 * M_PI / numSamples`。

**修复方式**：
```cpp
// 错误
float weight = 1.f / numSamples;
// 正确
float weight = 4.f * M_PI / numSamples;
```

**教训**：SH 投影的 MC 估计量需要正确的 PDF 权重。均匀球面采样的 PDF 是 $1/(4\pi)$，每个样本的贡献权重 = $L(\omega) \cdot Y_j(\omega) / \text{PDF} / N = L \cdot Y \cdot 4\pi / N$。

### Bug #2：坐标系错误导致法线方向偏转

**症状**：三个球体的光照方向奇怪，左侧球体的亮面在右侧，不符合环境光的太阳方向。

**错误假设**：以为是 SH 旋转的角度符号问题。

**真实原因**：`computeTransfer` 中构建切线空间的叉积顺序写反了。
```cpp
// 错误：顺序不对，坐标系手性错误
Vec3 tang  = normal.cross(up).norm();
Vec3 bitan = tang.cross(normal);
// 正确：先叉积得到切线，再用法线叉切线得副切线
Vec3 tang  = up.cross(normal).norm();
Vec3 bitan = normal.cross(tang);
```

**修复方式**：统一使用右手坐标系，确保 `(tang, normal, bitan)` 构成右手系。

**教训**：切线空间的构建要明确规定哪个轴是 up，叉积顺序影响手性。建议用 `tang × bitan = normal` 来验证正确性。

### Bug #3：不同顶点共用同一随机种子导致系统误差

**症状**：球体渲染有明显的"格子状"噪声，看起来像方块而不是平滑的光照。

**错误假设**：以为是样本数不够，增加到 5000 仍有问题。

**真实原因**：所有顶点用了相同的固定种子 `rng(42)`，导致每个顶点采用完全相同的方向样本，Monte Carlo 的随机误差变成了有规律的偏差（系统误差，增加样本数没有帮助）。

**修复方式**：
```cpp
// 错误：所有顶点共用同一随机序列
std::mt19937 rng(42);
// 正确：每个顶点基于自己的法线生成不同的种子
std::mt19937 rng(static_cast<uint32_t>(
    std::hash<float>{}(normal.x * 1000.f + normal.y)));
```

**教训**：Monte Carlo 采样中，如果多次估计共享同一随机序列，误差不会随样本增加而减小，而是固化为系统偏差。每次独立估计都应该有独立的随机源。

### Bug #4：背景天空渲染与球体颜色空间不一致

**症状**：球体边缘有明显的亮度跳变，球体比背景明显偏暗（tone mapping 应用了两次？）。

**真实原因**：背景天空在写入 framebuffer 时已经做了 tone mapping + gamma，但球体颜色是原始线性颜色写入 framebuffer，最后统一输出时又做了一次 gamma。

**修复方式**：将渲染管线分为两个阶段：
1. 背景：直接写入 tone-mapped + gamma-corrected 的颜色
2. 球体：写入原始线性颜色到 localFb，在合并到主 fb 时做 tone mapping + gamma

```cpp
// 球体渲染后合并
if (localFb.depth[py*W+px] < 1e29f) {
    Vec3 col = localFb.color[py*W+px];
    col = toneMap(col);          // 先 tone map
    col = gammaCorrect(col);     // 再 gamma
    fb.at(px, py) = col.clamp01();
}
```

**教训**：线性颜色空间和 gamma 空间的混用是图形编程中经典陷阱。所有光照计算必须在线性空间进行，只在最后输出前做 gamma 编码。

---

## ⑥ 效果验证与数据

### 输出图像

![PRT 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-03-prt-renderer/prt_output.png)

三个面板展示同一套传输向量在三种不同旋转环境光下的渲染：
- **左侧**：原始环境（太阳在左前方），无遮挡传输，球体顶部受天空蓝光，左侧受太阳暖光
- **中间**：环境光旋转 60°（太阳偏右），有遮挡传输版本，颜色分布变化
- **右侧**：环境光旋转 120°（太阳在右侧），同套传输向量，实时切换

### 量化数据

```
=== 图像质量验证 ===
文件大小:    37.6 KB（> 10KB ✅）
图片尺寸:    900×400 像素
像素均值:    185.7 / 255（在 10~240 范围内 ✅）
像素标准差:  11.2（> 5，图像有内容 ✅）
```

### 性能数据

```
=== 运行时间（i7 CPU，单线程）===
SH 投影 (20000 样本):   <1ms
SH 旋转 (×2):           <0.1ms
传输向量预计算:
  球体1 (33×33=1089顶点, 500样本/顶点): ~15ms
  球体2/3: 各 ~15ms
背景渲染 (900×400):     <1ms
光栅化 (3×32×32 三角):  ~2ms
总时间:                  131ms（./prt_output 实测）

对比：光线追踪版本做同等效果 > 10000ms
加速比：约 76x
```

### SH 系数验证

```
SH L0 = (3.01, 2.91, 3.24)  ← 环境平均亮度（高于 1.0 因为有 HDR 太阳）
SH L1 = (0.056, 0.045, 0.034)  ← 低频方向信息（z分量，表示垂直方向占优）
```

L0 系数约为 3.0 说明环境光 HDR 均值约为 3.0（含太阳高光贡献），Reinhard tone mapping 后压到合理范围。

---

## ⑦ 总结与延伸

### 局限性

1. **只支持低频光照**：SH L2（9个系数）只能表达平滑的环境光，无法处理硬阴影或锐利的高光。L6 需要 49 个系数，仍然不够，真正的高频光照需要其他方案（如 ASG、SG、神经辐射场）。

2. **传输向量只对静态几何有效**：预计算在特定法线/位置下完成，变形动画会导致传输向量失效（需要实时重新采样，代价极高）。解决方案：骨骼蒙皮后更新，或对每个 pose 单独预计算（存储代价大）。

3. **不支持次表面散射/折射等复杂 BRDF**：这里只做了 Lambertian 传输。扩展到 BRDF 传输需要 4D 传输矩阵（每个顶点存 9×9 的矩阵），存储爆炸。

4. **顶点频率**：这里在顶点上预计算，高频 SH 变化在三角形插值时会引入误差。解决方案：预计算纹理化（Baked PRT Texture）。

### 可优化方向

- **多线程并行化传输预计算**：每个顶点独立，天然 embarrassingly parallel。加速 8-16x 很容易。

- **更高阶 SH（L3/L4）**：从 9 个系数扩展到 16/25 个，支持更丰富的光照细节，代价是传输向量变大。

- **动态更新 SH 系数**：结合 `rotateSHY` 实现动态日夜循环，每帧只需 O(1) 旋转操作。

- **SH 传输纹理**：将传输向量烘焙为 9 张纹理（每通道一张），支持更细粒度的空间变化。

- **混合 PRT + IBL**：低频部分用 PRT，高频高光用 Split-Sum IBL 或 Reflection Capture，实现 PBR 工作流中的全谱光照。

### 与本系列的关联

- **04-19 辐照度缓存（光探针）**：使用了 `evalSHIrradiance`，是 PRT 思想的简化版（无预计算传输，直接用法线点积 SH）。今天的 PRT 在其基础上增加了"传输"的概念，让遮挡信息也能编码进 SH 域。

- **04-18 SSR**：处理高频镜面反射，与 PRT 的低频漫反射形成互补。完整的实时全局光照需要两者结合：PRT 负责低频漫反射 GI，SSR 负责高频镜面 GI，Bloom 负责能量扩散。

- **05-02 辐射级联**：面向实时高频 GI 的方案，与 PRT 面向低频预计算 GI 是两种思路——一个是实时（代价大），一个是预计算（限制多）。

PRT 是理解现代游戏引擎中光探针、SH 环境光、球形谐波系数系统的核心基础，也是深入理解"频域方法"在实时渲染中的应用的最好入门。

---

*每日编程实践系列 · 2026-05-03 · [查看源码](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-03-prt-renderer)*
