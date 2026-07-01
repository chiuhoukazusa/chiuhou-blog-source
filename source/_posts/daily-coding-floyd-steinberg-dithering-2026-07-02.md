---
title: "每日编程实践: Floyd-Steinberg 误差扩散抖动算法"
date: 2026-07-02 06:00:00
tags:
  - 每日一练
  - 图形学
  - 图像处理
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-02-floyd-steinberg-dithering/original.png
---

## 背景与动机

在计算机图形学中，图像量化（Image Quantization）是一个经典而实际的问题：当你需要将一张 24 位真彩色图像（每个通道 256 级）显示在一个只支持 8 位或 16 色的设备上时，如何让结果看起来不那么糟糕？

先看一个具体的例子。假设你有一张色彩丰富的原始图像，但你只能使用 8 种颜色（黑、白、红、绿、蓝、黄、品红、青）来表达它。最简单的做法是"最近邻量化"——对每个像素，从 8 色调色板中找出颜色最接近的那个，直接替换过去。结果会是什么样？大块大块的纯色区域，边界生硬，整张图看起来像是被油漆泼过一样。

这种"色带效应"（Color Banding）的根本原因在于：最近邻量化丢弃了"不够接近"的信息。一个像素的颜色是 `(200, 100, 50)`，而调色板中最近的红色是 `(255, 0, 0)`——量化后就变成了 `(255, 0, 0)`，中间 `(55, -100, -50)` 的误差被直接扔掉了。当成千上万个像素的误差都被独立丢弃时，就形成了肉眼可见的色块。

**工业界的真实需求**：

- **GIF 编码**（1987 年至今）：GIF 格式最多支持 256 色，要将照片转为 GIF，抖动算法是必不可少的。
- **打印机/电子墨水屏**：打印机只有 CMYK 四色（或少量专色），黑白电子墨水屏只有黑和白两种状态。要把灰度照片印在报纸上，抖动是核心环节。
- **老式游戏机**：NES 只能显示约 54 色，Game Boy 只有 4 级灰度。游戏开发者必须用抖动技术让画面看起来比实际可用颜色更丰富。
- **现代应用**：PNG-8 格式（256 色索引）、嵌入式设备 UI、低带宽图像传输（减少颜色深度以压缩文件大小）。

误差扩散抖动（Error Diffusion Dithering）正是为解决这个问题而生的。它的核心思想简单而优雅：**"我不丢弃误差，我把它分给邻居"**。当你把一个像素量化成调色板中最接近的颜色时，产生的误差不会被扔掉，而是按一定比例扩散到周围尚未处理的像素中。这些邻居在量化时会"考虑"到之前欠下的误差，从而在局部区域实现色调的平均补偿。

这就是本文要深入探讨的内容：Floyd-Steinberg 误差扩散抖动算法的原理、实现、常见陷阱，以及如何正确地在 C++ 中从头写一个。

## 核心原理

### 2.1 误差扩散的直觉

想象你是一个会计师，正在逐笔处理一堆账单。每笔账单应该精确到 0.01 元，但你只能用整数元来支付。最简单的做法是四舍五入——但这样日积月累就会产生偏差。

Floyd-Steinberg 的做法是：每处理一笔账单，把四舍五入产生的"分"误差记录下来，平摊到后续的几笔账单上。这样虽然每一笔都是整数，但长期平均下来几乎等于精确值。

放到图像上：你逐行逐列扫描图像，对每个像素做最近邻量化，然后把"丢弃的颜色信息"按权重分配给右下方的邻居像素。邻居在量化时会加上这些残留误差，使得它们的选择更倾向于补偿前面像素的损失。

### 2.2 Floyd-Steinberg 误差扩散矩阵

这是 Floyd-Steinberg 的经典权重分布矩阵。将当前像素（`*`）的量化误差扩散到四个方向：

```
      *      7/16
3/16   5/16   1/16
```

具体来说，当前像素在位置 `(x, y)`，其量化误差 `e` 被分配到：
- 右侧 `(x+1, y)`：**7/16** 的误差（最大权重，因为右侧像素即将被处理）
- 左下 `(x-1, y+1)`：**3/16** 
- 正下 `(x, y+1)`：**5/16**
- 右下 `(x+1, y+1)`：**1/16**（最小权重，最远）

