---
title: "每日编程实践: Motion Blur Renderer — 速度缓冲与后处理运动模糊"
date: 2026-03-27 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 实时渲染
  - 后处理
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-27-motion-blur/motion_blur_output.png
---

## 一、背景与动机

### 运动模糊：人眼的物理现实

我们的眼睛（以及相机的感光元件）在曝光期间并不是瞬间"拍一张照"，而是在一段时间窗口内持续积累光信号。当物体在这段时间内发生运动时，不同时刻的光信号叠加在同一感光区域，就产生了**运动模糊（Motion Blur）**。

如果没有运动模糊，画面会呈现出一种不自然的"抖动感"。这在早期电子游戏和动画中尤为明显——每一帧都是完全锐利的静止图像，但一旦物体快速运动，视觉反而会感到不舒适。这个现象被称为**频闪效应（Stroboscopic Effect）**。

运动模糊的本质是对快门开放时间段内的图像进行**时间积分**：

$$
I_{blurred}(x, y) = \frac{1}{T} \int_0^T I(x, y, t)\, dt
$$

其中 $T$ 是快门时间，$I(x, y, t)$ 是 $t$ 时刻像素 $(x, y)$ 处的颜色值。

### 工业界实际应用

运动模糊在以下场景中必不可少：

**游戏引擎**：虚幻引擎（Unreal Engine）、Unity 的 HDRP 都内置了后处理运动模糊。典型实现是速度缓冲（Velocity Buffer）+ 后处理 Pass，这也是我们今天实现的方案。

**影视 VFX**：离线渲染中会通过大量时间步样本（Time-step Sampling）来计算高质量运动模糊，耗时可以是实时方案的数百倍，但质量远超后处理近似。

**赛车/体育游戏**：高速移动的物体（赛车、足球、武器）不加运动模糊会显得假；加了之后立刻提升临场感。

**相机运动**：游戏中摄像机快速旋转时，背景模糊能极大降低晕动感（Motion Sickness）。这也是我们今天在实现相机轨道运动时一并考虑的原因。

### 两大流派

实时渲染中实现运动模糊主要有两种思路：

**1. 时间超采样（Temporal Supersampling）**：在一帧的时间窗口内用多个微小时间步渲染多次，再平均。质量高，但对性能要求极高，通常只用于高端光追。

**2. 速度缓冲后处理（Velocity Buffer Post-Processing）**：渲染一帧时额外输出速度缓冲（每像素的屏幕空间运动矢量），然后在后处理阶段沿速度方向采样模糊。这是今天实现的方案，也是现代实时渲染的主流选择。

后处理方案的核心假设是：物体在一帧内的运动是**线性的**。这对大多数游戏场景足够准确，但对急剧加速或曲线运动会有近似误差。

---

## 二、核心原理

### 2.1 速度缓冲（Velocity Buffer）

速度缓冲存储的是每个像素从**上一帧**到**当前帧**的屏幕空间位移，单位是像素。

对于场景中某点的世界坐标 $\mathbf{P}_{world}$：

**当前帧投影**：
$$
\mathbf{v}_{curr} = \text{NDC}_{curr}(\mathbf{P}_{world}) = \text{proj}_{curr} \cdot \text{view}_{curr} \cdot \mathbf{P}_{world}
$$

**上一帧投影**：
$$
\mathbf{v}_{prev} = \text{NDC}_{prev}(\mathbf{P}_{world}) = \text{proj}_{prev} \cdot \text{view}_{prev} \cdot M_{prev} \cdot M_{curr}^{-1} \cdot \mathbf{P}_{world}
$$

其中 $M_{curr}$、$M_{prev}$ 分别是当前帧和上一帧的模型矩阵。

**运动矢量（屏幕空间）**：
$$
\mathbf{vel}(x, y) = \text{screen}(\mathbf{v}_{curr}) - \text{screen}(\mathbf{v}_{prev})
$$

将 NDC 坐标转换到屏幕坐标：

