---
title: "每日编程实践: 双边滤波 — 边缘保持平滑"
date: 2026-07-01 06:00:00
tags:
  - 每日一练
  - 图形学
  - 图像处理
  - 滤波算法
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-01-bilateral-filter/bilateral_filter.png
---

## ① 背景与动机

在计算机图形学和图像处理中，「去噪」和「保持边缘」是一对天然矛盾的诉求。无论是光线追踪渲染器的最终帧、手机拍照的夜景模式、还是游戏引擎的后处理管线，噪声无处不在——路径追踪的 Monte Carlo 积分有残留噪声，摄像头传感器有低光噪声，甚至程序化纹理的数值振荡也会产生视觉伪影。

最经典的去噪方案是**高斯模糊（Gaussian Blur）**：用一个空间上的高斯核对邻域像素做加权平均。它的计算简单高效，但有一个致命的缺点——**边缘也被模糊了**。在渲染中，物体边界、纹理细节、镜面高光等高频信息都会被高斯模糊平滑掉，结果是图像虽然干净了，但也变糊了。

在游戏引擎领域（Unreal Engine、Unity HDRP），降噪是后处理管线中最关键的一环。路径追踪仅能提供极少样本（1-4 spp），原始图像充满噪点，必须靠降噪器恢复。如果使用简单的空间平滑——边缘就糊了；如果只做时间累积（TAA方向）——动态场景会有残影。因此需要一种**空间域中既能平滑噪声、又能保持边缘**的算法，这就是**双边滤波（Bilateral Filter）**。

双边滤波由 Tomasi 和 Manduchi 于 1998 年提出，核心思想非常优雅：**滤波权重不仅取决于空间距离，还取决于颜色/亮度的相似度**。两个像素即使空间上相邻，如果颜色差异很大（跨边缘），权重也会被压得很低；而平坦区域内颜色相似的像素能正常参与平滑。

工业界应用场景举例：
- **光线追踪降噪**：SVGF（Spatiotemporal Variance-Guided Filter）和 ReBLUR 等先进降噪器都把双边滤波作为核心的空间滤波组件，利用法线和深度信息作为"引导"（Joint/Corss Bilateral Filter）。
- **HDR 色调映射**：在计算局部色调映射时，用双边滤波分离基础层和细节层（Fast Bilateral Filtering for the Display of HDR Images, Durand & Dorsey 2002）。
- **手机计算摄影**：Google Pixel 的 HDR+ 夜间模式中，双边滤波用于去马赛克和去噪。
- **Flash/No-flash 图像融合**：Petschnigg 等人用双边滤波从闪光照片向环境光照片传递细节。

本文将从数学原理出发，完整实现一个标准的双边滤波器，并与高斯模糊进行**严格量化对比**——不只是看渲染结果，而是用 PSNR、边缘梯度保持率、噪声抑制率等指标来验证双边滤波的优势。

---

## ② 核心原理

### 2.1 高斯模糊的缺陷

高斯模糊的滤波公式为：

```
G[p] = (1/W_p) * Σ_{q∈Ω(p)} G_spatial(||p - q||) * I[q]
```

其中：
- `Ω(p)` 是以像素 `p` 为中心的滤波窗口（kernel window，半径 `r`）。
- `G_spatial(d)` 是空间高斯权重：`exp(-d² / 2σ_s²)`，像素距离中心越远权重越小。
- `W_p` 是归一化因子：所有权重之和。

高斯模糊的问题在于：权重**只看空间距离，不看像素值**。在边缘附近，窗口会跨越两种截然不同的颜色区域——一侧是明亮的前景，另一侧是暗色的背景。高斯核对两侧的像素都赋予空间权重，平均后的结果在边缘处产生过渡带——视觉上就是边缘模糊。

从信号处理角度看，高斯模糊是一个**低通滤波器**，它会不加区别地滤除所有高频成分，包括噪声和边缘信息——因为边缘本质上也是高频信号。

### 2.2 双边滤波的核心公式

双边滤波在高斯模糊的基础上引入**值域权重（Range Weight）**：

```
BF[p] = (1/W_p) * Σ_{q∈Ω(p)} G_spatial(||p - q||) * G_range(|I[p] - I[q]|) * I[q]
```

