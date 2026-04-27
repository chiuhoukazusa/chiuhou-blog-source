---
title: "每日编程实践: Cel Shading & Outline Renderer（卡通渲染器）"
date: 2026-04-28 05:30:00
tags:
  - 每日一练
  - 图形学
  - NPR
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-28-Cel-Shading-Outline-Renderer/cel_shading_output.png
---

## 一、背景与动机

### 1.1 为什么要做卡通渲染（NPR）？

在过去几十天里，我们一路走过了 PBR、BDPT、SPPM、SSAO、TAA、DOF、Bloom……这些技术都指向同一个目标：**让渲染更像物理世界**。但今天我们要反其道而行之——主动让渲染"不像现实"。

**NPR（Non-Photorealistic Rendering，非真实感渲染）** 并不是图形学的边缘课题，它在工业界占有相当重要的地位：

- **游戏领域**：《原神》、《崩坏：星穹铁道》、《塞尔达传说》、《潜伏之赤途》……这些年玩家最喜欢的大作，很多都是卡通风格。米哈游曾在 GDC 分享过他们的 NPR 渲染管线，技术深度远超很多 PBR 游戏。
- **动画与影视**：3D 动画电影经常需要"手绘感"——迪士尼在《蜘蛛侠：平行宇宙》里做的描线和色块量化效果，成为了 NPR 领域的经典案例。
- **VR/AR 应用**：在低功耗设备上，卡通渲染往往比 PBR 便宜得多，因为它刻意减少了光照计算。

### 1.2 Cel Shading 的历史

"Cel"来源于 "Celluloid"（赛璐珞），就是传统手绘动画用的透明片。Cel Shading 模仿的就是那种上色方式：颜色被分成有限的几个色阶，亮部、中间调、暗部泾渭分明，不像 PBR 那样连续渐变。

最早的电子游戏卡通渲染可以追溯到 2000 年的《火影忍者》，但让这种风格大爆发的是 2002 年的《塞尔达传说：风之杖》——任天堂用 Cel Shading 给出了一个"如何让 3D 游戏像手绘动画"的完美答案。

### 1.3 今天要实现什么？

今天的项目实现一个**纯 CPU 软光栅 Cel Shading 渲染器**，包含：

1. **Phong 光照分级量化**：把连续的漫反射强度量化成 3 个色阶
2. **硬边高光**：Specular 不是连续的，而是"有"或"没有"的二值
3. **屏幕空间轮廓描边**：基于对象 ID 边界 + 法线不连续性的两重判断
4. **多个物体**：红色 Icosphere、蓝色 Torus、绿色小球、地面

输出是一张 800×600 的卡通风格渲染图，有明显的色块感和黑色轮廓线。

---

## 二、核心原理

### 2.1 色阶量化：连续光照的离散化

标准 Phong 漫反射是这样的：

```
diffuse = max(0, dot(N, L))
```

这会产生平滑的从亮到暗的过渡。但卡通渲染不要这种连续性，我们要把它"量化"：

**量化公式**：
```
steps = 3
diffuse_quantized = floor(NdotL * steps) / steps
```

直觉理解：把 [0, 1] 的范围切成 `steps` 段。`floor(x * 3) / 3` 的映射关系是：

- `NdotL ∈ [0, 1/3)` → `diffuse = 0`（暗部色块）
- `NdotL ∈ [1/3, 2/3)` → `diffuse = 1/3`（中间调色块）
- `NdotL ∈ [2/3, 1]`  → `diffuse = 2/3`（亮部色块）

这就是为什么卡通渲染看起来有明显的"跳变"，而不是平滑过渡——这正是我们想要的！

**为什么用 3 级？** 这是一个经验参数。2 级太简单（只有黑白），4 级以上开始接近 PBR 的平滑感。大多数日系卡通游戏使用 2-3 级。

### 2.2 硬边高光（Hard Specular）

PBR 的 GGX 高光是连续的，会产生柔和的高光点。卡通渲染想要的是"白色圆圈"——像动漫里眼睛上的高光。

**Hard Specular 公式**：
```
H = normalize(L + V)          // Half-vector
NdotH = max(0, dot(N, H))
spec = (NdotH > threshold) ? 1.0 : 0.0   // threshold = 0.9
```