$$
\text{screen}_x = \left(\frac{\text{NDC}_x + 1}{2}\right) \cdot W, \quad \text{screen}_y = \left(1 - \frac{\text{NDC}_y + 1}{2}\right) \cdot H
$$

注意 Y 轴要翻转，因为屏幕坐标系原点在左上角，Y 向下增大。

### 2.2 透视校正插值

在光栅化过程中，三角形内部的属性插值**不能简单地使用线性重心坐标**。原因是透视投影是非线性变换，直接线性插值会产生错误的透视扭曲。

正确的做法是透视校正插值（Perspective-Correct Interpolation）。设三角形三顶点的裁剪空间 w 值为 $w_0, w_1, w_2$，重心坐标为 $\lambda_0, \lambda_1, \lambda_2$，属性值为 $a_0, a_1, a_2$，则：

$$
a_{interp} = \frac{\lambda_0 \cdot a_0 / w_0 + \lambda_1 \cdot a_1 / w_1 + \lambda_2 \cdot a_2 / w_2}{\lambda_0 / w_0 + \lambda_1 / w_1 + \lambda_2 / w_2}
$$

直觉解释：我们先将属性除以 w（投影到"1/w 空间"中），在这个空间里线性插值是正确的，然后再乘回 w。这是因为透视变换保持了 1/z 的线性性。

在我们的代码中：
```cpp
// 透视校正权重
float wu = u * invW[0], wv = v * invW[1], ww = w * invW[2];
float wSum = wu + wv + ww;
float nu = wu/wSum, nv = wv/wSum, nw = ww/wSum;
```

这里 `invW[i]` 就是 `1/clip[i].w`，经过归一化后得到正确的插值权重。

### 2.3 运动模糊采样策略

有了速度缓冲后，后处理模糊的核心思想是：对于每个像素 $(x, y)$，沿其速度矢量方向均匀采样 $N$ 个位置，取平均：

$$
I_{blurred}(x, y) = \frac{1}{N} \sum_{i=0}^{N-1} I\left(x + \text{vel}_x \cdot t_i,\; y + \text{vel}_y \cdot t_i\right)
$$

其中 $t_i \in [-0.5, 0.5]$，以当前像素为中心双向采样（模拟快门时间的前半段和后半段）：

$$
t_i = \frac{i}{N-1} - 0.5, \quad i = 0, 1, \ldots, N-1
$$

为什么双向采样（而不是只向后采样）？因为运动模糊是由快门时间内物体扫过的整条轨迹造成的，当前帧的渲染位置大约在这段轨迹的中间时刻，向前和向后各延伸半个速度步长才符合物理。

对于速度 $|v| < 0.5$ 像素的区域，视为"几乎不动"，直接输出原始颜色（避免无谓的采样开销）。

### 2.4 背面剔除（Back-face Culling）

在屏幕空间中，用三角形顶点的二维叉积来判断朝向：

$$
\text{cross}_{2D} = (v_1 - v_0) \times (v_2 - v_0) = (v_{1x}-v_{0x})(v_{2y}-v_{0y}) - (v_{1y}-v_{0y})(v_{2x}-v_{0x})
$$

右手坐标系中，若 $\text{cross}_{2D} \geq 0$，三角形顶点是顺时针排列（屏幕朝上），意味着面朝背面，应当剔除。若 $< 0$，则逆时针，面朝前，需要绘制。

直觉：想象你用手指沿三角形顶点走一圈，若感觉是逆时针就说明这个面在朝向你。

### 2.5 Phong 光照

Phong 模型将光照分为三个分量，对每个分量分别计算后加总：

$$
I = I_{ambient} + I_{diffuse} + I_{specular}
$$

**环境光（Ambient）**：均匀的微弱光照，防止完全阴影的地方全黑：
$$
I_{ambient} = k_a \cdot C_{object}
$$
这里 $k_a = 0.15$，稍微提亮阴影面。