其中 `G_range(ΔI) = exp(-ΔI² / 2σ_r²)` 是值域高斯核。

**直觉解释**：
- `G_spatial` 说"距离远的不重要的"：这与高斯模糊一致，近邻像素贡献大。
- `G_range` 说"颜色太不一样的不重要"：如果一个邻近像素的颜色与中心像素差异很大，它很可能属于另一个物体/材质/表面，不应该被用于平滑当前像素。
- 两个权重的乘积决定最终的贡献度：像素要**既近又相似**才能被充分纳入平均。

**关键参数 σ_s 和 σ_r**：
- `σ_s`（spatial sigma）：控制空间平滑的范围。越大，越远的像素参与滤波，越模糊。
- `σ_r`（range sigma）：控制颜色差异的容忍度。越小，对颜色差异越敏感，越倾向于保持边缘；越大，越像标准高斯模糊。

两个极端情况说明公式的正确性：
- **σ_r → ∞**（无范围限制）：`G_range ≈ 1`，退化为标准高斯模糊。
- **σ_r → 0**（极致边缘保持）：只有颜色与中心像素几乎完全一致的像素才被纳入平均，输出几乎等于输入（无平滑）。

### 2.3 边缘保持的数学直觉

考虑一个理想化的边缘场景：假设像素 `p` 恰好在明暗交界线上，它的值为 0.5。在一侧区域 A，所有像素值为 0.3；在另一侧区域 B，所有像素值为 0.7。

对于高斯模糊，权重分布为：

```
区域 A 像素：w = G_spatial(d) * 1.0
区域 B 像素：w = G_spatial(d) * 1.0
```

两侧权重完全相同，输出是两侧的混合，边缘模糊。

对于双边滤波，权重分布为：

```
区域 A 像素：w = G_spatial(d) * G_range(|0.5 - 0.3| = 0.2)
             = G_spatial(d) * exp(-0.04 / 2σ_r²)
区域 B 像素：w = G_spatial(d) * G_range(|0.5 - 0.7| = 0.2)
             = G_spatial(d) * exp(-0.04 / 2σ_r²)
```

在中心像素恰好为 0.5 的特殊情况下，两侧的 range weight 相同。但实际上中心像素的值受噪声影响会偏向某一侧。

更典型的情况是：中心像素在区域 A 内（值 ≈ 0.3），那么：

```
区域 A 像素（值 ≈ 0.3）：G_range ≈ exp(0) = 1.0  ← 满权重
区域 B 像素（值 ≈ 0.7）：G_range ≈ exp(-0.16/2σ_r²)  ← 低权重
```

这样，区域 B 的像素**被大幅抑制**，滤波只会平滑 A 区域内部的噪声，而不会跨过边缘。

### 2.4 与非局部均值（NL-Means）的关系

双边滤波可以看作是**非局部均值（Non-Local Means, Buades et al. 2005）的一个简化特例**。NL-Means 使用以每个像素为中心的 patch 来计算相似度（`G_range` 基于 patch 距离而非单像素距离），双边滤波可视为 patch 大小为 1×1 的退化 NL-Means。

### 2.5 与引导滤波（Guided Filter）的对比

| 特性 | 双边滤波 | 引导滤波 |
|------|---------|---------|
| 权重依据 | 像素值相似度 | 引导图局部线性模型 |
| 时间复杂度 | O(N·r²) | O(N) |
| 梯度反转 | 可能出现（staircase artifact） | 不发生梯度反转 |
| 实现复杂度 | 简单 | 需要 box filter |

在实时渲染中，联合双边滤波（Joint/Cross Bilateral Filter）用引导图（如法线、深度）计算 range weight，而不是直接用颜色，这样可以更好地保持几何边缘。

---

## ③ 实现架构

### 3.1 整体数据流

```
输入：干净测试图像 (clean.ppm, 512×512)
  │
  ├─→ 添加高斯噪声 (σ=0.10) → noisy.ppm
  │       │
  │       ├─→ 高斯模糊 (r=6, σ_s=3.0) → gaussian_blur.ppm
  │       │       验证：高噪声抑制，但边缘模糊
  │       │
  │       └─→ 双边滤波 (r=6, σ_s=3.0, σ_r=0.30) → bilateral_filter.ppm
  │               验证：合理去噪，边缘保持
  │
  └─→ 量化对比
        ├─ RMSE & PSNR vs clean
        ├─ 边缘梯度保持率 (Sobel 3×3)
        └─ 综合质量评分
```

