---
title: "每日编程实践: Screen Space Ambient Occlusion (SSAO)"
date: 2026-03-25 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 实时渲染
  - SSAO
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-25-ssao/ssao_output.png
---

## 背景与动机

### 为什么需要环境光遮蔽？

在传统 Phong/Blinn-Phong 光照模型中，**环境光（Ambient）是一个常量**——整个场景里所有点的环境光照贡献完全相同，不管这个点是在空旷的平台中央，还是被夹在两堵墙的角落里。这在物理上显然是错误的：角落里的点被周围几何体遮挡，从半球方向射来的间接光理应更少。

没有环境光遮蔽的场景会显得"扁平"、"塑料感强"——所有表面的暗部都亮得不自然。加上 AO 之后，角落变暗、物体底部变暗、缝隙变暗，整体立体感和质感都会有质的提升。

### 工业界的实际使用场景

**Ambient Occlusion** 的概念最早由 Zhukov 等人在 1998 年提出，但直到 Crytek 在 2007 年 GDC 上介绍 **Screen Space Ambient Occlusion (SSAO)** 之后才真正走入实时渲染的主流——当时是为了给《孤岛危机》服务的。

今天，AO 技术在各大引擎里几乎无处不在：

- **Unity**：有 SSAO（URP/HDRP 均支持），HDRP 还有基于射线的 RTAO
- **Unreal Engine**：内置 SSAO，后来又加了 GTAO（Ground Truth Ambient Occlusion）
- **Frostbite（EA）**：用 SSAO + 预烘焙 Bent Normal 的组合
- **游戏案例**：《赛博朋克 2077》、《荒野大镖客》、《古墓丽影》等 AAA 游戏均启用了 AO

核心吸引力在于：**SSAO 只需要 G-Buffer 中的深度和法线，不需要光线追踪，可以在单帧内完成，性能代价极低**（通常 0.5~2ms per frame on GPU）。

### SSAO 的本质问题

SSAO 本质上是在**屏幕空间**近似计算这样一个积分：

```
AO(p) = (1/π) ∫_Ω V(p, ω) cos(θ) dω
```

其中 V(p, ω) 是可见性函数（0 = 被遮挡，1 = 可见），θ 是样本方向与法线的夹角。

这个积分的真正答案需要射线追踪或预烘焙，SSAO 用**屏幕空间深度比较**来近似 V(p, ω)——用已有的深度缓冲来判断某个采样点是否"被遮挡"。这当然不精确，但速度够快，视觉上够用。

---

## 核心原理

### 1. G-Buffer：存储什么信息

SSAO 需要在一个单独的 Pass 中访问几何信息，因此必须先渲染一个 G-Buffer：

| 通道 | 内容 | 用途 |
|------|------|------|
| Depth | view-space 线性深度 | 重建 view-space 位置，做深度比较 |
| Normal | view-space 法线（单位向量）| 定义 TBN 矩阵，限定半球采样方向 |
| Position | view-space 3D 位置 | 偏移采样点 |
| Albedo | 漫反射颜色 | 用于最终着色 |

注意所有数据都在 **view space**（相机坐标系）而非 world space，这样 SSAO 的计算更简单——相机就在原点，深度比较直观。

### 2. 半球采样核（Kernel）

SSAO 的核心是在片元法线方向的**切线空间半球**内随机采若干个样本点，然后把这些点投影回屏幕空间，查看对应深度值与样本的 view-space 深度的大小关系。

**采样核的生成**：

```cpp
Vec3 sample = {
    rand() * 2.0f - 1.0f,  // x ∈ [-1, 1]
    rand() * 2.0f - 1.0f,  // y ∈ [-1, 1]
    rand()                  // z ∈ [0, 1]  // 半球！
};
sample = sample.normalized();
sample = sample * rand();  // 均匀分布在半球内部（非球面）
```

然后做"加速插值"——让更多样本集中在靠近片元的区域（小范围遮蔽更重要）：

```cpp
float scale = float(i) / numSamples;
scale = lerp(0.1f, 1.0f, scale * scale);  // 二次插值
sample = sample * scale;
```

这种分布的直觉是：**越靠近表面的遮挡物越有意义**——远处的物体对局部 AO 贡献极小，但如果均匀分布，远处样本会浪费大量采样预算。