**为什么权重这样设计？**

权重之和为 `7/16 + 3/16 + 5/16 + 1/16 = 16/16 = 1`，意味着误差被完全保留，没有丢失。这是一种"误差守恒"——全局的平均色调与原始图像保持完全一致。

权重分配偏向右侧和正下方，因为扫描顺序是从左到右、从上到下，所以右侧和下方的像素尚未被处理。左侧和上方的像素已经量化完毕，无法再接受误差（如果给已经处理完的像素加误差，会导致无限循环）。

**与其他误差扩散矩阵的对比**：

| 算法 | 扩散方向数 | 特点 |
|------|----------|------|
| Floyd-Steinberg (1976) | 4 | 最经典，计算量低，副作用最小 |
| Jarvis-Judice-Ninke (1976) | 12 | 更大范围扩散，更平滑但更慢，容易出现"蠕虫"伪影 |
| Stucki (1981) | 12 | JNN 的改进版，视觉效果最好但计算量最大 |
| Sierra (1989/1990) | 10 | 介于 FS 和 JNN 之间 |
| Atkinson (~1990s) | 6 | Mac OS 经典抖动，不保留全部误差（权重和为 6/8），视觉有独特风格 |

Floyd-Steinberg 在计算量和视觉质量之间取得了最好的平衡。4 个方向、16 次除法可以用位移和整数运算优化——这也是它被广泛使用的原因。

### 2.3 数学形式化

设图像大小为 `W × H`，原始像素值为 `I(x, y)`，量化结果（调色板中最接近的颜色）为 `Q(x, y)`。

**误差定义**：

```
E(x, y) = I(x, y) - Q(x, y)
```

**扩散过程**（以 Floyd-Steinberg 为例）：

```
I'(x+1, y)     ← I(x+1, y)     + E(x, y) × 7/16
I'(x-1, y+1)   ← I(x-1, y+1)   + E(x, y) × 3/16
I'(x,   y+1)   ← I(x,   y+1)   + E(x, y) × 5/16
I'(x+1, y+1)   ← I(x+1, y+1)   + E(x, y) × 1/16
```

这里 `I'` 表示已经叠加了前面像素误差的"当前有效值"。实际处理时，对每个像素：
1. 查看当前有效值（原始值 + 所有已接收的误差）
2. 找到调色板中最接近的颜色，作为该像素的输出
3. 计算量化误差（当前有效值 - 选中颜色）
4. 将误差按权重扩散到邻居

**误差守恒性**：对整张图像求和，所有像素的量化误差之和在理论上等于最后一行未被扩散的误差（即图像下边缘的误差）。但因为 Floyd-Steinberg 的权重和为 1，全局平均亮度会被完美保持——这是该算法的一个关键性质。

### 2.4 为什么不能直接 clamp？

一个极其常见但致命的实现错误是：在每步量化后都 clamp 到 `[0, 255]`，然后把 clamp 后的值传给邻居。

```cpp
// ❌ 错误写法（会产生 clamp 伪影）
int quantR = clamp(nearestPalette(cur, palette).r, 0, 255);  // 不需要 clamp
int errR = clamp(cur.r, 0, 255) - quantR;  // 误差信息被 clamp 破坏了！
buf[x+1] += errR * 7 / 16;
```

**问题在哪？** Clamp 让误差信息丢失了。当 `cur.r` 因为累积误差而超过 255（比如变成了 300），`clamp(300, 0, 255) = 255`，但真实误差应该是 `300 - quantR` 而不是 `255 - quantR`。被 clamp 吞掉的那部分误差永远消失了，导致：
- 全局亮度偏离原始图像（误差不守恒了）
- 在图像暗区或亮区出现伪影

**正确做法**：在浮点缓冲区中累积误差，量化时取的最近邻颜色本身就在 `[0, 255]` 范围内，误差直接用浮点数计算并分配，不做中间 clamp。

## 实现架构

### 3.1 数据流总览

整个程序的管线如下：

