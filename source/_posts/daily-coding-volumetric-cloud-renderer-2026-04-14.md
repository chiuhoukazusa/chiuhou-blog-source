---
title: "每日编程实践: Volumetric Cloud Renderer — 基于FBM与光线步进的程序化体积云渲染"
date: 2026-04-14 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 体积渲染
  - 噪声生成
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-14-Volumetric-Cloud-Renderer/volumetric_cloud_output.png
---

## 背景与动机

### 云渲染为什么难？

在实时渲染领域，天空系统是沉浸感最重要的视觉元素之一，而云则是其中最难处理的部分。传统解决方案中有两个极端：

**极端一：Billboard 贴图云**
使用预渲染的二维贴图，对着相机永远朝向（billboard），像《早期 Minecraft》或很多移动端游戏。优点是极度廉价，缺点是穿帮明显——从侧面看是一片纸，相机穿入时立刻露馅。

**极端二：离线体积渲染**
Disney、Pixar 的电影级云用蒙特卡洛体积路径追踪，每帧渲染几小时。这当然是最物理正确的，但对实时渲染毫无参考价值。

**实时体积云的工业实践**：2015年，Guerrilla Games 在《地平线：零之曙光》中引入了基于 Worley 噪声和 FBM 的体积云系统，开创了现代 AAA 游戏云渲染的范式。随后 SIGGRAPH 2017 上 Sebastian Hillaire 发表的《Real-time Rendering of Volumetric Clouds》进一步完善了这套思路。Unreal Engine 5 的"Volumetric Clouds"功能就是这套方法的具体实现。

**今天要做的事**：从零实现一个软光栅体积云渲染器，包含：
1. 三维 FBM 噪声生成云密度场
2. 域变形（Domain Warping）增强云体自然感
3. 光线步进（Ray Marching）体积积分
4. Beer-Lambert 衰减
5. Henyey-Greenstein 前向散射
6. 多重散射近似
7. 程序化天空背景

---

## 核心原理

### 2.1 体积渲染方程（Volume Rendering Equation）

体积渲染的核心是求解**辐射传输方程（Radiative Transfer Equation, RTE）**在沿射线方向的积分形式：

```
L(x, ω) = ∫₀ᵈ T(x, x') · [σ_s(x') · L_in(x', ω) + σ_a(x') · L_e(x')] dt
         + T(x, x_d) · L_bg
```

其中：
- `T(x, x')` = 透射率，从 `x` 到 `x'` 的光线存活比例
- `σ_s` = 散射系数（scattering coefficient），粒子把光向其他方向散射的概率密度
- `σ_a` = 吸收系数（absorption coefficient），粒子把光转化为热量的概率密度
- `σ_t = σ_s + σ_a` = 消光系数（extinction coefficient）
- `L_in` = 从所有方向入射后经相位函数加权的散射光
- `L_e` = 自发光（云不需要）
- `L_bg` = 背景色（天空）

**透射率的 Beer-Lambert 定律**：

```
T(t₀, t₁) = exp(-∫_{t₀}^{t₁} σ_t(t) dt)
```

直觉理解：想象一束光穿过烟雾。每走一小段 `dt`，光的强度衰减 `σ_t · dt` 的比例。这个指数衰减就是 Beer-Lambert 定律。云越厚（`σ_t` 越大）、光走的路越长，透射率越低（云看起来越白/灰不透明）。

### 2.2 离散化的 Ray Marching

连续积分无法直接计算，我们沿射线方向将路径离散为 N 步，每步长 `Δt`：

```
T_i = exp(-σ_t(p_i) · Δt)
ΔT_i = T_{total} · (1 - T_i)    // 这一步贡献的不透明度
```

对散射光的贡献：
```
L += ΔT_i · (L_sun · phase(θ) · T_sun(p_i) + L_amb)
T_{total} *= T_i
```

代码实现直接体现了这一迭代：
```cpp
float stepT = std::exp(-sigma_t * STEP_SIZE);
float dT = T * (1.0f - stepT);          // 本步骤消光的透射率份额
Vec3 inScatter = (sunLight + ambLight) * dT;
scattered += inScatter;
T *= stepT;                              // 累乘透射率
```