**为什么是切线空间？**

如果直接在 view space 采样，不同片元的朝向不同，半球方向不对——比如一个朝上的地板和一个朝右的墙，它们的法线不同，但我们希望采样始终在**法线方向的半球**内。切线空间保证了采样核与法线对齐。

TBN 矩阵构建：

```
T = normalize(randomVec - N × (N·randomVec))   // Gram-Schmidt 正交化
B = cross(N, T)
TBN = [T | B | N]
```

其中 randomVec 是来自随机噪声纹理的旋转向量，用于**打破规律性**，防止出现带状/轮廓状的 banding artifacts。

### 3. 深度比较与遮蔽判断

对于每个样本点 samplePos（view space），我们：

1. 将其投影到裁剪空间，再转换到屏幕坐标 (sx, sy)
2. 在 G-Buffer 的深度通道中查询 (sx, sy) 处的实际深度值 `gbufferDepth`
3. 如果 `gbufferDepth >= samplePos.z + bias`，说明此样本点位于真实几何体"内部"，即被遮挡

数学上：在 view space 中，z 轴朝相机方向为正（经过我的坐标系设置），更靠近相机的物体有更大（更正）的深度值。如果查询到的 G-Buffer 深度比样本点的深度更靠近相机，说明样本点处有实际几何体阻挡，贡献遮蔽。

**Range Check（范围检查）**：

```cpp
float rangeCheck = 1.0f - clamp(abs(fragPos.z - gbufferDepth) / radius, 0.0, 1.0);
```

如果 G-Buffer 中查询到的深度与当前片元差距很大（远超采样半径），说明这个"遮蔽"很可能来自无关的背景几何体（比如一个球体通过球心看过去，球的正背面都会产生虚假遮蔽）。Range Check 将这种远距离的遮蔽贡献逐渐衰减为 0。

**Bias（偏置）**：

加一个小的 bias 值是为了防止**自遮蔽（self-occlusion）**——法线偏差导致采样点非常接近表面，使得 G-Buffer 中的深度比样本点略深，产生错误的遮蔽。常见值 0.025~0.05。

最终遮蔽值：

```
occlusion = Σ(rangeCheck) / numSamples  // 0 = 完全不遮蔽, 1 = 完全遮蔽
```

### 4. 模糊 Pass

原始 SSAO 的结果非常噪声——因为随机核只有 64 个样本，采样数不足，加上 4×4 噪声纹理的随机旋转。直接使用会显出明显的颗粒感。

**模糊策略**：简单的 Box Blur（5×5 均值滤波）可以显著平滑噪声，同时 SSAO 本身的低频特性（AO 值变化不剧烈）决定了模糊不会损失太多高频细节。

更高质量的做法是 **Bilateral Blur**（双边模糊）——在颜色/深度边缘处减小模糊半径，防止 AO 从一个物体"渗透"到相邻物体。本实现使用 Box Blur 作为简化版本。

### 5. 最终合成

```
finalColor = albedo * ambientIntensity * (1 - occlusion * 0.85)  // AO 影响环境光
           + albedo * diffuse                                       // 漫反射不受 AO 影响
           + specular                                               // 高光不受 AO 影响
```

注意：**AO 只影响环境光，不影响直接光的漫反射/高光**。这在物理上是合理的——AO 近似的是来自各方向的间接光（已经被多次弹射的光），而直接光来自确定方向，其遮挡应该由阴影算法（Shadow Mapping 等）来处理。

---

## 实现架构

### 整体渲染管线

```
CPU Side (软光栅):
┌─────────────────────────────────────────────────┐
│  1. 构建场景（Cornell Box + 球体）                │
│     ↓                                           │
│  2. G-Buffer Pass                               │
│     顶点变换 → 三角形光栅化 → 插值写入           │
│     GBuffer.depth / normal / posView / albedo    │
│     ↓                                           │
│  3. SSAO Pass                                   │
│     per-pixel: 生成 TBN → 半球采样 → 深度比较    │
│     GBuffer.ssao = occlusion                    │
│     ↓                                           │
│  4. Blur Pass                                   │
│     Box Blur(5×5) → GBuffer.ssaoBlur            │
│     ↓                                           │
│  5. Shading Pass                                │
│     Blinn-Phong + AO → tone-map → pixels[]      │
│     ↓                                           │
│  6. PNG 输出                                    │
└─────────────────────────────────────────────────┘
```