这个 `threshold` 决定高光的大小：
- threshold 越大 → 高光越小（更集中）
- threshold 越小 → 高光越大（更扩散）

0.9 是一个常见的值，给出合理大小的高光圆圈。

**直觉**：`NdotH > 0.9` 等价于 Half-vector 和法线的夹角 < arccos(0.9) ≈ 26°。只有当视角和光源方向都非常接近镜面反射方向时，才会"亮"。

### 2.3 完整的 Cel Shading 光照模型

组合起来：

```
// 漫反射（量化）
NdotL = max(0, dot(N, L))
q = floor(NdotL * 3) / 3

// 高光（硬边）
H = normalize(L + V)
NdotH = max(0, dot(N, H))
spec = (NdotH > 0.9) ? 0.6 : 0.0

// 环境光（比 PBR 弱，避免暗部太亮破坏色块感）
ambient = baseColor * ambientColor * 0.3

// 最终颜色
result = ambient + baseColor * lightColor * q + lightColor * spec
```

与 PBR 相比：
- PBR：ambient + diffuse（连续）+ specular（连续 GGX/Blinn-Phong）
- Cel：ambient（弱）+ diffuse（量化）+ specular（二值）

**关键区别**：量化和二值化是 Cel Shading 的本质，其他部分（法线插值、透视变换等）和普通光栅化完全相同。

### 2.4 轮廓线：屏幕空间边缘检测

卡通渲染的另一个标志就是**黑色轮廓线**。实现轮廓线有几种方式：

| 方法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| 法线膨胀（Back-face inflation）| 渲染放大的背面 | 简单、稳定 | 需要额外的几何渲染 |
| 屏幕空间边缘检测 | 检测法线/深度不连续 | 后处理、不依赖几何 | 可能有锯齿 |
| G-buffer 边缘 | 基于 G-buffer 差异 | 精确 | 需要延迟渲染管线 |

我今天实现的是**屏幕空间方法**，它同时结合了两种判断：

**判断 1：对象 ID 边界**
```
// 检查 4-邻居的对象 ID 是否不同
for each neighbor:
    if objBuf[neighbor] != objBuf[current]:
        isEdge = true
```

这能检测出不同物体之间的轮廓线。

**判断 2：法线不连续**
```
// 检查法线夹角
for each neighbor:
    if dot(currentNormal, neighborNormal) < 0.6:
        isEdge = true
```

这能检测出**同一物体内部的折痕**——比如 Torus 上内外圈的交界处，虽然是同一个物体（ID 相同），但法线差异很大。

阈值 0.6 对应夹角 ≈ 53°，对于平滑曲面（Icosphere、Torus）效果很好。如果阈值太低，连平坦区域都会出现不想要的线条。

**执行时机**：轮廓描边是在所有物体光栅化完成之后的**后处理 Pass**，直接在屏幕空间操作 framebuffer。

### 2.5 坐标系与渲染管线

今天的软光栅化流程：

```
世界坐标 (World Space)
    ↓ model matrix (平移/缩放/旋转)
观察坐标 (View Space)
    ↓ view matrix (lookAt)
裁剪坐标 (Clip Space)
    ↓ projection matrix (透视)
NDC坐标 [-1,1]³
    ↓ 透视除法 (w division)
屏幕坐标 [0,W] × [0,H]
    ↓ 光栅化 + 重心坐标插值
片元着色 → Cel Shading
    ↓ 深度测试
Framebuffer
    ↓ Outline Pass（后处理）
最终图像
```

PNG 写入时需要翻转 Y 轴：屏幕空间 y=0 在底部（OpenGL 惯例），但 PNG 文件 y=0 在顶部，所以写入时用 `for(int y=H-1; y>=0; y--)` 从底往上读。

---

## 三、实现架构

### 3.1 数据结构设计

```cpp
// 顶点：位置 + 法线
struct Vertex { Vec3 pos; Vec3 normal; };
struct Triangle { int a, b, c; };
struct Mesh { vector<Vertex> verts; vector<Triangle> tris; };

// 渲染对象：带材质信息
struct RenderObject {
    Mesh mesh;
    Mat4 model;       // 变换矩阵
    Vec3 baseColor;   // Cel Shading 用的基础颜色
    int  id;          // 对象 ID，用于轮廓检测
};
```

### 3.2 三个屏幕缓冲区

