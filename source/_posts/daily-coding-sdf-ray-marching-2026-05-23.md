---
title: "每日编程实践: SDF Ray Marching Scene Renderer"
date: 2026-05-23 05:35:00
tags:
  - 每日一练
  - 图形学
  - C++
  - Ray Marching
  - SDF
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-23-sdf-ray-marching/sdf_scene_output.png
---

# SDF Ray Marching Scene Renderer

今天的项目是一个完整的 **SDF（Signed Distance Functions）Ray Marching 场景渲染器**——使用有向距离函数描述几何体，通过光线步进算法渲染完整3D场景，包含软阴影、环境光遮蔽（AO）、Blinn-Phong着色和ACES色调映射。

![SDF Ray Marching 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-23-sdf-ray-marching/sdf_scene_output.png)

渲染时间：**0.75秒**，输出 800×600，96KB PNG。

---

## ① 背景与动机

### 传统光栅化管线的几何局限

在传统光栅化渲染中，几何体必须用三角面片表示。想画一个球？你需要至少 960 个三角形（经纬细分），还会有明显的锯齿边缘。想做两个球体的平滑融合？你需要一个复杂的网格合并算法，并且每帧都要重新计算。

SDF Ray Marching 从根本上绕开了这个问题。

### 什么是 SDF？

**SDF（Signed Distance Field，有向距离场）**：对于场景中任意一点 `p`，SDF 函数 `f(p)` 返回该点到最近几何表面的有符号距离：
- `f(p) > 0`：点在几何体外部
- `f(p) = 0`：点在几何体表面
- `f(p) < 0`：点在几何体内部

这个看似简单的定义，带来了极其强大的组合能力：几何体的并集、交集、差集、平滑融合，全部通过简单的 min/max/smooth 操作实现，不需要任何网格修改。

### 工业界和游戏中的实际应用

SDF 技术并非学术玩具：

1. **Claybook（2018）**：Epic Games 的演示作品，整个游戏世界完全用 SDF 表示，支持实时雕刻和变形
2. **Epic Nanite**：Unreal Engine 5 的距离场软阴影和距离场 AO，基于 DFAO（Distance Field AO）技术
3. **Valve Dota 2**：大量使用 SDF 字体渲染（Valve 2007年发明的 SDF 字体技术）
4. **Inigo Quilez（iq）**：Shadertoy 大量作品证明 SDF + Ray Marching 可以用几百行实现令人叹服的场景
5. **物体轮廓/软阴影**：Lumen（UE5 的全局光照系统）在屏幕外区域使用 SDF ray march 而非光栅化

Ray Marching 的优点：
- **无需显式几何表示**：几何体只是一个 C++ 函数
- **任意精度**：球体无论多近都是完美圆形，没有三角面片锯齿
- **几何操作简单**：融合、雕刻、扭曲只需几行数学代码
- **自然支持软阴影和 AO**：这在光栅化中需要额外的 Shadow Map + PCSS

---

## ② 核心原理

### 2.1 球体 SDF

球体是最基础的 SDF。球心在原点、半径为 `r` 的球体：

```
f(p) = |p| - r
```

直觉：点到球心的距离减去半径，就是到球面的距离。
- 当 `p = (2, 0, 0)`，`r = 1` 时，`f = 2 - 1 = 1`（在球外1个单位）
- 当 `p = (0.5, 0, 0)`，`r = 1` 时，`f = 0.5 - 1 = -0.5`（在球内）

```cpp
float sdSphere(Vec3 p, float r) {
    return p.length() - r;
}
```

### 2.2 盒子 SDF

盒子 SDF 稍微复杂，但有一个优雅的构造：

```
f(p) = |max(|p| - b, 0)| + min(max(px-bx, py-by, pz-bz), 0)
```

分解来看：
1. `|p| - b`：各轴上的"超出量"，如果点在盒子内某轴则为负
2. `max(..., 0)`：只保留超出部分（负值截为0）→ 这是**外部距离**（角落/边/面的min）
3. `min(max(components), 0)`：如果点在盒子内，这一项给出负值 → **内部距离**

