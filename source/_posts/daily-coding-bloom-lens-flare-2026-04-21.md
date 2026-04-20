---
title: "每日编程实践: Physically Based Bloom & Lens Flare"
date: 2026-04-21 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 后处理
  - 光学效果
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-21-bloom-lens-flare/bloom_output.png
---

## 背景与动机

打开任何一款现代 3A 游戏或商业级渲染器，你都会看到两种无处不在的视觉效果：**泛光（Bloom）**和**镜头光晕（Lens Flare）**。它们乍看像是"装饰"，但实际上背后有严肃的物理基础——前者模拟相机传感器在曝光高亮区域时的光散射，后者模拟真实镜头组内部的多次反射与衍射。

没有 Bloom 的画面往往让人感觉"假"或"干燥"。原因在于：人眼对高亮度区域存在生理散射（视网膜光晕），相机胶片/传感器在高曝光下也会发生光渗（halation / blooming）。当我们渲染的场景里存在远比周围亮 10 倍乃至 100 倍的光源时，如果没有 Bloom，光源只是一个亮像素，缺乏"炽热感"。

**Lens Flare** 的动机同样来自物理。任何由多片透镜组成的光学系统（相机、望远镜、人眼）在面对强光源时都会产生：
- **鬼影（Ghosts）**：光在镜片间的内部反射，沿光轴对称出现
- **光晕（Halo）**：大气/镜片微粒散射形成的圆环
- **星芒（Starburst）**：光圈叶片边缘的衍射，产生辐射状光刺

这些效果在**电影摄影**中是刻意保留甚至增强的，因为它们传达了"通过相机/镜头看到的真实感"，也是区分照片与纯 CG 渲染的重要线索之一。

**工业界应用**：
- **虚幻引擎 5**：后处理 Volume 中的 Bloom 使用了 Convolution Bloom（卷积泛光）与 FFT 加速，Lens Flare 为完整材质驱动系统
- **Unity HDRP**：物理相机模型 + Multi-pass Bloom，支持 Dirt Mask（镜头灰尘）
- **寒霜引擎（DICE BF 系列）**：著名的"DICE Lens Flare"是实时游戏镜头光晕的标杆实现，使用 SpriteFlare + Ghost + Halo 系统
- **影视 VFX**：Nuke 的 Glow node、Adobe AE 的 Optical Flares 插件都是专门处理这类效果的工具

今天的实现聚焦于**软件光栅化 + CPU 后处理**，完整展示从 HDR 场景渲染到最终 LDR 输出的全流程，包括：
1. HDR 场景渲染（夜景 + 多彩光源 + 星空 + 地面反射）
2. 亮度阈值提取（Bright-Pass Filter）
3. 四级高斯模糊金字塔（Bloom 核心）
4. 镜头光晕：星芒 + 鬼影 + 光晕环
5. ACES Filmic 色调映射 + Gamma 校正

---

## 核心原理

### 1. HDR 与 Bloom 的本质联系

理解 Bloom 必须先理解 **HDR（High Dynamic Range）**。

在物理世界中，光的亮度范围极大：室外阳光约 100,000 lux，室内约 100~500 lux，相差三个数量级。人眼通过**自动曝光**适应这个范围，但显示器只能显示 [0, 255] 范围（8 bit LDR）。

HDR 渲染管线的核心思路：
1. 在线性空间中渲染，允许亮度值 > 1.0（比如太阳光源值可以是 100.0）
2. 后处理：应用色调映射把 HDR 压缩到 [0, 1]
3. Bloom 在色调映射**之前**提取高亮区域，这样光源的"超亮"信息就被 Bloom 编码为光晕

如果在 LDR 图像上做 Bloom，光源已经被截断为 255，丢失了"它到底有多亮"的信息，Bloom 效果会很假。

**Bloom 的物理对应**：相机传感器（CCD/CMOS）在强光照射下，光电子会溢出到相邻像素（blooming effect），产生扩散发光。胶片时代称为 halation（光晕），本质是乳胶层的光散射。