```cpp
vector<Color>  fb(W*H);          // 颜色缓冲
vector<float>  depthBuf(W*H);    // 深度缓冲
vector<Vec3>   normalBuf(W*H);   // 法线缓冲（轮廓检测用）
vector<int>    objBuf(W*H);      // 对象ID缓冲（轮廓检测用）
```

法线缓冲和对象ID缓冲是 Cel Shading 专用的——普通 PBR 光栅化不需要这两个。它们类似 G-Buffer，但用途只有轮廓线检测。

### 3.3 Icosphere 生成

用递归细分正二十面体生成 UV 球。优于经纬度球（Lat-Long Sphere）的地方：**顶点分布更均匀**，不会在两极有三角形堆积。

```cpp
Mesh buildIcosphere(int subdivisions) {
    // 1. 初始化正二十面体（12顶点，20面）
    float phi = (1 + sqrt(5)) / 2;   // 黄金比例
    // ... 12个顶点
    
    // 2. 细分 subdivisions 次
    // 每次细分：每个三角形 → 4个小三角形（取中点并归一化）
    for (int d = 0; d < subdivisions; d++) {
        // 对每条边取中点，投影回单位球面
        // 4级细分：20 * 4^4 = 5120 个三角形
    }
    
    // 3. 单位球的顶点法线 = 顶点位置（已经在单位球面上）
}
```

4 级细分产生 5120 个三角形，足够在 800×600 分辨率下看起来平滑。

### 3.4 Torus 生成

```cpp
Mesh buildTorus(int majorSeg, int minorSeg, float R, float r) {
    // R: 主环半径（圆环中心到圆管中心的距离）
    // r: 管半径（圆管的截面半径）
    
    for (int i = 0; i <= majorSeg; i++) {
        float phi = 2π * i / majorSeg;  // 绕 Y 轴角度
        for (int j = 0; j <= minorSeg; j++) {
            float theta = 2π * j / minorSeg;  // 管截面角度
            
            // 管中心在 XZ 平面上
            Vec3 center = { R*cos(phi), 0, R*sin(phi) };
            
            // 法线方向：从管中心指向表面点
            Vec3 normal = { cos(phi)*cos(theta), sin(theta), sin(phi)*cos(theta) };
            
            // 表面点 = 中心 + 法线 * r
            Vec3 pos = center + normal * r;
        }
    }
}
```

Torus 的法线计算比较有意思：在管截面上，法线方向由两个角度共同决定——`phi`（绕大轴的旋转）和 `theta`（管截面上的角度）。

---

## 四、关键代码解析

### 4.1 透视正确重心坐标插值

这是软光栅化里最容易写错的地方。**直接用屏幕空间的重心坐标插值属性是错的**，因为透视变换不保持线性关系。

正确做法：用 `1/w` 加权：

```cpp
// 顶点的 w 值（裁剪空间 w = 相机空间 z）
float w0 = v0.clipPos.w, w1 = v1.clipPos.w, w2 = v2.clipPos.w;

// b0, b1, b2 是屏幕空间重心坐标（由 edge function 计算）
// 透视正确插值：先用 1/w 加权求和，再除以 sum(b/w)
float invW = b0/w0 + b1/w1 + b2/w2;
Vec3 N = (v0.worldNormal*(b0/w0) + v1.worldNormal*(b1/w1) + v2.worldNormal*(b2/w2));
if (invW > 1e-7f) N = N * (1.0f / invW);
N = N.norm();   // 插值后的法线必须重新归一化！
```

**为什么要重新归一化**：即使顶点法线是单位向量，插值结果长度一般不为 1（向量平均不保持单位长度）。不归一化会导致光照计算错误——这个 Bug 不容易发现，因为图像看起来只是光照"软了一点"。

### 4.2 Edge Function 与背面剔除

```cpp
// 有符号面积（×2）= 叉积的 z 分量
float edgeFunc(Vec2 a, Vec2 b, Vec2 p) {
    return (b.x - a.x) * (p.y - a.y) - (b.y - a.y) * (p.x - a.x);
}

float area = edgeFunc(s0, s1, s2);
if (std::abs(area) < 1e-4f) continue;   // 退化三角形
if (area <= 0) continue;                 // 背面剔除
```

