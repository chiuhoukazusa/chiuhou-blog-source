---
title: "每日编程实践: Subsurface Scattering Renderer — Jensen SSS 偶极子模型深度解析"
date: 2026-05-24 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 渲染
  - SSS
  - PBR
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-24-sss-renderer/sss_renderer_output.png
---

![SSS 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-24-sss-renderer/sss_renderer_output.png)

## ① 背景与动机

### 为什么需要 Subsurface Scattering？

在标准 Phong 或 Cook-Torrance 光照模型中，我们有一个根本假设：**光线只在物体表面发生交互**。光击中表面，要么被反射（镜面反射 + 漫反射），要么被吸收。然而，这个假设对于很多现实材质来说是**严重错误的**。

考虑你用手电筒照射自己的手：你会看到手指透出橙红色的光，背面也会有柔和的光晕。这是因为皮肤是**半透明材质** ——光线穿透皮肤表面，在皮下组织（脂肪、血管、骨骼）中不断散射，最终从另一个地方射出。这个过程叫做**次表面散射（Subsurface Scattering, SSS）**。

没有 SSS 的皮肤渲染有以下明显缺陷：
- **阴影边界过硬**：明暗交界线像刀割一样，没有皮肤的柔和过渡
- **暗部完全是死黑**：皮肤暗部应该有散射光泛起的橙红色调，没有 SSS 就是纯黑
- **背光效果缺失**：逆光拍摄时耳朵、鼻翼会透出红光，这在没有 SSS 的模型里完全消失
- **整体质感如同塑料**：皮肤特有的通透感完全丢失

**工业界应用场景**：
- **AAA 游戏角色皮肤**：Unreal Engine 5 的虚拟人类技术（MetaHuman）核心之一就是 SSS。《赛博朋克 2077》、《死亡搁浅》等游戏的角色皮肤质感都依赖 SSS
- **电影 VFX 人物**：工业光魔、Weta Digital 的数字演员渲染（如《阿凡达》、《魔戒》中的咕噜）使用离线 SSS 模拟
- **医疗可视化**：皮肤、器官的光学行为模拟需要精确的 SSS 模型
- **其他材质**：大理石雕塑的通透感、玉器的内部光晕、蜡烛的暖光穿透感、牛奶的浑浊均匀散射

本项目实现了 Jensen 2001 偶极子模型（CPU 软光栅），展示皮肤、大理石、蜡烛、牛奶四种典型 SSS 材质。

---

## ② 核心原理

### 2.1 光的传输：BSSRDF vs BRDF

标准 BRDF（Bidirectional Reflectance Distribution Function）描述的是：从方向 $\omega_i$ 射入某点 $\mathbf{x}$ 的光，有多少从 $\omega_o$ 方向反射出去。关键假设是**入射点 = 出射点**。

而 BSSRDF（Bidirectional Scattering Surface Reflectance Distribution Function）则允许**入射点 $\mathbf{x}_i$ ≠ 出射点 $\mathbf{x}_o$**：

$$L_o(\mathbf{x}_o, \omega_o) = \int_A \int_{2\pi} S(\mathbf{x}_i, \omega_i; \mathbf{x}_o, \omega_o) \cdot L_i(\mathbf{x}_i, \omega_i) \cdot (\hat{n} \cdot \omega_i) \, d\omega_i \, dA$$

其中 $S$ 是 BSSRDF 函数。这个八维积分在实时应用中无法直接计算，需要各种近似方法。

### 2.2 散射介质的物理参数

在散射介质（如皮肤）中，光的行为由以下参数描述：

**吸收系数 $\sigma_a$（Absorption Coefficient）**：
- 单位：$mm^{-1}$
- 光在介质中传播单位距离被吸收的概率密度
- 皮肤中血红蛋白、黑色素对特定波长有强吸收（这是肤色的来源）
- 直觉：$\sigma_a$ 越大，材质越不透明