### 2. Bright-Pass 亮度阈值提取

Bloom 的第一步是**只保留场景中最亮的部分**。公式如下：

**感知亮度（Perceived Luminance）**：
$$L = 0.2126 \cdot R + 0.7152 \cdot G + 0.0722 \cdot B$$

这三个系数来自 BT.709 标准，对应人眼对 RGB 三原色的感知权重——绿色通道对亮度贡献最大（71.52%），因为人眼中的中锥细胞对绿光最敏感。

**阈值提取**：
$$\text{bright}(p) = p \cdot \text{clamp}\left(\frac{L - T}{T + \epsilon}, 0, 1\right)$$

其中 $T$ 是阈值（本实现取 1.2），$\epsilon$ 是平滑因子（取 0.5）。

这个公式的意义：当像素亮度 $L < T$ 时，输出为 0（完全屏蔽）；当 $L > T + \epsilon$ 时，输出接近原像素值；中间有一个平滑过渡区，避免边缘出现锯齿状截断。

为什么不直接用 `if (L > T) keep else discard`？因为硬截断会在 Bloom 边缘产生明显的"切割感"，平滑的 Shoulder 曲线才能产生自然的光晕过渡。

### 3. 多级高斯模糊金字塔（Bloom Core）

单次高斯模糊产生的光晕半径有限，而且计算成本随半径 $r$ 急剧增长（$O(r)$ 采样）。现代引擎的解法是**多分辨率模糊金字塔**：

```
原图 (512x512) → Bright-Pass → Level 0 (512x512, 小半径模糊)
                                     ↓ 降采样 2x
                              Level 1 (256x256, 模糊后上采样)
                                     ↓ 降采样 2x
                              Level 2 (128x128, 模糊后上采样)
                                     ↓ 降采样 2x
                              Level 3 (64x64,  模糊后上采样)
                                     ↓
                              加权合并
```

**为什么这样有效**？

低分辨率图像上做固定半径的高斯模糊，等价于在高分辨率上做**更大半径**的模糊。具体来说：
- Level 1（1/2分辨率）上 radius=8 的模糊 ≈ Level 0 上 radius=16 的模糊
- Level 2（1/4分辨率）上 radius=8 的模糊 ≈ Level 0 上 radius=32 的模糊
- Level 3（1/8分辨率）上 radius=8 的模糊 ≈ Level 0 上 radius=64 的模糊

**计算复杂度**：直接在 512x512 做 radius=64 的高斯模糊需要 $(512 \times 512) \times (128 \times 2)$ = 约 6700 万次采样；多级金字塔总采样量约 (512² + 256² + 128² + 64²) × 16 × 2 ≈ 110 万次，**节省约 60 倍**。

**加权合并**：各级别的权重决定了 Bloom 的"形状"——
- Level 0（高权重 0.40）：细腻的光源核心光晕
- Level 1（权重 0.30）：中等扩散
- Level 2（权重 0.20）：大范围柔和发光
- Level 3（权重 0.10）：远端微弱扩散

这四个权重加起来 = 1.0，保持能量守恒。

### 4. 高斯核数学

高斯函数：
$$G(x, \sigma) = \frac{1}{\sqrt{2\pi}\sigma} e^{-\frac{x^2}{2\sigma^2}}$$

离散实现时，通常用 **分离卷积（Separable Convolution）**：先水平方向做 1D 卷积，再垂直方向做 1D 卷积，等价于 2D 高斯卷积。

这利用了高斯函数的分离性：
$$G(x,y,\sigma) = G(x,\sigma) \cdot G(y,\sigma)$$

**计算量节省**：2D 卷积需要 $(2r+1)^2$ 次乘加；分离卷积只需 $2 \times (2r+1)$ 次，对于 radius=8 的核，节省约 $\frac{17^2}{2 \times 17} = 8.5$ 倍。