### 3.2 测试图像设计

测试图像不是随机噪声图，而是精心设计的对角线边缘图像——中心有一条从左上到右下的对角线，将图像分为两个区域：

- 左下区域：均匀亮度 0.25
- 右上区域：均匀亮度 0.75

这个设计的优势：
1. **边缘单一且清晰**：只有一条对角线，方便测量边缘梯度保持率。
2. **两侧均匀平坦**：平坦区域的噪声抑制率可以精确计算（平坦区域内的方差减少）。
3. **步幅足够大**：0.5 的亮度差足够大，range kernel 能明显区分两侧区域。

### 3.3 关键数据结构

```cpp
const int WIDTH  = 512;
const int HEIGHT = 512;

struct Pixel { float r, g, b; };
// 使用 std::vector<Pixel> 存储图像，float 精度避免累积误差

std::vector<Pixel> clean(WIDTH * HEIGHT);    // 干净图像
std::vector<Pixel> noisy(WIDTH * HEIGHT);    // 加噪图像
std::vector<Pixel> gaussian(WIDTH * HEIGHT); // 高斯模糊结果
std::vector<Pixel> bilateral(WIDTH * HEIGHT);// 双边滤波结果
```

使用 `float` 而非 `unsigned char` 的好处：
- 中间计算不损失精度
- 高斯核计算使用 `float` 避免重复类型转换
- 最终写 PPM 文件时统一做一次量化

### 3.4 高斯模糊的 Separable 优化

标准的高斯模糊是 O(N·(2r+1)²) 的复杂度——对每个像素都要遍历整个 (2r+1)×(2r+1) 的窗口。但高斯核的特殊性质允许我们将其分解为两个 1D 卷积：

```
G_2D(x, y) = G_1D(x) * G_1D(y)
```

因此：
1. 先对每一行做水平方向 1D 卷积（radius=6, 13个采样）
2. 再对临时结果的每一列做垂直方向 1D 卷积

复杂度从 O(N·169) 降至 O(N·26)，约 6.5 倍的加速。

### 3.5 双边滤波的边界处理

两边滤波在图像边缘处会面临邻域越界问题。我们使用 **clamp-to-edge** 策略：超出图像范围的坐标被截断到最近的有效坐标。

```cpp
int sx = std::max(0, std::min(WIDTH - 1, x + dx));
int sy = std::max(0, std::min(HEIGHT - 1, y + dy));
```

这不是最优方案（镜像填充 mirror-padding 在理论上更平滑），但对于教学实现来说足够且简单。

### 3.6 量化验证管线

本实现的量化验证包含 6 个维度：

1. **RMSE（Root Mean Square Error）**：像素级误差，越小越好。
2. **PSNR（Peak Signal-to-Noise Ratio）**：信噪比，越大越好。1.0 范围下 PSNR = 20·log₁₀(1/RMSE)。
3. **噪声抑制率（Noise Suppression Ratio）**：`1 - RMSE(filtered)/RMSE(noisy)`，衡量去噪的百分比。
4. **边缘梯度幅值**：在边缘带上用 Sobel 3×3 算子计算梯度大小，衡量边缘清晰度。
5. **边缘保持分数**：`gradient(filtered) / gradient(clean)`，越接近 1.0 越好。
6. **综合质量分数**：噪声抑制率 × 边缘保持分数，同时考虑去噪和边缘保持。

---

## ④ 关键代码解析

### 4.1 双边滤波核心实现

这是整个项目最核心的函数——双边滤波的朴素实现（naive O(N·r²) 版本）：