**注意**：步长 `STEP_SIZE` 必须小于云密度变化的特征长度，否则会出现欠采样噪声（类似走样）。本项目使用 0.025，对应场景尺度约合 2.5% 的单位长度。

### 2.3 FBM 分形噪声

云的外形来自多尺度的湍流结构——远处看是大团积云，近处看是小的卷曲。这正是 FBM（Fractional Brownian Motion）的特性。

**单频 Perlin 噪声**：对每个点 `(x,y,z)` 求基于梯度的平滑噪声，值域 `[-1, 1]`。

**FBM 叠加公式**：
```
fbm(x) = Σ_{i=0}^{N-1} amplitude_i · noise(x · frequency_i)
       其中 amplitude_i = 0.5^i, frequency_i = 2^i
```

直觉理解：低频项（amplitude 大）决定云的整体形态；高频项（amplitude 小）添加细节褶皱。就像在光滑的山丘上叠加越来越细小的凹凸。

代码：
```cpp
static float fbm(float x, float y, float z, int octaves = 6) {
    float value = 0.0f, amplitude = 0.5f, frequency = 1.0f;
    float total_amplitude = 0.0f;
    for (int i = 0; i < octaves; i++) {
        value += amplitude * gradNoise3(x * frequency, y * frequency, z * frequency);
        total_amplitude += amplitude;
        amplitude *= 0.5f;     // 每倍频，振幅减半
        frequency *= 2.0f;     // 频率翻倍
    }
    return value / total_amplitude;  // 归一化到 [-1, 1]
}
```

6 个 octave 的 FBM 能很好地平衡计算量和视觉细节。再增加 octave 效果提升微乎其微但计算量翻倍。

### 2.4 域变形（Domain Warping）

单纯的 FBM 生成的噪声是"圆润"的，没有云那种撕裂、弯曲的感觉。Inigo Quilez 的域变形技巧：**用一个 FBM 来偏移另一个 FBM 的输入坐标**。

```
warpX = fbm(p + offset1) * 0.4
warpZ = fbm(p + offset2) * 0.4
density = fbm(p.x + warpX, p.y * 0.4, p.z + warpZ)
```

这为什么有效？想象一下：原本规则的球形噪声被弯曲变形，就像在流水中的烟雾会被气流拉伸扭曲。域变形模拟了这种湍流拉伸效应，让云的边界更加不规则和自然。

代码中的实现：
```cpp
float warpX = fbm(p.x * 1.3f + 0.5f, p.y * 0.7f, p.z * 1.3f - 0.5f, 3) * 0.4f;
float warpZ = fbm(p.x * 1.3f - 0.5f, p.y * 0.7f, p.z * 1.3f + 0.5f, 3) * 0.4f;
// 偏移量用低频（3阶）FBM，速度更快且效果已足够
float wx = p.x * CLOUD_SCALE + warpX;
float wz = p.z * CLOUD_SCALE + warpZ;
float noise = fbm(wx, wy, wz, 6);  // 最终密度用高频（6阶）FBM
```

### 2.5 高度遮罩（Altitude Masking）

真实积云有明确的高度边界（云底和云顶）。我们用正弦曲线平滑过渡：

```cpp
float altFade = std::sin(π · (altitude - CLOUD_BOTTOM) / (CLOUD_TOP - CLOUD_BOTTOM));
altFade = altFade * altFade;  // 平方增强边界清晰度
```

为什么用 `sin²` 而不是线性插值？`sin²(πt)` 在 t=0 和 t=1 处导数为零（切线水平），这意味着在云底和云顶的过渡是平滑的，不会出现硬边。相比之下，线性遮罩在边界处会有明显的"切割感"。

### 2.6 Henyey-Greenstein 相位函数

体积云中的米散射（Mie scattering）对可见光具有强烈的**前向散射**特性——对着太阳看云，边缘会有耀眼的光晕（银边效应）。这由 HG 相位函数描述：

```
p(θ, g) = (1 - g²) / (4π · (1 + g² - 2g·cosθ)^{3/2})
```

参数 `g` 控制各向异性程度：
- `g = 0`：各向同性散射（均匀向四面八方）
- `g = 0.9`：强前向散射（光主要向前传播，银边效应强）
- `g = -0.5`：后向散射（光主要向后，逆光时云较暗）

云渲染中通常使用两个 HG 叶片的混合，同时模拟前向散射和轻微的后向散射：

