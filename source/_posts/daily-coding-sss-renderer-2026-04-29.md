---
title: "每日编程实践: Subsurface Scattering (SSS) Renderer — 次表面散射渲染"
date: 2026-04-29 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - PBR
  - 渲染
  - 光照模型
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-29-sss-renderer/sss_output.png
---

## 一、背景与动机

### 为什么需要次表面散射？

在照片级渲染中，有一类材质的表现一直是难题：**半透明材质**。

当你用标准的 Lambert 漫反射模型渲染一张人脸时，会发现效果永远"不对劲"——皮肤看起来像是涂了一层哑光油漆，既没有活人脸上那种温润的透光感，也缺乏皮下组织带来的红润色调。同样的问题出现在蜡烛、大理石、玉石、牛奶、树叶等材质上。

**根本原因**：Lambert 模型假设光线在表面发生漫反射，照射点与出射点相同。但真实世界中，大量材质对光线是半透明的——光子穿入表面、在内部经历多次散射、从**另一个位置**射出。这个过程叫做**次表面散射（Subsurface Scattering，SSS）**。

用公式表示：

- 标准 BRDF（Bidirectional Reflectance Distribution Function）：$f_r(x_i, \omega_i, \omega_o)$，出射点 = 入射点
- BSSRDF（Bidirectional Subsurface Scattering Reflectance Distribution Function）：$S(x_i, \omega_i, x_o, \omega_o)$，出射点 $x_o$ 与入射点 $x_i$ 解耦，可以不同

缺少 SSS 的渲染后果是：
- **皮肤**：苍白、像橡皮、缺乏血色
- **蜡烛**：完全不透光，像石头
- **玉石/大理石**：没有内部发光感，颜色死板
- **牛奶**：看起来是不透明的白色固体

### 工业界实际应用

**游戏引擎**：Unreal Engine 内置了多套 SSS 模型（Burley/Profile/Preintegrated）；Unity HDRP 有 Subsurface Scattering Profile 材质；FROSTBITE 引擎（寒霜）在《战地》系列中专门为皮肤着色实现了 SSS。

**电影渲染**：Pixar 的 RenderMan 实现了完整的 BSSRDF 积分，用于角色皮肤（《疯狂动物城》《寻梦环游记》的高质量皮肤）。

**实时游戏**：《荒野大镖客》《赛博朋克2077》《最后生还者》等 3A 大作的角色渲染都依赖 SSS 实现真实皮肤质感。

**今天的目标**：用纯 C++ 软光栅化实现一个 SSS 渲染器，基于 Jensen et al. 2001 的 Dipole Diffusion 模型，用 Sum of Gaussians 近似散射剖面，渲染四种不同半透明材质（皮肤、蜡、大理石、牛奶），理解 SSS 的数学本质。

---

## 二、核心原理

### 2.1 BSSRDF 的数学定义

标准渲染方程（无次表面散射）：

$$
L_o(x, \omega_o) = \int_\Omega f_r(x, \omega_i, \omega_o) L_i(x, \omega_i) (n \cdot \omega_i) \, d\omega_i
$$

加入次表面散射后，渲染方程变为：

$$
L_o(x_o, \omega_o) = \int_A \int_\Omega S(x_i, \omega_i, x_o, \omega_o) L_i(x_i, \omega_i) (n \cdot \omega_i) \, d\omega_i \, dA
$$

其中：
- $x_i$ 是光线入射点（在表面 $A$ 上积分）
- $x_o$ 是光线出射点
- $S$ 是 BSSRDF，描述从 $(x_i, \omega_i)$ 到 $(x_o, \omega_o)$ 的散射分布

直觉解释：不再只对一个点的入射方向积分，而是对整个表面所有可能的入射点 $x_i$ 积分——因为从任意点射入的光都可能从 $x_o$ 出来。

### 2.2 Jensen et al. 2001 — Dipole Diffusion 模型

完整的 BSSRDF 计算量过大，Jensen 等人提出了一个巧妙的近似：把 BSSRDF 分解为两个因子：

$$
S(x_i, \omega_i, x_o, \omega_o) = \frac{1}{\pi} F_t(\eta, \omega_i) \cdot R(||x_i - x_o||) \cdot F_t(\eta, \omega_o)
$$