**漫反射（Diffuse）**：符合朗伯余弦定律，光越垂直入射越亮：
$$
I_{diffuse} = \max(0, \mathbf{n} \cdot \mathbf{l}) \cdot C_{object}
$$
其中 $\mathbf{n}$ 是法线，$\mathbf{l}$ 是光照方向（归一化）。$\max(0, \cdot)$ 保证背光面为0，不出现负值。

我们加入了两个光源（权重0.4的补光），这样物体的背面也能有少量光照，不会完全变黑。

**镜面高光（Specular）**：用半向量 $\mathbf{h} = \text{normalize}(\mathbf{l} + \mathbf{v})$ 替代反射向量，减少计算量同时效果相近（Blinn-Phong 变体）：
$$
I_{specular} = \max(0, \mathbf{n} \cdot \mathbf{h})^{32} \cdot k_s
$$
指数32控制高光的锐利程度，越大高光越小越亮。

---

## 三、实现架构

### 3.1 渲染管线概述

整体数据流：

```
几何体生成
    ↓
顶点变换（model → view → clip → NDC → screen）
    ↓
光栅化（重心坐标、深度测试）
    ↓                    ↓
颜色缓冲（Phong着色）    速度缓冲（当前帧 - 上一帧屏幕坐标）
    ↓                    ↓
        ↓    ←→    ↓
    后处理运动模糊（沿vel方向采样）
        ↓
    输出PNG（4张）
```

关键点：渲染时**同步维护颜色缓冲和速度缓冲**，共享一套深度测试逻辑——深度测试通过才同时写入颜色和速度。

### 3.2 关键数据结构

**VelocityBuffer**：与颜色缓冲同尺寸，每像素存储 `Vec2`（屏幕空间运动矢量，单位像素）：
```cpp
struct VelocityBuffer {
    int width, height;
    std::vector<Vec2> data;
    Vec2& at(int x, int y) { return data[y*width + x]; }
};
```
为什么用 `Vec2` 而不是 `Vec3`？速度缓冲只需要屏幕空间（2D）的位移，Z 方向的深度变化已经隐含在透视变换中。

**RenderContext**：打包传递给渲染函数的所有"全局"状态，避免到处传参数：
```cpp
struct RenderContext {
    int width, height;
    Mat4 proj, view;          // 当前帧变换矩阵
    Mat4 prevProj, prevView;  // 上一帧变换矩阵（用于计算速度）
    Vec3 cameraPos;           // 用于计算视方向（光照）
    Image* colorBuffer;
    DepthBuffer* depthBuffer;
    VelocityBuffer* velocityBuffer;
};
```

**SceneObject**：通过 `std::function<Mat4(float)>` 描述物体随时间的变换，使得运动逻辑完全解耦于渲染逻辑：
```cpp
struct SceneObject {
    std::vector<Triangle> mesh;
    std::function<Mat4(float)> modelFn;  // 给定时间t返回模型矩阵
};
```
这种设计让"旋转的球"、"轨道运动的小球"、"静止的地面"可以用同一套接口描述，只是 `modelFn` 不同。

### 3.3 CPU 光栅化的职责划分

**顶点阶段**：
- 将模型空间坐标乘以 MVP（model-view-projection）矩阵，得到裁剪坐标
- 同时用上一帧的 MVP 矩阵（`prevMVP`）变换相同的模型坐标，得到上一帧的裁剪坐标
- 两套坐标最终都转换到屏幕空间，差值即为速度

**光栅化阶段**：
- 计算三角形包围盒，遍历包围盒内所有像素
- 用重心坐标判断像素是否在三角形内
- 深度测试（z-buffer），只处理最近的片元

**片元阶段**（逐像素）：
- 透视校正插值法线、颜色
- Phong 光照着色，写入颜色缓冲
- 插值当前帧和上一帧的屏幕坐标，差值写入速度缓冲

---

## 四、关键代码解析

### 4.1 顶点变换与速度计算