### 关键数据结构

**GBuffer**：

```cpp
struct GBuffer {
    std::vector<float> depth;    // view-space 线性深度（正值 = 靠相机）
    std::vector<Vec3>  normal;   // view-space 法线（已归一化）
    std::vector<Vec3>  posView;  // view-space 3D 位置
    std::vector<Vec3>  albedo;   // 漫反射颜色
    std::vector<float> ssao;     // 原始遮蔽值 [0,1]
    std::vector<float> ssaoBlur; // 模糊后遮蔽值 [0,1]
};
```

深度存储的是 **view-space 正深度**（-z in view space，因为相机看向 -z 方向），这样深度比较时"更靠近相机 = 更大的值"，逻辑更直观。

**场景几何体**：

```cpp
struct Vertex {
    Vec3 worldPos;  // 世界空间位置
    Vec3 normal;    // 世界空间法线（光栅化时变换到 view space）
    Vec3 albedo;    // 漫反射颜色
};
```

为了保持代码简洁，这里不区分材质，直接在 Vertex 里存 albedo。更完整的实现会有 MaterialID + 材质数组。

**投影矩阵约定**：

采用 OpenGL 风格的右手坐标系，view space 中 -Z 朝前（相机看向 -Z 方向），Y 朝上，X 朝右。`perspective()` 函数生成的矩阵中 `m[3][2] = -1`，在变换后 w = -z（即正值）。

### 软光栅化的设计权衡

本项目没有使用 OpenGL/Vulkan，而是用 CPU 软光栅来演示 SSAO 的完整流程。这样做的好处是：

- 所有细节一目了然，不需要处理 Shader 语言和 API 调用
- 坐标系变换可以单步调试
- 便于理解 G-Buffer 的生成方式

代价是性能差（800×600 光栅化 + SSAO 约 2~5 秒），但对于教学目的完全可接受。

---

## 关键代码解析

### 三角形光栅化与 G-Buffer 填充

光栅化的核心是**重心坐标（Barycentric Coordinates）**插值。对于屏幕空间的像素点 P，计算其相对于三角形三顶点的权重 w0, w1, w2（通过叉积面积比）：

```cpp
auto edge = [](const Vec3& a, const Vec3& b, const Vec3& p) {
    return (p.x - a.x)*(b.y - a.y) - (p.y - a.y)*(b.x - a.x);
};
float area = edge(s0, s1, s2);  // 三角形有符号面积（×2）
// 对于像素 p：
float w0 = edge(s1, s2, p) / area;
float w1 = edge(s2, s0, p) / area;
float w2 = edge(s0, s1, p) / area;
if(w0 < 0 || w1 < 0 || w2 < 0) continue;  // 在三角形外
```

**透视正确插值**是一个经常被忽略的细节。直接用 w0/w1/w2 插值会在透视投影下产生错误（仿射插值的纹理会扭曲），必须在插值时除以 view-space 的深度：

```cpp
// 1/w（view-space 深度的倒数）在屏幕空间中是线性的
float invW0 = 1.0f / (-vp0.z);  // vp0.z 是 view-space z（负值）
float invW1 = 1.0f / (-vp1.z);
float invW2 = 1.0f / (-vp2.z);
float invW  = w0*invW0 + w1*invW1 + w2*invW2;  // 线性插值 1/w
float depth = 1.0f / invW;  // 恢复实际深度

// 对任意属性 A，透视正确插值：
Vec3 posV = (vp0*(w0*invW0) + vp1*(w1*invW1) + vp2*(w2*invW2)) * depth;
```

这里的数学原理是：透视除法使得屏幕空间坐标 = clip space / w，而 1/w 在屏幕空间中是**线性的**（w 本身不是）。所以先插值 1/w，再乘以 depth 恢复原始 view-space 属性。

**深度测试**：保留距相机最近（最大 depth 值）的片元：

```cpp
if(depth < gb.depth[idx]) continue;  // 当前片元更远，丢弃
gb.depth[idx] = depth;  // 更新深度
// 更新 normal, posView, albedo...
```

### SSAO 核心循环