核的归一化：计算所有权重的总和 $S = \sum_i G(x_i, \sigma)$，每个权重除以 $S$，保证卷积后总能量不变（不增亮也不减暗）。

### 5. 镜头鬼影（Lens Ghosts）的几何原理

鬼影（Ghosts）是光在镜头组多个镜片之间来回反射形成的。关键几何特性：

**鬼影的对称轴**：镜头光轴（屏幕中心）。

设光源在屏幕坐标 $(s_x, s_y)$，屏幕中心 $(c_x, c_y)$，向量 $\mathbf{d} = (s_x - c_x, s_y - c_y)$。

第 $k$ 个鬼影位置：
$$\mathbf{g}_k = (c_x, c_y) - t_k \cdot \mathbf{d}$$

其中 $t_k$ 是鬼影相对于光源-中心向量的位置参数：
- $t_k < 0$：鬼影在光源同侧但更近
- $t_k = 0$：正好在中心点
- $0 < t_k < 1$：在光源和中心之间
- $t_k > 1$：在中心的另一侧（光源的对面方向）

**真实光学依据**：每个鬼影对应一条从光源出发、在不同镜片对之间反射的光路。反射次数越多，鬼影越靠近光轴（能量逐渐衰减）。不同鬼影的色彩差异来自镜片玻璃的色散：不同波长折射率略有不同，导致鬼影带有彩色边缘。

### 6. 星芒（Starburst）的物理：衍射

星芒由相机光圈叶片（aperture blades）边缘的**光衍射**产生。对于 N 片光圈叶片的相机，会产生 N 条（N 为偶数时）或 2N 条（N 为奇数时）星芒。

数学模型：傅里叶变换。理论上正确的模拟需要计算光圈形状的 2D FFT，本实现用**解析近似**：

$$\text{spike}(\theta) = \frac{1}{N}\sum_{i=0}^{N-1} \left|\cos\left(\theta - \frac{i\pi}{N}\right)\right|^{P}$$

其中 $P$ 是尖锐度指数（本实现取 40），$N$ 是光芒数量（取 6，对应 6 叶片光圈）。

这个公式的直觉：每对相对的叶片产生一个方向上的衍射脊，$\cos^P$ 函数创造了尖锐的角度响应。

**整体星芒强度**：
$$\text{starburst}(r, \theta) = e^{-r/R_0} \cdot (0.3 + 0.7 \cdot \text{spike}(\theta))$$

$e^{-r/R_0}$ 是径向衰减（距离越远越暗），$R_0$ 是特征半径（本实现取 14 像素）。

### 7. 光晕环（Halo）

光晕（Halo）是由大气中的水汽、冰晶或镜头表面微粒散射形成的圆形亮环，在 22° 太阳光晕中尤为明显。

模拟公式：
$$\text{halo}(r) = e^{-\frac{(r - R_{\text{halo}})^2}{2\sigma_{\text{halo}}^2}}$$

这是一个以 $R_{\text{halo}}$ 为中心、宽度 $\sigma_{\text{halo}}$ 的高斯环。本实现取 $R_{\text{halo}} = 60$，$\sigma_{\text{halo}} = 12$。

光晕颜色偏冷（轻微蓝白色）：物理上是由于瑞利散射（Rayleigh scattering）对短波长（蓝光）散射更强。

### 8. ACES Filmic 色调映射

ACES（Academy Color Encoding System）是电影工业标准色彩空间，其 Filmic 曲线被广泛用于游戏渲染（UE4/5 默认色调映射）。

精确版本很复杂，Krzysztof Narkowicz 提供了一个高精度近似：

$$f(x) = \frac{x(ax + b)}{x(cx + d) + e}$$

参数：$a=2.51, b=0.03, c=2.43, d=0.59, e=0.14$

**Filmic 曲线的特点**：
- **Toe（暗部）**：曲线略有上凸，使暗部更有层次感（比简单 gamma 更保留暗部细节）
- **Linear Section（中间调）**：接近线性，保持色彩准确
- **Shoulder（亮部）**：曲线向上弯曲并饱和，强光不会直接截断为纯白，而是"优雅地压缩"
- **输出范围**：始终在 [0, 1]，不需要额外的 clamp