两项相加：点在外面时第二项为0，给出到最近面/边/角的距离；点在里面时第一项为0，给出到最近面的负距离。

```cpp
float sdBox(Vec3 p, Vec3 b) {
    Vec3 q = { fabsf(p.x)-b.x, fabsf(p.y)-b.y, fabsf(p.z)-b.z };
    Vec3 qp = { std::max(q.x,0.0f), std::max(q.y,0.0f), std::max(q.z,0.0f) };
    return qp.length() + std::min(std::max(q.x, std::max(q.y, q.z)), 0.0f);
}
```

注意 `fabsf(p.x)` 利用了盒子的对称性——只需处理第一象限，然后用绝对值折叠其他象限。

### 2.3 圆环面（Torus）SDF

圆环面：大半径 `R`（环中心到管中心的距离），小半径 `r`（管的截面半径）：

```
f(p) = |(|(px, pz)| - R, py)| - r
```

构造步骤：
1. `sqrt(px² + pz²) - R`：在 xz 平面上到圆环中心轴的距离，减去大半径 → 得到 xz 平面上的"相对距离"
2. 配合 y 坐标 → 构成一个二维向量
3. 求该向量长度减小半径 → 到圆管表面的距离

```cpp
float sdTorus(Vec3 p, float R, float r) {
    float qx = sqrtf(p.x*p.x + p.z*p.z) - R;
    return sqrtf(qx*qx + p.y*p.y) - r;
}
```

### 2.4 胶囊体（Capsule）SDF

胶囊体是线段 AB 的等距外扩。核心：先求点 p 到线段 AB 的距离，再减去半径：

```
t = clamp(dot(p-A, B-A) / dot(B-A, B-A), 0, 1)
closest = A + t*(B-A)
f(p) = |p - closest| - r
```

`t` 是 p 在线段 AB 上的投影参数，clamp 保证它不超出线段端点。

```cpp
float sdCapsule(Vec3 p, Vec3 a, Vec3 b, float r) {
    Vec3 pa = p - a, ba = b - a;
    float h = clampf(pa.dot(ba) / ba.dot(ba), 0.0f, 1.0f);
    return (pa - ba*h).length() - r;
}
```

### 2.5 SDF 组合操作

SDF 最强大的地方：几何操作通过简单的数学函数实现。

**并集（Union）**：两个几何体合并 → 取两个 SDF 的最小值
```
f_union(p) = min(f1(p), f2(p))
```
直觉：距离最近的物体"胜出"，射线会先打到它。

**差集（Subtraction）**：从 A 中挖去 B → 
```
f_sub(p) = max(f_A(p), -f_B(p))
```
直觉：`-f_B` 翻转了 B 的内外，然后取 max → 只有同时在 A 外部、B 内部（即被挖去的空间）才是"实体"。

**平滑融合（Smooth Union）**：两个物体平滑混合，没有硬边缘：
```
h = clamp(0.5 + 0.5*(d2-d1)/k, 0, 1)
f_smooth(p) = mix(d2, d1, h) - k*h*(1-h)
```
- `k` 是融合强度（越大越平滑）
- `h` 是插值权重，在两个距离相近时平滑过渡
- `k*h*(1-h)` 项进一步"压低"融合区域，使表面向内凹陷，产生有机感

```cpp
float sdfSmoothUnion(float d1, float d2, float k) {
    float h = clampf(0.5f + 0.5f*(d2-d1)/k, 0.0f, 1.0f);
    return mixf(d2, d1, h) - k*h*(1.0f-h);
}
```

这个公式是 Inigo Quilez 推导的，也是 Shadertoy 和实时 SDF 渲染的标准配置。

### 2.6 Ray Marching 算法

Ray Marching 的核心思想：**步进安全距离**。

传统 Ray Casting 需要解析方程（如与球体的二次方程），对复杂几何体往往无解。Ray Marching 的做法：