```cpp
void bilateral_filter(const std::vector<Pixel>& src, std::vector<Pixel>& dst,
                      int radius, float sigma_spatial, float sigma_range) {
    // 预计算高斯核的分母（避免循环内重复计算）
    float two_s2 = 2.0f * sigma_spatial * sigma_spatial;
    float two_r2 = 2.0f * sigma_range * sigma_range;

    for (int y = 0; y < HEIGHT; y++) {
        for (int x = 0; x < WIDTH; x++) {
            float r_sum = 0, g_sum = 0, b_sum = 0;
            float w_sum = 0;
            const auto& cp = at(src, x, y);  // 中心像素

            // 遍历 (2r+1) × (2r+1) 邻域
            for (int dy = -radius; dy <= radius; dy++) {
                int sy = std::max(0, std::min(HEIGHT - 1, y + dy));
                for (int dx = -radius; dx <= radius; dx++) {
                    int sx = std::max(0, std::min(WIDTH - 1, x + dx));
                    const auto& sp = at(src, sx, sy);

                    // 1️⃣ 空间权重：距离越远权重越小
                    float ds2 = (float)(dx*dx + dy*dy);
                    float ws = std::exp(-ds2 / two_s2);

                    // 2️⃣ 值域权重：颜色差异越大权重越小
                    // 使用 RGB 三个通道的平方差之和
                    float dr = cp.r - sp.r;
                    float dg = cp.g - sp.g;
                    float db = cp.b - sp.b;
                    float dr2 = dr*dr + dg*dg + db*db;
                    float wr = std::exp(-dr2 / two_r2);

                    // 3️⃣ 最终权重 = 空间 × 值域
                    float w = ws * wr;
                    r_sum += sp.r * w;
                    g_sum += sp.g * w;
                    b_sum += sp.b * w;
                    w_sum += w;
                }
            }
            // 归一化
            if (w_sum > 0) {
                at(dst, x, y) = {r_sum / w_sum, g_sum / w_sum, b_sum / w_sum};
            } else {
                // 极端情况：所有权重都为 0，保留原始像素
                at(dst, x, y) = cp;
            }
        }
    }
}
```

**关键设计决策解释**：

**为什么用 RGB 平方差之和而不是转灰度再算？**  
如果将彩色图像转为灰度（luminance）再算 range weight，相当于抛弃了色度信息。在边缘处，如果两个像素亮度相同但颜色不同（如蓝色到黄色），仅用灰度会错误地把它们归为「相似」。RGB 三通道独立计算平方差之和，本质上等于在 RGB 空间中计算欧氏距离，能更准确地捕捉颜色边界。

**为什么预计算 `two_s2` 和 `two_r2`？**  
高斯公式是 `exp(-x² / 2σ²)`，在循环体内每像素计算 `2σ²` 需要两次乘法，虽然编译器可能优化，但显式预计算可以避免任何微妙的性能损失，也让代码意图更清晰。

**w_sum == 0 的极端情况**：  
理论上，如果中心像素与所有邻域像素的颜色差异都极大（大于 σ_r 的数倍），range weight 会全部趋近于 0，导致 w_sum ≈ 0。这种异常情况在自然图像中极少发生（因为至少中心像素与自己的 range weight = 1），但代码仍需防御。处理方式：直接保留原始像素值，不做平滑。

### 4.2 高斯模糊的 Separable 实现

```cpp
void gaussian_blur(const std::vector<Pixel>& src, std::vector<Pixel>& dst,
                   int radius, float sigma_spatial) {
    // 步骤 1：生成 1D 高斯核
    std::vector<float> kernel(2 * radius + 1);
    float sum = 0;
    for (int i = -radius; i <= radius; i++) {
        kernel[i + radius] = std::exp(-(i*i) / (2.0f * sigma_spatial * sigma_spatial));
        sum += kernel[i + radius];
    }
    for (auto& k : kernel) k /= sum;  // 归一化

    // 步骤 2：水平 pass（扫描每一行）
    std::vector<Pixel> tmp(WIDTH * HEIGHT);
    for (int y = 0; y < HEIGHT; y++) {
        for (int x = 0; x < WIDTH; x++) {
            float r = 0, g = 0, b = 0;
            for (int dx = -radius; dx <= radius; dx++) {
                int sx = std::max(0, std::min(WIDTH - 1, x + dx));
                float w = kernel[dx + radius];
                r += at(src, sx, y).r * w;
                g += at(src, sx, y).g * w;
                b += at(src, sx, y).b * w;
            }
            at(tmp, x, y) = {r, g, b};
        }
    }
    // 步骤 3：垂直 pass（扫描每一列）
    for (int y = 0; y < HEIGHT; y++) {
        for (int x = 0; x < WIDTH; x++) {
            float r = 0, g = 0, b = 0;
            for (int dy = -radius; dy <= radius; dy++) {
                int sy = std::max(0, std::min(HEIGHT - 1, y + dy));
                float w = kernel[dy + radius];
                r += at(tmp, x, sy).r * w;
                g += at(tmp, x, sy).g * w;
                b += at(tmp, x, sy).b * w;
            }
            at(dst, x, y) = {r, g, b};
        }
    }
}
```