对比 **Reinhard 色调映射** $f(x) = x/(1+x)$：
- Reinhard 在极亮区域会严重偏黄（R 通道先饱和），ACES 对三个通道处理一致
- ACES 的 Shoulder 更平，高光区域的色彩还原更准确

---

## 实现架构

### 整体数据流

```
renderScene()
    ├── 背景天空梯度
    ├── 随机星空（mt19937 + uniform_distribution）
    ├── 地平线 glow（高斯近似）
    ├── 地面反射（水平翻转光源位置）
    └── 光源圆盘（cubic falloff）
         │
         ▼
HDRBuffer (512x512, Vec3 float)
         │
         ├──→ brightPass() → Bright-pass Buffer
         │         ├──→ gaussianBlur(radius=8, σ=4) → Level 0
         │         ├──→ downsample2x → gaussianBlur → upsampleTo → Level 1
         │         ├──→ downsample2x×2 → gaussianBlur → upsampleTo → Level 2
         │         └──→ downsample2x×3 → gaussianBlur → upsampleTo → Level 3
         │                       │
         │                 加权合并 (0.4+0.3+0.2+0.1) → bloomBuffer
         │
         ├──→ computeLensFlare()
         │         ├──→ starburst (6-ray, P=40)
         │         ├──→ 7 ghosts (t = -0.25, 0.4, 0.65, 0.85, 1.1, 1.35, 1.6)
         │         └──→ halo ring (R=60, σ=12)
         │                       │
         │                 lensFlareBuffer
         │
         └── composite = scene + bloom×0.85 + lensFlare×0.55
                    │
              acesFilmic() per-pixel
                    │
              gammaCorrect(2.2)
                    │
              uint8_t RGB → PNG
```

### 关键数据结构

```cpp
struct HDRBuffer {
    int w, h;
    std::vector<Vec3> pixels;  // 线性存储，row-major
    
    Vec3 sample(float u, float v) const;  // 双线性插值
    void addScaled(const HDRBuffer& other, float scale);  // 加权合并
};
```

**设计决策**：
- 使用 `std::vector<Vec3>` 而非 `Vec3**`：连续内存，缓存友好，支持范围循环
- `pixels[y*w + x]` 行主序：与图像写入库（stb_image_write）的内存布局一致
- `sample()` 双线性插值：上采样时避免马赛克/块状感

**职责划分**：
- `renderScene()`：生成 HDR 场景（CPU "着色器"）
- `brightPass()`：纯像素变换，无邻域操作
- `gaussianBlur{H,V}()`：分离卷积，各向同性高斯
- `downsample2x()`：2×2 box filter 均值下采样
- `upsampleTo()`：双线性上采样到指定尺寸
- `computeBloom()`：组装多级模糊，加权合并
- `computeLensFlare()`：几何驱动的光学效果
- `main()`：流程控制 + 合成 + 色调映射 + 输出

---

## 关键代码解析

### 1. 光源圆盘渲染

```cpp
float lightDisc(float dx, float dy, float radius) {
    float d = std::sqrt(dx*dx + dy*dy);
    float t = clamp(1.f - d/radius, 0.f, 1.f);
    return t * t * t;  // 三次方衰减
}
```

**为什么用三次方而不是线性或高斯**？

- 线性 `t`：光源边缘有明显的硬边，不自然
- 高斯 `exp(-d²/σ²)`：衰减太"均匀"，中心不够突出
- 三次方 `t³`：产生漂亮的 S 形衰减——中心高亮突出，边缘快速衰落，符合 Lambertian + 逆平方衰减的组合视觉效果

光源 Intensity 取值（如 28.0 for 主光源）远大于 1.0，这是 HDR 的核心——允许超过显示范围的值，之后由色调映射决定如何压缩。

### 2. 高斯核生成与分离卷积