1. 从相机出发，沿光线方向前进
2. 在当前位置 `p = ro + t*rd` 处，计算 `f(p)` = 到最近几何体的距离
3. **安全步进**：向前移动 `f(p)` 的距离——这个距离内保证不会越过任何几何体
4. 如果 `f(p) < ε`（极小值），说明击中了几何体表面
5. 如果 `t > 最大距离` 或迭代次数超限，说明光线进入了背景

```
for i in range(MAX_STEPS):
    p = ro + t * rd
    d = scene_sdf(p)
    if d < SURF_DIST: return t  # 击中
    t += d                       # 安全步进
    if t > MAX_DIST: break       # 未击中
return -1  # miss
```

```cpp
SDFResult rayMarch(Vec3 ro, Vec3 rd) {
    float t = 0.02f;  // 起始偏移，避免自交
    for(int i = 0; i < MAX_STEPS; i++) {
        Vec3 p = ro + rd * t;
        SDFResult r = sceneMap(p);
        if(r.dist < SURF_DIST) {
            hit.dist = t;
            hit.matID = r.matID;
            return hit;
        }
        t += r.dist;
        if(t > MAX_DIST) break;
    }
    return hit; // miss: dist=-1
}
```

**为什么这是"安全"的？**  
SDF 的 Lipschitz 条件保证：函数值 `f(p)` 是点 `p` 到最近表面的真实距离，因此在步进 `f(p)` 的距离内，不可能越过任何几何体表面——就像知道最近的墙离你 2 米，你安全地走 2 米，不可能穿墙。

### 2.7 法线计算：中心差分

击中表面后，我们需要该点的法线用于着色。SDF 梯度方向就是法线方向（SDF 是光滑函数，其梯度恒为单位向量）：

```
n = normalize(grad f(p))
grad f ≈ (f(p+ε,0,0)-f(p-ε,0,0), f(p+0,ε,0)-f(p-0,ε,0), f(p+0,0,ε)-f(p-0,0,ε)) / (2ε)
```

中心差分相比前向差分精度更高（O(ε²) vs O(ε)）：

```cpp
Vec3 calcNormal(Vec3 p) {
    const float eps = 0.001f;
    float dx = sceneMap(p + Vec3(eps,0,0)).dist - sceneMap(p - Vec3(eps,0,0)).dist;
    float dy = sceneMap(p + Vec3(0,eps,0)).dist - sceneMap(p - Vec3(0,eps,0)).dist;
    float dz = sceneMap(p + Vec3(0,0,eps)).dist - sceneMap(p - Vec3(0,0,eps)).dist;
    return Vec3(dx, dy, dz).normalized();
}
```

每次法线计算需要 6 次 SDF 求值，这是 Ray Marching 的主要性能开销之一。

### 2.8 软阴影：Ray Marching 的天然礼物

软阴影（Penumbra，半影）是阴影边缘的模糊区域，由面光源的部分遮挡产生。在传统 Shadow Map 中实现软阴影需要 PCSS 等复杂算法。

在 Ray Marching 中，软阴影几乎是免费的：从表面点向光源发射一条射线，在 march 过程中，记录最小"接近度"：

```
f_shadow(p_surf, p_light):
    shadow = 1.0
    t = mint
    while t < maxt:
        p = p_surf + t * dir_to_light
        h = scene_sdf(p)
        if h < ε: return 0  # 完全被遮挡
        shadow = min(shadow, k * h / t)  # k 控制半影软度
        t += h
    return clamp(shadow, 0, 1)
```

`k * h / t` 的直觉：当光线距离障碍物很近（h 小）且距离出发点不远（t 小）时，遮挡度高 → 阴影深；反之边界处 h/t 有一个渐变，形成半影。参数 `k` 越大，半影越锐利。

```cpp
float softShadow(Vec3 ro, Vec3 rd, float mint, float maxt, float k) {
    float res = 1.0f;
    float t = mint;
    for(int i = 0; i < 64; i++) {
        float h = sceneMap(ro + rd * t).dist;
        if(h < 0.001f) return 0.0f;
        res = std::min(res, k * h / t);
        t += h;
        if(t > maxt) break;
    }
    return clampf(res, 0.0f, 1.0f);
}
```