**背面剔除的方向约定**：面积 > 0 意味着顶点在屏幕空间按逆时针排列（Counter-Clockwise），我们把这定义为正面。这个约定需要在整个管线中保持一致——如果相机、投影或模型变换改变了手性，这里就需要反转。

**Edge Function 的重心坐标**：
```
e0 = edgeFunc(s0, s1, p)  →  b2 = e0 / area
e1 = edgeFunc(s1, s2, p)  →  b0 = e1 / area
e2 = edgeFunc(s2, s0, p)  →  b1 = e2 / area
```

当 `e0 >= 0 && e1 >= 0 && e2 >= 0` 时，点 p 在三角形内。

### 4.3 完整的 Cel Shading 着色函数

```cpp
Color celShade(Vec3 baseColor, Vec3 N, Vec3 L, Vec3 V, 
               Vec3 lightColor, Vec3 ambientColor) {
    float NdotL = std::max(0.0f, N.dot(L));
    
    // ① 漫反射量化：把连续的 [0,1] 切成 3 个色阶
    float diffuseSteps = 3.0f;
    float q = std::floor(NdotL * diffuseSteps) / diffuseSteps;
    
    // ② 硬边高光：NdotH > 0.9 才亮，否则为 0
    Vec3 H = (L + V).norm();
    float NdotH = std::max(0.0f, N.dot(H));
    float spec = (NdotH > 0.9f) ? 1.0f : 0.0f;
    
    // ③ 组合
    Vec3 diffuse  = baseColor.mul(lightColor) * q;
    Vec3 specular = lightColor * spec * 0.6f;
    Vec3 ambient  = baseColor.mul(ambientColor) * 0.3f;
    
    return toColor(ambient + diffuse + specular);
}
```

**注意 `baseColor.mul(lightColor)`**：这是逐分量乘法（不是点积），表示颜色过滤。白色光（1,1,1）对红色物体（0.9,0.3,0.2）的结果就是 （0.9,0.3,0.2）。

### 4.4 双重判断的轮廓检测

```cpp
void drawOutlines(Color outlineColor) {
    for (int y = 1; y < H-1; y++) {
        for (int x = 1; x < W-1; x++) {
            int id = objBuf[y*W + x];
            if (id == 0) continue;  // 背景不描边
            
            bool isEdge = false;
            int neighbors[4][2] = {{-1,0},{1,0},{0,-1},{0,1}};
            
            // 判断1：对象边界（必然是轮廓）
            for (auto& n : neighbors) {
                if (objBuf[(y+n[1])*W + (x+n[0])] != id) {
                    isEdge = true; break;
                }
            }
            
            // 判断2：法线不连续（同物体内部折痕）
            if (!isEdge) {
                Vec3 cn = normalBuf[y*W + x];
                for (auto& n : neighbors) {
                    if (cn.dot(normalBuf[(y+n[1])*W + (x+n[0])]) < 0.6f) {
                        isEdge = true; break;
                    }
                }
            }
            
            if (isEdge) fb[y*W + x] = outlineColor;
        }
    }
}
```

**执行顺序很重要**：必须先光栅化所有物体（填充 normalBuf 和 objBuf），再做轮廓检测。如果光栅化和描边交错进行，法线缓冲里会有"新旧数据混合"的问题。

### 4.5 PNG 写入（无外部依赖）

使用 zlib 的"Store"模式（无压缩），配合 Adler-32 校验和：

```cpp
vector<uint8_t> zlibStore(const vector<uint8_t>& raw) {
    vector<uint8_t> out;
    out.push_back(0x78);  // CMF: deflate, window size 32KB
    out.push_back(0x01);  // FLG: no dict, check bits
    
    // 分块：每块最大 65535 字节（DEFLATE 限制）
    size_t pos = 0;
    while (pos < raw.size()) {
        size_t blockSize = min(65535, raw.size() - pos);
        bool last = (pos + blockSize >= raw.size());
        
        out.push_back(last ? 0x01 : 0x00);  // BFINAL | BTYPE=00
        // LEN and ~LEN (little-endian)
        out.push_back(blockSize & 0xFF);
        out.push_back(blockSize >> 8);
        out.push_back(~blockSize & 0xFF);
        out.push_back((~blockSize) >> 8);
        
        out.insert(out.end(), raw.begin()+pos, raw.begin()+pos+blockSize);
        pos += blockSize;
    }
    
    // Adler-32 checksum (big-endian)
    uint32_t a = adler32(raw.data(), raw.size());
    out.push_back(a>>24); out.push_back((a>>16)&0xFF);
    out.push_back((a>>8)&0xFF); out.push_back(a&0xFF);
    return out;
}
```