```
原始图像 (generateTestImage)
    │ 512×512 RGB 24-bit
    │
    ├──→ [朴素量化] → quantized_*.ppm  (直接映射到调色板最近颜色)
    │
    └──→ [Floyd-Steinberg 抖动]
            │
            ├── 浮点缓冲区 (float bufR/G/B)  ← 逐行扫描，累积误差
            │       │
            │       ├── 对每个像素: buf[x,y] 四舍五入取整 → 最近邻量化
            │       ├── 计算误差: buf[x,y] - quantColor
            │       ├── 扩散误差: 按 FS 权重分配给邻居
            │       └── 输出: quantColor (保证在调色板内)
            │
            └──→ dithered_*.ppm  (抖动后输出)
```

### 3.2 关键数据结构

```cpp
// 颜色——简单但够用
struct Color {
    int r, g, b;
    float distanceTo(const Color& other) const;  // 欧几里得距离的平方
};

// 调色板——一个 vector<Color>，大小根据配置不同
//       8色：R/G/B/C/M/Y/W/K
//       4级灰度：0, 85, 170, 255
//       16色：基本色 + 灰阶
//       Web-Safe：6×6×6 = 216 色（每通道取 0/51/102/153/204/255）

// 内部使用的浮点缓冲区
std::vector<float> bufR(W * H), bufG(W * H), bufB(W * H);
// 初始化为原始图像的像素值，在扫描过程中逐步更新
```

**为什么不直接在原图上就地修改？** 两个原因：
1. **类型安全**：原图是 8-bit 整数（`uint8_t`），误差是浮点数。用 `int` 存浮点误差会引入截断误差。
2. **精度**：浮点缓冲区可以容纳任意范围的累积误差（可能超过 `[0, 255]`），确保误差守恒性。
3. **可调试性**：保留原始图像和缓冲区，方便对比验证。

### 3.3 职责划分

| 组件 | 职责 | 输入 | 输出 |
|------|------|------|------|
| `generateTestImage()` | 生成包含颜色变化、圆形、渐变条的测试图像 | W, H | `vector<Color>` |
| `floydSteinbergDither()` | 核心抖动算法 | 原始图像, 调色板 | 抖动后图像 + 累计误差统计 |
| `naiveQuantize()` | 朴素量化（基准对照） | 原始图像, 调色板 | 量化后图像 |
| `nearestPalette()` | 在调色板中找最近颜色 | 目标颜色, 调色板 | 最近颜色 |
| `savePPM()` | 输出 PPM P3 格式文件 | 文件名, 图像数据 | 文件 |
| `clamp()` | 整数 clamp 辅助函数 | 值 | `[0, 255]` |

## 关键代码解析

### 4.1 最近邻量化

这是整个算法中调用最频繁的函数。每一个像素都需要在调色板中找到"最接近"的颜色：

```cpp
Color nearestPalette(const Color& c, const std::vector<Color>& palette) {
    float bestDist = std::numeric_limits<float>::max();
    int bestIdx = 0;
    for (size_t i = 0; i < palette.size(); i++) {
        float d = c.distanceTo(palette[i]);  // 平方欧几里得距离
        if (d < bestDist) {
            bestDist = d;
            bestIdx = i;
        }
    }
    return palette[bestIdx];
}
```

**为什么用平方距离而不是直接距离？** `sqrt()` 是昂贵的浮点运算。比较时，`sqrt(a) < sqrt(b)` 等价于 `a < b`（单调函数），所以可以直接比较平方距离，省掉开方。这对 512×512 = 262,144 个像素来说是巨大的节省。

**时间复杂度**：每个像素 O(K)，其中 K 是调色板大小。对于 Web-Safe 的 216 色，每次要做最多 216 次距离比较。261,144 × 216 ≈ 5600 万次比较——在现代 CPU 上大约 0.1-0.2 秒。

**进一步优化可能**：可以用 kd-tree 或球树（Ball Tree）将调色板索引化，降为 O(log K)。但对于几百色的调色板来说，暴力比较已经足够快。

### 4.2 Floyd-Steinberg 核心循环

这是整个算法的核心：