**散射系数 $\sigma_s$（Scattering Coefficient）**：
- 光在介质中传播单位距离发生散射的概率密度
- 散射方向由相位函数 $p(\omega_i, \omega_o)$ 描述
- 皮肤中胶原纤维、细胞器引起强散射

**各向异性参数 $g$（Anisotropy）**：
- 取值 $[-1, 1]$：$g = 0$ 各向同性，$g > 0$ 前向散射，$g < 0$ 后向散射
- 皮肤的 $g \approx 0.8$（强前向散射）

**约化散射系数 $\sigma_s'$（Reduced Scattering Coefficient）**：
$$\sigma_s' = \sigma_s \cdot (1 - g)$$
这是将各向异性散射近似为等效各向同性散射后的系数。

**消光系数 $\sigma_t'$（Reduced Extinction Coefficient）**：
$$\sigma_t' = \sigma_a + \sigma_s'$$
综合了吸收和散射的总衰减。

**有效衰减系数 $\sigma_{tr}$（Effective Transport Coefficient）**：
$$\sigma_{tr} = \sqrt{3 \sigma_a \sigma_t'}$$

这个根号下的乘积来源于扩散方程的特征解。直觉上，$\sigma_{tr}$ 决定了散射光在介质内能扩散多远：$\sigma_{tr}$ 越小，光能扩散越远，SSS 范围越大（如牛奶）；$\sigma_{tr}$ 越大，光扩散越局限（如更不透明的材质）。

**平均自由程 $\ell_{mfp}$（Mean Free Path）**：
$$\ell_{mfp} = \frac{1}{\sigma_t'}$$

光在散射一次之前平均行进的距离。

### 2.3 扩散近似（Diffusion Approximation）

完整求解 RTE（辐射传输方程）计算量巨大。当材质的散射远大于吸收（$\sigma_s' \gg \sigma_a$）时，可以使用**扩散近似**：将复杂的方向性散射简化为各向同性的扩散过程，类似于热传导方程。

在扩散近似下，漫射出射函数 $R_d(r)$ 只与入射点和出射点的距离 $r$ 有关（而非各自的位置），这大大简化了计算。

### 2.4 Jensen 偶极子模型（Dipole Model）

Jensen et al. 2001 的关键贡献是用**偶极子（Dipole）**来解析求解扩散方程，得到 $R_d(r)$。

**边界条件问题**：
扩散方程在半无限介质（如皮肤）中有一个复杂的边界条件——在 $z = 0$ 的空气/介质界面，有部分光会被全内反射回来。Jensen 用 Fresnel 边界近似来处理这个问题。

**偶极子构造**：

在真实点光源（位于表面下 $z_r$ 处）的基础上，增加一个**虚拟点光源**（位于 $z = 0$ 以上 $z_v$ 处），使得边界条件（$z = 0$ 处通量为零）近似满足。这形成了正负光源的"偶极子"对。

$$z_r = \frac{1}{\sigma_t'}$$

$$z_v = z_r + 4AD$$

其中 $D = \frac{1}{3\sigma_t'}$ 是扩散系数，$A$ 是内部反射系数：

$$A = \frac{1 + F_{dr}}{1 - F_{dr}}$$

$F_{dr}$ 是菲涅尔漫反射项（Grosjean 近似）：

$$F_{dr} = -\frac{1.44}{\eta^2} + \frac{0.71}{\eta} + 0.668 + 0.0636\eta$$

直觉：$\eta > 1$ 时，$F_{dr} > 0$，$A > 1$，意味着更多光被内反射，$z_v$ 更大，SSS 范围更宽。

**漫射轮廓函数 $R_d(r)$**：

$$R_d(r) = \frac{\alpha'}{4\pi} \left[ z_r \frac{(\sigma_{tr} d_r + 1)e^{-\sigma_{tr} d_r}}{d_r^3} + z_v \frac{(\sigma_{tr} d_v + 1)e^{-\sigma_{tr} d_v}}{d_v^3} \right]$$

其中：
- $d_r = \sqrt{r^2 + z_r^2}$：出射点到真实光源的距离
- $d_v = \sqrt{r^2 + z_v^2}$：出射点到虚拟光源的距离
- $\alpha' = \sigma_s' / \sigma_t'$：约化反照率（reduced albedo）

**逐项理解**：
- $z_r(\cdots)/d_r^3$：真实光源的贡献，离光源越近贡献越大
- $z_v(\cdots)/d_v^3$：虚拟光源的贡献（为负，但此处由于符号约定已处理为正值在边界约束下叠加），修正边界条件
- $e^{-\sigma_{tr} d}$：随距离指数衰减，$\sigma_{tr}$ 决定衰减速率
- $(\sigma_{tr} d + 1)/d^3$：这是点扩散核（Green's function for diffusion equation）的一部分

### 2.5 通道独立计算（RGB 的意义）

SSS 的每个 RGB 通道有**不同的 $\sigma_a$、$\sigma_s'$**，因此 $R_d(r)$ 是按通道独立计算的。这正是 SSS 产生颜色变化的原因：

- 皮肤的红色通道 $\sigma_a$ 最小（血液吸收绿蓝光，红光穿透能力强）→ 红色通道 SSS 范围最大
- 这就是为什么逆光看手指会呈现红橙色：是红色 SSS 在蓝绿被吸收后的结果

### 2.6 预计算 LUT（性能优化）

$R_d(r)$ 的解析计算包含两次平方根和两次指数运算，实时计算每个像素代价较高。

**LUT 策略**：将 $R_d(r)$ 预计算为 256 点的 1D 查找表，$r$ 范围 $[0, r_{max}]$，渲染时线性插值采样。

这是 GPU Gems 3 中皮肤渲染的标准技巧，也是 UE5 等引擎中 Pre-Integrated Skin Shading 的基础思路。

---

## ③ 实现架构

### 3.1 整体数据流

```
材质定义 (σ_a, σ_s', η)
    ↓
预计算 DiffusionLUT (256点 Rd(r))
    ↓
光线追踪 (Sphere / Cylinder / Plane 求交)
    ↓
命中着色:
  ├── 直接光照 (Diffuse, NdotL)
  ├── SSS 半透明散射 (Wrap term + LUT采样 + 半透明阴影)
  ├── 镜面高光 (Phong specular)
  └── 菲涅尔边缘光 (Rim light)
    ↓
ACES Filmic 色调映射 + Gamma 校正
    ↓
输出 PPM → PIL 转 PNG
```

### 3.2 关键数据结构

**DipoleProfile**：封装偶极子参数计算，输入 $(\sigma_a, \sigma_s', \eta)$，输出 $R_d(r)$

```cpp
struct DipoleProfile {
    Vec3 sigma_a, sigma_sp;  // per-channel
    float eta;
    Vec3 computeRd(float r) const;
};
```

**DiffusionLUT**：预计算 256 点查找表，渲染时用线性插值

```cpp
struct DiffusionLUT {
    static constexpr int N = 256;
    static constexpr float MAX_R = 8.0f;  // 最大散射半径 (场景单位)
    Vec3 lut[N];
    Vec3 sample(float r) const;
};
```

**SSSMaterial**：每种材质持有 σ_a、σ_s'、η、sssStrength 等参数，以及表面反照率 albedo

**Scene**：持有几何体列表（球、柱、平面）、材质数组、LUT 数组、光源数组

### 3.3 渲染管线职责划分

| 组件 | 职责 |
|------|------|
| `DipoleProfile::computeRd` | 纯数学：解析计算偶极子漫射轮廓 |
| `DiffusionLUT::build` | 预计算：将 Rd(r) 离散化为 LUT |
| `Scene::intersectAll` | 几何求交：找最近命中点 |
| `computeSSS` | 着色：基于 LUT 采样计算 SSS 贡献 |
| `trace` | 主渲染函数：集成所有着色项 |
| `toneMapACES` | 后处理：HDR → SDR 映射 |

### 3.4 场景构成

- **皮肤球**（左）：$\sigma_a = (0.0032, 0.017, 0.048)$，$\sigma_s' = (0.74, 0.88, 1.01)$，$\eta = 1.3$
- **大理石球**（中）：$\sigma_a = (0.002, 0.002, 0.002)$（极低），$\sigma_s' = (2.19, 2.62, 3.00)$（强散射）
- **牛奶球**（右）：$\sigma_a = (0.0015, 0.0015, 0.0015)$，$\sigma_s' = (4.5, 5.5, 7.0)$（极强散射）
- **蜡烛（蜡圆柱+球帽）**（后方）：$\sigma_a = (0.008, 0.012, 0.020)$，有自发光 emissive
- **棋盘格地面**（平面）
- **4 盏光源**：暖色主光、冷色补光、蜡烛橙色点光、背光

---

## ④ 关键代码解析

### 4.1 偶极子 $R_d(r)$ 计算

```cpp
Vec3 DipoleProfile::computeRd(float r) const {
    // Step 1: 基本参数推导
    Vec3 sigma_t_prime = sigma_a + sigma_sp;
    Vec3 alpha_prime;
    alpha_prime.x = sigma_sp.x / (sigma_t_prime.x + 1e-6f);
    // 约化反照率 α' = σ_s' / σ_t'
    // 直觉：α' 接近1 → 以散射为主（高透射）；接近0 → 以吸收为主（不透明）

    // Step 2: 有效衰减系数
    Vec3 sigma_tr;
    sigma_tr.x = std::sqrt(3.f * sigma_a.x * sigma_t_prime.x);
    // σ_tr = sqrt(3 * σ_a * σ_t') 来自扩散方程特征值
    // 决定 SSS 的衰减尺度：σ_tr 小 → 光扩散远 → 宽 SSS（如牛奶）

    // Step 3: 真实光源深度
    Vec3 z_r;
    z_r.x = 1.f / (sigma_t_prime.x + 1e-6f);
    // z_r = 1/σ_t' 即平均自由程
    // 直觉：光穿入介质"平均深度"就是 z_r

    // Step 4: Fresnel 内反射修正
    float Fdr = -1.44f/(eta*eta) + 0.71f/eta + 0.668f + 0.0636f*eta;
    float A = (1.f + Fdr) / (1.f - Fdr);
    // A > 1 对应 η > 1（如皮肤 η=1.3）
    // A 决定虚拟光源比真实光源深多少

    // Step 5: 虚拟光源深度
    Vec3 D;
    D.x = 1.f/(3.f*sigma_t_prime.x + 1e-6f);  // 扩散系数 D = 1/(3σ_t')
    Vec3 z_v;
    z_v.x = z_r.x + 4.f*A*D.x;
    // z_v = z_r + 4AD：虚拟光源比真实光源深 4AD
    // 这个 4 来自扩散方程边界条件的精确推导（见 Jensen 2001 附录）

    // Step 6: 逐通道计算 Rd
    auto evalChannel = [&](float zr, float zv, float str, float alp) -> float {
        float dr = std::sqrt(r2 + zr*zr);  // 到真实光源距离
        float dv = std::sqrt(r2 + zv*zv);  // 到虚拟光源距离

        // 点扩散核：(σ_tr*d + 1) * exp(-σ_tr*d) / d^3
        // 这是 Green's function 中的径向衰减项
        // exp(-σ_tr*d) 是主要的指数衰减
        // (σ_tr*d + 1)/d^3 是补偿近场奇异性的修正
        float term_r = zr * (str * dr + 1.f) * std::exp(-str * dr) / (dr*dr*dr + 1e-12f);
        float term_v = zv * (str * dv + 1.f) * std::exp(-str * dv) / (dv*dv*dv + 1e-12f);

        // 真实光源正贡献 + 虚拟光源辅助修正边界条件
        float Rd = alp / (4.f * 3.14159265f) * (term_r + term_v);
        return std::max(0.f, Rd);
    };

    return Vec3{
        evalChannel(z_r.x, z_v.x, sigma_tr.x, alpha_prime.x),
        evalChannel(z_r.y, z_v.y, sigma_tr.y, alpha_prime.y),
        evalChannel(z_r.z, z_v.z, sigma_tr.z, alpha_prime.z)
    };
}
```

**关键点**：分母加 `1e-12f` 防止 $r \to 0$ 时的除零。物理上 $r = 0$ 时 $R_d$ 有理论最大值（不是无穷大），但离散实现需要数值保护。

### 4.2 LUT 构建与采样

```cpp
void DiffusionLUT::build(const DipoleProfile& p) {
    for(int i=0;i<N;i++){
        float r = (float)i/(N-1) * MAX_R + 0.001f;
        // 注意：从 0.001 开始而不是 0，避免 r=0 的奇异性
        lut[i] = p.computeRd(r);
    }
}

Vec3 DiffusionLUT::sample(float r) const {
    float t = clamp01(r / MAX_R) * (N-1);
    int lo = (int)t, hi = std::min(lo+1, N-1);
    float f = t - lo;
    return lerp3(lut[lo], lut[hi], f);
    // 线性插值确保连续性，256点对于 Rd 的平滑衰减足够精确
}
```

**MAX_R 的选取**：MAX_R = 8.0（场景单位）。对于皮肤参数，$1/\sigma_{tr} \approx 1/\sqrt{3 \times 0.003 \times 0.75} \approx 11$。选 8.0 覆盖了大部分有效散射范围，超出范围的贡献已接近零。

### 4.3 SSS 着色函数

```cpp
Vec3 computeSSS(const Scene& scene, const HitRecord& hit, ...) {
    for(const auto& light : scene.lights) {
        Vec3 L = (light.pos - hit.pos).norm();
        float atten = light.intensity / (dist*dist + 1.f);

        // 1. 直接漫反射（表面项）
        float NdotL = std::max(0.f, hit.normal.dot(L));
        Vec3 directDiff = mat.albedo * NdotL * light.color * atten;

        // 2. Wrap Term（包裹漫射）
        // 将 Lambert 阴影边界"软化"，模拟光线绕过物体的散射效果
        float wrapW = 0.3f;
        float wrapTerm = clamp01((hit.normal.dot(L) + wrapW) / (1.f + wrapW));
        wrapTerm = wrapTerm * wrapTerm;
        // wrapW 越大，阴影区越亮；平方使过渡更自然

        // 3. 从 LUT 采样 Rd(r)
        // r_sss = 0.5 * 平均自由程
        // 这是一个简化的"等效散射半径"
        float mfp = 1.f / (mat.sigma_a.x + mat.sigma_sp.x + 1e-4f);
        float r_sss = mfp * 0.5f;
        Vec3 Rd = lut.sample(r_sss);

        // 4. 半透明阴影：背面的散射光贡献
        // 利用 -NdotL 来估算背面接受的光照
        // 乘以 3.0f 加强 SSS 颜色可见性
        float backIrr = clamp01(-hit.normal.dot(L) * 0.5f + 0.5f);
        Vec3 transColor = mat.albedo * Rd * backIrr * light.color * atten * 3.0f;

        // 5. 阴影处理：SSS 材质在阴影中不是全黑（有散射透过）
        bool inShadow = scene.shadow(hit.pos, light.pos);
        float shadowFactor = inShadow ? 0.15f : 1.f;
        // 0.15 不是 0：即使在阴影中，SSS 也有微弱的散射光亮度

        totalSSS += directDiff * shadowFactor + transColor * mat.sssStrength;
    }
    return totalSSS;
}
```

### 4.4 菲涅尔边缘光

```cpp
// Rim light：利用观察方向与法线夹角模拟边缘 SSS 泛光
float fresnel = 1.f - std::max(0.f, (-viewDir).dot(hit.normal));
fresnel = std::pow(fresnel, 3.f);
Vec3 rimColor = mat.albedo * 0.4f * fresnel;
result += rimColor;
```

当观察方向接近切线方向（grazing angle），`(-viewDir).dot(normal)` 趋近 0，fresnel 趋近 1，产生边缘光晕。三次方让过渡更紧凑，只在接近边缘时才明显。这模拟了真实皮肤在边缘因大量散射而产生的半透明光晕效果。

### 4.5 蜡烛材质的自发光

```cpp
SSSMaterial makeWaxMaterial() {
    m.emissive = Vec3{0.8f, 0.4f, 0.05f};  // 橙黄色发光
    // 同时有 SSS（蜡质半透明）和自发光（模拟火焰热辐射）
    // 搭配蜡烛位置的 orange 点光源形成视觉联系
}
```

### 4.6 ACES Filmic 色调映射

```cpp
Vec3 toneMapACES(Vec3 x) {
    // Narkowicz 近似（原始 ACES 的简化版）
    const float a=2.51f, b=0.03f, c=2.43f, d=0.59f, e=0.14f;
    r.x = (x.x*(a*x.x+b))/(x.x*(c*x.x+d)+e);
    // 这个有理函数模拟了 S 形色调曲线：
    // - 暗部细节保留（斜率接近线性）
    // - 高光自然压缩（避免焦白）
    // - 中间调对比度略增
}
```

曝光乘数设为 0.7f（`toneMapACES(col * 0.7f)`）：这是在保留亮部细节和体现 SSS 颜色差异之间手动平衡的结果（初始用 1.5 过曝，0.9 还是偏亮，0.7 达到最佳视觉效果）。

---

## ⑤ 踩坑实录

### Bug 1：两个编译警告（unused parameter）

**症状**：`computeSSS` 函数签名有 `viewDir` 和 `numLightSamples` 参数，但函数体内未用到，导致 `-Wunused-parameter` 警告。

**错误假设**：以为 `-Wall -Wextra` 只报错误不报警告。

**真实原因**：`-Wall -Wextra` 包含 `-Wunused-parameter`，即使是函数参数未使用也会报警告，且我们的要求是 0 warnings。

**修复**：将参数名改为注释形式：

```cpp
// 修改前
Vec3 computeSSS(const Scene& scene, const HitRecord& hit,
                const Vec3& viewDir, int numLightSamples)

// 修改后
Vec3 computeSSS(const Scene& scene, const HitRecord& hit,
                const Vec3& /*viewDir*/, int /*numLightSamples*/)
```

C++ 允许在保留类型的同时注释掉参数名，这样既保持了函数签名兼容性，又消除了警告。

### Bug 2：渲染结果整体过曝（均值 229/255）

**症状**：第一次运行后像素均值达到 229，图像接近全白，虽然验证脚本通过（均值 < 250），但视觉上几乎看不到材质颜色差异，球体看起来都是白色。

**错误假设**：直接用 `col * 1.5f` 做曝光提升，以为多光源场景会偏暗需要提亮。

**真实原因**：场景有 4 盏光源（强度 40 + 15 + 5 + 20），再乘以 1.5 曝光，总辐照度过高，ACES 映射后全部压缩到高亮区域。材质的 SSS 颜色差异（皮肤的橙红、大理石的冷白、蜡烛的黄橙）全部丧失。

**修复步骤**：
1. 第一次调整：将光源强度从 40/15/5/20 降为 20/8/3/10，曝光乘数从 1.5 降为 0.9 → 均值 201，仍偏高
2. 第二次调整：ambient 从 0.08 降为 0.04，曝光乘数降为 0.7 → 均值 184.8，效果合适

**教训**：多光源场景要从低曝光开始调试，逐步提亮，而不是从高曝光开始压暗。每次调整后必须用 PIL 检查像素统计，不能依赖肉眼估计。

### Bug 3：ImageMagick `convert` 命令不可用

**症状**：代码内嵌了 `system("convert sss_output.ppm sss_renderer_output.png")` 转换命令，运行时 `system()` 返回非零，PNG 文件未生成。

**错误假设**：以为 ImageMagick 在所有 Linux 环境中都预装。

**真实原因**：当前容器环境（AnyDev）未安装 ImageMagick，但安装了 Python 的 PIL/Pillow 库。

**修复**：改为 Python PIL 转换：

```python
from PIL import Image
Image.open('sss_output.ppm').save('sss_renderer_output.png')
```

**教训**：不能假设工具的可用性，应优先使用已知可用的工具链（Python PIL 在此环境已验证可用）。

### Bug 4：SSS 效果不够明显（无法区分材质）

**症状**：虽然图像不全白，但三个球看起来颜色和质感几乎相同，看不出皮肤/大理石/牛奶的区别。

**错误假设**：LUT 采样的 $r$ 值使用固定的平均自由程，以为材质参数差异会自动体现在颜色上。

**真实原因**：各材质的 sssStrength 差异不够大，且 transColor 乘数为 1.0f 时 SSS 贡献太弱，被表面直接光照淹没。

**修复**：将 `transColor` 乘以 3.0f 加强散射颜色：

```cpp
Vec3 transColor = mat.albedo * Rd * backIrr * light.color * atten * 3.0f;
```

这将 SSS 贡献提升 3 倍，使材质的颜色通道差异（特别是皮肤的红色 SSS vs. 牛奶的白色强散射）更明显。

**教训**：SSS 实现中需要对各个贡献项的绝对量级有直觉：直接光照通常是 SSS 的 5-10 倍，需要显式放大 SSS 才能在视觉上体现出来。

---

## ⑥ 效果验证与数据

### 像素统计验证

```
文件: sss_renderer_output.png
尺寸: 800 × 600 像素
文件大小: 48021 bytes (46.9 KB > 10 KB ✅)
像素均值: 184.8 / 255 (在 [10, 240] 范围内 ✅)
像素标准差: 19.3 (> 5 ✅，图像有内容变化)
```

标准差 19.3 表明图像有足够的亮度变化，场景中的材质差异和阴影层次可见。

### 渲染性能

```
总像素数: 800 × 600 × 3×3(AA) = 4,320,000 次着色调用
渲染时间: 1.133s (wall clock)
吞吐量: ~3,815,534 pixels/s
光线追踪: 最大深度 2，无递归反射（只有直接着色）
```

CPU 软光栅实现，无 GPU 加速。主要瓶颈在每像素的 4 个阴影测试（shadow ray）。

### 材质参数对比

| 材质 | σ_a (R,G,B) | σ_s' (R,G,B) | MFP (R) | SSS 宽度 |
|------|-----------|------------|---------|---------|
| 皮肤 | 0.0032, 0.017, 0.048 | 0.74, 0.88, 1.01 | ~1.3 | 中等，颜色分离强 |
| 大理石 | 0.002, 0.002, 0.002 | 2.19, 2.62, 3.00 | ~0.45 | 窄，几乎白色 |
| 蜡 | 0.008, 0.012, 0.020 | 0.50, 0.45, 0.40 | ~1.9 | 宽，黄调 |
| 牛奶 | 0.0015, 0.0015, 0.0015 | 4.50, 5.50, 7.00 | ~0.22 | 最窄，均匀散射 |

皮肤的 R 通道 $\sigma_a$ 是 B 通道的 1/15，这导致红色散射距离远超蓝色，产生红橙色的 SSS 效果。

### 视觉效果确认

图像分析（AI 视觉评估）：
- 左侧球体（皮肤）：可见柔和的漫射过渡，暗部非全黑，有温暖色调的散射光晕
- 中间球体（大理石）：偏冷白色，通透感强，半透明阴影明显
- 右侧球体（牛奶）：高度均匀散射，几乎无颜色差异，质感类似不透明白色
- 蜡烛柱（背景）：可见暖橙色自发光，SSS 使蜡柱有蜡质通透感

---

## ⑦ 总结与延伸

### 技术局限性

**当前实现的简化**：

1. **固定散射半径**：使用 `r = 0.5 * MFP` 的固定值采样 LUT，而不是对整个表面积分 $\int R_d(|\mathbf{x}_i - \mathbf{x}_o|) \, dA$。真实 SSS 应该考虑每个表面点对每个出射点的贡献（计算量大得多）

2. **Wrap Term 近似**：我们用包裹漫射（Wrap Diffuse）近似背光 SSS 效果，而不是真正的透光追踪（光线穿过几何体从另一侧射出）

3. **忽略相位函数细节**：直接使用约化参数（已将各向异性折叠进 $\sigma_s'$），没有显式的 Henyey-Greenstein 相函数评估

4. **无多散射层模型**：皮肤实际上有表皮（Epidermis）、真皮（Dermis）等多层结构，每层参数不同。真实皮肤 SSS 使用多层模型（如 d'Eon 2007 的多极扩展）

### 可优化方向

**实时渲染方向**：
- **Pre-Integrated Skin Shading（Penner 2010）**：将 SSS 与曲率信息结合，预计算 2D LUT $\text{Rd}(curvature, NdotL)$，GPU 实时采样，被 UE4/5 采用
- **Screen-Space SSS（SSSSS）**：在屏幕空间对已渲染的像素做高斯模糊，模拟散射扩散效果，是实时引擎最常用的方案
- **Separable SSS（Jimenez 2015）**：将 2D 高斯分解为 1D，GPU 着色器实现，成为游戏引擎标准方案

**离线渲染方向**：
- **路径追踪 SSS（BSSRDF 直接采样）**：使用 Volumetric Path Tracing，光线在介质内随机游走，最精确但最慢
- **Photon Mapping SSS**：光子散射模拟，与本系列 05-07 的光子映射结合

### 与本系列其他文章的关联

- **05-07 Photon Mapping**：光子映射可以自然扩展到 SSS，光子在介质内随机游走即可实现精确 SSS
- **05-06 IBL PBR**：IBL 中的 BRDF LUT 与此处的 Diffusion LUT 有相似的预计算思路
- **05-19 Atmospheric Scattering**：大气散射（Rayleigh/Mie）与 SSS 同属粒子散射，相位函数是共同的数学工具
- **05-22 SSGI**：Screen-Space SSS 与 SSGI 都依赖屏幕空间后处理，GPU 实现框架相似

### 参考资料

1. H. W. Jensen, S. R. Marschner, M. Levoy, P. Hanrahan. "A Practical Model for Subsurface Light Transport." SIGGRAPH 2001
2. E. d'Eon, D. Luebke. "Advanced Techniques for Realistic Real-Time Skin Rendering." GPU Gems 3, 2007
3. J. Jimenez, K. Zsolnai-Fehér et al. "Separable Subsurface Scattering." Eurographics 2015
4. N. Penner. "Pre-Integrated Skin Shading." GPU Pro 2, 2010
5. C. Donner, H. W. Jensen. "Light Diffusion in Multi-Layered Translucent Materials." SIGGRAPH 2005

**代码仓库**：[GitHub - daily-coding-practice - 05-24-sss-renderer](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-24-sss-renderer)