```cpp
std::vector<float> makeGaussianKernel(int radius, float sigma) {
    int size = 2*radius + 1;
    std::vector<float> k(size);
    float sum = 0;
    for (int i = 0; i < size; i++) {
        float x = (float)(i - radius);
        k[i] = std::exp(-x*x / (2*sigma*sigma));
        sum += k[i];
    }
    for (auto& v : k) v /= sum;  // 归一化！
    return k;
}
```

**容易犯错的地方**：
1. **忘记归一化**：未归一化的高斯核会改变图像整体亮度（累积权重 ≠ 1）
2. **边界处理**：使用 `clamp(x+k, 0, w-1)` 而非 wrap（不然边缘像素会"泄漏"到对面）
3. **sigma 与 radius 的关系**：经验规则 $r \approx 3\sigma$；本实现 radius=8, sigma=4 正好满足（3×4=12 > 8，覆盖 99.7% 的高斯分布）

**分离卷积的关键**：

```cpp
HDRBuffer gaussianBlur(const HDRBuffer& src, int radius, float sigma) {
    return gaussianBlurV(gaussianBlurH(src, radius, sigma), radius, sigma);
    //     先水平 ──────────────────────── 再垂直
}
```

先水平后垂直，或先垂直后水平，结果相同（高斯可分离）。但**不能同时在同一个 buffer 上修改**（in-place），因为垂直方向需要读取水平方向已经模糊过的结果，而不是原始像素。

### 3. Bright-Pass 提取

```cpp
HDRBuffer brightPass(const HDRBuffer& src, float threshold) {
    HDRBuffer dst(src.w, src.h);
    for (int i = 0; i < src.w * src.h; i++) {
        const Vec3& p = src.pixels[i];
        float lum = 0.2126f*p.x + 0.7152f*p.y + 0.0722f*p.z;
        float factor = clamp((lum - threshold) / (threshold + 0.5f), 0.f, 1.f);
        dst.pixels[i] = src.pixels[i] * factor;
    }
    return dst;
}
```

**关键设计**：
- `factor` 乘以原像素（而非白色）：保留颜色信息，Bloom 颜色与光源颜色一致
- 分母 `threshold + 0.5f`：控制过渡区宽度；较小的值 = 更锐利的阈值截断，较大的值 = 更柔和的过渡
- 使用感知亮度而非 RGB 均值：避免偏色光源（纯红色光源）被错误过滤

### 4. 双线性插值上采样

```cpp
Vec3 HDRBuffer::sample(float u, float v) const {
    float px = u * (w-1);
    float py = v * (h-1);
    int x0 = clamp((int)px, 0, w-2);
    int y0 = clamp((int)py, 0, h-2);
    float fx = px - x0;  // 小数部分 = 插值权重
    float fy = py - y0;
    Vec3 c00 = at(x0,   y0);
    Vec3 c10 = at(x0+1, y0);
    Vec3 c01 = at(x0,   y0+1);
    Vec3 c11 = at(x0+1, y0+1);
    return mix(mix(c00,c10,fx), mix(c01,c11,fx), fy);
    //     先插 x 方向 ──────────── 再插 y 方向
}
```

**为什么用 `w-2` 作为 x0 的上限**？

因为 `x0+1` 必须存在（最大为 `w-1`），所以 `x0` 最大为 `w-2`。如果用 `w-1` 作为上限，访问 `x0+1 = w` 就越界了。这是双线性插值非常容易出现的 off-by-one 错误。

**`mix` 函数**：`mix(a, b, t) = a*(1-t) + b*t`，标准线性插值。连续调用两次实现双线性：先在 X 方向插值两次（上下行），再在 Y 方向插值一次。

### 5. 镜头鬼影几何

```cpp
// 鬼影位置 = 屏幕中心 - t * (光源 - 中心)
// 即：相对于中心，向光源反方向偏移 t 倍距离
float gx = cx - g.t * dx;
float gy = cy - g.t * dy;
```

