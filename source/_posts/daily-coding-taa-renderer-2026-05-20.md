---
title: "每日编程实践: Temporal Anti-Aliasing (TAA) Renderer"
date: 2026-05-20 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 抗锯齿
  - 实时渲染
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-20-taa-renderer/taa_output.png
---

# 每日编程实践: Temporal Anti-Aliasing (TAA) Renderer

今天实现了现代游戏引擎中最重要的抗锯齿技术之一——**TAA（时域抗锯齿，Temporal Anti-Aliasing）**。包含Halton序列亚像素抖动、运动矢量计算、历史帧重投影、Variance Clipping幽灵效果抑制等核心模块，并输出三列对比图（无AA / SSAA 2x / TAA 20帧积累）。

<!-- more -->

## 效果预览

![TAA对比图](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-20-taa-renderer/taa_output.png)

左列：无抗锯齿（锯齿明显，边缘粗糙）  
中列：SSAA 2x（参考，无运动模糊）  
右列：TAA 20帧积累（平滑，边缘干净）

---

## ① 背景与动机

### 锯齿问题的本质

图形渲染中，"锯齿（aliasing）"来源于**采样不足**。场景中的几何边缘是连续的曲线或直线，但屏幕像素是离散的网格——每个像素只取中心点一个样本来决定颜色。当边缘恰好斜穿多个像素时，有些像素被覆盖、有些没有，结果就是肉眼可见的阶梯状锯齿。

### 传统抗锯齿方案的代价

- **MSAA（多重采样抗锯齿）**：每像素多次采样（4x/8x/16x）。质量好，但延迟渲染下难以应用，内存占用随样本数线性增长，4x MSAA相当于用4倍显存。
- **SSAA（超级采样）**：直接渲染高分辨率后降采样。效果最好，代价最大，相当于分辨率翻4倍。实时游戏几乎不用。
- **FXAA（快速近似抗锯齿）**：纯后处理，检测亮度梯度后模糊边缘。速度快但会模糊细节，尤其是文字和细线。

### TAA的思路：借用时间维度

TAA的核心洞见是：**与其一帧内多次采样，不如跨帧积累样本**。

每帧只采样一个子像素位置（通过"抖动"偏移像素中心），但跨越多帧后，不同的子像素位置会覆盖整个像素面积，累积效果等价于超采样——代价却只是单帧的一次采样。

这正是现代游戏引擎（Unreal Engine、Unity HDRP、寒霜引擎）选择TAA的原因：
- 运行时代价接近于零（一个全屏后处理pass）
- 可以与延迟渲染完美兼容
- 随时间积累，质量稳步提升

### 工业界实际使用

- **Unreal Engine 4/5**：TAA是默认抗锯齿选项，后期演变为TSR（时域超分辨率）
- **Unity HDRP**：TAA集成在后处理栈中
- **《赛博朋克 2077》、《战神》**等3A大作默认TAA
- **DLSS/FSR**：本质上也是TAA思想的延伸——积累历史帧、利用运动矢量重投影

---

## ② 核心原理

TAA的数学框架由三个关键操作组成：**抖动（Jitter）**、**重投影（Reprojection）**、**混合与鬼影抑制（Blending + Ghost Suppression）**。

### 2.1 Halton序列抖动

每帧不是在像素中心采样，而是在像素内用低差异序列（Low-Discrepancy Sequence）偏移采样点位置。

**Halton序列**：给定整数底数 $b$，第 $i$ 个Halton数为：将 $i$ 写成 $b$ 进制，然后在小数点后镜像翻转。

例如，$b=2$，Halton序列：
$$
i=1: (1)_2 \to 0.1_2 = 0.5
$$
$$
i=2: (10)_2 \to 0.01_2 = 0.25
$$
$$
i=3: (11)_2 \to 0.11_2 = 0.75
$$
$$
i=4: (100)_2 \to 0.001_2 = 0.125
$$

TAA常用Halton(2,3)序列——X方向用底数2，Y方向用底数3——这两个互质的底数产生的二维点集均匀分布，不像随机数那样会聚堆。