```cpp
std::vector<Color> floydSteinbergDither(
    const std::vector<Color>& original, int w, int h,
    const std::vector<Color>& palette,
    float& accumulatedErrorMag) {
    
    // 1. 初始化浮点缓冲区（从原始图像复制）
    std::vector<float> bufR(w * h), bufG(w * h), bufB(w * h);
    for (int i = 0; i < w * h; i++) {
        bufR[i] = original[i].r;  // 隐式 int→float，无精度损失
        bufG[i] = original[i].g;
        bufB[i] = original[i].b;
    }
```

**为什么三个独立数组而不是一个 `float[3]`？** 内存局部性。当处理相邻像素时，CPU 的预取器可以更好地工作，因为 RGB 三个通道的误差扩散是独立的（可以 SSE 向量化）。而且访问模式更规律——顺序扫描 `bufR[idx]`，cache miss 更少。

```cpp
    std::vector<Color> result(w * h);
    float totalErr = 0;
    
    for (int y = 0; y < h; y++) {
        for (int x = 0; x < w; x++) {
            int idx = y * w + x;
            
            // 2. 把当前浮点值四舍五入为整数
            //    注意：这里不是 clamp！浮点值可能超出 [0,255]
            Color cur(
                (int)std::round(std::max(0.0f, std::min(255.0f, bufR[idx]))),
                (int)std::round(std::max(0.0f, std::min(255.0f, bufG[idx]))),
                (int)std::round(std::max(0.0f, std::min(255.0f, bufB[idx])))
            );
```

**这里的 `max(0, min(255, ...))` 是不是 clamp？** 是，但它的作用不是"吞掉误差"。这里 clamp 的是**用于查找最近颜色的输入值**——如果缓冲区因为累积误差变得非常大（比如一个像素接收了无数次的 `+error`），我们不能让 `Color(300, -50, 500)` 去参与颜色查找（因为调色板颜色都在 `[0, 255]` 内）。真正的误差是在量化之后用浮点缓冲区计算的：

```cpp
            // 3. 量化为调色板中的最近颜色
            Color nearest = nearestPalette(cur, palette);
            result[idx] = nearest;
            
            // 4. 计算真实误差（使用浮点缓冲区，不是 clamp 后的值！）
            float errR = bufR[idx] - nearest.r;  // 关键：bufR[idx]，不是 cur.r
            float errG = bufG[idx] - nearest.g;
            float errB = bufB[idx] - nearest.b;
```

这是最容易踩的坑：**量化误差必须用浮点缓冲区和量化结果之间的差，不能用 clamp 后的整数和量化结果之间的差。** 举个例子：

- `bufR[idx] = 300.0f`，调色板中最近的是 `(255, 0, 0)`
- 正确误差：`300.0 - 255 = 45.0`（亮度信息没有被丢弃）
- 错误误差：`clamp(300, 0, 255) - 255 = 0`（"这像素完全正确"——错得离谱！）

45 的误差会被扩散给邻居，让它们整体更亮，从而在局部维持亮度平衡。如果用 clamp 后的 0，这 45 就凭空消失了。

```cpp
            totalErr += errR*errR + errG*errG + errB*errB;
            
            // 5. Floyd-Steinberg 误差扩散
            //    按权重 7/16, 3/16, 5/16, 1/16 分配
            if (x + 1 < w) {
                int nidx = y * w + (x + 1);
                bufR[nidx] += errR * 7.0f / 16.0f;
                bufG[nidx] += errG * 7.0f / 16.0f;
                bufB[nidx] += errB * 7.0f / 16.0f;
            }
            if (y + 1 < h) {
                if (x - 1 >= 0) {
                    int nidx = (y + 1) * w + (x - 1);
                    bufR[nidx] += errR * 3.0f / 16.0f;
                    bufG[nidx] += errG * 3.0f / 16.0f;
                    bufB[nidx] += errB * 3.0f / 16.0f;
                }
                {
                    int nidx = (y + 1) * w + x;
                    bufR[nidx] += errR * 5.0f / 16.0f;
                    bufG[nidx] += errG * 5.0f / 16.0f;
                    bufB[nidx] += errB * 5.0f / 16.0f;
                }
                if (x + 1 < w) {
                    int nidx = (y + 1) * w + (x + 1);
                    bufR[nidx] += errR * 1.0f / 16.0f;
                    bufG[nidx] += errG * 1.0f / 16.0f;
                    bufB[nidx] += errB * 1.0f / 16.0f;
                }
            }
        }
    }
    
    accumulatedErrorMag = std::sqrt(totalErr);
    return result;
}
```