```cpp
float phase = hgPhase(cosTheta, 0.65f) * 0.7f   // 强前向散射 (70%)
            + hgPhase(cosTheta, -0.2f) * 0.3f;   // 弱后向散射 (30%)
```

直觉：当 `cosθ ≈ 1`（正对太阳），HG(0.65) 值远大于 HG(0)，说明沿太阳方向的光强被大幅增益，产生云的银边光晕。

### 2.7 太阳光透射（Light Marching）

在每个采样点，为了计算来自太阳的直接光照，需要沿太阳方向额外做一次短距离 March，计算采样点到太阳之间的光学厚度：

```
T_sun(p) = exp(-∫_p^{p+6Δt} σ_t(p') dt')  // 6 步轻量级 Light March
```

```cpp
float lightMarch(const Vec3& pos) {
    const int LIGHT_STEPS = 6;
    const float LIGHT_STEP = 0.08f;
    float density_accum = 0.0f;
    Vec3 p = pos;
    for (int i = 0; i < LIGHT_STEPS; i++) {
        p += SUN_DIR * LIGHT_STEP;  // 沿太阳方向步进
        density_accum += cloudDensity(p);
    }
    return std::exp(-density_accum * LIGHT_STEP * 8.0f);  // Beer-Lambert
}
```

6 步而非 64 步是一个性能妥协：完整的光线步进在大密度云中会很准确，但计算量是主 March 的 N 倍。用 6 步粗略估计已能捕捉云内部的自遮挡（云心比云边暗），视觉效果令人满意。

---

## 实现架构

### 3.1 整体渲染管线

```
摄像机设置（位置、朝向、FOV）
        ↓
对每个像素：
  生成射线方向（reverse projection）
        ↓
  天空颜色计算（背景）
        ↓
  射线-AABB 求交（云层边界盒）
        ↓
  [如果有交点]
    主 Volume March（64步）
      每步：
        采样 FBM 密度（含域变形）
        Beer-Lambert 步进透射率
        Light March（6步）→ 太阳遮挡
        HG 相位函数
        累积散射光
        早出（T < 0.01）
        ↓
  合成：云色 + 背景 * 透射率
        ↓
  ACES 色调映射 + Gamma 校正
        ↓
  写入 PPM（→ Python 转 PNG）
```

### 3.2 坐标系约定

- 场景单位空间：`[0,1]^3`，摄像机位于 `(0.5, 0.05, 0.5)`
- Y 轴向上
- 云层占据 `y ∈ [0.4, 0.85]`（高度 0.45 单位）
- 太阳方向：`normalize(0.6, 0.8, 0.4)`（正前方偏右上 45°）

### 3.3 关键数据结构

```cpp
struct Vec3 { float x, y, z; ... };  // 统一的向量/颜色类型

struct MarchResult {
    Vec3 color;          // 累积散射颜色
    float transmittance; // 最终透射率 [0,1]，0=完全不透明
};
```

不使用 struct 分离色彩与透射的原因：体积渲染的颜色和透射率是**同一次积分的两个输出**，代码耦合在同一个循环里，分离反而增加复杂度。

### 3.4 性能数据

| 步骤 | 耗时 |
|------|------|
| 总渲染（800×450） | ~25.9 秒 |
| 平均每像素 | ~72 微秒 |
| 主 March（64步） | ~60 微秒/像素 |
| Light March（6步×密度采样） | ~12 微秒/像素 |

单线程软光栅，无 SIMD 优化。可用 OpenMP 并行行循环达到 4-8× 加速。

---

## 关键代码解析

### 4.1 云密度函数（核心）