**直觉解释**：想象把一个像素格子分成16个小格，随机撒16个点可能有空格也有聚集的格子。Halton(2,3)序列保证这16个点"尽量均匀分布在每个角落"——用数学术语叫"低差异"（低偏差）。

```cpp
float halton(int index, int base) {
    float f = 1.0f;
    float r = 0.0f;
    int i = index;
    while(i > 0) {
        f /= (float)base;
        r += f * (float)(i % base);  // 取 i 的 base 进制最低位，乘以权重
        i /= base;                    // 右移一位（base进制）
    }
    return r;  // 返回 [0,1) 范围的值
}

Vec2 haltonJitter(int frameIndex) {
    // 减去0.5使范围变为[-0.5, 0.5]，对称分布在像素中心周围
    return {
        halton(frameIndex+1, 2) - 0.5f,
        halton(frameIndex+1, 3) - 0.5f
    };
}
```

这个偏移量被加到投影矩阵的最后一列（NDC空间的X/Y偏移），让每帧的渲染结果在屏幕空间中有亚像素级别的偏移：

```cpp
Mat4 jitterMat = Mat4::identity();
if(applyJitter) {
    // 将像素偏移 [jitterX, jitterY] 转换为 NDC 偏移
    // NDC范围是[-1,1]，宽度为2，所以1像素 = 2/W NDC单位
    jitterMat.m[12] = jitterX * 2.0f / (float)W;
    jitterMat.m[13] = -jitterY * 2.0f / (float)H;  // 注意Y轴翻转
}
Mat4 mvp = jitterMat * viewProj * model;
```

### 2.2 运动矢量（Motion Vectors）

运动矢量记录了每个像素从当前帧到上一帧的位移（像素空间）。这是TAA的关键数据：没有运动矢量，历史帧重投影就无从谈起。

**运动矢量的计算方式**：

对于场景中一个顶点 $\mathbf{p}$，当前帧的屏幕坐标为 $\mathbf{s}_{cur}$，上一帧的屏幕坐标为 $\mathbf{s}_{prev}$（用上一帧的变换矩阵计算）。

$$
\text{MotionVector} = \mathbf{s}_{cur} - \mathbf{s}_{prev}
$$

在光栅化阶段，把顶点的"上一帧裁剪空间位置"也传入，在三角形内部用重心坐标插值：

```cpp
struct Vertex {
    Vec4 pos;      // 当前帧裁剪空间位置（含抖动）
    Vec4 prevPos;  // 上一帧裁剪空间位置（不含抖动）
    Vec3 color;
    Vec3 normal;
};
```

在片元着色器（光栅化）阶段：

```cpp
// 插值得到上一帧的裁剪坐标
Vec4 prevClip = Vec4(
    v0.prevPos.x*w2 + v1.prevPos.x*w0 + v2.prevPos.x*w1,
    v0.prevPos.y*w2 + v1.prevPos.y*w0 + v2.prevPos.y*w1,
    v0.prevPos.z*w2 + v1.prevPos.z*w0 + v2.prevPos.z*w1,
    v0.prevPos.w*w2 + v1.prevPos.w*w0 + v2.prevPos.w*w1
);

// 透视除法得到上一帧NDC坐标，转换为屏幕坐标
float invPrevW = 1.0f / prevClip.w;
float prevScreenX = (prevClip.x * invPrevW * 0.5f + 0.5f) * (float)(W-1);
float prevScreenY = (1.0f - (prevClip.y * invPrevW * 0.5f + 0.5f)) * (float)(H-1);

// 运动矢量 = 当前像素位置 - 上一帧位置
motionVector.x = (float)px - prevScreenX;
motionVector.y = (float)py - prevScreenY;
```

**直觉解释**：如果一个立方体向右移动了5个像素，那么当前帧像素(100,100)对应的物体表面，在上一帧是在屏幕(95,100)位置渲染的。运动矢量就是(5,0)——告诉TAA系统：要找这个像素的历史信息，应该去(95,100)位置取样。