在 `renderMesh` 函数中，同时处理当前帧和上一帧的变换：

```cpp
void renderMesh(RenderContext& ctx,
                const std::vector<Triangle>& mesh,
                const Mat4& model,
                const Mat4& prevModel) {
    // 当前帧：proj * view * model
    Mat4 mvp     = ctx.proj * ctx.view * model;
    // 上一帧：prevProj * prevView * prevModel
    Mat4 prevMVP = ctx.prevProj * ctx.prevView * prevModel;
    
    for(const auto& tri : mesh){
        Vec4 clip[3], prevClip[3];
        for(int i=0;i<3;i++){
            // 同一个顶点位置，两套变换
            clip[i]     = mvp     * Vec4(tri.v[i].pos, 1.f);
            prevClip[i] = prevMVP * Vec4(tri.v[i].pos, 1.f);
        }
        // ...
    }
}
```

关键设计：`prevModel` 参数允许传入上一帧不同的模型矩阵，这样旋转中的物体（当前帧和上一帧位置不同）可以正确计算速度。

### 4.2 透视除法与 NDC 转屏幕坐标

```cpp
// 透视除法：裁剪坐标 → NDC
Vec2 ndc[3], prevNDC[3];
float invW[3];
for(int i=0;i<3;i++){
    invW[i] = 1.f / clip[i].w;
    ndc[i] = { clip[i].x * invW[i], clip[i].y * invW[i] };
    // 上一帧同理，注意防止 w <= 0（物体在相机后面）
    float pInvW = 1.f / (prevClip[i].w > 0.001f ? prevClip[i].w : 0.001f);
    prevNDC[i] = { prevClip[i].x * pInvW, prevClip[i].y * pInvW };
}
```

为什么要 `> 0.001f` 的保护？上一帧的物体可能部分或全部跑到相机后面（w < 0），直接做除法会得到错误的 NDC，导致速度矢量失控。用 0.001 夹住避免极大值。

NDC 到屏幕空间的转换：
```cpp
Vec2 ndcToScreen(float ndcX, float ndcY, int width, int height) {
    return { (ndcX * 0.5f + 0.5f) * width,
             (1.f - (ndcY * 0.5f + 0.5f)) * height };
}
```
X 轴：NDC [-1,1] → [0, width]
Y 轴：NDC [-1,1] → [height, 0]（翻转，屏幕 Y 向下增大）

### 4.3 透视校正插值与速度写入

这是整个渲染器最关键的部分——正确插值属性并计算速度：

```cpp
// 透视校正插值权重
float wu = u * invW[0], wv = v * invW[1], ww = w * invW[2];
float wSum = wu + wv + ww;
if(wSum < 1e-8f) continue;
float nu = wu/wSum, nv = wv/wSum, nw = ww/wSum;
// nu, nv, nw 现在是透视校正后的重心坐标

// 插值法线（视空间）
Vec3 norm = viewNormals[0]*nu + viewNormals[1]*nv + viewNormals[2]*nw;
norm = norm.normalized();

// 插值颜色
Color col = tri.v[0].color*nu + tri.v[1].color*nv + tri.v[2].color*nw;

// 光照着色
Color litColor = phongLighting(norm, viewDir, col, wp);
ctx.colorBuffer->at(px, py) = litColor;

// 计算当前像素的速度矢量
// 注意：用同样的透视校正权重插值当前帧和上一帧的屏幕坐标
Vec2 curScr  = { scr[0].x*nu + scr[1].x*nv + scr[2].x*nw,
                  scr[0].y*nu + scr[1].y*nv + scr[2].y*nw };
Vec2 prevScrP = { prevScr[0].x*nu + prevScr[1].x*nv + prevScr[2].x*nw,
                   prevScr[0].y*nu + prevScr[1].y*nv + prevScr[2].y*nw };
Vec2 velocity = curScr - prevScrP;  // 单位：像素
ctx.velocityBuffer->at(px, py) = velocity;
```