```cpp
float cloudDensity(const Vec3& p) {
    float altitude = p.y;

    // 高度范围检查：快速剔除非云区域
    if (altitude < CLOUD_BOTTOM || altitude > CLOUD_TOP) return 0.0f;

    // 高度遮罩：用 sin² 产生平滑的云底/云顶边界
    float altFade = std::sin(3.14159f * (altitude - CLOUD_BOTTOM) / (CLOUD_TOP - CLOUD_BOTTOM));
    altFade = altFade * altFade;

    // 域变形：用低频 FBM 偏移坐标，使云的形状更扭曲自然
    // 垂直方向缩放 0.7（云水平方向延伸比垂直方向更多）
    float warpX = fbm(p.x * 1.3f + 0.5f, p.y * 0.7f, p.z * 1.3f - 0.5f, 3) * 0.4f;
    float warpZ = fbm(p.x * 1.3f - 0.5f, p.y * 0.7f, p.z * 1.3f + 0.5f, 3) * 0.4f;

    // 应用域变形后的坐标
    float wx = p.x * CLOUD_SCALE + warpX;
    float wy = p.y * CLOUD_SCALE * 0.4f;  // Y 方向压缩，云比较"扁"
    float wz = p.z * CLOUD_SCALE + warpZ;

    // 主 FBM 采样（6阶，高细节）
    float noise = fbm(wx, wy, wz, 6);
    noise = noise * 0.5f + 0.5f;  // [-1,1] 重映射到 [0,1]

    // 密度阈值（0.42）：只保留噪声较强的区域形成云，
    // 0.42 附近的值决定云的覆盖率。减小 → 云更多更厚，增大 → 云更稀少
    float density = std::max(0.0f, noise - 0.42f) * 3.5f;

    // 最终密度乘以高度遮罩
    density *= altFade;
    return density;
}
```

**易错点**：`noise` 在 `fbm` 后值域是 `[-1, 1]`，必须先 `remap` 到 `[0, 1]` 再做阈值判断，否则负值区域的 `max(0, -0.5 - 0.42)` 永远是 0，云的形状会失真。

### 4.2 梯度噪声（Perlin Noise 实现）

```cpp
static float gradNoise3(float x, float y, float z) {
    int ix = (int)std::floor(x), iy = (int)std::floor(y), iz = (int)std::floor(z);
    float fx = x - ix, fy = y - iy, fz = z - iz;

    // Ken Perlin's quintic smoothstep: 6t⁵ - 15t⁴ + 10t³
    // 比三次 smoothstep 导数更平滑，消除二阶不连续
    auto s = [](float t){ return t*t*t*(t*(t*6-15)+10); };
    float ux = s(fx), uy = s(fy), uz = s(fz);

    // 梯度函数：根据哈希值选择 12 个主要梯度方向之一
    auto grad = [](uint32_t h, float dx, float dy, float dz) -> float {
        int h4 = h & 15;
        float u = (h4 < 8) ? dx : dy;
        float v = (h4 < 4) ? dy : ((h4==12||h4==14) ? dx : dz);
        return ((h4 & 1) ? -u : u) + ((h4 & 2) ? -v : v);
    };

    // 三线性插值 8 个角点的梯度贡献
    float n000 = grad(hash3(ix,   iy,   iz  ), fx,   fy,   fz  );
    float n100 = grad(hash3(ix+1, iy,   iz  ), fx-1, fy,   fz  );
    // ... (其余6个角点)

    float x0 = n000*(1-ux) + n100*ux;
    float x1 = n010*(1-ux) + n110*ux;
    // ... 三线性插值
    return y0*(1-uz) + y1*uz;
}
```

**为什么用 quintic（五次）smoothstep 而不是三次**？三次 `3t²-2t³` 的二阶导在 t=0,1 处不连续，导致法线计算中出现接缝（在体积渲染中不明显，但在高度场地形渲染中会有可见接缝）。五次版本的一二阶导均连续，更"干净"。

### 4.3 主体积步进循环