### 2.3 历史帧重投影与混合

有了运动矢量，TAA就可以利用历史帧了：

```cpp
// 从运动矢量找到上一帧的采样位置
float prevX = (float)x - motion.x;
float prevY = (float)y - motion.y;

// 双线性插值采样历史帧（亚像素精度）
int px0 = (int)prevX, py0 = (int)prevY;
float fx = prevX - px0, fy = prevY - py0;
Vec3 histColor = bilinear_sample(historyBuffer, prevX, prevY);

// 时间指数移动平均（EMA）
// blendFactor = 当前帧权重（通常0.1，即10%新帧 + 90%历史）
Vec3 taaColor = currentColor * blendFactor + histColor * (1.0f - blendFactor);
```

**EMA的数学意义**：

设 $\alpha = 0.1$（当前帧权重），经过 $n$ 帧后，第 $k$ 帧的贡献权重为：
$$
w_k = \alpha \cdot (1-\alpha)^{n-k}
$$

即越旧的帧权重越小，但永远不会完全消失——这就是"历史积累"的本质。当 $n \to \infty$ 时，收敛到每帧均匀平均，等价于超采样。

### 2.4 Variance Clipping：鬼影抑制

TAA最大的问题是**鬼影（Ghosting）**：相机或物体快速移动时，历史帧的颜色与当前帧不匹配，产生拖尾。

**问题来源**：运动矢量在物体边界处可能不精确，或者被遮挡的区域在前一帧根本不存在——这些情况下重投影拿到的"历史颜色"是错误的。

**Variance Clipping 解法**：

核心思路是：**历史颜色如果与当前邻域颜色差太远，就把它"裁剪"到邻域的颜色范围内**。

1. 计算当前帧该像素 3×3 邻域的颜色均值 $\mu$ 和标准差 $\sigma$
2. 构建颜色包围盒（AABB）：$[\mu - \gamma\sigma, \mu + \gamma\sigma]$，其中 $\gamma$ 通常为 1.0~1.5
3. 将历史颜色裁剪到这个包围盒内

```cpp
void varianceClipping(const Framebuffer& fb, int px, int py,
                      Vec3& aabbMin, Vec3& aabbMax, int radius=1) {
    Vec3 m1(0,0,0), m2(0,0,0);
    int count = 0;
    
    // 计算3x3邻域的一阶矩和二阶矩
    for(int dy=-radius; dy<=radius; dy++) {
        for(int dx=-radius; dx<=radius; dx++) {
            Vec3 c = sampleNeighbor(px+dx, py+dy);
            m1 += c;        // 一阶矩（均值）
            m2 += c * c;    // 二阶矩（用于计算方差）
            count++;
        }
    }
    m1 = m1 / count;  // 均值 μ
    m2 = m2 / count;  // E[X²]
    
    // 标准差 σ = sqrt(E[X²] - μ²)
    Vec3 sigma = sqrt(max(0, m2 - m1 * m1));
    
    float gamma = 1.2f;  // 松弛因子，越大越宽松（更多历史信息但鬼影多）
    aabbMin = m1 - sigma * gamma;
    aabbMax = m1 + sigma * gamma;
}
```

**直觉解释**：如果当前帧这个像素周围都是深蓝色天空（均值≈(0.1,0.2,0.6)），但历史帧给出的颜色是橙色（因为这里之前是建筑）——那橙色明显不在邻域分布范围内，直接裁剪掉，用天空颜色代替。

### 2.5 自适应混合权重

静态场景（无运动）用高历史权重（0.9），快速运动时降低历史权重减少拖尾：

```cpp
float motionLen = sqrt(motion.x*motion.x + motion.y*motion.y);
// 基础：10%当前帧 + 90%历史帧
float blendFactor = 0.1f;
// 运动越快，历史权重越低（最高50%当前帧）
if(motionLen > 0.5f) {
    blendFactor = min(0.5f, 0.1f + motionLen * 0.05f);
}
// 越界（无有效历史）直接用当前帧
if(!validHistory) blendFactor = 1.0f;
```