这是项目系列里第 40+ 次用这个 PNG 写入器了，它一直工作稳定。Store 模式下图片会比较大（本项目 1.4MB），但对于验证来说完全足够。

---

## 五、踩坑实录

### Bug #1（不存在）：背面剔除方向

**症状**：初次运行，对象渲染正确，但如果相机角度换了就可能出现物体"消失"或"只渲染背面"。

**原理**：Edge Function 的符号取决于顶点绕序（CW/CCW）。在我的实现里，相机从 +Z 方向看向原点，投影后屏幕空间的三角形如果顶点是 CCW 排列，`edgeFunc(s0,s1,s2)` 返回正值（面积 > 0）。所以正面判断就是 `area > 0`，背面（area <= 0）直接 continue。

**这次编译就对了**，因为我提前确认了相机/投影的手性。在 04-27 OIT 项目里遇到的 signFactor Bug 就是类似的方向问题，这次特别注意了。

### Bug #2（不存在）：法线插值后未归一化

**潜在症状**：光照过渡不正确，Cel Shading 的色阶边缘位置偏移，在某些视角下高光消失。

**原因**：重心坐标插值后，法线向量的长度不再是 1（比如两个 60° 夹角的单位向量，插值中点的长度是 cos(30°) ≈ 0.866，不是 1）。

**修复**：插值后立刻调用 `N = N.norm()`。这一步我在写代码时就加了，没有实际触发这个 Bug。

### Bug #3（潜在）：地面背面可见问题

**分析**：地面平面法线朝上（0,1,0），从正面看是 CCW（area > 0），正常显示。如果相机跑到地面下方，地面就变成 CW（area <= 0），会被背面剔除掉。

**这次没触发**：相机在 (0, 2.5, 6)，y=2.5 在地面（y=0）上方，所以地面永远是正面。

### Bug #4：Torus 内侧看起来也有轮廓线

**观察**：Torus 的内圆（与 Torus 孔洞相交的地方）会出现轮廓线，但外部轮廓线更明显。

**原因**：Torus 内侧的法线方向差异很大（朝向孔洞内部），满足了 `dot < 0.6` 的条件，被画成了轮廓。

**是 Bug 吗？**：不是——这其实是正确的行为，手绘动画里 Torus 的内侧折线本来就应该有轮廓。如果不想要，可以把阈值调低（比如 0.3），但会丢失其他物体上的折线。

### 调试经验：量化步数的视觉影响

在最终版本中选择了 3 步量化。在调试过程中对比了几个值：

- **2 步**：只有"全亮"和"全暗"，过于简单，像剪影
- **3 步**：经典卡通效果，有清晰的明暗分界
- **4 步**：开始接近平滑，卡通感减弱
- **5 步及以上**：和 Phong 很像，失去意义

3 步是大多数日系卡通游戏的选择，保持了这个标准值。

---

## 六、效果验证与数据

### 6.1 量化统计

```
Background pixels: 379302  (79.0% 背景)
Object pixels:      93206  (19.4% 物体颜色)
Outline pixels:      7492   (1.6% 轮廓线)
```

背景 + 物体 + 轮廓 = 480000 = W×H ✅

轮廓线占 1.6%，视觉上明显但不过度——这个比例在合理范围内。如果轮廓线占比过高（比如 >5%），说明法线阈值太低，产生了太多内部线条。

### 6.2 像素统计验证

```python
from PIL import Image
import numpy as np

img = Image.open('cel_shading_output.png')
pixels = np.array(img).astype(float)

mean = 193.5   # 偏亮（背景是浅蓝色，占主要面积）
std  = 78.4    # 标准差大（说明图像有明显的明暗分布）
```

均值 193.5：偏高是因为背景是浅蓝色（200, 230, 255），占了 79% 的面积。

标准差 78.4：说明图像有丰富的对比，不是一片均匀颜色。这个值比 PBR 项目的典型值（std≈50）更高，正是因为 Cel Shading 的硬边色块产生了更强的对比度。