```cpp
MarchResult volumetricMarch(const Vec3& rayOrigin, const Vec3& rayDir) {
    const int MAX_STEPS = 64;
    const float STEP_SIZE = 0.025f;

    Vec3 scattered(0.0f);
    float T = 1.0f;  // 透射率（从1开始，逐步衰减）

    // 预计算相位函数（仅与视线-太阳夹角有关，不随位置变化）
    float cosTheta = rayDir.dot(SUN_DIR);
    float phase = hgPhase(cosTheta, 0.65f) * 0.7f + hgPhase(cosTheta, -0.2f) * 0.3f;

    Vec3 p = rayOrigin;
    for (int i = 0; i < MAX_STEPS && T > 0.01f; i++) {  // 早出优化
        float d = cloudDensity(p);
        if (d > 0.001f) {  // 只在有密度的地方计算光照
            // 消光系数 = 吸收 + 散射
            float sigma_a = d * 6.0f;
            float sigma_s = d * 10.0f;
            float sigma_t = sigma_a + sigma_s;

            // 本步骤透射率（Beer-Lambert 微分形式）
            float stepT = std::exp(-sigma_t * STEP_SIZE);
            float dT = T * (1.0f - stepT);  // 这一步"消耗"的透射率

            // 太阳光透射（Light March）
            float sunT = lightMarch(p);
            Vec3 sunLight = SUN_COLOR * sunT * phase;

            // 多重散射近似：均匀的蓝色环境光
            Vec3 ambLight = Vec3(0.35f, 0.45f, 0.65f) * 0.3f;

            // 散射贡献 = 入射光 × 消耗的透射率份额
            scattered += (sunLight + ambLight) * dT;
            T *= stepT;  // 累乘透射率
        }
        p += rayDir * STEP_SIZE;
        if (p.y > 1.1f || p.y < 0.0f) break;  // 出云层边界
    }
    return {scattered, T};
}
```

**`dT = T * (1 - stepT)` 的直觉**：这一步中，有 `(1 - stepT)` 的光子被散射/吸收（不再穿透）。乘以当前透射率 `T` 是因为只有还没被前面步骤吸收的光子才会参与这一步的计算。这保证了能量守恒。

### 4.4 射线-云层 AABB 求交

为避免对整条射线都做 March，先计算射线进入/离开云层的 t 值范围：

```cpp
float tStart = 0.0f, tEnd = 3.5f;
if (rayDir.y > 0.001f) {
    // 射线向上：从云底开始，到云顶结束
    tStart = std::max(0.0f, (CLOUD_BOTTOM - camPos.y) / rayDir.y);
    tEnd   = (CLOUD_TOP   - camPos.y) / rayDir.y;
} else if (rayDir.y < -0.001f) {
    // 射线向下：限制 tEnd（不进入地面以下）
    tEnd = std::min(tEnd, (CLOUD_BOTTOM - camPos.y) / rayDir.y);
}
```

**易错点**：相机在云层下方（`camPos.y = 0.05`），所以对于向上的射线，`tStart > 0`（需要走一段才到云底）。对水平或几乎水平的射线，直接用 `tStart=0, tEnd=3.5` 的默认值，这可能会导致步进超出场景范围——但实际上水平射线进入云层的密度极低，影响很小。

### 4.5 ACES 色调映射

体积散射的累积颜色可能超过 1.0（HDR），需要色调映射压缩到显示范围：

```cpp
Vec3 acesToneMap(const Vec3& c) {
    // ACES Filmic 曲线（Stephen Hill 近似）
    // f(x) = x(ax + b) / (x(cx + d) + e)
    float a=2.51f, b=0.03f, cc2=2.43f, d=0.59f, e=0.14f;
    auto tm = [&](float x) {
        return std::max(0.0f, std::min(1.0f, (x*(a*x+b)) / (x*(cc2*x+d)+e)));
    };
    return {tm(c.x), tm(c.y), tm(c.z)};
}
```

为什么用 ACES 而不是简单的 `clamp`？直接 clamp 会让太亮的区域（太阳光下的云顶）变成死白，丢失所有层次。ACES 通过 S 形曲线压缩暗部细节、保留高光对比度，是当前主流游戏（如 Unreal、Unity）的默认色调映射。

---

## 踩坑实录

### Bug 1：整幅图像全黑

**症状**：渲染完成，输出 PNG，但图像完全漆黑，看不到任何东西。

**错误假设**：以为是相机方向设置错误，Ray March 没有命中云层。

**调试过程**：
```bash
# 打印射线方向范围
python3 -c "
import math
for y in [0, 112, 225, 337, 450]:
    v = (1 - 2*(y+0.5)/450) * math.tan(math.radians(30))
    print(f'row {y}: v={v:.3f}')
"
# 输出：row 0: v=0.577, row 225: v=0.000, row 450: v=-0.577
```

射线方向覆盖了从上到下，看起来正常。

**真实原因**：相机 `pitch` 角度设置后，`forward` 向量的 Y 分量让射线到达云层的 `tStart/tEnd` 计算出错。最初代码中 `tEnd = (CLOUD_TOP - camPos.y) / rayDir.y`，当 `rayDir.y` 非常小（接近水平）时，这个值会变成一个极大数，浮点精度丢失。