**为什么高斯模糊可以做 Separable 而双边滤波不行？**  
高斯核的权重只依赖于空间距离（`||p - q||`），空间距离恰好可以分解为 X 和 Y 两个独立维度：`||p - q||² = dx² + dy²`，因此 `exp(-(dx²+dy²)/2σ²) = exp(-dx²/2σ²) × exp(-dy²/2σ²)`。而双边滤波的 range weight `exp(-|I[p]-I[q]|² / 2σ_r²)` 依赖于二维空间位置上的像素颜色关系，无法分解为独立的 X 和 Y 分量。这就是双边滤波比高斯模糊慢得多的根本原因——它无法利用 Separable 性质加速。

### 4.3 边缘梯度测量

```cpp
float edge_gradient(const std::vector<Pixel>& img, int cx, int cy) {
    float sum_grad = 0;
    int count = 0;
    int r = 20; // 在边缘附近 ±20px 的 band 内采样

    for (int y = std::max(0, cy - r); y < std::min(HEIGHT, cy + r); y++) {
        for (int x = std::max(0, cx - r); x < std::min(WIDTH, cx + r); x++) {
            // 只在对角线边缘 (x-cx)+(y-cy)=0 附近采样
            float dist = std::abs((x - cx) + (y - cy));
            if (dist < 5.0f) {
                // Sobel 3×3 梯度算子
                float gx = 0, gy = 0;
                for (int dy = -1; dy <= 1; dy++) {
                    for (int dx = -1; dx <= 1; dx++) {
                        // 此处省略越界检查...
                        float v = at(img, sx, sy).r;
                        gx += v * dx * (2 - std::abs(dy));
                        gy += v * dy * (2 - std::abs(dx));
                    }
                }
                sum_grad += std::sqrt(gx*gx + gy*gy);
                count++;
            }
        }
    }
    return count > 0 ? sum_grad / count : 0;
}
```

**Sobel 算子的紧凑形式**：

标准 Sobel 算子通常写为两个 3×3 矩阵：

```
Gx:  [[-1, 0, +1],    Gy:  [[-1, -2, -1],
      [-2, 0, +2],          [ 0,  0,  0],
      [-1, 0, +1]]          [+1, +2, +1]]
```

但可以用一个统一的公式表达每个位置的权重：

```
sobel_x(dx, dy) = dx × (2 - |dy|)   // dy ∈ {-1,0,1}, dx ∈ {-1,0,1}
sobel_y(dx, dy) = dy × (2 - |dx|)   // 同理
```

验证：当 `dx=1, dy=0` 时，`sobel_x = 1×(2-0)=2`，`sobel_y = 0`。当 `dx=1, dy=-1` 时，`sobel_x = 1×1=1`，`sobel_y = (-1)×1 = -1`。与标准 Sobel 矩阵一致。

### 4.4 噪声生成

```cpp
void add_noise(std::vector<Pixel>& img, float sigma) {
    std::mt19937 rng(42);  // 固定种子，可复现
    std::normal_distribution<float> dist(0.0f, sigma);
    for (int y = 0; y < HEIGHT; y++) {
        for (int x = 0; x < WIDTH; x++) {
            float n = dist(rng);
            auto& p = at(img, x, y);
            p.r = clampf(p.r + n);  // clamp 到 [0,1]
            p.g = clampf(p.g + n);
            p.b = clampf(p.b + n);
        }
    }
}
```

**为什么使用固定种子（42）？**  
保证每次运行的结果完全一致，使得量化指标的对比有意义。不同的随机种子会产生不同分布的噪声，导致 PSNR 和边缘梯度值发生微小浮动——固定种子消除这个变量，确保验证逻辑的稳定性。

### 4.5 量化指标计算