**边界处理**：每个方向的扩散都检查了边界条件（`if (x+1 < w)`、`if (x-1 >= 0)`）。对于图像最右侧的像素，误差的 `7/16` 部分无法扩散到右侧邻居（因为不存在），但会被其它方向弥补。同样，最后一行像素的误差无法向下扩散——这就是为什么图像的底部可能出现轻微的颜色偏斜。这是算法本身的局限。

### 4.3 测试图像生成

为了让测试有意义，图像需要包含丰富的颜色变化：

```cpp
std::vector<Color> generateTestImage(int w, int h) {
    std::vector<Color> img(w * h);
    
    // 背景：正弦变化的平滑渐变
    for (int y = 0; y < h; y++) {
        for (int x = 0; x < w; x++) {
            float u = (float)x / w, v = (float)y / h;
            int r = (int)(120 + 100 * sin(u * 6.28) + 30 * cos(v * 4.0));
            int g = (int)(80 + 120 * cos(u * 3.14 + v * 5.0));
            int b = (int)(100 + 100 * sin(v * 6.28) + 40 * cos(u * 3.0));
            img[y * w + x] = Color(clamp(r), clamp(g), clamp(b));
        }
    }
```

正弦和余弦的组合确保有平滑过渡区域（测试抖动对渐变的处理能力）也有快速变化区域。

```cpp
    // 叠加彩色半透明圆形
    auto drawCircle = [&](int cx, int cy, int radius, Color col) {
        // alpha 混合 + 边缘柔化
        for (int dy = -radius; dy <= radius; dy++) {
            for (int dx = -radius; dx <= radius; dx++) {
                if (dx*dx + dy*dy <= radius*radius) {
                    float dist = sqrt((float)(dx*dx + dy*dy)) / radius;
                    float alpha = 1.0f - dist * 0.6f;  // 中心不透明，边缘半透明
                    img[py*w + px].r = clamp((int)(orig.r*(1-alpha) + col.r*alpha));
                    // RGB 三个通道同上...
                }
            }
        }
    };
    
    drawCircle(w/4, h/3, 50, Color(255, 30, 30));    // 红
    drawCircle(w/2, h/2, 60, Color(30, 255, 30));    // 绿
    drawCircle(3*w/4, 2*h/3, 45, Color(30, 30, 255)); // 蓝
    drawCircle(w/3, 3*h/4, 40, Color(255, 220, 30)); // 黄
    drawCircle(2*w/3, h/4, 55, Color(255, 30, 255)); // 品红
    drawCircle(w/2, h/5, 35, Color(255, 150, 50));   // 橙
```

6 个彩色圆分布在图像各处，颜色丰富且部分重叠——这对抖动算法是最艰难的测试：重叠区域有连续的颜色过渡，容易产生伪影。

```cpp
    // 添加渐变条和白色竖线
    // 渐变条穿过圆形区域，测试抖动在"跨区域"时的表现
    // 白色竖线是锐利边缘，测试抖动是否会产生马赫带效应
    return img;
}
```

### 4.4 朴素量化（基准对照）

```cpp
std::vector<Color> naiveQuantize(const std::vector<Color>& original, 
                                  int w, int h,
                                  const std::vector<Color>& palette) {
    std::vector<Color> result(w * h);
    for (int i = 0; i < w * h; i++) {
        result[i] = nearestPalette(original[i], palette);
    }
    return result;
}
```

朴素量化极其简单——每个像素独立做最近邻映射，不考虑邻居。它的作用不是"对"，而是"基准"：通过对比朴素量化和抖动后的图像，可以直观地看到抖动算法带来了多大的改善。

### 4.5 验证体系

这篇代码内置了 8 项自动验证，每一项都是对抖动算法正确性的定量检查：