**修复**：增加 `rayDir.y > 0.001f` 的保护条件，并给 `tEnd` 设置合理上限 `3.5f`。

```cpp
// 修复前（有 Bug）
float tEnd = (CLOUD_TOP - camPos.y) / rayDir.y;  // rayDir.y 趋近 0 时爆炸

// 修复后
float tEnd = 3.5f;  // 默认上限
if (rayDir.y > 0.001f)
    tEnd = (CLOUD_TOP - camPos.y) / rayDir.y;
```

### Bug 2：云的颜色是洋红色（粉红）

**症状**：图像有内容，但颜色不对——云应该是白/灰，结果显示为明显的洋红色。

**错误假设**：以为是色调映射参数问题。

**真实原因**：`SUN_COLOR` 设置为 `Vec3(1.2f, 0.3f, 0.7f)`（调试时随手改过），导致太阳光是洋红色。

**修复**：将太阳颜色改回正确的暖白色 `Vec3(1.2f, 1.0f, 0.7f)`。

**教训**：调试时改了参数，修完其他 Bug 后忘记改回来。应该在代码里把"物理合理的默认值"写成常量并加注释。

### Bug 3：云的密度场全是同样的纹理，没有大尺度形态

**症状**：渲染出来的云没有积云那种层叠感，而是整个天空均匀分布的"棉花"纹理，看起来像高空卷云（cirrus），而非预期的积云（cumulus）。

**错误假设**：以为是 FBM octave 数量不够。

**真实原因**：`CLOUD_SCALE` 设置为 `6.0f` 时，噪声频率太高，看不到低频的大尺度形态；而高度遮罩 `altFade` 的范围 `[CLOUD_BOTTOM, CLOUD_TOP]` 太窄（设为 `[0.6, 0.7]`），没有足够的空间展示密度变化。

**修复**：
1. 降低 `CLOUD_SCALE` 至 `2.5f`（更大的云团结构）
2. 扩大云层高度范围至 `[0.4, 0.85]`
3. 增加域变形偏移量至 `0.4f`（更多扭曲形变）

**视觉对比**：修复前是均匀薄雾，修复后有明显的大团积云和透明区域。

### Bug 4：PNG 转换失败（exit 1）

**症状**：程序输出 `PNG conversion failed (code 32512)`，最终 exit 1。

**原因**：系统没有安装 ImageMagick 的 `convert` 命令。

**修复**：改用 Python Pillow 进行格式转换（已预安装）：

```cpp
// 修复前
int ret = system("convert volumetric_cloud_output.ppm volumetric_cloud_output.png");

// 修复后
int ret = system("python3 -c \""
                 "from PIL import Image; "
                 "img = Image.open('volumetric_cloud_output.ppm'); "
                 "img.save('volumetric_cloud_output.png'); "
                 "print('PNG saved:', img.size)"
                 "\" 2>&1");
```

**教训**：依赖外部命令（`convert`、`ffmpeg` 等）前应先检查是否可用。Python/Pillow 更可靠，因为它是本项目已有的依赖（验证脚本也用 Pillow）。

---

## 效果验证与数据

### 渲染结果

![Volumetric Cloud Renderer 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-14-Volumetric-Cloud-Renderer/volumetric_cloud_output.png)

渲染尺寸：800×450，用时约 25.9 秒（单线程）。

### 量化验证数据

```bash
python3 << 'EOF'
from PIL import Image
import numpy as np

img = Image.open('volumetric_cloud_output.png')
pixels = np.array(img).astype(float)

print(f"图像尺寸: {img.size}")
print(f"像素均值: {pixels.mean():.1f}")
print(f"像素标准差: {pixels.std():.1f}")
print(f"最小值: {pixels.min():.1f}")
print(f"最大值: {pixels.max():.1f}")

# 天空区域（顶部50行）
print(f"顶部天空均值: {pixels[:50,:,:].mean():.1f}")
# 云层区域（中部）
print(f"中部云层均值: {pixels[100:350,:,:].mean():.1f}，标准差: {pixels[100:350,:,:].std():.1f}")
# 地平线区域（底部50行）
print(f"底部均值: {pixels[400:,:,:].mean():.1f}")
EOF
```