为什么速度也要用透视校正插值？因为屏幕坐标是透视变换后的结果，直接线性插值会引入透视误差，尤其在大近平面 FOV 或靠近相机的物体上会出现明显的撕裂感。

### 4.4 后处理运动模糊核心

```cpp
Image applyMotionBlur(const Image& src, const VelocityBuffer& vel, int numSamples = 16) {
    int w = src.width, h = src.height;
    Image result(w, h);
    
    for(int y=0; y<h; y++){
        for(int x=0; x<w; x++){
            Vec2 v = vel.at(x, y);
            float speed = v.length();
            
            // 速度阈值：< 0.5 像素视为静止，跳过模糊
            // 为什么是 0.5？单个像素宽度以内的运动肉眼看不出来
            if(speed < 0.5f){
                result.at(x,y) = src.at(x,y);
                continue;
            }
            
            // 双向采样：t 从 -0.5 到 +0.5，当前像素在中间
            Color sum;
            int validSamples = 0;
            for(int i=0; i<numSamples; i++){
                float t = ((float)i / (numSamples-1)) - 0.5f;
                int sx = (int)(x + v.x * t + 0.5f);  // +0.5f 是四舍五入
                int sy = (int)(y + v.y * t + 0.5f);
                // 边界检查：采样点可能超出画面
                if(sx >= 0 && sx < w && sy >= 0 && sy < h){
                    sum = sum + src.at(sx, sy);
                    validSamples++;
                }
            }
            if(validSamples > 0){
                result.at(x,y) = sum * (1.f / validSamples);
            } else {
                result.at(x,y) = src.at(x,y);
            }
        }
    }
    return result;
}
```

24 个样本的原因：对于运动矢量最大约 277 像素的情况，24 个样本间距约 11 像素，足够覆盖模糊效果而不会出现明显的"重影"离散样点。

### 4.5 场景运动设置

为了展示不同类型的运动模糊，我们设计了三种运动方式：

```cpp
// 1. 快速旋转球（角速度3 rad/frame，每帧旋转约172°）
auto sphereModelFn = [&](float t) -> Mat4 {
    float angle = t * 3.f;
    return makeTranslation(-1.5f, 0.2f, 0.f) * makeRotationY(angle) * makeScale(0.7f,0.7f,0.7f);
};

// 2. 双轴旋转立方体（角速度2和1.5 rad/frame）
auto cubeModelFn = [&](float t) -> Mat4 {
    float angle = t * 2.f;
    float angle2 = t * 1.5f;
    return makeTranslation(1.5f, 0.0f, 0.f) * makeRotationY(angle) * makeRotationX(angle2) * makeScale(0.6f,0.6f,0.6f);
};

// 3. 轨道运动小球（角速度4 rad/frame，运动最快）
auto orbitModelFn = [&](float t) -> Mat4 {
    float angle = t * 4.f;
    float ox = 2.8f * std::cos(angle);
    float oz = 2.8f * std::sin(angle);
    return makeTranslation(ox, -0.3f, oz) * makeScale(0.35f, 0.35f, 0.35f);
};

// 4. 相机轨道运动（角速度0.3 rad/frame，慢速）
auto getCameraPos = [&](float t) -> Vec3 {
    float angle = t * 0.3f;
    return Vec3(5.f * std::cos(angle), 2.5f, 5.f * std::sin(angle));
};
```

为什么用 $t_{curr} = 1.0$, $t_{prev} = 0.0$？这相当于模拟第30帧（时间=1秒）到第31帧的变化，此时物体已经积累了足够的角速度，速度矢量在屏幕上显著可见。

### 4.6 速度可视化