**t 参数的几何意义**（设光源在右上方 (dx>0, dy<0)）：
- `t=0.4`：鬼影在中心偏光源方向 40% 处（在中心和光源之间）
- `t=1.0`：鬼影正好在中心
- `t=1.35`：鬼影在中心另一侧，距离为 0.35×|d|

这正好对应真实镜头鬼影的分布规律——多个鬼影沿镜头光轴对称分布，有些在光源同侧，有些在中心对侧。

**七个鬼影的颜色设计**：
- 暖橙 (1.0, 0.7, 0.3)：模拟玻璃折射偏红端
- 冷蓝 (0.5, 0.8, 1.0)：模拟折射偏蓝端（色散效果）
- 紫洋红 (0.9, 0.4, 0.8)：中波段残留
- 翠绿 (0.3, 1.0, 0.6)：多层镀膜的绿色反射
- 不同颜色模拟了真实镜头组中各层镀膜对不同波长的选择性反射

### 6. 星芒实现

```cpp
float starburst(float dx, float dy, float r, int rays) {
    float angle = std::atan2(dy, dx);
    float dist  = std::sqrt(dx*dx + dy*dy);
    float base  = std::exp(-dist / r);  // 径向衰减
    float spike = 0.f;
    float pi    = 3.14159265f;
    for (int i = 0; i < rays; i++) {
        float a = (float)i * pi / (float)rays;
        float c = std::abs(std::cos(angle - a));
        spike += std::pow(c, 40.f);  // P=40 产生尖锐光刺
    }
    spike /= (float)rays;  // 归一化
    return base * (0.3f + 0.7f * spike);
}
```

**P=40 的选择**：
- P=2（平方）：非常宽的角度响应，几乎看不出星芒
- P=10：明显的星芒，但边缘模糊
- P=40：清晰的 6 条光刺，类似 f/8 小光圈镜头
- P=100：极细的光刺，类似衍射模拟

**`pi/rays` 而不是 `2*pi/rays`**：因为每条光刺有两个方向（正反），间隔 π，所以用 `i * π / N` 覆盖 [0, π] 就够了。

**`0.3 + 0.7 * spike`**：保留 30% 的基础圆形发光，70% 的光刺，让效果既有核心亮点又有星芒方向性。

### 7. ACES + Gamma 流水线

```cpp
Vec3 acesFilmic(Vec3 x) {
    float a = 2.51f, b = 0.03f, c = 2.43f, d = 0.59f, e = 0.14f;
    auto apply = [&](float v) {
        return clamp((v*(a*v+b)) / (v*(c*v+d)+e), 0.f, 1.f);
    };
    return {apply(x.x), apply(x.y), apply(x.z)};
}

Vec3 gammaCorrect(Vec3 col, float gamma = 2.2f) {
    float inv = 1.f / gamma;
    return {std::pow(std::max(0.f,col.x), inv), ...};
}
```

**为什么先 ACES 后 gamma？**

1. ACES 在线性光照空间中操作（物理正确）
2. Gamma 校正将线性空间转换为感知均匀空间（sRGB），用于显示器显示

如果反过来（先 gamma 后 ACES），ACES 曲线在非线性空间中会产生错误的颜色偏移。

**ACES 参数来源**：Krzysztof Narkowicz 的博文 "ACES Filmic Tone Mapping Curve"，通过曲线拟合得到这 5 个参数，在 [0, 10] 范围内与真实 ACES 变换误差 < 0.002。

---

## 踩坑实录

### Bug 1：`std::max` 三参数调用错误

**症状**：编译错误：
```
error: '__comp' cannot be used as a function
/usr/include/c++/12/bits/stl_algobase.h:303:17: error: '__comp' cannot be used as a function
```

**错误代码**：
```cpp
float d = std::max(std::abs(dx), std::max(std::abs(dy),
                   std::abs(dx * 0.5f + dy * 0.866f),
                   std::abs(dx * 0.5f - dy * 0.866f)));
```