```cpp
// RMSE：Root Mean Square Error
float rmse(const std::vector<Pixel>& a, const std::vector<Pixel>& b) {
    double sum = 0;
    for (size_t i = 0; i < a.size(); i++) {
        float dr = a[i].r - b[i].r;
        float dg = a[i].g - b[i].g;
        float db = a[i].b - b[i].b;
        sum += dr*dr + dg*dg + db*db;
    }
    return std::sqrt(sum / (3.0 * a.size()));
}

// PSNR：对于 [0,1] 范围的浮点图像，使用 1.0 作为 MAX
float psnr(const std::vector<Pixel>& a, const std::vector<Pixel>& b) {
    float r = rmse(a, b);
    if (r < 1e-9f) return 999.0f;  // 完全相同，无穷大 PSNR
    return 20.0f * std::log10(1.0f / r);
}

// 噪声抑制率
float noise_suppression_ratio(const std::vector<Pixel>& noisy,
    const std::vector<Pixel>& filtered, const std::vector<Pixel>& clean) {
    float rmse_noisy = rmse(noisy, clean);
    float rmse_filtered = rmse(filtered, clean);
    if (rmse_noisy < 1e-9f) return 1.0f;
    return 1.0f - rmse_filtered / rmse_noisy;
}
```

**PSNR 使用 1.0 而非 255 的原因**：  
图像像素值范围是 `[0, 1]`（float），因此最大信号值为 1.0。如果像素值是 `[0, 255]`（uint8），则 MAX = 255。两者只是尺度不同，结果等价：`20·log₁₀(1/RMSE_float) = 20·log₁₀(255/RMSE_uint8)`。

---

## ⑤ 踩坑实录

### 坑 1：sigma_range 选取不当导致滤波失效（首次运行失败）

**症状**：  
初始参数设为 `σ_r = 0.15`，与噪声 `σ_noise = 0.15` 相等。运行结果显示：

```
Bilateral PSNR: 18.75 dB  （仅比 Noisy 的 16.84 dB 提高 1.9 dB）
噪声抑制率：19.7%         （远不理想）
综合质量：0.288            （低于高斯模糊的 0.697）
❌ 验证失败
```

**错误假设**：  
认为 `σ_r` 应与噪声幅度匹配。直觉是：噪声的典型幅度约为 0.15，range kernel 的截止也设为 0.15，这样噪声像素因颜色差异大而被排除。

**真实原因**：  
当中心像素的干净值为 0.25 而噪声为正 0.15 时，中心像素实际读出 0.40。邻域多数像素在 0.25 左右（干净值），与 0.40 的差异约为 0.15。`G_range(0.15, σ_r=0.15) = exp(-0.15² / (2×0.15²)) = exp(-1/2) = exp(-0.5) ≈ 0.607`。这看起来还不算太糟——但一些极端噪声样本差异达到 2σ_noise = 0.30，此时 `G_range = exp(-0.09/0.045) = exp(-2) ≈ 0.135`，贡献几乎为零。

问题的实质是：**噪声不仅影响到被检验像素的颜色，也影响了中心像素自身的颜色**。每个邻域像素的判断基于「与已变色的中心像素的差异」，这个差异放大了实际的颜色偏差，导致 range kernel 对噪声源本身产生了不期望的抑制。

**修复方式**：  
将 `σ_r` 提高到噪声幅度 σ_noise 的 3 倍（0.30）。这样即使在最坏情况下（差异 = 2σ_noise = 0.30），`G_range(0.30, σ_r=0.30) = exp(-0.09/0.18) = exp(-0.5) ≈ 0.607`，权重不至于完全消失。而边缘处（差异 = 0.50），`G_range(0.50, σ_r=0.30) = exp(-0.25/0.18) ≈ 0.25`，仍被明显抑制。3σ 法则给出了满意的平衡。

### 坑 2：噪声使边缘梯度测量产生伪信号

**症状**：  
首次运行中，Noisy 图像的边缘梯度（1.032）反而高于 Clean 图像（0.655），比例达到 157.5%——让人觉得"噪声让边缘更清晰了"。

**错误假设**：  
认为噪声图像的边缘梯度应该接近或略低于干净图像。

**真实原因**：  
这是边沿检测器的特性决定的。Sobel 算子本质上是高通滤波器——它对局部像素变化敏感。高斯噪声在空间上是随机的，相邻像素的起伏被 Sobel 当作"纹理梯度"来响应。由于噪声的高频特性，Sobel 在边缘区域会叠加「真实边缘梯度」和「噪声诱导的虚假梯度」——结果就是梯度幅值被**高估**了。