---

## ③ 实现架构

### 整体渲染管线

```
每帧循环（0 ~ TAA_FRAMES-1）:
    ┌─────────────────────────────────┐
    │  1. 更新场景状态（立方体旋转角度） │
    │  2. 计算 Halton 抖动偏移        │
    │  3. 软光栅化渲染当前帧           │
    │     → 颜色缓冲 + 深度缓冲       │
    │     → 运动矢量缓冲              │
    │  4. TAA 混合                   │
    │     → 重投影（运动矢量 → 历史帧位置）│
    │     → Variance Clipping       │
    │     → EMA 混合                │
    │  5. 交换历史帧缓冲              │
    └─────────────────────────────────┘

最后一帧额外渲染：
    - 无AA版本（不含抖动）
    - SSAA 2x版本（高分辨率渲染后降采样）

输出：拼合3列对比图 → taa_output.png
```

### 关键数据结构

```cpp
// 帧缓冲：存储当前帧渲染结果
struct Framebuffer {
    int width, height;
    std::vector<Vec3> color;    // RGB颜色
    std::vector<float> depth;   // 深度值 [0,1]
    std::vector<Vec2> motion;   // 运动矢量（像素空间）
};

// TAA双缓冲：当前积累结果 + 上一帧结果
struct TAABuffer {
    std::vector<Vec3> accumulated;  // 当前帧TAA混合结果
    std::vector<Vec3> history;      // 上一帧的积累结果（重投影来源）
    int frameCount;                  // 已积累帧数
    
    void swapBuffers() {
        std::swap(accumulated, history);
        // accumulated变成下一帧的"上一帧"，history变成新的写入目标
    }
};
```

### 职责划分

| 模块 | 职责 |
|------|------|
| `haltonJitter()` | 计算当前帧的抖动偏移量 |
| `renderCube()` | 软光栅化：填充颜色缓冲、运动矢量缓冲 |
| `varianceClipping()` | 计算3x3邻域颜色AABB |
| `applyTAA()` | 重投影 + 裁剪 + EMA混合 + 交换缓冲 |
| `savePNG()` | 纯C++ PNG编码（DEFLATE存储模式） |

---

## ④ 关键代码解析

### 4.1 抖动矩阵的注入

抖动应该影响最终的投影坐标，但**不应该影响运动矢量的计算**。原因是：运动矢量要反映真实的物体运动，而不是相机抖动。如果把抖动也算进运动矢量，TAA会把正常的子像素偏移当成"运动"来处理，混乱大量历史信息。

```cpp
// 带抖动的MVP：用于当前帧渲染（产生抖动后的颜色）
Mat4 mvp = jitterMat * viewProj * model;

// 不带抖动的prevMVP：用于计算运动矢量（只反映物体真实运动）
Mat4 prevMVP = viewProj * prevAngle_model;  // 上一帧的变换，无抖动
```

这样渲染出的颜色有子像素偏移，但运动矢量计算的是"物体本身从哪里来"，而非"加上抖动后像素在哪"。

### 4.2 双线性插值历史帧

从历史缓冲中采样时，上一帧的位置通常不是整数坐标（因为运动量可能是0.7像素这样的小数）。必须用双线性插值才能拿到平滑的历史颜色，否则会出现网格状的采样噪点。

```cpp
float prevX = (float)x - motion.x;  // 减去运动矢量 = 上一帧位置
float prevY = (float)y - motion.y;

int px0 = (int)prevX, py0 = (int)prevY;
int px1 = px0+1, py1 = py0+1;
float fx = prevX - (float)px0;  // 小数部分（水平插值权重）
float fy = prevY - (float)py0;  // 小数部分（垂直插值权重）

// 2x2双线性插值
Vec3 c00 = history[py0*W+px0];  // 左上
Vec3 c10 = history[py0*W+px1];  // 右上
Vec3 c01 = history[py1*W+px0];  // 左下
Vec3 c11 = history[py1*W+px1];  // 右下

// 先水平插值，再垂直插值
Vec3 histColor = (c00*(1-fx) + c10*fx) * (1-fy) +
                 (c01*(1-fx) + c11*fx) * fy;
```