**错误假设**：以为 `std::max` 可以接受多于 2 个参数，或者重载了 N 元版本。

**真实原因**：C++ 标准库的 `std::max(a, b)` 只接受 2 个参数（第三个参数是可选的比较器 `comp`，不是第三个值）。`std::max(a, b, c)` 中 `c` 被解析为比较函数，导致类型错误。

**修复方式**：
```cpp
float a1 = std::abs(dx);
float a2 = std::abs(dy);
float a3 = std::abs(dx * 0.5f + dy * 0.866f);
float a4 = std::abs(dx * 0.5f - dy * 0.866f);
float d = std::max(a1, std::max(a2, std::max(a3, a4)));
```

或者使用 initializer_list 版本：`std::max({a1, a2, a3, a4})`（C++11 起支持）。

### Bug 2：第三方库警告污染编译输出

**症状**：编译时出现大量 `stb_image_write.h: warning: missing initializer for member` 警告，干扰对 main.cpp 错误的判断。

**错误假设**：以为这些是 main.cpp 的问题，花时间排查自己的代码。

**真实原因**：`stb_image_write.h` 是第三方单头文件库，其内部用 `{0}` 初始化结构体，GCC -Wextra 对此产生警告（成员未显式初始化）。这不是我们代码的问题。

**修复方式**：使用 `#pragma GCC diagnostic` 抑制特定警告，仅针对该文件：

```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"
#pragma GCC diagnostic pop
```

`push/pop` 保证警告抑制**仅作用于 stb 头文件**，我们自己的代码继续享受完整的 -Wall -Wextra 保护。

**教训**：区分"我的代码警告"和"第三方库警告"。编译自己的代码时，第三方库警告通常应该通过 `-isystem` 或 `#pragma` 抑制，而不是降低自己代码的警告级别。

### Bug 3（设计阶段预防）：双线性插值越界

**潜在症状**：运行时访问越界，产生 segfault 或乱码像素。

**陷阱代码**：
```cpp
int x0 = clamp((int)px, 0, w-1);  // 错误！x0+1 可能越界
```

**正确代码**：
```cpp
int x0 = clamp((int)px, 0, w-2);  // x0+1 最大为 w-1
```

这个 bug 只有在像素坐标接近右边界（u→1.0）时触发，不会在单元测试中出现，容易遗漏。

### Bug 4（设计阶段预防）：地面反射坐标计算

地面反射的 Y 坐标是以地平线为对称轴的镜像：

```cpp
float reflY = 2.f * horizonY - lt.cy;
```

如果写成 `reflY = horizonY - (lt.cy - horizonY)` 也等价，但更容易出现符号错误。

写完后检查：若光源在 `cy = 0.78H`，地平线在 `horizonY = 0.82H`，则 `reflY = 2*0.82H - 0.78H = 0.86H`，位于地面中，正确。

---

## 效果验证与数据

### 量化验证结果

```
文件大小:  105K bloom_output.png (105,xxx 字节)
图像尺寸:  512×512 px
像素均值:  59.1  (范围 10~240 ✅)
像素标准差: 33.1  (> 5 ✅)
天空亮度均值 (前100行):    51.4
光源区域亮度均值 (360~420行): 103.8
坐标系: 天空暗 (51.4) < 光源区域亮 (103.8) ✅
```

**编译性能**：
- 编译耗时：< 2s（单文件，约 550 行 C++）
- 渲染耗时：70ms（512×512，含完整后处理链）
- 内存占用：约 12MB（4 个 512×512 HDR buffer × 3 float/pixel × 4 字节 = 12MB）

**Bloom 性能细分（估算）**：
- Bright-pass：512×512 × 9 ops ≈ 2.3M 运算
- Level 0 高斯（512x512, r=8）：2 × 512² × 17 ≈ 8.9M 采样
- Level 1 高斯（256x256, r=8）：2 × 256² × 17 ≈ 2.2M 采样
- Level 2+3：合计约 0.8M 采样
- 总计：约 14M 采样操作，70ms 对应约 200M 次浮点运算/秒（单线程 CPU，符合预期）