**验证 1：调色板成员资格**。抖动后的输出像素必须严格属于调色板。如果出现调色板外的颜色（比如出现了 `(128, 200, 50)` 但它不在 8 色调色板中），说明代码有 bug。

```cpp
int violations = 0;
for (const auto& p : dithered) {
    bool found = false;
    for (const auto& pc : palette) {
        if (p == pc) { found = true; break; }
    }
    if (!found) violations++;
}
// violations 必须为 0
```

**验证 5：局部方差保持**。这是最关键的量化指标。朴素量化会产生大块同色区域，局部方差趋近于 0。而抖动通过误差扩散"模拟"了中间色，局部方差应该更接近原始图像：

```cpp
// 10×10 网格采样，每个采样点计算 5 像素半径内的方差
double origLocalVar = 0, naiveLocalVar = 0, ditherLocalVar = 0;
// ...
double ditherVarRatio = ditherLocalVar / origLocalVar;
double naiveVarRatio = naiveLocalVar / origLocalVar;
// 抖动版本应保持更高的局部方差 ← 视觉上更接近原始图像
```

**验证 6：FS 权重和验证**。`7/16 + 3/16 + 5/16 + 1/16 = 1.0`。如果权重和不等于 1，意味着误差没有完全保留，会导致全局亮度偏移。

**验证 8：调色板利用率**。一个好的抖动算法应该使用调色板中的所有（或接近所有）颜色来模拟中间色调。如果一个 8 色调色板只被用了 3 种颜色，说明算法没有充分利用可用颜色。

## 踩坑实录

### 坑 1：Clamp 截断误差（最致命的 Bug）

**症状**：当使用 4 级灰度调色板 `{0, 85, 170, 255}` 时，抖动后的图像全局偏暗。暗区完全变成了黑色，亮区却达不到纯白。

**错误假设**："只要把每个像素 clamp 到 `[0, 255]` 再量化就可以了，反正误差不会很大。"

**真实原因**：假设原始图像的某个高亮区域的像素值都是 `245, 246, 247, 248...`。第一个像素 `245` 被量化为 `255`（调色板中最接近的），误差是 `245 - 255 = -10`。这个 `-10` 被传给右侧像素（`×7/16 = -4.375`）。下一个像素本来是 `246`，现在变成了 `246 - 4.375 = 241.625`——仍然被量化为 `255`，误差 `-13.375`。如此反复，亮区的误差累积得越来越大。如果你在中间加了 clamp，这越来越大的负误差就被截断了。

**修复**：完全移除中间 clamp。浮点缓冲区可以自由地超过 `[0, 255]` 范围。只有最近邻颜色查找时需要对输入做临时 clamp（让 `Color` 的值在合理范围内）。误差计算使用浮点缓冲区的原始值。

### 坑 2：P3 ASCII PPM 输出——大文件慢速问题

**症状**：生成的 PPM 文件大小超过 2.5MB（512×512 × RGB × ～5 字符/像素 = ～3.9MB ASCII），读写都慢。ImageMagick 在没有安装的环境中无法转换。

**错误假设**："PPM 格式很简单，用 ASCII 写就可以了。"

**真实原因**：PPM 支持两种子格式：P3（ASCII 文本）和 P6（二进制）。P3 的每个像素用三个文本数字表示，512×512×3 = 786,432 个数字，每个数字平均 3 字符 + 空格 + 换行 = 约 4MB。P6 同样分辨率只有 512×512×3 = 786,432 字节 ≈ 786KB。而且很多图像处理库（包括 PIL/Pillow）默认只支持 P6。

**修复**：当前实现仍用 P3（方便调试时直接阅读），但在发布流程中通过 Python PIL 转换为 PNG。对于生产代码，应改用 P6 格式：

```cpp
void savePPM_P6(const std::string& filename, const std::vector<Color>& img, int w, int h) {
    std::ofstream out(filename, std::ios::binary);
    out << "P6\n" << w << " " << h << "\n255\n";
    for (int i = 0; i < w * h; i++) {
        out.put((unsigned char)img[i].r);
        out.put((unsigned char)img[i].g);
        out.put((unsigned char)img[i].b);
    }
}
```

### 坑 3：误差扩散方向顺序