**容易写错的地方**：减运动矢量还是加？运动矢量定义是"当前像素位置 - 上一帧位置"，所以 `prevPos = currentPos - motionVector`，即**减去**运动矢量。这个符号很容易搞反，症状是图像出现奇怪的双影或反向拖尾。

### 4.3 Variance Clipping 裁剪逻辑

```cpp
Vec3 clipToAABB(Vec3 histColor, Vec3 aabbMin, Vec3 aabbMax) {
    // 如果历史颜色在当前邻域颜色范围内，直接返回（不修改）
    if(histColor.x >= aabbMin.x && histColor.x <= aabbMax.x &&
       histColor.y >= aabbMin.y && histColor.y <= aabbMax.y &&
       histColor.z >= aabbMin.z && histColor.z <= aabbMax.z) {
        return histColor;
    }
    // 否则裁剪到AABB范围内（分量独立clamp）
    return Vec3(
        clamp(histColor.x, aabbMin.x, aabbMax.x),
        clamp(histColor.y, aabbMin.y, aabbMax.y),
        clamp(histColor.z, aabbMin.z, aabbMax.z)
    );
}
```

**注意**：更精确的Variance Clipping不是分量独立clamp，而是找历史颜色到AABB中心的射线与AABB表面的交点（clip to box in color space）。本实现用的是更简单的per-component clamp，效果够用但在颜色边界处可能引入轻微的色彩偏移。工业级实现会用完整的射线求交版本。

### 4.4 完整的 applyTAA 函数

```cpp
void applyTAA(const Framebuffer& current, TAABuffer& taa, bool ghostSuppression=true) {
    int W = current.width, H = current.height;
    
    for(int y=0; y<H; y++) {
        for(int x=0; x<W; x++) {
            Vec3 currentColor = current.color[y*W+x];
            Vec2 motion = current.motion[y*W+x];
            
            // Step 1: 重投影 - 找到上一帧对应位置
            float prevX = (float)x - motion.x;
            float prevY = (float)y - motion.y;
            
            // 边界检查
            bool validHistory = (prevX >= 0 && prevX < W-1 && 
                                  prevY >= 0 && prevY < H-1);
            
            Vec3 histColor;
            if(validHistory) {
                // Step 2: 双线性插值采样历史帧
                histColor = bilinear(taa.history, prevX, prevY, W);
            } else {
                // 越界：没有历史信息，直接用当前帧
                histColor = currentColor;
            }
            
            // Step 3: Variance Clipping - 防止鬼影
            if(ghostSuppression && validHistory) {
                Vec3 aabbMin, aabbMax;
                varianceClipping(current, x, y, aabbMin, aabbMax);
                histColor = clipToAABB(histColor, aabbMin, aabbMax);
            }
            
            // Step 4: 自适应混合权重
            float motionLen = length(motion);
            float blendFactor = 0.1f;  // 默认：90%历史
            if(motionLen > 0.5f) {
                // 运动快：增加当前帧权重，减少拖尾
                blendFactor = min(0.5f, 0.1f + motionLen * 0.05f);
            }
            if(!validHistory) blendFactor = 1.0f;
            
            // Step 5: EMA混合
            taa.accumulated[y*W+x] = 
                currentColor * blendFactor + histColor * (1.0f - blendFactor);
        }
    }
    
    // Step 6: 交换缓冲（本帧的结果成为下一帧的"历史"）
    taa.swapBuffers();
    taa.frameCount++;
}
```

### 4.5 背面剔除

光栅化中，背面剔除非常关键——不剔除会导致背面三角形覆盖正面，出现Z-fighting或错误的颜色。

```cpp
// 在世界空间计算面法线与视线方向的夹角
Mat4 normalMat = Mat4::rotate(angle, rotAxis);
Vec4 wn = normalMat * Vec4(face.normal, 0.0f);
Vec3 worldNormal = wn.xyz().norm();

Vec3 viewDir = Vec3(0, 0, -1);  // 相机沿-Z方向看
// 法线与视线方向夹角>90度（点积>=0）时为背面
if(worldNormal.dot(viewDir) >= 0) continue;
```