### 视觉效果分析

渲染图展示：

![Physically Based Bloom & Lens Flare 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-21-bloom-lens-flare/bloom_output.png)

**可见效果**：
- **星空背景**：200 颗随机星点，符合夜空密度
- **地平线辉光**：暖色（橙黄）散射，模拟大气霾层
- **光源发光**：5 个不同颜色/大小的光源，均有软边发光圆盘
- **地面水面反射**：光源以地平线为对称轴在地面产生镜像倒影
- **Bloom 泛光**：大范围柔和光晕，主光源（暖黄色）的 Bloom 扩散半径约 80 像素
- **星芒**：主光源附近可见 6 条辐射状光刺
- **镜头鬼影**：从主光源沿屏幕中心轴分布的 7 个彩色光斑（橙、蓝、紫、绿等色）
- **光晕环**：主光源周围隐约可见的白色圆环（半径约 60px）

**有无 Bloom 对比**：
- 无 Bloom：光源只是几个亮像素点，没有"发光感"，画面干燥、"假"
- 有 Bloom：光源具有强烈的辐射感，高亮区域自然向周围"溢出"，符合相机/人眼感知

---

## 总结与延伸

### 技术局限性

1. **CPU 软件渲染**：本实现全部在 CPU 上运行，无 GPU 加速。512×512 只需 70ms，但 1920×1080 会需要约 700ms（线性缩放），不适合实时应用
2. **Bloom 没有 Temporal 稳定**：帧间 Bloom 不稳定（抖动），实际游戏引擎会用 TAA 平滑历史帧
3. **镜头光晕没有 Occlusion**：真实镜头光晕在光源被遮挡时应该消失，本实现无遮挡检测
4. **无 Dirt Mask**：UE5/HDRP 支持镜头灰尘贴图，影响 Bloom 的不均匀分布
5. **光谱 Bloom**：波长依赖的 Bloom（蓝色 Bloom 扩散更广）未实现

### 可优化方向

1. **FFT Convolution Bloom**：用 FFT 卷积实现任意形状的 Bloom kernel（如心形光晕、蝴蝶散景），O(NlogN) 复杂度
2. **GPU 实现**：使用 compute shader，将 Bloom 降低到 < 1ms 的 pass 耗时
3. **Anamorphic Lens Flare**：宽幅电影镜头产生的水平拉伸光晕（如《星际迷航》重启版的蓝色横条光晕）
4. **物理遮挡**：Lens Flare 随光源遮挡程度动态淡出（需要遮挡查询或深度图）
5. **Bloom Dirt Texture**：用不规则纹理调制 Bloom 分布，模拟真实镜头污渍
6. **Eye Adaptation**：自动曝光（Automatic Exposure），根据场景平均亮度动态调整 Bloom 强度

### 与系列文章的关联

本文是每日编程实践系列的第 66 篇。

相关文章：
- **TAA（03-26）**：Bloom + TAA 是现代游戏后处理管线的"黄金搭档"，TAA 处理 Bloom 的时序抖动
- **HDR Tone Mapping（03-29）**：深入讲解了 ACES/Reinhard/Uncharted2 等色调映射曲线
- **Depth of Field（03-28）**：另一个基于镜头物理的后处理效果，薄透镜模型
- **Spectral Dispersion（04-13）**：色散与镜头光晕鬼影的彩色边缘有相同的物理根源（波长依赖折射率）
- **Volumetric Light（04-12）**：体积光 + Bloom 组合可以创造极具冲击力的丁达尔效应

### 代码与资源

完整代码：[GitHub - daily-coding-practice/2026/04/04-21-bloom-lens-flare](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-21-bloom-lens-flare)

核心文件：`main.cpp`（约 400 行，零外部依赖，仅需 stb_image_write.h）

```bash
# 编译运行
g++ main.cpp -o output -std=c++17 -O2
./output
# 输出: bloom_output.png (512x512)
```