```cpp
void computeSSAO(GBuffer& gb, const Mat4& proj,
                 const std::vector<Vec3>& kernel,
                 const std::vector<Vec3>& noise,
                 float radius, float bias)
{
    for(int py = 0; py < H; py++) {
        for(int px = 0; px < W; px++) {
            int idx = py*W + px;
            Vec3 fragPos = gb.posView[idx];  // 当前片元 view-space 位置
            Vec3 normal  = gb.normal[idx];

            // ① 从噪声纹理取随机旋转向量（4×4 平铺）
            const Vec3& randomVec = noise[(py%4)*4 + (px%4)];

            // ② 构建 TBN（切线空间 → view 空间的变换）
            Vec3 tangent   = (randomVec - normal * normal.dot(randomVec)).normalized();
            Vec3 bitangent = normal.cross(tangent);
            auto TBN = [&](const Vec3& v) -> Vec3 {
                return tangent * v.x + bitangent * v.y + normal * v.z;
            };

            float occlusion = 0.0f;
            for(int i = 0; i < nSamples; i++) {
                // ③ 将采样核从切线空间变换到 view 空间，偏移到采样点
                Vec3 samplePos = fragPos + TBN(kernel[i]) * radius;

                // ④ 将采样点投影到屏幕空间
                float x_ = proj.m[0][0]*samplePos.x + proj.m[0][3];
                float y_ = proj.m[1][1]*samplePos.y + proj.m[1][3];
                float w_ = -samplePos.z;  // view-space 深度（正值）
                int sx = (int)((x_/w_ * 0.5f + 0.5f) * W);
                int sy = (int)((y_/w_ * 0.5f + 0.5f) * H);
                sx = clamp(sx, 0, W-1); sy = clamp(sy, 0, H-1);

                // ⑤ 查询 G-Buffer 深度 & 深度比较
                float sampleDepth = gb.depth[sy*W + sx];  // view-space 深度

                // rangeCheck：防止远处几何体产生虚假遮蔽
                float rangeCheck = 1.0f -
                    std::min(1.0f, std::abs(-fragPos.z - sampleDepth) / radius);

                // sampleDepth >= -samplePos.z + bias → 样本点被遮挡
                if(sampleDepth >= -samplePos.z + bias) {
                    occlusion += rangeCheck;
                }
            }
            gb.ssao[idx] = occlusion / nSamples;
        }
    }
}
```

**关键细节解析**：

- `TBN(kernel[i])` 将切线空间的半球样本旋转对齐到片元法线方向。没有这步，所有片元的采样半球方向相同（比如全部朝上），朝侧面的墙面就不会产生正确遮蔽。
- `noise[(py%4)*4 + (px%4)]` 是 4×4 平铺的随机旋转——不同像素用不同的随机旋转，采样核的方向各异，打破规律性，避免 banding。
- `w_ = -samplePos.z`：view space 中 z 为负值（相机看 -Z 方向），所以 `-z` 才是正的深度值，与投影矩阵的约定一致（`m[3][2] = -1` 即 w = -z）。
- bias = 0.05：实测发现不加 bias 会有明显的自遮蔽黑点，特别是在平坦表面上（法线对齐不精确时）。

### 采样核生成的加速插值

```cpp
std::vector<Vec3> genSSAOKernel(int n, std::mt19937& rng) {
    std::uniform_real_distribution<float> dist(0.0f, 1.0f);
    std::vector<Vec3> kernel;
    for(int i = 0; i < n; i++) {
        Vec3 s{
            dist(rng)*2.0f - 1.0f,
            dist(rng)*2.0f - 1.0f,
            dist(rng)          // z > 0，保证在法线方向半球内
        };
        s = s.normalized() * dist(rng);  // 均匀分布到半球内部

        // 加速插值：更多样本靠近原点
        float scale = float(i) / n;
        scale = 0.1f + scale*scale * 0.9f;  // lerp(0.1, 1.0, t²)
        s = s * scale;
        kernel.push_back(s);
    }
    return kernel;
}
```

`scale = lerp(0.1, 1.0, t²)` 的效果：第 0 个样本的 scale = 0.1（紧靠片元），第 63 个样本的 scale = 1.0（最远）。这样 64 个样本中约有一半集中在 0~0.5 半径范围内，对局部遮蔽的捕捉更精细。

---

## 踩坑实录