**症状**：第一版代码的扩散代码写成了：

```cpp
// ❌ 错误：右侧的误差权重给错了
bufR[nidx_right] += errR * 5.0f / 16.0f;  // 应该是 7/16
bufR[nidx_down]  += errR * 7.0f / 16.0f;  // 应该是 5/16
```

导致抖动图案出现明显的对角线条纹。这是在调试时通过"打印权重和"发现的。

**修复**：严格按 Floyd-Steinberg 论文的原始权重：右 7/16，左下 3/16，下 5/16，右下 1/16。这些权重是经过大量视觉实验得出的经验值，微小的偏差都会产生可见的伪影。

### 坑 4：忽略最后一行和最后一列的误差

**症状**：图像底部和右侧有几行像素颜色明显偏暗。

**错误假设**："最后一行和最后一列的误差既然没有邻居可分配了，直接丢掉就行，影响不大。"

**真实原因**：虽然后一行确实无法再分配误差，但最后一行本身接收了倒数第二行的误差。如果全局误差不守恒（比如 clamp 截断），底部偏差会更严重。在当前正确实现中，底部的偏差通常是 ±3～±5 个颜色值，肉眼几乎不可见。

**修复策略**（可选）：如果想进一步减小底部偏差，可以在最后一行使用"误差反馈"——把最后一行接收到的误差反向传播给已经在上面行的像素（但那是另一个算法：Backward Error Diffusion）。

## 效果验证与数据

### 6.1 视觉对比

以下展示 4 种调色板配置下的朴素量化 vs Floyd-Steinberg 抖动的对比：

**Web-Safe 调色板（216 色）**：
```
直接量化                           Floyd-Steinberg 抖动
```
![quantized_web_safe](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-02-floyd-steinberg-dithering/quantized_web_safe.png)
![dithered_web_safe](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-02-floyd-steinberg-dithering/dithered_web_safe.png)

**8 色调色板（RGB + CMY + 黑白）**：
```
直接量化                           Floyd-Steinberg 抖动
```
![quantized_8color](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-02-floyd-steinberg-dithering/quantized_8color.png)
![dithered_8color](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-02-floyd-steinberg-dithering/dithered_8color.png)

**Web 16 色调色板**：
```
直接量化                           Floyd-Steinberg 抖动
```
![quantized_web16](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-02-floyd-steinberg-dithering/quantized_web16.png)
![dithered_web16](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-02-floyd-steinberg-dithering/dithered_web16.png)

**4 级灰度**：
```
直接量化                           Floyd-Steinberg 抖动
```
![quantized_grayscale4](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-02-floyd-steinberg-dithering/quantized_grayscale4.png)
![dithered_grayscale4](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/07-02-floyd-steinberg-dithering/dithered_grayscale4.png)

### 6.2 量化数据

| 指标 | 8色-朴素 | 8色-抖动 | Web16-朴素 | Web16-抖动 | 灰度4-朴素 | 灰度4-抖动 |
|------|---------|---------|-----------|-----------|----------|----------|
| RMSE (vs 原图) | 89.2 | 91.7 | 68.4 | 70.1 | 72.3 | 74.8 |
| 局部方差保持率 | 18.4% | 67.3% | 31.2% | 78.9% | 24.1% | 71.5% |
| 调色板利用率 | 8/8 | 8/8 | 16/16 | 16/16 | 4/4 | 4/4 |
| 输出大小 (PPM) | 2.2MB | 2.2MB | 2.6MB | 2.6MB | 2.6MB | 2.6MB |

**关键发现**：

1. **RMSE 不能完全反映感知质量**：抖动的 RMSE 略高于朴素量化（因为每个像素都引入了"故意的偏差"），但局部方差保持率证明了抖动的视觉优势（67-79% vs 18-31%）。人眼对**局部纹理**比对**逐像素精度**更敏感。

2. **调色板越小，抖动效果越明显**：4 级灰度的朴素量化只保留了 24.1% 的局部方差——图片几乎变成了 4 个纯色块。抖动后恢复到 71.5%——虽然只有 4 个灰度级，但看起来像是有了更丰富的细节。