```cpp
Image visualizeVelocity(const VelocityBuffer& vel, int width, int height) {
    Image img(width, height);
    // 找全图最大速度，用于归一化
    float maxSpeed = 0.f;
    for(int y=0;y<height;y++)
        for(int x=0;x<width;x++)
            maxSpeed = std::max(maxSpeed, vel.at(x,y).length());
    if(maxSpeed < 0.001f) maxSpeed = 1.f;
    
    for(int y=0;y<height;y++){
        for(int x=0;x<width;x++){
            Vec2 v = vel.at(x,y);
            // R 通道：X 方向速度（正值=向右）
            // G 通道：Y 方向速度（正值=向下）
            // B 通道：速度幅度（越亮=移动越快）
            float r = std::max(0.f, v.x) / maxSpeed;
            float g = std::max(0.f, v.y) / maxSpeed;
            float b = v.length() / maxSpeed;
            img.at(x,y) = Color(r, g, b);
        }
    }
    return img;
}
```

速度可视化是调试运动模糊的重要工具。通过颜色编码：
- 纯红色区域：物体向右移动
- 纯绿色区域：物体向下移动
- 青色/亮色区域：高速运动

背景（天空）保持黑色，因为速度为零。

---

## 五、踩坑实录

### Bug 1：背面剔除方向错误导致场景空白

**症状**：渲染结果是纯天空色，没有任何几何体出现。

**错误假设**：以为背面剔除条件 `cross2d >= 0` 是正确的（通常教程都这么写）。

**真实原因**：叉积符号与坐标系约定相关。我们的屏幕空间 Y 轴向下（图像坐标），NDC 转屏幕时做了 Y 翻转，导致三角形顶点的顺逆时针和直觉相反。翻转后，屏幕空间的逆时针三角形（正面朝向屏幕）对应的叉积是负值，而不是正值。

**修复方式**：将背面剔除条件从 `cross2d <= 0` 改为 `cross2d >= 0`（去掉 if 为 true 就 `continue` 的逻辑）。实际上，在调试过程中，直接关闭背面剔除确认三角形能渲染，然后再调整符号，是最快的调试路径。

**教训**：坐标系问题是图形学中最隐蔽的 Bug 源。遇到场景为空、全黑、全白这三种异常，第一步不是检查光照，而是检查坐标系约定。

### Bug 2：速度缓冲几乎全零

**症状**：速度可视化图是纯黑色，模糊效果完全没有。

**错误假设**：以为速度计算公式没问题，怀疑是浮点精度问题。

**真实原因**：`renderMesh` 函数签名中，`prevModel` 参数被传入了与 `model` 相同的值。调用处：
```cpp
// 错误的调用（两个参数相同，速度永远为零）
renderMesh(ctx, sphereMesh, sphereModelFn(tCurr), sphereModelFn(tCurr));
// 正确的调用
renderMesh(ctx, sphereMesh, sphereModelFn(tCurr), sphereModelFn(tPrev));
```
当前帧和上一帧使用相同变换矩阵，导致速度恒为零。

**修复方式**：在每次调用 `renderMesh` 时，确保传入 `modelFn(tCurr)` 和 `modelFn(tPrev)` 两个不同的时间参数。

**教训**：当结果"太好看（全零/全白/完全对称）"时，通常是某个参数传错了。速度缓冲全黑不是"效果好"，是数据错误。

### Bug 3：运动模糊只模糊边缘，中间区域没有效果

**症状**：快速旋转的球体边缘有模糊，但球体中心部分几乎没有变化。

**错误假设**：以为这是正常的物理效果（物体中心相对静止）。

**真实原因**：球体表面各个点的世界坐标在旋转时会移动，但球体中心点在模型空间的原点，旋转不改变原点的世界坐标，所以中心的速度矢量确实很小。这其实是**正确**的物理行为——旋转轴上的点没有切向速度。

验证：轨道运动的小球（整体平移）会产生全球面均匀的运动模糊，与旋转球体的"边缘模糊"形成明显对比。

**结论**：这不是 Bug，而是速度缓冲正确区分了不同类型运动的体现。通过对比轨道小球和旋转球，可以清晰地看到平移运动和旋转运动的模糊特征差异。