本项目 `k=12`，产生的阴影边缘柔和且有层次感。

### 2.9 环境光遮蔽（AO）

AO 估算表面点附近几何体对环境光的遮蔽程度。正式做法是在半球方向上采样，但有一种轻量近似：沿法线方向做多次 SDF 采样。

在距离表面 `hr` 处，理想情况下 SDF 值应该恰好是 `hr`（如果附近没有其他几何体）。如果实际 SDF 值 `d < hr`，说明附近有遮挡物，遮蔽量为 `hr - d`：

```
occ = sum_over_i( (h_i - SDF(p + h_i * n)) * decay^i )
AO = clamp(1 - c*occ, 0, 1)
```

```cpp
float calcAO(Vec3 pos, Vec3 nor) {
    float occ = 0.0f;
    float sca = 1.0f;
    for(int i = 0; i < 5; i++) {
        float hr = 0.01f + 0.12f * float(i) / 4.0f;
        Vec3 aopos = nor * hr + pos;
        float dd = sceneMap(aopos).dist;
        occ += (hr - dd) * sca;
        sca *= 0.95f;  // 衰减因子，远处遮挡权重更低
    }
    return clampf(1.0f - 3.0f * occ, 0.0f, 1.0f);
}
```

5次采样，距离从 0.01 到 0.13，衰减系数 0.95。这是一个 O(1) 的 AO 近似，效果出人意料地好。

---

## ③ 实现架构

### 整体渲染管线

```
相机参数 (ro, rd)
       ↓
   rayMarch(ro, rd)
       ↓           ↘ miss → skyColor(rd)
   hit point pos
       ↓
   calcNormal(pos)
       ↓
   getMaterial(matID)
       ↓
   shade(pos, nor, rd, mat, lights)
       │→ softShadow(pos, lightDir)
       │→ calcAO(pos, nor)
       │→ Blinn-Phong: diff + spec + ambient
       ↓
   雾效混合
       ↓
   tonemapACES
       ↓
   gammaCorrect
       ↓
   输出像素 [0-255]
```

### 关键数据结构

**SDFResult**：场景 SDF 的返回值，同时携带距离和材质ID。这个设计允许场景中不同物体有不同材质，而不需要两次求值：

```cpp
struct SDFResult {
    float dist;   // 到最近表面的距离
    int matID;    // 最近物体的材质ID（-1表示miss）
};
```

**Material**：简化的 PBR 参数集，用 roughness/metallic 驱动 Blinn-Phong 的高光指数和反射颜色混合：

```cpp
struct Material {
    Vec3 albedo;      // 基础颜色
    float roughness;  // 粗糙度：驱动高光shininess
    float metallic;   // 金属度：驱动specColor和diffuse衰减
    float emissive;   // 自发光强度
};
```

### SDF 场景组织：顺序最小化

场景 SDF 函数 `sceneMap` 按顺序测试所有几何体，取距离最小者。这是最直接的组织方式，对于物体数量 < 20 的小场景足够高效：

```cpp
SDFResult sceneMap(Vec3 p) {
    SDFResult res = { sdPlane(p, -1.0f), 0 };  // 地面作为初始值
    
    float d = sdSphere(p - sphereCenter, 1.5f);
    if(d < res.dist) { res.dist = d; res.matID = 1; }
    
    // ... 其他几何体类似处理
    return res;
}
```

每次 Ray March 步进都调用一次 `sceneMap`，128 步最多调用 128 次，每次测试约 7 个基元，总计约 896 次 SDF 求值/像素。800×600 = 480000 像素 × 896 ≈ 4.3亿次 SDF 求值，但每次只是几个加减乘除，现代 CPU 0.75 秒跑完合理。

### 相机模型

使用标准透视相机，通过 FOV 和宽高比构建光线方向：