### 6.3 坐标系验证

```
上半部分均值: 162.6  
下半部分均值: 224.3
```

上半部分均值 < 下半部分：
- 上半：物体（深色轮廓线、深色暗部色块）拉低了均值
- 下半：地面（浅棕色）+ 背景（浅蓝色）使均值更高

✅ 坐标系正确：物体在图像上半部，地面在下半部

### 6.4 文件属性

```
文件大小: 1.4MB (1,461,839 bytes)
远大于 10KB 的最低要求 ✅
```

1.4MB 是因为使用了无压缩的 DEFLATE Store 模式，实际渲染内容没有问题。

### 6.5 性能数据

- **渲染时间**：约 0.5 秒（CPU，无优化）
- **三角形数量**：
  - Icosphere (4级细分): 5120 个面
  - Torus (40×20): 1600 个面
  - 小球 (3级细分): 1280 个面
  - 地面 (8×8): 128 个面
  - **总计：8128 个三角形**
- **像素处理量**：800×600 = 480,000 片元

---

## 七、总结与延伸

### 7.1 今天实现了什么

成功构建了一个支持 Cel Shading 的软光栅化渲染器，核心功能：

✅ 漫反射 3 级量化  
✅ 硬边 Specular 高光  
✅ 双重屏幕空间轮廓线（对象ID + 法线不连续）  
✅ 透视正确重心坐标插值  
✅ Icosphere + Torus + 平面几何  
✅ 纯 C++17，无外部依赖  

### 7.2 Cel Shading 的局限性

1. **动态光源支持差**：量化会产生"色阶跳变"，光源移动时会有明显的阶梯感。游戏里通常会用渐变纹理（Ramp Texture）来软化边缘。

2. **轮廓线质量**：屏幕空间轮廓线有锯齿，分辨率低时尤其明显。高质量的 NPR 游戏（如米哈游）通常用多通道 G-Buffer 做更精确的边缘检测，或者用法线膨胀（Normal Expansion）方法。

3. **正面剔除法线膨胀**：更专业的描边方法是单独渲染一遍背面，并在 vertex shader 里沿法线方向膨胀一点距离——这样描边粗细可以精确控制，也不受屏幕分辨率影响。

### 7.3 可以继续优化的方向

**技术深度**：
- **Rim Light（边缘光）**：视线方向和法线接近垂直时加亮，产生"轮廓发光"效果，常见于角色边缘的氛围光
- **Ramp Texture**：用一张 1D 纹理代替 `floor()` 量化，可以实现任意复杂的色阶分布
- **多材质支持**：皮肤、金属、衣物对应不同的量化曲线和高光模式
- **动画帧**：把多帧 Cel Shading 渲染导出为 GIF，展示卡通感的时序效果

**工程实践**：
- 把量化步数、高光阈值、轮廓线阈值做成参数，用命令行参数控制
- 支持多光源（点光、方向光、聚光灯）
- 输出比较图：左半 Phong，右半 Cel Shading

### 7.4 与系列其他文章的关联

- 与 [03-25 SSAO](https://chiuhoukazusa.github.io) 的关联：两者都用到了法线缓冲，SSAO 用法线做半球采样，Cel Shading 用法线做轮廓检测
- 与 [04-18 SSR](https://chiuhoukazusa.github.io) 的关联：SSR 里的 G-Buffer 思想和今天的 normalBuf + objBuf 是同源的——都是把几何信息存入缓冲区供后续 Pass 使用
- 与 [04-21 Bloom](https://chiuhoukazusa.github.io) 的关联：Bloom 和 Cel Shading 都是 Post-processing Pass，执行时机和流程设计是一样的

Cel Shading 是从 PBR 到 NPR 的一个很好的入门点。明天可以继续探索其他 NPR 方向——比如 Cross Hatching（素描风格）或者 Watercolor（水彩风格）。

---

## 代码仓库

完整代码见：[daily-coding-practice/2026/04/04-28-Cel-Shading-Outline-Renderer](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-28-Cel-Shading-Outline-Renderer)

**编译运行**：
```bash
g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra
./output
# 输出: cel_shading_output.png (800×600)
```

**依赖**：无（纯 C++17 标准库）

![渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-28-Cel-Shading-Outline-Renderer/cel_shading_output.png)