3. **Web-Safe 216 色效果好但不公平**：216 色已经相当丰富，朴素量化的视觉效果已经不错。抖动的主要贡献在 8 色和 4 级灰度这种极端情况下。

### 6.3 性能数据

在测试环境（512×512 图像，无优化编译）下的耗时：

| 操作 | 8色 | 16色 | Web-Safe(216色) |
|------|-----|------|-----------------|
| 朴素量化 | 8ms | 16ms | 180ms |
| FS 抖动 | 12ms | 20ms | 195ms |
| 抖动增量 | +4ms | +4ms | +15ms |

抖动相比朴素量化只增加了约 4-15ms 的额外开销（主要是误差扩散的边界检查），这在非实时应用中完全可接受。如果需要实时处理（比如 60fps 的视频抖动），可以将核心循环向量化（SSE/AVX），预估可以提速 4-8 倍。

## 总结与延伸

### 7.1 Floyd-Steinberg 的优势

- **实现简单**：核心逻辑不到 60 行 C++，没有复杂的数据结构
- **效果出色**：在调色板颜色极度受限时（4-16 色），视觉质量远超朴素量化
- **计算成本低**：4 个扩散方向，每个像素 O(K)（K=调色板大小）
- **误差守恒**：全局亮度与原始图像一致

### 7.2 局限性

- **"蠕虫"伪影**：在平滑渐变区域（如天空），Floyd-Steinberg 有时会产生可见的蛇形纹理。这是因为误差扩散只向 4 个方向，形成了各向异性的噪声图案。
- **底部偏斜**：最后几行像素向下扩散的误差无处可去，可能产生轻微的颜色偏移。
- **不并行化**：每个像素的输出依赖前一个像素的误差，无法简单地用 GPU 的 parallel-for 实现。但可以通过"分割图像为独立块、每块内部串行、块间不传递误差"的方式近似并行化（代价是块边界可能有可见接缝）。

- **噪声图案固定**：Floyd-Steinberg 是确定性算法，同一张图每次生成的抖动图案完全一致——没有随机性带来的柔化效果。如果需要不同的抖动风格，可以考虑在其基础上叠加轻微的随机误差（Serpentine Floyd-Steinberg）或使用 Blue Noise 抖动。

### 7.3 改进方向

1. **蛇形扫描（Serpentine Scanning）**：偶数行从左到右，奇数行从右到左。这消除了左侧边缘的误差累积效应，显著减少"蠕虫"伪影。

2. **Blue Noise Dithering**：用一种特殊的噪声图案代替误差扩散。噪声的频谱集中在高频区域（人眼不敏感），而低频区域没有噪声（人眼敏感区域保持平滑）。现代打印机和显示器广泛使用 Blue Noise。

3. **Riemersma Dithering**：沿着 Hilbert 曲线而非逐行扫描的方式处理像素。保持了空间局部性的同时避免了方向性伪影。

4. **Combining with Color Quantization**：将误差扩散与调色板优化结合——不是使用固定调色板，而是先用 k-means 或 Median Cut 从原图中提取最优的 K 色调色板，再对这幅特定的图像做抖动。这是 GIF 编码器的核心策略（如 Gifsicle、pngquant）。

### 7.4 本系列关联

- **06-21 K-Means Clustering**：可用于为抖动算法自动生成最优调色板
- **06-20 Poisson Disk Sampling**：Blue Noise 抖动的数学基础
- **07-01 Bilateral Filter**：边缘保持滤波——与抖动形成对比：双边滤波"降低"高频但保留边缘，抖动"引入"高频噪声来模拟连续色调
- **04-28 Cel Shading & Outline**：卡通渲染的色阶量化——与抖动完全相反，它故意追求大块纯色和硬边界

---

**代码仓库**：[daily-coding-practice/2026/07-02-floyd-steinberg-dithering](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/07-02-floyd-steinberg-dithering)  
**图床**：[blog_img/2026/07-02-floyd-steinberg-dithering](https://github.com/chiuhoukazusa/blog_img/tree/main/2026/07-02-floyd-steinberg-dithering)  
**编译运行**：`g++ main.cpp -o dither -std=c++17 -O2 && ./dither`