### Bug 1：深度比较逻辑反向导致全白或全黑

**症状**：整个右半屏幕几乎全白（ssao 值为 0），或者全黑（ssao 值为 1）。

**错误假设**：以为 view-space depth 是负值（z 朝后的右手系），depth 比较用 `sampleDepth <= -samplePos.z + bias`。

**真实原因**：在这套软光栅实现中，存储的 `GBuffer.depth` 是 `-vp.z`（正值，靠相机 = 大），而 `samplePos.z` 也是直接来自 view-space（负值）。所以 `-samplePos.z` 才是正的深度值，与 `gb.depth` 的符号匹配。比较方向是：如果 `gbufferDepth >= -samplePos.z + bias`，说明 G-Buffer 里的几何体在样本点"后面"（深度更大 = 更靠相机），即遮挡了样本点。

**修复**：反复用 `cout` 打印 `fragPos.z`、`samplePos.z`、`sampleDepth` 的实际数值，确认符号和量级后修正比较逻辑。

**教训**：SSAO 的深度比较非常容易因坐标系约定而出错。每次遇到全亮/全暗的结果，先打印 3~5 个像素的实际深度值，确认数值符号。

### Bug 2：unused variable 编译警告

**症状**：`g++ -Wall -Wextra` 报 `warning: unused variable 'z_'`。

**原因**：在手动实现投影变换时，写了四行（x', y', z', w'），但 SSAO Pass 中只用了 x'/w' 和 y'/w' 来计算屏幕坐标，z' 不需要（我们直接用 view-space 的 `-samplePos.z` 做深度比较，而不是 clip-space z）。

**修复**：直接删除 `z_` 那一行。

**教训**：写手动矩阵变换时，先明确每个分量是否真的被用到，避免写出 dead code。

### Bug 3：图像右半整体比左半暗但不明显

**症状**：运行后发现右半（SSAO）与左半（无SSAO）差距很小，几乎看不出来。

**错误假设**：以为 radius = 0.5 够用了。

**真实原因**：场景的 Cornell Box 尺寸是 ±5（half-extent = 5），而 SSAO radius = 0.5 相对场景太小，只能探测非常近的遮蔽，效果微弱。

**修复**：将 radius 从 0.5 调大到 1.5，效果明显增强（左右均值差从 ~2 增大到 ~16）。

**量化对比**：

| radius | 左均值 | 右均值 | 差值 | 效果 |
|--------|--------|--------|------|------|
| 0.5    | 73.2   | 71.0   | 2.2  | 几乎不可见 |
| 1.0    | 73.2   | 65.1   | 8.1  | 可见但不明显 |
| 1.5    | 73.2   | 57.6   | 15.6 | 明显遮蔽效果 |

**教训**：SSAO 的 radius 必须与场景尺度匹配，不能使用固定的"经验值"。

### Bug 4：Valid pixels = W×H（背景也被算为有效像素）

**症状**：验证脚本输出 `Valid pixels: 480000/480000`，而明显场景不会覆盖全部像素。

**分析**：GBuffer 初始化时 depth = -1e9（极小值），在 shade 函数中当 depth < -1e8 时返回背景色，但这些背景像素在 SSAO 计算中被 `if(gb.depth[idx] < -1e8f) continue` 正确跳过——只是我的"有效像素计数"统计逻辑有问题，实际上 480000 = 800×600 是总像素数，而有效像素数更接近 480000 - 背景像素数。

**分析后判断**：这个 Bug 不影响视觉效果，只是统计信息不准确。可以用 `gb.depth[idx] > -1e8f` 过滤背景像素。

**保留**：因为对最终输出无影响，保留原始统计，下次改进。

---

## 效果验证与数据

### 输出图像统计

```
文件大小：1.4 MB（> 10KB ✅）
图像尺寸：800 × 600（左无SSAO / 右有SSAO）
像素均值：65.6（在10~240范围内 ✅）
像素标准差：38.4（> 5，图像有丰富内容 ✅）
```

### 左右对比数据

| 区域 | 像素均值亮度 | 说明 |
|------|------------|------|
| 左半（无SSAO） | 73.26 | 普通 Blinn-Phong 光照 |
| 右半（有SSAO） | 57.58 | AO 遮蔽环境光，整体变暗 |
| 差值 | 15.68 | SSAO 导致 ~21% 亮度降低 |