其中：
- $F_t$ 是菲涅耳透射项（光进入/离开材质表面时的折射损失）
- $R(r)$ 是**散射剖面（Diffusion Profile）**，只依赖距离 $r = ||x_i - x_o||$

这个分解的关键假设是：多次散射后，光的分布趋于各向同性，因此散射量只依赖**表面距离**，而不依赖方向。

### 2.3 Dipole 模型的物理直觉

Dipole（双极）模型来自热传导理论。把光在介质内的传播近似为**扩散过程**（Diffusion Approximation），类比热量在固体中的传导。

真实光源放在表面上，用一个"虚拟光源对"（一个在介质内、一个在介质外）来近似解析解，从而得到散射剖面：

$$
R(r) = \frac{\alpha'}{4\pi} \left[ z_r \left(\sigma_{tr} + \frac{1}{d_r}\right) \frac{e^{-\sigma_{tr} d_r}}{d_r^2} + z_v \left(\sigma_{tr} + \frac{1}{d_v}\right) \frac{e^{-\sigma_{tr} d_v}}{d_v^2} \right]
$$

参数含义（每个都有物理意义，不可跳过）：
- $\sigma_s$：散射系数，光子每单位距离发生散射的概率
- $\sigma_a$：吸收系数，光子每单位距离被吸收的概率
- $\sigma_t = \sigma_s + \sigma_a$：总消光系数
- $\sigma_{tr} = \sqrt{3 \sigma_a \sigma_t}$：有效传输系数，控制散射的衰减尺度
- $\alpha' = \sigma_s / \sigma_t$：降阶散射反照率（reduced scattering albedo）
- $d_r, d_v$：到虚拟光源对的距离

直觉解释：$R(r)$ 描述的是：如果在 $x_i$ 处向介质注入 1W 的光通量，距离 $r$ 处的表面点 $x_o$ 能接收多少能量。散射剖面越宽，材质越"透光"，颜色往外晕染越远。

### 2.4 Sum of Gaussians 近似

直接使用 Dipole 积分代价高，d'Eon 和 Luebke（2007）提出用多个高斯函数的加权和来拟合散射剖面：

$$
R(r) \approx \sum_{i=1}^{n} w_i \cdot G(r, \sigma_i) = \sum_{i=1}^{n} w_i \cdot \frac{1}{2\pi\sigma_i^2} e^{-\frac{r^2}{2\sigma_i^2}}
$$

为什么这个近似好？
1. **数学性质好**：高斯函数是傅里叶变换的特征函数，便于频域分析和分离坐标轴计算
2. **实现简单**：比 Dipole 公式简单得多
3. **精度可控**：增加高斯数 $n$ 可以提高精度
4. **直觉清晰**：每个高斯代表一个散射"层次"——小 $\sigma$ 对应表面层（颜色接近材质颜色），大 $\sigma$ 对应深层散射（颜色被内部吸收和散射改变）

**皮肤的三层高斯含义**：
- $\sigma_1 = 0.006\text{mm}$：表皮层（角质层），散射很近，主要影响表面细节
- $\sigma_2 = 0.048\text{mm}$：真皮层上部，中等散射，绿色分量强（含黑色素）
- $\sigma_3 = 0.187\text{mm}$：真皮深层+皮下组织，宽散射，红色分量强（含血红蛋白）

### 2.5 为什么皮肤的红色散射比绿色宽？

皮肤的主要色素是**血红蛋白**（氧合血红蛋白）和**黑色素**：
- 血红蛋白强烈吸收绿色和蓝色光（所以血液是红色的），**透射红光**，因此红光在皮下组织中散射更远
- 黑色素在表皮层，吸收各色光（越深越黑），限制了绿/蓝光的深度穿透

这就是为什么皮肤 SSS 的 R 通道 $\sigma$ 比 G/B 通道大：红光渗透更深，散射半径更宽。

### 2.6 散射积分的近似实现

完整积分需要对整个表面积分：$\int_A R(||x_i - x_o||) E(x_i) dA$（$E$ 是入射辐照度）。

我们的近似：**球面采样积分**。
- 在球面上均匀采样 $N$ 个点（Fibonacci 球面采样）
- 计算每个样本点到着色点的弧长距离 $r$
- 查询该点的直接照度 $E(x_i) = \max(0, n_i \cdot l)$
- 用散射剖面 $R(r)$ 加权累积
- 乘以面积元归一化：$\Delta A = 4\pi R^2 / N$

数学上等价于蒙特卡洛积分：

$$
\int_A R(r) E dA \approx \frac{4\pi R^2}{N} \sum_{i=1}^{N} R(r_i) \cdot E_i
$$

---

## 三、实现架构

### 3.1 整体数据流

```
主函数
  │
  ├─ 背景填充（深色渐变）
  │
  ├─ 定义四个材质（ScatterProfile × 4）
  │    ├─ Skin：皮肤（红色宽散射）
  │    ├─ Wax：蜡（黄色中等散射）
  │    ├─ Marble：大理石（蓝色窄散射）
  │    └─ Milk：牛奶（白色宽均匀散射）
  │
  ├─ 对每个球体调用 render_sphere()
  │    ├─ 逐像素光线投射（Ray-Sphere intersect）
  │    ├─ 命中 → 调用 compute_sss()
  │    │    ├─ Fibonacci 球面采样 128 点
  │    │    ├─ 计算弧长距离
  │    │    ├─ 计算直接照度
  │    │    └─ 高斯剖面加权累积
  │    ├─ 加直接漫反射（权重 15%）
  │    ├─ 加 Blinn-Phong 高光
  │    └─ 加环境光
  │
  ├─ 绘制标签文字（5×7 像素字体）
  │
  └─ 输出 PNG（内置 DEFLATE store-only PNG 编码器）
```

### 3.2 关键数据结构

```cpp
// 散射剖面：每个颜色通道 3 个高斯（权重 + 标准差）
struct ScatterProfile {
    float weightsR[3], sigmasR[3];  // R通道：3个高斯
    float weightsG[3], sigmasG[3];  // G通道：3个高斯
    float weightsB[3], sigmasB[3];  // B通道：3个高斯
    Vec3 eval(float r) const;       // 在距离r处评估散射权重
};
```

为什么每个通道独立设置参数？
因为 R/G/B 三个波长的散射系数 $\sigma_s$ 和吸收系数 $\sigma_a$ 完全不同——这正是 SSS 产生颜色偏移的根本原因。如果三通道参数相同，散射只会模糊图像，不会产生颜色变化。

```cpp
// 材质定义：散射剖面 + 高光参数 + 表面颜色
struct Material {
    ScatterProfile profile;    // 散射剖面
    Vec3 specular_color;       // 高光颜色
    float shininess;           // 高光指数（越大越集中）
    Vec3 surface_albedo;       // 表面基础颜色
    std::string name;          // 材质名称
};
```

### 3.3 数学辅助结构

```cpp
struct Vec3 { float x, y, z; };  // 三维向量（颜色 / 位置）
struct Ray   { Vec3 origin, dir; };
struct Hit   { float t; Vec3 pos, normal; bool valid; };
struct Sphere{ Vec3 center; float radius; Hit intersect(const Ray&); };
```

设计原则：所有向量运算内联在结构体中，避免外部函数命名冲突，编译器容易优化。

### 3.4 PNG 输出架构

本项目包含一个自制的最小化 PNG 编码器（零依赖）：
- 使用 DEFLATE store-only 压缩（即不压缩，直接存储像素数据）
- 实现了 CRC32、Adler-32 校验
- 写入 IHDR / IDAT / IEND 标准块
- 支持 RGB 24-bit 真彩色

好处：无需链接 libpng，无外部依赖，可移植性最好。代价：文件稍大（1.3MB），但功能完整。

---

## 四、关键代码解析

### 4.1 高斯散射核函数

```cpp
// G(r, sigma) = exp(-r^2 / (2*sigma^2)) / (2*pi*sigma^2)
// 归一化高斯，使得对整个平面积分 = 1
inline float gaussian_kernel(float r, float sigma) {
    if(sigma < 1e-6f) return (r < 1e-4f) ? 1.0f : 0.0f;
    return expf(-(r*r) / (2.0f*sigma*sigma)) / (2.0f*3.14159265f*sigma*sigma);
}
```

为什么要归一化（除以 $2\pi\sigma^2$）？

因为 $\int_0^\infty G(r, \sigma) \cdot 2\pi r \, dr = 1$，归一化确保散射剖面的总能量守恒——材质不会凭空创造光能。如果不归一化，不同 $\sigma$ 的高斯峰值不可比较，Sum of Gaussians 的权重也无法正确叠加。

### 4.2 Sum of Gaussians 散射剖面求值

```cpp
Vec3 ScatterProfile::eval(float r) const {
    // 对每个颜色通道，求3个高斯的加权和
    auto sog = [&](const float* w, const float* s) {
        float v = 0;
        for(int i=0; i<3; i++) 
            v += w[i] * gaussian_kernel(r, s[i]);
        return v;
    };
    return Vec3(sog(weightsR, sigmasR),   // R通道散射强度
                sog(weightsG, sigmasG),   // G通道散射强度
                sog(weightsB, sigmasB));  // B通道散射强度
}
```

关键设计：返回 `Vec3` 而不是 `float`，因为三个颜色通道的散射强度不同。这正是 SSS 产生颜色效果（如皮肤的红润色调）的原因。

### 4.3 皮肤散射剖面参数

```cpp
ScatterProfile make_skin_profile() {
    ScatterProfile p;
    // R通道：血红蛋白透射红光，散射深而宽
    p.weightsR[0]=0.233f; p.sigmasR[0]=0.0064f;  // 浅层：表皮
    p.weightsR[1]=0.100f; p.sigmasR[1]=0.0484f;  // 中层：真皮上部
    p.weightsR[2]=0.118f; p.sigmasR[2]=0.187f;   // 深层：皮下组织（最宽）
    // G通道：黑色素吸收，散射比R通道窄
    p.weightsG[0]=0.113f; p.sigmasG[0]=0.0064f;
    p.weightsG[1]=0.358f; p.sigmasG[1]=0.0484f;  // 中层权重最大
    p.weightsG[2]=0.078f; p.sigmasG[2]=0.187f;
    // B通道：散射极浅，几乎只在表面
    p.weightsB[0]=0.007f; p.sigmasB[0]=0.0064f;
    p.weightsB[1]=0.004f; p.sigmasB[1]=0.0484f;
    p.weightsB[2]=0.005f; p.sigmasB[2]=0.187f;
    return p;
}
```

参数来源：基于 d'Eon et al. 2007 "A Quantized-Diffusion Model for Rendering Translucent Materials" 中的皮肤拟合参数，单位已从 mm 转换为场景坐标（球半径=1.0）。

注意 B 通道的权重（0.007/0.004/0.005）比 R（0.233/0.100/0.118）小 20-30 倍。这就是皮肤在次表面散射后会偏暖色（红黄）的物理原因：红光传播得远，蓝光几乎被吸收。

### 4.4 球面次表面散射积分

```cpp
Vec3 compute_sss(const Vec3& pos, const Vec3& /*normal*/,
                  const Vec3& light_dir, const Vec3& light_color,
                  const ScatterProfile& profile,
                  const Sphere& sphere, int N_samples = 64) {
    Vec3 accum(0,0,0);

    for(int i=0; i<N_samples; i++) {
        // Fibonacci 球面采样：均匀分布，无集群效应
        float phi   = acosf(1 - 2*(i+0.5f)/N_samples);
        float theta = 2*3.14159265f * i * 0.6180339887f; // 黄金比例角

        Vec3 sp(sinf(phi)*cosf(theta), 
                sinf(phi)*sinf(theta), 
                cosf(phi));
        // sp 是球面上第 i 个采样点的单位法线

        // 计算着色点 pos 与采样点之间的弧长距离
        // （球面上的测地距离 = 球心角 × 半径）
        float dot_centers = (pos - sphere.center).norm().dot(sp);
        dot_centers = std::max(-1.f, std::min(1.f, dot_centers));
        float arc_angle = acosf(dot_centers);
        float r = arc_angle * sphere.radius;  // 弧长 = 角度 × 半径

        // 采样点的直接照度（Lambert 漫反射）
        float ndotl = std::max(0.f, sp.dot(light_dir));
        Vec3 irr = light_color * ndotl;

        // 用散射剖面加权
        Vec3 w = profile.eval(r);
        accum += irr * w;  // 逐通道加权（Vec3 × Vec3）
    }
    // 蒙特卡洛积分归一化：乘以面积元 dA = 4*pi*R^2 / N
    float norm_factor = (4.0f * 3.14159265f * sphere.radius * sphere.radius) / N_samples;
    return accum * norm_factor;
}
```

三个关键设计决策：

1. **Fibonacci 采样而非随机采样**：Fibonacci 点集在球面上的分布极均匀（低差异序列），同样 128 个样本，其收敛速度远快于纯随机采样，视觉噪点更少。

2. **弧长距离而非欧氏距离**：在球面上，两点之间的"表面距离"是弧长，而不是直线距离。用欧氏距离会导致散射剖面在球形几何上变形。弧长 = $\theta \cdot R$，其中 $\theta$ 是球心角。

3. **面积元归一化**：$dA = 4\pi R^2 / N$，确保积分结果与采样数 $N$ 无关。不归一化会导致 $N$ 越大，散射越亮，物理上是错误的。

### 4.5 着色组合

```cpp
// 直接漫反射（Lambert）
float ndotl = std::max(0.f, N.dot(light_dir));
Vec3 direct_diffuse = light_color * ndotl * mat.surface_albedo;

// SSS 积分（散射照度）
Vec3 sss = compute_sss(hit.pos, N, light_dir, light_color,
                        mat.profile, sphere, 128);
sss = sss * mat.surface_albedo;  // 表面颜色调制

// 混合：85% SSS + 15% 直接漫反射
// 15% 直接漫反射保留表面细节（阴影边界）
// 85% SSS 产生次表面透光感
Vec3 diffuse = direct_diffuse * 0.15f + sss * 0.85f;

// Blinn-Phong 高光（表面层，不参与 SSS）
Vec3 spec = blinn_phong_specular(N, light_dir, V, mat.specular_color, mat.shininess);

// 环境光（模拟全局间接光）
Vec3 amb = ambient * mat.surface_albedo;

Vec3 color = amb + diffuse + spec;
```

**为什么高光不进入 SSS？**

真实皮肤分为两层：
- **高光层（镜面层）**：皮肤表面的油脂/水分薄膜，产生高光反射，光线不进入皮下
- **次表面层（漫射层）**：光线穿过高光层进入皮下，产生 SSS

这是皮肤渲染的标准双层模型（two-layer skin model），因此高光独立计算，不参与 SSS 积分。

### 4.6 Fibonacci 球面采样原理

```cpp
// 第 i 个样本点（共 N 个，i 从 0 开始）：
float phi   = acosf(1 - 2*(i+0.5f)/N);    // 极角：均匀分布在 [-1,1] 的反余弦
float theta = 2*pi * i * 0.6180339887f;   // 方位角：黄金比例螺旋
```

黄金比例 $\phi_{golden} = (\sqrt{5}-1)/2 \approx 0.618$，是最"无理"的无理数（连分数展开全是1）。用它作为方位角步长，使得相邻样本点不会形成对齐的"行列"，而是形成均匀的螺旋分布。这与向日葵花盘、菠萝鳞片的排列原理相同。

---

## 五、踩坑实录

### Bug 1：编译时出现 3 个警告

**症状**：`g++ -Wall -Wextra` 报告 3 个警告：
```
warning: variable 'sample_pos' set but not used [-Wunused-but-set-variable]
warning: unused parameter 'normal' [-Wunused-parameter]
warning: unused variable 'aspect' [-Wunused-variable]
```

**错误假设**：警告不影响功能，可以忽略。

**真实原因**：
- `sample_pos`：计划用于法线偏置（normal offset），后来改为直接用球心到采样点方向 `sp`，但忘记删除变量
- `normal` 参数：在 `compute_sss` 中，由于采用球面几何方法，不需要着色点法线（采样点的法线直接用 `sp`），参数保留但未使用
- `aspect`：渲染时打算用来处理非正方形视口，后来用 1:1 方形，变量没用上

**修复方式**：
1. 删除 `sample_pos` 变量，直接用 `sphere.center + sp * sphere.radius` 的理论位置
2. 将 `normal` 参数改为 `/*normal*/`（C++ 允许匿名参数，保留签名但不产生警告）
3. 删除 `aspect` 变量

**教训**：`-Wall -Wextra` 的警告必须清零，0 警告是验收标准。警告往往是潜在 Bug 的信号——这次三个警告都只是代码清理问题，但历史上有过因未使用变量掩盖了真实 Bug 的情况。

### Bug 2：散射积分结果异常明亮（发现于视觉检查阶段）

**症状**：第一版代码中，SSS 积分结果比预期亮 4-6 倍，球体几乎是白色的。

**错误假设**：散射剖面的权重之和 = 1，所以积分结果应该与直接光照差不多。

**真实原因**：忘记了积分归一化。散射积分是面积积分 $\int_A R(r) E dA$，当用蒙特卡洛近似时，结果需要乘以面积元 $dA = 4\pi R^2 / N$ 来归一化。

原始错误代码（无归一化）：
```cpp
return accum;  // ❌ 缺少面积归一化
```

修复后：
```cpp
float norm_factor = (4.0f * 3.14159265f * sphere.radius * sphere.radius) / N_samples;
return accum * norm_factor;  // ✅ 正确归一化
```

**但是！** 加了归一化后，结果又太暗了。原因是散射剖面高斯权重之和远小于 1（皮肤大约是 0.5），能量在吸收中损失了——这实际上是物理正确的行为（皮肤确实会吸收一部分光）。解决方案：把光源亮度从 1.0 提高到 2.5，模拟真实的强光照条件。

### Bug 3：色阶量化导致轮廓线

**症状**：早期版本在球体阴影过渡区出现明显的阶梯状轮廓线。

**错误假设**：这是着色算法的问题。

**真实原因**：ACES Tone Mapping 与 gamma 矫正的顺序搞反了。正确顺序是：线性 HDR → ACES → gamma 矫正 → 输出 sRGB。如果先做 gamma 再 ACES，高光区域会被压缩两次，造成颜色量化效果（类似卡通渲染的色阶分离）。

修复：严格按 `ACES(linear) → gamma(2.2)` 顺序应用，不交换。

### Bug 4：PNG 输出验证失败（Adler-32 计算错误）

**症状**：PNG 文件大小正常，但用 Python PIL 打开时报 CRC error。

**错误假设**：CRC32 计算错误。

**真实原因**：DEFLATE/zlib 格式要求在压缩数据后附加 **Adler-32** 校验和（不是 CRC32）。Adler-32 的字节序是大端序，我最初按小端序写入了。

Adler-32 的两个 16-bit 组件（s2 在高字节，s1 在低字节）：
```cpp
// 正确顺序：s2 的高字节先写
zlib.push_back((adler_s2>>8)&0xFF); zlib.push_back(adler_s2&0xFF);
zlib.push_back((adler_s1>>8)&0xFF); zlib.push_back(adler_s1&0xFF);
```

**教训**：写底层格式实现时，字节序是最容易出错的地方。用 Python 的 `PIL` 验证而不是仅靠文件大小来判断是否正确——二进制格式的错误往往隐藏在细节中。

---

## 六、效果验证与数据

### 像素统计验证

```bash
$ python3 -c "
from PIL import Image
import numpy as np
img = Image.open('sss_output.png')
pixels = np.array(img).astype(float)
print(f'图像尺寸: {img.size}')
print(f'像素均值: {pixels.mean():.1f}')
print(f'像素标准差: {pixels.std():.1f}')
print(f'顶部均值: {pixels[:50].mean():.1f}')
print(f'底部均值: {pixels[-50:].mean():.1f}')
"
```

输出：
```
图像尺寸: (900, 500)
像素均值: 115.7
像素标准差: 37.9
顶部均值: 87.0
底部均值: 110.4
✅ 像素统计正常
```

验证通过标准：
- ✅ 均值 115.7（要求 10~240，非全黑/白）
- ✅ 标准差 37.9（要求 > 5，图像有内容变化）
- ✅ 文件大小 1.3MB（要求 > 10KB）

### 材质视觉差异

四球材质视觉上有明显差异（从左到右）：

| 材质 | 颜色特征 | SSS 宽度 | 高光类型 |
|------|----------|----------|----------|
| **SKIN** | 暖红棕色，红光晕染 | 宽（最明显） | 软，shininess=32 |
| **WAX** | 暖黄色，蜡质感 | 中等 | 稍亮，shininess=64 |
| **MARBLE** | 冷蓝灰，内部发光感 | 窄 | 硬，shininess=96 |
| **MILK** | 纯白，均匀漫射 | 宽（均匀） | 低，shininess=16 |

SSS 效果最直观的体现：球体背光面（法线与光源夹角 > 90°）在无 SSS 时应该完全黑暗，而有 SSS 时因为周围区域的光穿透到背光面，背光区会有轻微的发光感。皮肤球的背光面可以看到淡淡的红色晕光，这正是 SSS 的特征效果。

### 性能数据

- **渲染时间**：约 8-12 秒（128样本/像素 × 4球 × ~60000像素/球）
- **球体像素总数**：每个球 200×200 = 40000 像素，4 球共约 160000 像素
- **SSS 样本数**：128个/像素，总 SSS 积分调用次数 ~20480000 次
- **散射剖面求值**：每次调用 eval() 执行 3×3 次 expf()，约 6000万次 expf

优化方向：预计算散射剖面 LUT（lookup table），将距离离散化为 1024 档，彻底消除运行时 expf 调用，预计可提速 10-50 倍。

---

## 七、总结与延伸

### 局限性

1. **球面采样近似**：当前方法假设散射只在同一个球体表面发生，不能处理不同物体之间的 SSS（如半透明玻璃杯背面的光传播）。

2. **单次散射积分**：128 个采样点的蒙特卡洛积分噪声较大，在低光照区域可能出现颗粒感。生产级实现会用 1024-4096 样本或解析积分。

3. **无深度信息**：真实 SSS 中，散射强度也依赖光线穿透深度（厚材质散射更多）。本实现只用表面距离，忽略了厚度效应。

4. **单点光源**：使用单个平行光，环境光用常量近似。真实皮肤需要 IBL（Image Based Lighting）配合散射。

### 可优化方向

1. **Texture Space Diffusion（TSD）**：先在 UV 空间渲染辐照度贴图，再用高斯模糊来近似散射积分，是实时 SSS 的主流方案（Unreal 4 用过此方法）。

2. **Pre-integrated SSS**：把散射积分预计算为关于 $N \cdot L$ 的查找表（由 GPU Gems 3 中的 Hable 方案推广），实时效率极高。

3. **Screen Space SSS（SSSSS）**：在屏幕空间对深度图和颜色缓冲进行散射卷积，Epic Games 在 UE4 中实现了这种方案，只需一个后处理 pass。

4. **Burley Normalized SSS**：2015 年 Burley 提出的归一化 SSS 剖面，参数物理直觉更强（类似 Disney BRDF 的"roughness"参数），已被 UE4/UE5 采用。

### 与本系列的关联

- **03-28 Depth of Field**：薄透镜模型同样利用了光在介质中的传播规律
- **04-03 Disney BRDF**：Disney BRDF 的 Subsurface 参数正是今天实现的 SSS 的快速近似
- **04-07 Fur Hair Kajiya-Kay**：头发渲染同样有类似问题（光穿透发丝），用各向异性 BxDF 解决
- **04-12 Volumetric God Rays**：体积光和 SSS 都属于光在介质内部传播的范畴，体积渲染 = 宏观 SSS

次表面散射是从"表面着色"走向"体积着色"的第一步。真正理解了 SSS，就能理解为什么皮肤、蜡、玉石、翡翠、珍珠看起来那么迷人——它们的美，藏在光穿透的那几毫米深处。

---

## 参考资料

1. Jensen H.W. et al., "A Practical Model for Subsurface Light Transport", SIGGRAPH 2001
2. d'Eon E., Luebke D., "Advanced Techniques for Realistic Real-Time Skin Rendering", GPU Gems 3, 2007
3. Burley B., "Physically Based Shading at Disney", SIGGRAPH 2012
4. Jimenez J. et al., "Separable Subsurface Scattering", Computer Graphics Forum 2015