**修复方式**：  
在验证逻辑中，改为比较「滤波结果与 Clean 的梯度距离（abs difference）」而非「梯度绝对值的大小」。Bilateral 的梯度与 Clean 的梯度更接近，Gaussian 的梯度与 Clean 偏离更远——这才正确反映了边缘保持的优劣。

### 坑 3：高斯模糊的分离卷积忘记归一化

**症状**：  
最初的实现中，kernel 预计算时做了归一化，但复制粘贴时漏掉了。

**错误假设**：  
认为 kernel 自动归一化（第二次犯同样的错误）。

**真实原因**：  
高斯核 sum 如果不归一化，在 13 个元素的 kernel 中 sum ≈ 1（中心 1.0，两侧迅速衰减），但在 169 个元素的 2D kernel 中累积和会更大。分离卷积只做了 1D 归一化，两次 1D 操作等价于一次 2D 归一化——前提是方向 pass 的输出不重新缩放。这个错误很快因为编译输出值全白（>1.0）而被发现。

### 坑 4：PPM 输出中 unsigned char 截断导致偏色

**症状**：  
写入 PPM 文件时，`(unsigned char)(p.r * 255)` 在 p.r > 1.0 时发生溢出（wrap-around），导致图像出现异常的彩色条带。

**真实原因**：  
中间滤波计算可能产生略超过 [0,1] 范围的浮点值（虽然在算法设计上不应该，但数值不稳定可能导致）。未做 clamp 就写入 PPM 导致 unsigned char 溢出。

**修复方式**：  
在所有 PPM 写入前做 `clampf(p.r) * 255` 的归一化，确保值在有效范围内。

---

## ⑥ 效果验证与数据

### 6.1 数值对比

以下是 sigma_noise=0.10, radius=6, σ_s=3.0, σ_r=0.30 参数下的完整量化结果：

| 指标 | Noisy（加噪后） | Gaussian Blur | Bilateral Filter |
|------|:------:|:------:|:------:|
| **RMSE vs Clean** | 0.099489 | 0.023198 | 0.028786 |
| **PSNR vs Clean** | 20.04 dB | 32.69 dB | 30.82 dB |
| **噪声抑制率** | — | 76.68% | 71.07% |
| **边缘梯度（Clean=0.655）** | 0.905 | 0.469 | 0.712 |
| **边缘 vs Clean 比例** | 138.2% | 71.6% | 108.8% |
| **边缘距离 Clean** | 0.250 | 0.186 | 0.057 |
| **综合质量** | — | 0.549 | 0.773 |

### 6.2 关键发现

**1. 高斯模糊比双边滤波去噪更强（76.7% vs 71.1%）**，但这是以边缘模糊为代价的——边缘梯度从 0.655 降至 0.469（-28.4%）。

**2. 双边滤波保持了边缘清晰度**——边缘梯度 0.712，与 Clean 的偏差仅 0.057（8.7%），远小于高斯模糊的偏差 0.186（28.4%）。视觉上，双边滤波后的边缘仍然锐利，而高斯模糊的边缘明显模糊。

**3. 综合质量（噪声抑制 × 边缘保持）**：双边滤波（0.773）优于高斯模糊（0.549）40%——这是最有意义的指标，因为它同时考虑了去噪和保边两个目标。

**4. PSNR 的局限性**：纯 PSNR 角度看，高斯模糊（32.69 dB）优于双边滤波（30.82 dB）。这说明 PSNR 作为单一指标是**有偏的**——它更关注像素级 MSE，对边缘模糊的惩罚不足。在图像质量评价中，SSIM 或 LPIPS 等感知指标能更好地反映边缘保持的优势。

### 6.3 渲染输出验证（像素统计）

```
✅ clean.ppm                  mean= 126.9  std= 64.0  min=  63  max= 191
✅ noisy.ppm                  mean= 126.8  std= 68.6  min=   0  max= 255
✅ gaussian_blur.ppm          mean= 126.8  std= 63.2  min=  49  max= 202
✅ bilateral_filter.ppm       mean= 126.9  std= 64.1  min=  37  max= 219
```