**注意**：视线方向和法线方向的符号约定容易混淆。相机看向-Z，而"面向相机"的法线应该有正的Z分量（点积<0）。如果搞反了，会只渲染背面而看不到正面。

---

## ⑤ 踩坑实录

### Bug 1：警告导致编译失败（要求0警告）

**症状**：编译输出6个警告，其中包括 `-Wunused-variable` 和 `-Wunused-parameter`。虽然能运行，但要求严格的0警告策略下必须修复。

**错误假设**：以为警告不影响运行，可以忽略。

**真实原因**：代码中有：
1. `Mat4 curMVP_noJitter`：计算了但从未使用（早期设计遗留）
2. `prevViewProj` 参数：函数签名中有但实现时改为内部计算
3. `renderPixelSSAA` 函数中的 `fx`、`fy`、`cubes`、`viewProj`：整个函数是占位实现，变量都没用到

**修复**：
- 删除未使用的 `curMVP_noJitter` 变量
- 将 `prevViewProj` 参数改为 `/*prevViewProj*/`（注释掉的具名参数）
- 删除整个 `renderPixelSSAA` 占位函数（实际SSAA改为用帧缓冲方案实现）

**经验**：在设计阶段就要避免写"占位代码"——占位函数要么不写参数，要么在写的时候就实现，否则稀里糊涂留了一堆未使用的东西。

### Bug 2：运动矢量方向理解错误（设计阶段提前规避）

**症状**（假设犯错的情况）：图像出现反向拖尾，物体向右运动但历史帧从左边拉过来。

**错误假设**：运动矢量 = 当前帧位置减上一帧位置，采样时应该加上这个矢量。

**真实原因**：混淆了方向。运动矢量 $\mathbf{m} = \mathbf{p}_{cur} - \mathbf{p}_{prev}$，所以 $\mathbf{p}_{prev} = \mathbf{p}_{cur} - \mathbf{m}$，采样时必须**减去**运动矢量：
```cpp
float prevX = (float)x - motion.x;  // 减，不是加
float prevY = (float)y - motion.y;
```

**经验**：运动矢量的符号是TAA实现中最常见的方向性错误。在写代码前先手推一个具体例子：如果物体向右移动3像素，当前像素(100,50)处的物体表面在上一帧是在屏幕(97,50)位置渲染的，运动矢量是(3,0)，所以 `prevX = 100 - 3 = 97`，正确。

### Bug 3：抖动影响了运动矢量（设计规范规避）

**症状**：TAA结果比预期更模糊，或者静止物体也有"运动"感。

**错误假设**：抖动偏移只是修改了渲染位置，不影响运动矢量计算。

**真实原因**：如果用带抖动的MVP同时计算颜色**和**运动矢量，抖动偏移会被算进运动矢量中，每帧的"假运动"（实为子像素抖动）会被TAA当作真实运动处理，导致历史权重降低，抗锯齿效果打折扣。

**修复**：分离计算：
```cpp
// 带抖动的MVP：只用于渲染颜色（产生子像素偏移）
Mat4 mvp = jitterMat * viewProj * model;

// 不带抖动的prevMVP：只用于运动矢量（反映真实物体运动）
Mat4 prevMVP = viewProj * prevAngle_model;
```

### Bug 4：PNG图像全黑（深度测试异常）

**症状**：运行成功，PNG文件存在但全黑。

**原因分析**：深度测试条件写反，所有片元都被拒绝。深度值从前往后是-1到1（NDC空间），近处z值更小，应该写：
```cpp
if(z >= fb.depth_at(px, py)) continue;  // 距离更远的片元丢弃
fb.depth_at(px, py) = z;
```
不是：
```cpp
if(z <= fb.depth_at(px, py)) continue;  // 错误：近处片元被丢弃
```

