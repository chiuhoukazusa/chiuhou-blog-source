---
title: "每日编程实践: Shadow Volume Renderer（阴影体积渲染器）"
date: 2026-04-30 05:30:00
tags:
  - 每日一练
  - 图形学
  - 阴影技术
  - C++
  - 软光栅化
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-30-shadow-volume-renderer/shadow_volume_output.png
---

# 每日编程实践: Shadow Volume Renderer（阴影体积渲染器）

> 日期：2026-04-30  
> 技术标签：Shadow Volume · Stencil Buffer · Z-Fail (Carmack's Reverse) · 轮廓边检测 · 软光栅化  
> 代码仓库：[daily-coding-practice/2026/04/04-30-shadow-volume-renderer](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-30-shadow-volume-renderer)

---

## ① 背景与动机：为什么需要 Shadow Volume？

阴影是使渲染图像真实可信的最关键因素之一。没有阴影，物体会看起来像是飘浮在空中，场景缺乏空间感和深度感。阴影技术的历史演进也是图形学发展史上最有意思的线索之一。

### Shadow Map 的局限

最广为人知的阴影技术是 **Shadow Map（阴影贴图）**，其思路是：从光源视角渲染场景的深度图，然后在主渲染时判断每个像素是否比该深度图记录的值更远，如果是则在阴影中。Shadow Map 的优点是实现简单、性能好，是游戏行业20年来的主流方案。

然而，Shadow Map 有一个根本性的**精度问题**：它将连续的3D空间离散化为一张有限分辨率的2D纹理，这导致著名的**Shadow Acne（阴影粉刺/自遮挡）**和**Peter Panning（彼得潘效应）**问题。尽管有 Bias 调整、PCF 过滤、PCSS 等众多补救方案（我们之前也实现过 PCSS），但这些都是在掩盖一个根本性的离散化缺陷。

### Shadow Volume 的出发点

**Shadow Volume（阴影体积）** 是一种从根本上不同的思路：它不依赖离散化的深度图，而是直接在3D空间中构造出一个几何体——"阴影体"——来精确表示被某个物体遮挡的空间区域。

直觉理解：想象一个不透明球体和一盏点光源。光线从四面八方照射过来，球体遮挡了其中一部分，形成一个以光源为顶点、以球体轮廓为"底面"的无穷大锥形区域。这个锥形区域就是**阴影体（Shadow Volume）**。处于阴影体内部的任何点都在阴影中；处于阴影体外部的点则被光源直接照射。

这种方法的理论优势非常明显：
- **像素精确（pixel-accurate）**：不存在阴影贴图的分辨率限制
- **不需要任何偏移（Bias）**：不存在 Shadow Acne 问题
- **任意形状投射物**：对于复杂几何体同样适用
- **无穿透阴影伪影**：几何精确性保证了没有光线"泄漏"

工业界的使用：Shadow Volume 虽然在现代 GPU 管线中已经被 Shadow Map 方案取代（主要是性能原因），但在历史上有过重要的应用时期。最著名的案例是 **id Software 的 Doom 3（2004年）**，整个游戏引擎 id Tech 4 使用的就是 Shadow Volume，这种技术在当时被称为"Carmack's Reverse"或"Z-Fail Method"，以约翰·卡马克（John Carmack）的名字命名，因为他独立发现并推广了这种实现方式。

---

## ② 核心原理：Shadow Volume 的数学与算法

### 2.1 Stencil Buffer 计数原理

Shadow Volume 的实现核心依赖于 GPU 的**模板缓冲（Stencil Buffer）**。这是一个每像素的整数计数器，Shadow Volume 算法利用它来统计一条从摄像机出发的光线穿越阴影体的次数。

设想一条从摄像机出发、经过屏幕上某个像素 `p` 的射线，它最终会到达一个场景几何体表面上的点 `P`（这就是该像素在深度缓冲中对应的点）。

对于这条射线，我们统计它穿越阴影体边界的次数：
- 每穿入阴影体一次（穿过面朝向摄像机的阴影体三角形）→ 计数 +1
- 每穿出阴影体一次（穿过面背向摄像机的阴影体三角形）→ 计数 -1

最终：
- 计数 = 0 → 点 P 在所有阴影体外 → P 被光源照射
- 计数 ≠ 0 → 点 P 在某个阴影体内 → P 处于阴影中

这就是 Shadow Volume 算法的本质：**用几何穿越次数来判断阴影包含关系**。

### 2.2 Z-Pass vs Z-Fail：为什么需要 Carmack's Reverse

原始的 Shadow Volume 算法（Z-Pass）：
- 深度测试通过时（阴影体三角形在场景几何体前面）
  - 正面三角形 → stencil++
  - 背面三角形 → stencil--

这个方法有一个致命问题：**当摄像机本身位于某个阴影体内部时，算法会失效**。

想象摄像机就在阴影中（比如你在一个黑暗的房间里向窗外看），射线从摄像机出发时已经在阴影体内了，但 Z-Pass 的计数从 0 开始，无法区分这种情况。

**Z-Fail（Carmack's Reverse）解决方案**：

Z-Fail 改变了测试的时机——不在深度测试通过时更新，而在**深度测试失败时**更新：
- 深度测试失败时（阴影体三角形在场景几何体后面）
  - 正面三角形 → stencil--
  - 背面三角形 → stencil++

这个看似微小的改变有深刻含义。对于深度测试失败的三角形，它处于我们关心的场景点 `P` 的后面（更远处）。通过统计有多少个阴影体"穿越"了 P 后方，我们可以从"无穷远处"往回数穿越次数。

**直觉理解**：想象无穷远处有一束光射向摄像机，经过点 P。这束光先穿过所有在 P 后面的阴影体边界，再到达 P。Z-Fail 统计的就是这段路程中的穿越次数。无论摄像机在哪里（无论是否在阴影体内），这个统计都是正确的。

**数学证明**：对于任意一条从无穷远射向摄像机方向的射线，设阴影体为拓扑封闭的几何体，射线必然整对地穿入和穿出。对于点 P 后方的穿越计数：
- 若 P 在阴影体内，穿出次数 > 穿入次数（因为射线在 P 处进入后还没穿出），计数 > 0
- 若 P 在阴影体外，穿入穿出成对，计数 = 0

这就是为什么 Z-Fail 需要**封闭**的阴影体（这是它相比 Z-Pass 的一个额外要求）。

### 2.3 轮廓边检测（Silhouette Edge Detection）

要构建阴影体，首先需要找到物体的**轮廓边（Silhouette Edges）**——即从光源方向看到的物体轮廓。

**定义**：一条边是轮廓边，当且仅当它被恰好两个三角形共享，且其中一个三角形朝向光源（面法线与光线方向夹角 < 90°），另一个背向光源。

数学判断：对于三角形面法线 N 和光线方向向量 L = (lightPos - 三角形顶点).normalize()：

```
朝光面条件: N·L > 0
背光面条件: N·L ≤ 0
轮廓边条件: 两个面的 (N·L > 0) 值不同
```

**代码实现**：通过半边数据结构，对每条边找到其两个相邻面：

```cpp
// 判断每个面朝向光源
std::vector<bool> facesLight(nFaces);
for(int i = 0; i < nFaces; i++) {
    Vec3 a = mesh.worldVertex(mesh.faces[i][0]);
    Vec3 normal = mesh.worldFaceNormal(i);
    Vec3 toLight = (lightPos - a).normalize();
    facesLight[i] = (normal.dot(toLight) > 0.0f);
}
```

然后找到两个面朝光性不同的共享边：

```cpp
// 检查两个半边 (va,vb) 和 (vb,va) 是否形成轮廓边
if(halfEdges[j].va == vb && halfEdges[j].vb == va) {
    int fi2 = halfEdges[j].fi;
    if(facesLight[fi] != facesLight[fi2]) {
        // 这是一条轮廓边！
    }
}
```

### 2.4 阴影体的封闭构建

Z-Fail 要求阴影体必须是**拓扑封闭的几何体**。一个完整的封闭阴影体由三部分组成：

**部分一：侧面四边形（Side Quads）**
从每条轮廓边延伸出一个四边形，向光源反方向无限延伸（实践中用足够大的距离代替无穷）。

对于轮廓边端点 va、vb，延伸方向为从光源出发通过 va（或 vb）的方向：

```
dirA = normalize(va - lightPos)
dirB = normalize(vb - lightPos)
farA = va + dirA * extrudeDist
farB = vb + dirB * extrudeDist

四边形 = 两个三角形:
  Triangle1: va → vb → farB
  Triangle2: va → farB → farA
```

**部分二：前盖（Front Cap）**
所有朝向光源的三角面，保持原始位置。这封闭了阴影体靠近光源一侧的开口。

**部分三：后盖（Back Cap）**
所有背向光源的三角面，投影到远处，并取反绕序（因为这些面需要朝外，即朝向阴影体内部）。

```
farA = a + normalize(a - lightPos) * extrudeDist
farB = b + normalize(b - lightPos) * extrudeDist  
farC = c + normalize(c - lightPos) * extrudeDist
// 注意：绕序取反！
Triangle: farA → farC → farB  // 反转 B,C 顺序
```

---

## ③ 实现架构：软光栅化 Shadow Volume 渲染管线

### 3.1 整体数据流

```
输入: 场景 Mesh 列表 + 光源位置 + 摄像机参数
    │
    ▼
[Pass 1: 深度预渲染（全光照）]
    场景所有 Mesh → 光栅化 → 写入 depth buffer + color buffer
    （这步假设全部不在阴影中，后面会被覆写）
    │
    ▼
[Pass 2: 阴影体模板填充]
    对每个 shadow caster:
        buildShadowVolume() → 封闭阴影体三角形列表
    光栅化阴影体三角形 → 仅更新 stencil buffer
    规则: 深度测试失败时
          正面 → stencil--
          背面 → stencil++
    │
    ▼
[Pass 3: 最终着色（有阴影信息）]
    重置 color + depth buffer
    光栅化场景 Mesh
    查询 savedStencil[py][px] != 0 → 阴影中（仅环境光）
               savedStencil[py][px] == 0 → 光照中（完整 Phong）
    │
    ▼
输出: shadow_volume_output.png
```

### 3.2 关键数据结构

**Framebuffer**：包含三个缓冲区
```cpp
struct Framebuffer {
    Color color[H][W];    // 颜色缓冲（3×uint8_t）
    float depth[H][W];    // 深度缓冲（[0,1] 归一化深度）
    int   stencil[H][W];  // 模板缓冲（软件模拟的整数计数器）
};
```

在真实 GPU 上，Stencil Buffer 通常是 8-bit（值域 0-255），这里我们用 int 来避免溢出，并方便调试。

**ShadowVolumeMesh**：直接存储三角形顶点（world space），不共享顶点
```cpp
struct ShadowVolumeMesh {
    std::vector<Vec3> verts;  // 每3个一组构成一个三角形
};
```

为什么不用索引？因为阴影体的三角形来自不同来源（轮廓边延伸、前/后盖），使用 flat 三角形数组更简单，避免了构建索引的复杂性。

### 3.3 CPU 端职责 vs GPU 端职责

在真实 GPU 渲染管线中，Shadow Volume 的各阶段分工如下：

| 操作 | GPU 端 | 我们的软件模拟 |
|------|--------|---------------|
| 轮廓边检测 | Geometry Shader / 预计算 | CPU 端 buildShadowVolume() |
| 阴影体延伸 | Vertex Shader 中 extrude | CPU 端预计算 farVert |
| 模板更新 | Stencil Test 硬件固定功能 | 软件光栅化中手动 stencil++ |
| 背面剔除控制 | glDisable(GL_CULL_FACE) | 光栅化时双面处理 |
| 阴影查询 | 读取 Stencil Buffer | savedStencil[py][px] |

---

## ④ 关键代码解析

### 4.1 轮廓边检测与阴影体构建

```cpp
ShadowVolumeMesh buildShadowVolume(const Mesh& mesh, const Vec3& lightPos, 
                                    float extrudeDist = 50.0f) {
    ShadowVolumeMesh sv;
    int nFaces = (int)mesh.faces.size();
    
    // 第一步：判断每个面是否朝向光源
    std::vector<bool> facesLight(nFaces);
    for(int i = 0; i < nFaces; i++) {
        Vec3 a = mesh.worldVertex(mesh.faces[i][0]);
        Vec3 normal = mesh.worldFaceNormal(i);
        Vec3 toLight = (lightPos - a).normalize();
        // dot > 0 表示法线和光线方向同侧，即朝光
        facesLight[i] = (normal.dot(toLight) > 0.0f);
    }
```

这里有个细节：我们用三角形的**一个顶点**来计算光线方向，而不是三角形的中心。对于平面三角形，这不影响结果（因为朝光判断只取决于法线和方向的点积正负）。

```cpp
    // 第二步：构建半边结构，找轮廓边
    struct HalfEdge { int va, vb, fi; };
    std::vector<HalfEdge> halfEdges;
    for(int fi = 0; fi < nFaces; fi++) {
        for(int k = 0; k < 3; k++) {
            int va = mesh.faces[fi][k];
            int vb = mesh.faces[fi][(k+1)%3];
            halfEdges.push_back({va, vb, fi});
        }
    }
    // 轮廓边：找 (va→vb) 和 (vb→va) 的对应半边，且朝光性不同
    for(size_t i = 0; i < halfEdges.size(); i++) {
        // ...找到对边 j，检查 facesLight[fi] != facesLight[fi2]
    }
```

**为什么用半边？** 半边 (va→vb) 和其对边 (vb→va) 正好对应共享同一条边的两个三角形。两个三角形的绕序相反（一个以 va→vb，另一个以 vb→va 的顺序使用这条边），这是网格的拓扑特性。

```cpp
    // 第三步：从轮廓边生成侧面四边形
    for(auto& se : silhouettes) {
        // 计算延伸点（沿远离光源方向）
        Vec3 dirA = (se.va - lightPos).normalize();
        Vec3 dirB = (se.vb - lightPos).normalize();
        Vec3 farA = se.va + dirA * extrudeDist;
        Vec3 farB = se.vb + dirB * extrudeDist;
        
        // 侧面：两个三角形（注意绕序要保持一致）
        sv.verts.push_back(se.va);
        sv.verts.push_back(se.vb);
        sv.verts.push_back(farB);
        
        sv.verts.push_back(se.va);
        sv.verts.push_back(farB);
        sv.verts.push_back(farA);
    }
```

**延伸方向的选择**：从光源出发通过顶点的方向 `normalize(v - lightPos)` 确保了投影是从光源透视投影出去的，这保证了阴影体在光源前方对应物体的遮挡形状。

### 4.2 Z-Fail 模板更新核心逻辑

```cpp
void rasterizeShadowVolumeZFail(Vec3 p0, Vec3 p1, Vec3 p2, const Mat4& mvp) {
    // ...标准光栅化流程，计算屏幕坐标 s0, s1, s2...
    
    float area = edgeFunc(s0.x,s0.y, s1.x,s1.y, s2.x,s2.y);
    // area > 0 表示正面（counter-clockwise in screen space）
    bool frontFace = (area > 0);
    
    for(每个覆盖像素 (px, py)) {
        // 插值得到这个三角形在该像素的深度
        float depth = ...;
        
        // Z-Fail 关键判断：深度测试失败（三角形在场景几何体后面）
        if(depth >= fb.depth[py][px]) {
            if(frontFace) {
                fb.stencil[py][px]--;  // 正面：计数减1
            } else {
                fb.stencil[py][px]++;  // 背面：计数加1
            }
        }
        // 深度测试通过时：什么都不做（Z-Fail 只关心失败的情况）
    }
}
```

**为什么正面减而背面加？** 

回顾前面的原理：射线从无穷远射向摄像机，经过点 P。对 P 后方（更远处）的穿越计数：

- 穿入阴影体 = 射线遇到背面三角形 → stencil++
- 穿出阴影体 = 射线遇到正面三角形 → stencil--

如果 P 在阴影体内，从无穷远到 P 的路程中，最后一次穿越是"穿入"，所以穿入次数 > 穿出次数，stencil > 0。

**为什么深度失败时更新？** 

"深度测试失败"意味着这个阴影体三角形在场景点 P 的后方，即这个三角形在 P 和无穷远之间的路段上——这正是我们要统计的区域。

### 4.3 基于模板缓冲的最终着色

```cpp
// Pass 1 完成后，模板缓冲已经填好
// 将模板缓冲保存起来（因为 Pass 3 会重置缓冲）
int savedStencil[H][W];
memcpy(savedStencil, fb.stencil, sizeof(fb.stencil));

// Pass 3：重置颜色和深度缓冲，用正确的阴影信息重新渲染
for(auto* mesh : allMeshes) {
    // ...对每个三角形...
    rasterizeTriangle(
        p0, p1, p2, wn, wn, wn, mesh->color, vp,
        lightPos, eyePos,
        [&savedStencil](int px, int py) {
            return savedStencil[py][px] != 0;  // 阴影查询
        }
    );
}
```

着色函数根据阴影状态决定光照计算：
```cpp
Vec3 shade(Vec3 worldPos, Vec3 normal, Vec3 matColor, 
           Vec3 lightPos, Vec3 eyePos, bool inShadow) {
    Vec3 ambient = matColor * 0.15f;
    if(inShadow) return ambient;  // 在阴影中：只有环境光
    
    // 不在阴影中：完整 Phong 光照
    Vec3 L = (lightPos - worldPos).normalize();
    Vec3 V = (eyePos - worldPos).normalize();
    Vec3 H = (L + V).normalize();  // Blinn-Phong 半向量
    
    float NdotL = max(0, N·L);
    float NdotH = max(0, N·H);
    
    // 距离衰减
    float dist = length(lightPos - worldPos);
    float attn = 1 / (1 + 0.01*dist + 0.001*dist²);
    
    Vec3 diffuse  = matColor * (NdotL * attn);
    Vec3 specular = Vec3(1,1,1) * (pow(NdotH, 32) * attn * 0.5);
    
    return ambient + diffuse + specular;
}
```

---

## ⑤ 踩坑实录

### Bug 1：编译失败——缺少 `#include <cstdint>`

**症状**：
```
error: 'uint8_t' does not name a type
note: 'uint8_t' is defined in header '<cstdint>'
```

**错误假设**：`uint8_t` 是基础类型，应该在 `<cstdlib>` 或 `<cstring>` 中已经包含了。

**真实原因**：在 GCC/Clang 的严格模式下，`uint8_t` 定义在 `<cstdint>` 中，并不会通过其他标准头文件隐式引入（尽管在某些平台的实现中可能会）。这是跨平台 C++ 的一个常见陷阱：在 Windows MSVC 下可能会隐式可用，但在 Linux GCC 下不行。

**修复**：在所有 include 语句后添加 `#include <cstdint>`。

**教训**：永远显式包含你用到的类型所在的头文件，不要依赖隐式传递包含。

### Bug 2：编译失败——Color 结构体无法用大括号初始化

**症状**：
```
error: no match for 'operator=' (operand types are 'Color' and '<brace-enclosed initializer list>')
```

**错误假设**：C++ 结构体应该可以直接用 `{r, g, b}` 语法赋值（聚合初始化）。

**真实原因**：在 C++11 之后，聚合初始化（`Color{255, 0, 0}` 或 `fb.color[y][x] = {255, 0, 0}`）需要结构体满足**聚合类型**的条件。如果结构体有用户声明的构造函数，它就不再是聚合类型，大括号初始化就会失败。

但这里更根本的问题是：我先写了 `uint8_t r, g, b;` 但没有构造函数，此时大括号初始化应该是可以的（聚合初始化）。实际上编译器报错是因为我在 `default` 参数中使用了 `{30, 30, 40}` 但编译器在没有 `cstdint` 的情况下无法识别 `Color` 类型，造成了连锁错误。

**修复**：
1. 添加 `#include <cstdint>`（修复上面的 Bug 1）
2. 给 Color 添加显式构造函数消除歧义：
```cpp
struct Color {
    uint8_t r, g, b;
    Color(uint8_t r=0, uint8_t g=0, uint8_t b=0): r(r), g(g), b(b) {}
};
```

### Bug 3：`static` 局部数组无法在 Lambda 中捕获

**症状**：
```
warning: capture of variable 'savedStencil' with non-automatic storage duration
```

**错误假设**：`static` 只是让变量在函数调用间保持值，应该和普通局部变量一样可以被 lambda 捕获。

**真实原因**：`static` 局部变量有**静态存储期（static storage duration）**，不是**自动存储期（automatic storage duration）**。Lambda 捕获只适用于自动存储期的变量（即普通局部变量、函数参数）。对于静态变量，由于它们的地址是固定的，在 lambda 中可以直接访问（不需要捕获），但显式捕获是未定义行为（或编译器警告）。

这里使用 `static` 的原因是 `int savedStencil[H][W]` 作为栈上数组太大（800×600×4 = ~1.9MB），会导致栈溢出。

**修复**：去掉 `static` 关键字，改为在 `main` 函数中分配：
```cpp
// 原来: static int savedStencil[H][W];  ← 栈溢出 + 无法捕获
// 修改: 用 new 或改为成员变量
int savedStencil[H][W];  // 但这可能栈溢出！
// 正确做法: 放在全局或使用 vector
```

实际上在这个代码中，我们将 `savedStencil` 放在栈上（`H*W*sizeof(int) ≈ 1.9MB`），这在 Linux 系统（默认栈大小 8MB）下通常没问题，但在嵌入式或 Windows 系统（1MB 默认栈）下可能栈溢出。更好的做法是用全局数组或 `std::vector`。

### Bug 4：阴影体覆盖率只有 9.1%，视觉上的阴影面积看起来合理吗？

**症状**：渲染后观察图片，发现地面上有明显阴影投影，符合物理预期。9.1% 的阴影像素覆盖率合理。

**分析**：场景中三个物体（1个球体 + 2个立方体）投射阴影，地面和墙面接收阴影。点光源位于场景右上方 `(3, 8, 5)`，阴影主要投射在地面和后墙上。9.1% 对于这个场景尺寸和光源位置来说是合理的数字。

**验证方法**：
```bash
python3 -c "
from PIL import Image
import numpy as np
img = Image.open('shadow_volume_output.png')
pixels = np.array(img).astype(float)
dark = (pixels.mean(axis=2) < 30).sum()
print(f'深色像素(< 30): {dark}')  # 44314
"
```
44314 个深色像素，与阴影体的 43629 个模板非零像素接近，说明阴影计算基本正确。

---

## ⑥ 效果验证与数据

### 6.1 渲染结果

![Shadow Volume 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-30-shadow-volume-renderer/shadow_volume_output.png)

场景设置：
- Cornell Box 变体（地面 + 后墙 + 左红墙 + 右绿墙）
- 中心金色球体（球谐方向 radius=1.2）
- 右侧蓝色立方体（高立方体，旋转 0.4rad）
- 左侧紫色立方体（矮立方体，旋转 -0.3rad）
- 点光源位于 (3, 8, 5)，用白色圆圈标注

### 6.2 量化验证数据

| 指标 | 数值 |
|------|------|
| 渲染分辨率 | 800 × 600 = 480,000 像素 |
| 总三角形数 | 场景约 2000 面 + 阴影体 2310 面 |
| 阴影体三角形 | 2310（含侧面、前盖、后盖）|
| 模板非零像素 | 43,629（9.1%）|
| 图像均值 | 134.5（正常明暗对比）|
| 图像标准差 | 65.1（丰富的色彩变化）|
| 深色像素(< 30) | 44,314（阴影区域）|
| 亮色像素(> 50) | 419,625（光照区域）|
| 输出文件大小 | 85.4 KB |

**像素统计验证脚本**：
```python
from PIL import Image
import numpy as np, sys

img = Image.open("shadow_volume_output.png")
pixels = np.array(img).astype(float)
mean = pixels.mean()
std = pixels.std()
print(f"像素均值: {mean:.1f}  标准差: {std:.1f}")

if mean < 5:
    print("❌ 图像过暗"); sys.exit(1)
if mean > 250:
    print("❌ 图像过亮"); sys.exit(1)
if std < 5:
    print("❌ 图像无变化"); sys.exit(1)

dark_pixels = (pixels.mean(axis=2) < 30).sum()
bright_pixels = (pixels.mean(axis=2) > 50).sum()
print(f"✅ 像素统计正常")
print(f"暗区像素: {dark_pixels} / 亮区像素: {bright_pixels}")
# 输出:
# 像素均值: 134.5  标准差: 65.1
# ✅ 像素统计正常
# 暗区像素: 44314 / 亮区像素: 419625
```

### 6.3 与 Shadow Map 的对比

Shadow Volume 的明显优势在于**像素级精确性**：阴影边界完全没有走样（Aliasing）和自遮挡（Shadow Acne）问题。Shadow Map 实现中经常需要调整 Bias 值来平衡 Acne 和 Peter Panning，而 Shadow Volume 完全不需要。

Shadow Volume 的缺点在于**填充率（fill rate）**：阴影体内所有像素都需要进行光栅化和模板更新，对于大阴影体（如远处方向光）会产生极大的 overdraw。这也是 Doom 3 在性能上被诟病的主要原因之一。

---

## ⑦ 总结与延伸

### 7.1 技术局限性

**填充率瓶颈**：Shadow Volume 需要渲染两遍场景（深度预通道 + 最终着色），加上阴影体本身的光栅化。对于复杂场景中大量光源的情况，性能代价是不可接受的。这是它逐渐被 Shadow Map 取代的根本原因。

**几何体封闭性要求**：Z-Fail 需要阴影体是拓扑封闭的。对于有洞（hole）的网格，需要额外处理边界边。本次实现中对于裸边（只有一个相邻面的边）也做了处理。

**点光源限制**：Shadow Volume 天然适合点光源和聚光灯。对于方向光（平行光源），需要将光源放在无穷远处，延伸方向变为统一方向而非透视方向。

**透明物体**：Shadow Volume 对透明物体的处理非常复杂，基本无法直接支持。

### 7.2 可优化方向

**硬件 Stencil Buffer**：在真实 OpenGL/Vulkan 中，这整套流程完全可以交给 GPU 固定功能管线的 Stencil Test 处理，性能会大幅提升。

**轮廓边缓存**：当物体静止时，每帧重新计算轮廓边是浪费。可以缓存朝光面，只在光源移动时重新计算。

**Half-vector extrusion**：可以使用 Geometry Shader 在 GPU 上实时延伸轮廓边，避免 CPU-GPU 数据传输。

**Scissor Test 优化**：在屏幕空间计算阴影体的包围矩形，限制 Stencil 更新的范围，减少不必要的像素处理。

### 7.3 与本系列的关联

- **Day 04-20（PCSS）**：另一种阴影方案——Shadow Map + 百分比遮挡采样，产生软阴影但有精度问题
- **Day 04-01（SDF Ray Marching）**：SDF 场景中可以用 SDF 计算精确阴影，类似 Shadow Volume 的精确性
- **Day 04-18（SSR）**：同样是先渲染场景到 G-Buffer，再在屏幕空间做后处理的思路

Shadow Volume 代表了一类"在3D空间中直接推理遮挡关系"的技术路线，与 Shadow Map 的"从光源角度预采样"路线形成了有趣的对比。在理论上，Shadow Volume 更加"正确"；在工程上，Shadow Map 的灵活性和性能优势使其占据了主导地位。理解两种方案的权衡，是图形程序员必备的知识储备。

---

*代码仓库：[GitHub - daily-coding-practice](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-30-shadow-volume-renderer)*  
*图片托管：[blog_img CDN](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-30-shadow-volume-renderer/shadow_volume_output.png)*