```cpp
Vec3 camFwd = (camTarget - camPos).normalized();
Vec3 camRight = camFwd.cross(camUp).normalized();
Vec3 camUpOrtho = camRight.cross(camFwd).normalized();
float tanHalfFov = tanf(fov * 0.5f * PI / 180.0f);

// 像素 (x,y) → NDC → 世界空间光线方向
float nx = (2.0f * (x + 0.5f) / W - 1.0f) * aspect * tanHalfFov;
float ny = (1.0f - 2.0f * (y + 0.5f) / H) * tanHalfFov;
Vec3 rd = (camFwd + camRight * nx + camUpOrtho * ny).normalized();
```

注意 ny 的符号：`1.0f - 2.0f * (y+0.5f)/H`——y 从屏幕顶部到底部递增，这里翻转使 y=0 对应图像顶部（天空），y=H-1 对应底部（地面），坐标系正确。

---

## ④ 关键代码解析

### 4.1 完整场景 SDF：7种基元 + 平滑融合

```cpp
SDFResult sceneMap(Vec3 p) {
    // 地面平面，高度 y=-1
    float dPlane = sdPlane(p, -1.0f);
    SDFResult res = { dPlane, 0 };
    
    // 中心大球（金属球），球心 (0, 0.5, 0)，半径 1.5
    float dSphere1 = sdSphere(p - Vec3(0, 0.5f, 0), 1.5f);
    if (dSphere1 < res.dist) { res.dist = dSphere1; res.matID = 1; }
    
    // 左侧蓝盒，中心 (-3.5, 0, 0.5)，半尺寸 (0.7, 1.0, 0.7)
    Vec3 pb = p - Vec3(-3.5f, 0.0f, 0.5f);
    float dBox = sdBox(pb, Vec3(0.7f, 1.0f, 0.7f));
    if (dBox < res.dist) { res.dist = dBox; res.matID = 2; }
    
    // 右侧圆环，中心 (3.5, 0.3, 0)，大半径 1.0，小半径 0.35
    Vec3 pt = p - Vec3(3.5f, 0.3f, 0.0f);
    float dTorus = sdTorus(pt, 1.0f, 0.35f);
    if (dTorus < res.dist) { res.dist = dTorus; res.matID = 4; }
    
    // 后方胶囊体：线段从 (-1.5, -0.5, -3.5) 到 (1.5, 0.5, -3.5)，半径 0.4
    Vec3 capA = Vec3(-1.5f, -0.5f, -3.5f);
    Vec3 capB = Vec3(1.5f, 0.5f, -3.5f);
    float dCapsule = sdCapsule(p, capA, capB, 0.4f);
    if (dCapsule < res.dist) { res.dist = dCapsule; res.matID = 3; }
    
    // 左后圆柱，中心 (-3, -0.5, -2.5)，半径 0.5，高 0.5
    Vec3 pcyl = p - Vec3(-3.0f, -0.5f, -2.5f);
    float dCyl = sdCylinder(pcyl, 0.5f, 0.5f);
    if (dCyl < res.dist) { res.dist = dCyl; res.matID = 6; }
    
    // 前景小球群：4个小球，用 smooth union 融合
    float dSmall = 1e10f;
    float smallPositions[4][3] = {
        { 2.0f, -0.7f, 2.5f},
        { 2.8f, -0.5f, 2.0f},
        { 1.5f, -0.6f, 1.8f},
        { 2.3f, -0.3f, 1.5f}
    };
    float smallRadii[4] = { 0.3f, 0.25f, 0.2f, 0.22f };
    for(int i = 0; i < 4; i++) {
        Vec3 pp = p - Vec3(smallPositions[i][0], smallPositions[i][1], smallPositions[i][2]);
        float d = sdSphere(pp, smallRadii[i]);
        dSmall = sdfSmoothUnion(dSmall, d, 0.3f);  // k=0.3，中等融合强度
    }
    if (dSmall < res.dist) { res.dist = dSmall; res.matID = 5; }
    
    return res;
}
```

**为什么地面用 matID=0 作初始值而不是 1e10f？**  
地面是场景的"背景托底"，几乎任何视角都能看到。用它作初始值减少了一次条件判断。但要注意这样做必须保证地面 SDF 实现正确，否则会影响整个场景。

### 4.2 着色器：Blinn-Phong + 阴影 + AO