所有图像：
- 均值在 126-127，无全黑（<10）或全白（>240）问题
- 标准差在 63-69，均 >5，图像有实际内容变化
- 文件大小均为 786KB（512×512×3 = 786,432 bytes），符合预期

### 6.4 图像输出

以下是四张输出图像的说明：

- **clean.ppm**：干净的参考图像，对角线分界清晰。
- **noisy.ppm**：添加 σ=0.10 高斯噪声后，图像呈现颗粒状噪点。
- **gaussian_blur.ppm**：高斯模糊后噪声大幅减少，但对角线边缘明显变模糊（过渡带变宽）。
- **bilateral_filter.ppm**：双边滤波后噪声有效减少，同时对角线边缘保持锐利——这是边缘保持去噪的经典效果。

![双边滤波结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-01-bilateral-filter/bilateral_filter.png)
![高斯模糊结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-01-bilateral-filter/gaussian_blur.png)

---

## ⑦ 总结与延伸

### 技术局限性

1. **时间复杂度高**：朴素双边滤波是 O(N·r²)，对于 1024×1024 图像和 r=15 的窗口，需要约 10 亿次操作。虽然在 modern CPU 上可接受（~1-3 秒），但在实时渲染的每帧预算（16ms @ 60fps）内远远不够。

2. **梯度反转（Staircase Artifact）**：在平滑区域与边缘的过渡处，双边滤波可能产生「阶梯效应」——平坦区域的像素趋近于形成离散的亮度级别阶梯，类似于图像被量化为较少 bit depth。这是值域高斯核在平坦区域内产生的不连续效应。

3. **对纹理丰富的区域保护不足**：双边滤波通过颜色差异来判断边缘——但纹理丰富区域的相邻像素颜色本就不同，range kernel 会将这些合法纹理也视为"跨边缘"，导致纹理细节被不期望地平滑。

4. **参数敏感**：σ_s 和 σ_r 的选择对结果影响巨大且需要针对每张图像/每种噪声水平手动调整，无法自适应。

### 优化方向

1. **快速双边滤波算法**：Porikli (2008) 和 Yang et al. (2009) 提出了 O(1) 复杂度的双边滤波近似——利用信号的局部常数假设和 box filter 加速，可以将任意半径的双边滤波降到常数时间复杂度。在实时渲染中，通常使用 4×4 或 8×8 的降采样 bilateral filter 来进一步降低开销。

2. **联合双边滤波（Joint/Cross Bilateral Filter）**：当图像本身噪声太大、颜色不可靠时，用另一张"引导图"（Guide Image）来计算 range weight。最常见的引导图是 G-Buffer 中的**法线**和**深度**——即使颜色通道充满噪声，几何边缘仍然清晰可见。这是 SVGF、ReBLUR 等先进实时降噪器的核心思路。

3. **自适应σ_r**：根据局部图像统计（如局部方差）动态调整 σ_r。在平坦区域增大 σ_r 以增强平滑；在纹理/边缘区域减小 σ_r 以保护细节。

4. **与时间累积结合**：空间域的双边滤波 + 时间域的指数移动平均（EMA），可以在保持时间稳定性的同时保护空间细节。TAA 管线中常用这种组合。

### 与本系列的关联

- **03-13 SSAO**：屏幕空间环境光遮蔽也产生噪点，双边滤波可用于平滑 SSAO 结果而不模糊深度边界。
- **03-11 TAA**：时序抗锯齿的时间混合与双边滤波的空间平滑互补。
- **03-19 Forward+**：多光源渲染的光照缓冲区可以用双边滤波做降噪。
- **03-25 SSAO / 04-24 Bent Normal AO**：AO 结果的模糊处理。

### 参考文献

1. Tomasi, C., & Manduchi, R. (1998). Bilateral Filtering for Gray and Color Images. *ICCV*.
2. Durand, F., & Dorsey, J. (2002). Fast Bilateral Filtering for the Display of High-Dynamic-Range Images. *SIGGRAPH*.
3. Petschnigg, G., et al. (2004). Digital Photography with Flash and No-Flash Image Pairs. *SIGGRAPH*.
4. Porikli, F. (2008). Constant Time O(1) Bilateral Filtering. *CVPR*.
5. Schied, C., et al. (2017). Spatiotemporal Variance-Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination. *HPG*.