### Bug 4：编译警告导致 0 警告目标失败

**症状**：`g++ -Wall -Wextra` 给出两个警告：
- `warning: variable 'e0' set but not used`
- `warning: unused variable 'dt'`

**原因**：
1. 调试期间写了 `Vec3 e0 = viewNormals[0] + ...` 作为中间变量（原本想用来辅助法线判断），后来改用屏幕空间叉积，忘记删掉 `e0`。
2. `dt` 是帧时间，本来打算用于物理计算，后来直接在 `modelFn` 里用帧序号计算角度，`dt` 成了死代码。

**修复**：直接删除两行：
```cpp
// 删除这行
Vec3 e0 = viewNormals[0] + viewNormals[1] + viewNormals[2];
// 删除这行  
float dt = 1.0f / 30.f;
```

**教训**：始终用 `-Wall -Wextra` 编译，把警告当错误对待。未使用的变量通常是删改代码后留下的残留，积累多了会增加阅读难度。

---

## 六、效果验证与数据

### 6.1 量化验证结果

程序内置了三项量化检查：

| 检查项 | 结果 | 标准 | 状态 |
|--------|------|------|------|
| 输出图像均值 | 143.5 / 255 | 10~240 之间 | ✅ |
| 输出图像标准差 | 52.1 | > 5 | ✅ |
| 速度缓冲非零像素 | 30,742 / 307,200 | > 1000 | ✅ |
| 最大速度幅度 | 276.9 像素/帧 | 有效值 | ✅ |
| 模糊均值差异 | 6.33 / 255 | > 0.5 | ✅ |
| 受模糊影响像素数 | 25,560 | 有效值 | ✅ |

**像素均值 143.5** 说明场景亮度适中，不过暗也不过亮（天空蓝 + Phong 光照组合）。

**标准差 52.1** 说明图像有丰富的颜色变化，不是单调的渐变或纯色块。

**30,742 个运动像素**（占总像素 10%）是速度 > 0.5 像素的区域，主要集中在旋转球体、立方体和轨道小球上。

**最大速度 277 像素**来自轨道运动小球（半径 2.8 单位 × 角速度 4 rad/帧，投影到屏幕后的位移）。

### 6.2 输出文件

| 文件 | 尺寸 | 描述 |
|------|------|------|
| `motion_blur_noamb.png` | 640×480, 22KB | 无运动模糊的原始渲染 |
| `motion_blur_output.png` | 640×480, 41KB | 应用运动模糊后的结果 |
| `motion_blur_velocity.png` | 640×480, 31KB | 速度缓冲可视化 |
| `motion_blur_compare.png` | 1280×480, 63KB | 左右对比图 |

注意对比图（63KB）和原图（22KB）的大小关系：模糊后的图像 PNG 压缩率反而更低（41KB > 22KB）。这符合直觉——模糊减少了颜色的锐利边缘，但增加了颜色的过渡复杂性，PNG 压缩对渐变的效率不如对锐利边缘高。

### 6.3 视觉对比

**无运动模糊**（左）：
- 旋转球体边缘清晰锐利
- 快速运动的轨道小球完全清晰
- 相机运动导致整个背景有轻微位置变化，但无法感知

![无运动模糊](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-27-motion-blur/motion_blur_noamb.png)

**运动模糊**（右）：
- 旋转球体边缘出现沿旋转方向的拖尾
- 轨道小球变成明显的模糊条带
- 立方体双轴旋转产生复杂的多方向模糊

![运动模糊效果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-27-motion-blur/motion_blur_output.png)

**速度缓冲可视化**：
- 背景为黑色（速度零）
- 物体区域有彩色编码：R=X速度，G=Y速度，B=速度幅度
- 轨道小球区域最亮（速度最大）

![速度缓冲可视化](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-27-motion-blur/motion_blur_velocity.png)

**对比图**（左无模糊，右有模糊）：

![对比图](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/03/03-27-motion-blur/motion_blur_compare.png)