```cpp
Vec3 shade(Vec3 pos, Vec3 nor, Vec3 rd, Material mat, Vec3 lightPos, Vec3 lightCol) {
    Vec3 lightDir = (lightPos - pos).normalized();
    float dist2Light = (lightPos - pos).length();
    
    // 漫反射：兰伯特余弦定律
    float diff = clampf(nor.dot(lightDir), 0.0f, 1.0f);
    
    // Blinn-Phong 高光：用半程向量代替反射向量，减少计算量
    Vec3 viewDir = (-rd).normalized();
    Vec3 halfDir = (lightDir + viewDir).normalized();
    float spec = 0.0f;
    if(diff > 0.0f) {
        // 从 roughness 推导 shininess：roughness=0 → shininess≈∞（镜面），roughness=1 → shininess≈1
        float shininess = 2.0f / (mat.roughness * mat.roughness + 1e-4f) - 2.0f;
        shininess = std::max(shininess, 1.0f);
        spec = powf(clampf(nor.dot(halfDir), 0.0f, 1.0f), shininess);
    }
    
    // 软阴影
    float shadow = softShadow(pos + nor * 0.01f, lightDir, 0.02f, dist2Light, 12.0f);
    
    // AO：5步法线方向采样
    float ao = calcAO(pos, nor);
    
    // 点光源衰减：1/(1 + kd²)，比 1/r² 更温和，避免近光源时过爆
    float attenuation = 1.0f / (1.0f + 0.05f * dist2Light * dist2Light);
    
    // 金属工作流：金属度控制 specColor（金属颜色带 albedo）和漫反射（金属无漫反射）
    Vec3 specCol = mix3(Vec3(0.04f, 0.04f, 0.04f), mat.albedo, mat.metallic);
    Vec3 diffCol = mat.albedo * (1.0f - mat.metallic);
    
    // 三项相加：ambient（含AO）+ diffuse（含阴影）+ specular（含阴影）
    Vec3 ambient = diffCol * Vec3(0.15f, 0.18f, 0.22f) * ao;
    Vec3 diffuse = diffCol * lightCol * diff * shadow * attenuation;
    Vec3 specular = specCol * lightCol * spec * shadow * attenuation;
    
    Vec3 color = ambient + diffuse + specular;
    
    // 自发光：torus 有轻微自发光效果
    if(mat.emissive > 0.0f) {
        color += mat.albedo * mat.emissive * 0.5f;
    }
    
    return color;
}
```

**`pos + nor * 0.01f` 的作用**：阴影射线起始点从表面法线方向偏移 0.01 个单位，避免自阴影（表面点由于数值精度导致 SDF 值略负，向自身投射阴影）。偏移量太小 → 仍然自阴影；太大 → 阴影悬空（Terminator Problem）。

### 4.3 ACES Filmic 色调映射

ACES（Academy Color Encoding System）是好莱坞标准色彩空间。对应的 Filmic 曲线：

```
f(x) = (x(ax+b)) / (x(cx+d)+e)
a=2.51, b=0.03, c=2.43, d=0.59, e=0.14
```

这是一条 S 形曲线：低光部分被压暗（增加对比度），高光部分被压缩（防止过曝），中间调过渡自然。

注意这里 Vec3 之间不支持直接除法，需要逐分量计算：

```cpp
Vec3 tonemapACES(Vec3 x) {
    float a = 2.51f, b = 0.03f, c = 2.43f, d = 0.59f, e = 0.14f;
    Vec3 num = x*(x*a + Vec3(b,b,b));
    Vec3 den = x*(x*c + Vec3(d,d,d)) + Vec3(e,e,e);
    // 逐分量除法：避免 Vec3/Vec3 运算符未定义的错误
    return clamp3(Vec3(num.x/den.x, num.y/den.y, num.z/den.z), 0.0f, 1.0f);
}
```

这正是第一次编译时遇到的错误：写成了 `Vec3(1,1,1) / (...)` 试图做 Vec3 除法，但 Vec3 只定义了 `operator/(float)`。修复方式：改为逐分量显式相除。

### 4.4 棋盘格地面

地面增加棋盘格纹理，增强空间感：