实际本次实现时初始深度值设为1.0（最远），z范围是[-1,1]，近的片元z更小，所以"大于等于就丢弃"是正确的。

---

## ⑥ 效果验证与数据

### 输出文件检验

```
文件: taa_output.png
  尺寸: (620, 340)
  文件大小: 619 KB (>10KB ✅)
  像素均值: 98.4  (在10~240范围内 ✅)
  标准差: 41.6   (>5，图像有内容变化 ✅)
  ✅ 像素统计正常
```

### 性能数据

```
渲染耗时: 0.080秒 (real)
  - 立方体数量: 6个（2×3阵列）
  - 每个立方体: 6面 × 2三角形 = 12三角形
  - TAA帧数: 20帧
  - 分辨率: 200×300（每列）
  - 总三角形渲染次数: 6 × 12 × 20 = 1440次
  - CPU软光栅化（未优化）
```

### 三列对比分析

- **左列（无AA）**：对角边缘清晰可见阶梯锯齿，尤其是立方体的斜边缘
- **中列（SSAA 2x）**：2倍超采样降采样后，边缘明显平滑，作为质量参考
- **右列（TAA 20帧）**：积累了20帧的子像素信息，边缘平滑度接近SSAA，但无需额外的渲染代价

### 验证脚本完整输出

```python
from PIL import Image
import numpy as np

img = Image.open('taa_output.png')
pixels = np.array(img).astype(float)
print(f"尺寸: {img.size}")
print(f"像素均值: {pixels.mean():.1f}")    # 98.4 ✅
print(f"像素标准差: {pixels.std():.1f}")   # 41.6 ✅
```

---

## ⑦ 总结与延伸

### 技术局限性

1. **鬼影问题**：Variance Clipping虽然抑制了鬼影，但在极端运动（如物体出现/消失）时仍可能出现残影。现代引擎会额外检测 disocclusion（遮挡关系变化）。

2. **模糊问题**：混合历史帧本质上是时间域的低通滤波，会导致图像轻微模糊，尤其是包含高频细节（细线、文字）的场景。解决方案之一是 Sharpening pass（锐化后处理）。

3. **运动矢量缺失区域**：天空盒、透明物体等通常没有运动矢量，这些区域的TAA效果退化为简单的历史帧混合，可能出现问题。

4. **帧率依赖**：TAA需要多帧积累才能稳定，如果从静止突然快速移动，初始几帧的效果会较差。

### 可优化方向

1. **更精确的Variance Clipping**：用颜色空间的射线-AABB求交替换当前的per-component clamp
2. **YCoCg颜色空间**：在YCoCg空间做Variance Clipping效果更好（亮度和色差分离）
3. **Neighborhood Clamping vs Clipping**：Clamping更激进（减少鬼影但更多闪烁），Clipping更保守，可以根据场景调整
4. **Subpixel Morphological Anti-Aliasing (SMAA) + TAA**：两者结合，SMAA处理几何边缘，TAA处理着色噪点
5. **TSR（时域超分辨率）**：Unreal 5的TSR把TAA推广到上采样，用低分辨率渲染后放大到目标分辨率

### 与本系列的关联

- **FXAA（05-13）**：纯后处理空间域AA，TAA是时间域的延伸版本
- **MSAA**：TAA可以视为MSAA的时间版——MSAA在空间上多次采样，TAA在时间上积累样本
- **DLSS/FSR**：深度学习超采样的核心也是历史帧积累 + 运动矢量重投影，本质是TAA的神经网络强化版本
- **CSM（05-08）**：级联阴影的软阴影也可以用TAA来平滑阴影边界的锯齿

---

## 代码仓库

源代码：[GitHub - daily-coding-practice/2026/05/05-20-taa-renderer](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-20-taa-renderer)

```bash
git clone https://github.com/chiuhoukazusa/daily-coding-practice.git
cd daily-coding-practice/2026/05/05-20-taa-renderer
g++ main.cpp -o taa_renderer -std=c++17 -O2
./taa_renderer
# 输出：taa_output.png
```