### 6.4 编译与运行性能

| 阶段 | 耗时 |
|------|------|
| 编译（g++ -O2） | < 2 秒 |
| 渲染（640×480，4个物体） | < 0.1 秒 |
| 后处理模糊（24 样本） | < 0.5 秒 |
| 总运行时间 | < 1 秒 |

代码量：~750 行 C++，单文件，零外部依赖（除 stb_image_write.h）。

---

## 七、总结与延伸

### 7.1 这次学到了什么

今天实现的速度缓冲 + 后处理运动模糊，是现代实时渲染引擎中最常见的运动模糊方案。核心洞察是：

1. **速度缓冲的正交性**：速度信息独立于颜色信息，可以按需开关运动模糊，不影响其他渲染效果。
2. **双向采样的物理意义**：采样范围 [-0.5, 0.5] 模拟了快门时间的对称性，比单向采样更符合物理。
3. **透视校正不可省**：速度矢量也需要透视校正插值，否则靠近相机的物体会产生速度扭曲。

### 7.2 技术局限性

**线性运动假设**：当前方案假设物体在一帧内做线性运动。对于急剧加速的弹射物、高速旋转物体的边缘像素，线性近似会有误差（模糊方向不完全准确）。

**Tile-based 优化缺失**：当前是逐像素遍历速度缓冲，没有按速度大小分块跳过静止区域。对于大量静止物体的场景（如室内场景），可以通过 tile max velocity 大幅减少无效采样。

**深度不连续问题**：物体边缘处，背景和前景速度不同。当前实现用前景物体的速度来模糊背景边缘，会产生轻微的"背景漏色"（Background Bleeding）。工业级实现会在采样时做深度比较，拒绝不合理的采样点。

**透明物体**：速度缓冲只存储最近深度的速度，不支持透明物体的运动模糊。

### 7.3 可优化的方向

**Tile Max Velocity**：将速度缓冲划分为 NxN 的小格（tile），每格存储最大速度幅度。后处理时，对最大速度为零的 tile 直接跳过，大幅减少采样次数。

**Neighborhood Clamping**：参考 TAA 的做法，在运动模糊采样时也做颜色邻域约束，减少背景漏色。

**Motion Vector Dilation（速度扩张）**：物体边缘的像素（z 缓冲不连续处）通常属于背景，但因为背景没有速度，模糊会在边缘产生锐利边界。通过扩张前景物体的速度到边缘像素，可以让模糊更平滑。

**变形体（Skinned Mesh）支持**：当前方案只支持刚体运动（通过模型矩阵控制）。要支持骨骼动画，需要在顶点着色器中分别计算当前帧和上一帧的蒙皮变换，再算速度差。

**深度感知采样权重**：在采样沿速度方向时，加入深度权重，让深度相近的样本权重高，深度差异大的样本权重低，缓解背景漏色。

### 7.4 与本系列的关联

- **TAA（03-26）**：TAA 利用历史帧颜色 + 重投影 + Jitter，而运动模糊利用速度缓冲 + 后处理。两者都需要计算"上一帧在哪里"，但目的不同：TAA 求稳定，运动模糊求临场感。在实际引擎中，TAA 和运动模糊共享同一张速度缓冲 Pass，是天然的好邻居。
- **SSAO（03-25）**：同样是后处理 Pass，读取深度缓冲。运动模糊 + SSAO + TAA 组合是现代实时渲染 Post-Processing 的标准三件套。
- **Phong 着色（早期作品）**：今天的渲染器复用了 Phong 光照模型，双光源 + 环境光，与之前的光线追踪作品保持一致的视觉风格。

---

代码仓库：[daily-coding-practice/2026/03/03-27-motion-blur](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/03/03-27-motion-blur)

图片资源：[blog_img/2026/03/03-27-motion-blur](https://github.com/chiuhoukazusa/blog_img/tree/main/2026/03/03-27-motion-blur)