```cpp
if(hit.matID == 0) {
    float cx = floorf(pos.x) + floorf(pos.z);
    float checker = fmodf(fabsf(cx), 2.0f) < 1.0f ? 1.0f : 0.85f;
    color = color * checker;
}
```

`floor(x) + floor(z)` 的奇偶性决定格子颜色——奇偶交替正好形成棋盘图案。这里 `fmodf(fabsf(cx), 2.0f) < 1.0f` 处理了负坐标的情况（`fabsf` 避免负数取模的符号问题）。

---

## ⑤ 踩坑实录

### 坑1：ACES 色调映射中 Vec3/Vec3 除法

**症状**：第一次编译，1个错误：`error: no match for 'operator/' (operand types are 'Vec3' and 'Vec3')`

**错误假设**：以为 `Vec3(1,1,1) / (x*c + ...)` 能表示逐元素除法，像 GLSL 的 `vec3 / vec3` 那样。

**真实原因**：我的 `Vec3` 结构体只定义了 `operator/(float t)`，没有 `operator/(Vec3)`。在 GLSL/HLSL 中 `vec / vec` 是内置的，但 C++ 需要手动定义或手动展开。

**修复方式**：将 `Vec3(1,1,1) / den` 改为逐分量 `Vec3(num.x/den.x, num.y/den.y, num.z/den.z)`。

**教训**：写 C++ SDF 渲染器时，所有类似 GLSL 的向量操作都需要手动检查是否在 C++ 中定义了对应的运算符。不能把 GLSL 习惯直接移植。

### 坑2：stb_image_write.h 的 -Wextra 警告

**症状**：编译通过（exit 0），但输出了大量 `warning: missing initializer` 警告，都来自 `stb_image_write.h`。

**问题**：SKILL.md 要求"0 errors 0 warnings"，但这些警告来自第三方库，不是我们代码的问题。

**解决方式**：使用 `-isystem..` 代替 `-I..` 引入 stb 目录。GCC/Clang 对 `-isystem` 路径下的头文件不报 `-Wextra/-Wall` 警告（只报严重错误），这是处理第三方库警告的标准做法。

**修复前**：`g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra`（14个警告）  
**修复后**：`g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra -isystem..`（0警告）

### 坑3：软阴影自交问题（设计预防）

**可能的症状**：整个场景看起来完全是黑色的阴影，或者地面上有奇怪的黑色噪点。

**原因**：阴影射线从表面点出发，如果起点就在几何体内部（浮点精度导致 SDF 略负），sceneMap 会立刻返回距离 < 0.001，softShadow 误判为被完全遮挡，返回 0。

**预防措施**：阴影射线起点沿法线方向偏移：`pos + nor * 0.01f`。这确保起点在表面外部一个安全距离，不会因精度问题触发自交。

本项目在设计时直接加入了这个偏移，没有踩到这个坑，但在早期测试中有意留意了这个问题。

### 坑4：地面棋盘格负坐标问题（设计预防）

**可能的症状**：地面棋盘格在相机前方正常，但在某些方向或位置出现错位。

**错误做法**：`fmodf(cx, 2.0f) < 1.0f`——当 `cx` 为负时，`fmodf` 的结果在 C++ 中是负数，导致 `< 1.0f` 判断错误。

**正确做法**：`fmodf(fabsf(cx), 2.0f) < 1.0f`——先取绝对值，再取模，保证结果始终在 `[0, 2)` 范围内。

### 坑5：Ray Marching 起始距离

**可能的症状**：相机附近的物体有奇怪的白色或黑色斑点。

**原因**：`t = 0.0f` 起步，相机本身的 SDF 求值位置可能就在某个物体内或表面，导致立刻"击中"。

**修复**：`t = 0.02f`，给一个小的起始偏移，跳过相机位置本身可能的数值问题区域。

---

## ⑥ 效果验证与数据

### 渲染结果

![SDF场景渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-23-sdf-ray-marching/sdf_scene_output.png)

### 量化数据