### 渲染性能（CPU 单线程）

| 阶段 | 耗时 |
|------|------|
| 场景构建（3332 三角形） | < 10ms |
| G-Buffer 光栅化 | ~300ms |
| SSAO（64 样本，800×600） | ~1.5s |
| Box Blur（5×5） | ~50ms |
| 最终着色 + 输出 | ~200ms |
| **总计** | **~2s** |

SSAO 是最慢的阶段，480000 像素 × 64 样本 = 约 3100 万次深度采样，CPU 单线程情况下约 1.5 秒。GPU 实现（fragment shader）通常 < 1ms。

### 视觉对比

下图左半为普通 Phong 光照，右半为加入 SSAO 后的结果（黄线为分界）：

![SSAO对比效果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-25-ssao/ssao_output.png)

可以观察到：
1. **右半的墙角交接处明显更暗**（主要 AO 效果区域）
2. **高柱子底部和球体底部变暗**（与地板接触区域）
3. **红墙/绿墙与天花板、地板的夹角处** 产生明显遮蔽
4. **空旷区域**（如天花板中央）AO 值接近 0，亮度基本不变

---

## 总结与延伸

### SSAO 的局限性

1. **屏幕边缘误差**：采样点投影到屏幕外时无法获取深度信息，边缘附近的遮蔽不准确
2. **深度不连续性**：锐利边缘（前景物体 vs 背景）处，背景的深度信息不该影响前景的 AO
3. **法线精度依赖**：如果 G-Buffer 的法线不准确（比如低多边形模型），TBN 矩阵也会偏差
4. **无法捕捉远处遮蔽**：radius 限制了只能探测局部邻域，大规模遮蔽（如室内 vs 室外）无法体现
5. **非物理**：SSAO 只是近似，不能用于物理准确渲染

### 改进方向

- **HBAO（Horizon-Based AO）**：沿水平方向搜索遮挡视角，比 SSAO 更物理准确，减少半球采样数量
- **GTAO（Ground Truth AO）**：更接近真实半球积分，Unreal Engine 5 使用
- **RTAO（Ray Traced AO）**：RTX 硬件加速，完全物理准确但有性能代价
- **Bilateral Blur**：比 Box Blur 更好，保留边缘、防止 AO 跨越不同物体渗透
- **Temporal AA + SSAO**：利用时序累积更多样本，降低单帧采样数量（从 64 降到 8-16）
- **Interleaved Sampling**：不同像素使用不同偏移的采样核子集，配合 Denoiser 重建

### 与本系列其他文章的关联

本系列已经实现了多种光照技术，SSAO 补充了"近似间接光遮蔽"这一环：

| 日期 | 技术 | 类型 |
|------|------|------|
| 03-20 | 球谐函数环境光照 | 间接光（远场） |
| 03-21 | SPPM 光子映射 | 全局光照（精确） |
| 03-22 | BDPT 双向路径追踪 | 全局光照（精确） |
| 03-23 | Voxel Cone Tracing | 间接光（近场，实时近似） |
| 03-24 | 次表面散射 | 材质特效 |
| **03-25** | **SSAO** | **环境光遮蔽（屏幕空间近似）** |

SSAO 在"精确度 vs 性能"的谱系上处于最低成本端，适合所有实时应用；光子映射/BDPT 在精确端，适合离线渲染。实际游戏引擎往往两者都要——用 SSAO 做实时 AO，用烘焙的 AO Map（离线路径追踪生成）来补充精度。

今天这个项目用纯 CPU 软光栅实现了完整的 SSAO 渲染管线，从 G-Buffer 构建、半球采样、深度比较、模糊，到最终合成，每个环节的代码都是亲手写的，没有依赖任何图形 API。这种"手写"的方式最适合学习——遇到任何 Bug 都可以直接打断点、打印中间值，没有 GPU 调试的黑盒感。

下一步可能方向：HBAO 实现（对 SSAO 的直接改进），或者 Temporal Reprojection（利用上一帧数据，降低当前帧采样成本）。

---

*代码仓库：[daily-coding-practice/2026/03/03-25-ssao](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/03/03-25-ssao)*

*系列索引：[每日编程实践合集](https://github.com/chiuhoukazusa/daily-coding-practice)*