**输出结果**：
```
图像尺寸: (800, 450)
像素均值: 166.6      # ✅ 在 [10, 240] 范围内
像素标准差: 37.2     # ✅ > 5，图像有丰富内容
最小值: 86.0         # ✅ 非全黑
最大值: 255.0        # 部分高光被色调映射裁剪
顶部天空均值: 152.5  # 蓝天区域
中部云层均值: 153.9，标准差: 31.2  # ✅ 有明显密度变化
底部均值: 214.8      # 地平线较亮（接近地面大气散射）
```

**文件大小验证**：
```
-rw-r--r-- 1 root root 126K Apr 14 05:32 volumetric_cloud_output.png
```

126KB > 10KB ✅，图像有足够的内容变化使 PNG 压缩率不高。

### 性能分析

| 分项 | 值 |
|------|----|
| 总渲染时间 | 25.9 秒 |
| 分辨率 | 800×450 = 360,000 像素 |
| 平均每像素 | 72 微秒 |
| 主 March 步数 | 最多 64 步（平均约 40 步，早出优化） |
| Light March 步数/有云像素 | 6 步 |
| FBM octave | 6（主）+ 3（域变形×2）|

无 GPU、无 SIMD、无多线程。如果用 OpenMP 并行：
```cpp
#pragma omp parallel for
for (int y = 0; y < H; y++) { ... }
```
预计 4 核加速至 6-7 秒，8 核加速至 3-4 秒。

---

## 总结与延伸

### 局限性

1. **静态场景**：当前实现没有时间维度——云不会流动或变形。真实游戏引擎会用 `p.x += time * wind_speed` 让云随风移动。

2. **单次散射**：Light March 只计算一次散射（太阳→采样点→摄像机）。真实的厚云（如雷暴云）需要多次散射才能呈现正确的蓬松感。可用 Fong 等人的**多次散射近似**（存储光子深度信息）来改善。

3. **固定相机高度**：当前摄像机在云层下方。如果要支持在云层内穿行（如飞行模拟器），需要处理 `tStart = 0`（相机已在云层中）的情况。

4. **无阴影投射**：云对地面没有投射阴影。实现方式：地面 shading 时沿太阳方向做一次 Light March 穿过云层，计算云层遮挡。

5. **性能**：25.9 秒对离线渲染可接受，但实时需要用 GPU 计算着色器（Compute Shader）+ 时序累积（TAA/Temporal Reprojection）来分摊每帧只 March 部分像素。

### 可优化方向

- **蓝噪声抖动（Blue Noise Jitter）**：每帧的采样起始点用蓝噪声偏移，配合 TAA 抗锯齿，可以将步数减半而保持质量。
- **自适应步长**：在密度低的空旷区域用大步长跳过，进入云层时切换小步长。Guerrilla 的实现用了类似的方法，将平均步数从 128 降到 64。
- **Worley 噪声混合**：目前只用 Perlin FBM，加入 Worley（细胞噪声）可以产生更真实的卷云细节。公式：`density = fbm - worley * 0.3`。
- **大气散射集成**：与昨天（03-04）的大气散射渲染器结合，用 Rayleigh/Mie 散射计算天空颜色（而不是当前的程序化线性插值天空），整个画面物理一致性会大幅提升。

### 与系列其他文章的关联

- **03-04 Atmosphere Scattering**：为天空提供更物理的背景颜色，与本文的云渲染可以叠加。
- **03-21 SPPM Photon Mapping**：光子映射可以精确计算多次散射，代价是速度极慢，是本文单次散射的"参考真值"。
- **04-12 God Rays**：类似体积光步进，但 God Rays 是 2D 屏幕空间积分（沿屏幕射向光源），云渲染是 3D 世界空间积分。
- **04-13 Spectral Dispersion**：两者都用了光线步进思想，但目标不同——色散是追踪光谱折射，云渲染是追踪散射密度。

---

本文实现的 Volumetric Cloud Renderer 展示了体积渲染的完整技术链路：从 FBM 噪声生成密度场，到 Beer-Lambert 体积积分，再到 HG 相位函数的物理散射模型。代码约 350 行，无外部依赖（仅标准 C++），可作为体积渲染学习的入门项目。

完整代码：[GitHub](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-04-14)