| 指标 | 数值 |
|------|------|
| 输出分辨率 | 800 × 600 |
| 文件大小 | 96KB（97558 bytes） |
| 渲染时间 | 0.75 秒（CPU，无多线程） |
| 像素均值 | 185.3 |
| 像素标准差 | 30.4 |
| R/G/B 均值 | 177.1 / 185.1 / 193.6 |
| 上方区域均值 | 212.6（天空较亮） |
| 下方区域均值 | 187.2 |
| 上方蓝色通道 | 224.6 |
| 下方蓝色通道 | 191.9 |

### 验证脚本输出

```
图像尺寸: (800, 600)
像素均值: 185.3  标准差: 30.4
R均值: 177.1, G均值: 185.1, B均值: 193.6
上方区域均值: 212.6  下方区域均值: 187.2
✅ 像素统计正常
✅ 文件大小: 97558 bytes > 10KB
上方蓝色通道: 224.6, 下方蓝色通道: 191.9
✅ 坐标系正确：天空（蓝色）在上方
=== 所有验证通过 ===
```

**坐标系验证**：上方蓝色通道（224.6）> 下方（191.9），天空确实在图像上方，地面在下方，坐标系正确。

### 性能分析

**理论求值次数估算**：
- 分辨率：800 × 600 = 480,000 像素
- 平均 Ray March 步数：约 40-80 步（受场景深度分布影响）
- 每步 `sceneMap` 求值：约 7 个 SDF 基元
- 命中后法线：6 次额外 SDF 求值
- 命中后软阴影：约 20-30 次 SDF 求值
- 命中后 AO：5 次 SDF 求值

粗略估算：480,000 × (60步×7 + 31 + 5) = 约 2.1 亿次 SDF 基元求值，0.75 秒 = 2.8 亿次/秒。

若加入 SSE/AVX SIMD 并行化，理论可提升 4-8 倍；若加入 OpenMP 多线程（8核），可再提升 8 倍，达到 0.01 秒量级。

---

## ⑦ 总结与延伸

### 本次实现的局限性

1. **单次反射**：目前没有实现镜面反射光线追踪，金属球的高光来自 Blinn-Phong 近似，而非真实反射
2. **无折射**：透明材质（玻璃、水）的折射光线未实现
3. **无体积光/参与介质**：雾效是简单的距离衰减，不是真正的体积散射
4. **单线程**：未使用 SIMD 或多线程，性能有较大提升空间
5. **场景 SDF 是硬编码的**：没有 BVH 等加速结构，场景物体多时线性增加开销
6. **无抗锯齿**：每个像素只采一条光线，没有 MSAA 或 TAA

### 可以继续优化的方向

1. **实现反射**：命中金属表面后，基于菲涅尔方程发射反射光线，继续 Ray March
2. **SDF 形变**：在场景 SDF 中加入 `twist`、`bend`、`repeat` 等空间变换，创造复杂几何
3. **加入时间变量**：让场景随时间变化（旋转、波浪、噪声位移），实现动画渲染
4. **SIMD 加速**：用 SSE/AVX 一次处理 4/8 个 SDF 求值
5. **分层 BVH**：对场景物体按空间位置分区，减少不相关的 SDF 求值
6. **全局光照**：加入路径追踪模式，发射次级光线计算间接光照

### 与本系列其他文章的关联

- **05-07 Photon Mapping**：同样处理全局光照，但使用双向 Monte Carlo 而非纯 Ray March
- **05-01 ReSTIR**：在光栅化管线中用 Reservoir 采样解决多光源问题，与本文的解析光照互补
- **04-30 Shadow Volume**：光栅化硬阴影的代表；本文的 Ray March 软阴影代表了完全不同的阴影计算路径
- **05-21 Thin Film Iridescence**：如果把彩虹色薄膜材质加到本场景的圆环面上，会产生非常漂亮的效果

SDF Ray Marching 是一种"函数即场景"的渲染哲学——与其问"三角面片在哪里"，不如问"你离最近的表面有多远"。这个视角的转变带来了极高的灵活性，是理解现代实时渲染（Lumen、DFAO、SDF字体）的重要基础。
