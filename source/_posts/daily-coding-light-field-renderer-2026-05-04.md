---
title: "每日编程实践: Light Field Renderer — 光场渲染器"
date: 2026-05-04 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 光场
  - 视角合成
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-04-light-field-renderer/light_field_output.png
---

# 每日编程实践：Light Field Renderer — 光场渲染器

今天实现了一个基于**双平面参数化（Two-Plane Parameterization）**的 4D 光场渲染器。光场这个概念在计算机图形学和计算摄影学的交汇处，它回答了一个根本性的问题：**如果我们能捕捉所有方向上所有光线的颜色，我们能重建任意视角的图像吗？**

![光场渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-04-light-field-renderer/light_field_output.png)

---

## 一、背景与动机

### 传统渲染的视角限制

传统的实时或离线渲染器，每一帧都需要从特定相机位置重新追踪光线或执行光栅化。如果你想改变观察角度，整套计算必须重来。在游戏引擎中，这通常是可以接受的——GPU 足够快，每帧 16ms 内完成渲染不是难事。

但想象以下场景：

1. **VR 头显中的头部追踪**：微小的头部移动（几毫米）就需要精确的视差变化。如果每次移动都重新渲染完整场景，延迟会让人晕眩。
2. **全息显示器**：需要从极其密集的角度阵列同时呈现不同图像，传统渲染完全无法应对。
3. **神经辐射场（NeRF）之前的自由视角视频**：捕捉真实场景后，想从任意角度观看，但不可能在后期重建完整的几何模型。
4. **光场相机（Lytro）**：拍摄后重新对焦，本质就是从捕捉到的光场中重新渲染图像。

光场渲染提供了一种不同的思路：**预先捕捉场景在多个视角下的外观，然后通过插值重建任意中间视角**，而不需要了解场景的几何结构。

### 工业界实际应用

- **Google Immersive View**：使用类似光场的技术在 Google Maps 中实现沉浸式鸟瞰视角
- **Meta Presence Platform**：Codec Avatars 项目利用密集相机阵列捕捉光场，实现照片级真实的虚拟形象
- **Lytro Cinema**：电影拍摄中使用光场相机，后期可以改变焦平面和视角
- **Microsoft HoloLens**：AR 显示中利用光场原理优化近眼显示的景深感知
- **游戏引擎的 Lightmap**：某种意义上，预烘焙光照贴图也是一种低分辨率的光场近似

### 光场 vs 其他视角合成方法

| 方法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| 光场渲染 | 捕捉多视角插值 | 不需要几何重建 | 存储量大，视角范围有限 |
| NeRF | 神经网络隐式表达 | 任意视角，存储小 | 训练慢，推理慢 |
| 3D Gaussian Splatting | 显式高斯基元 | 实时渲染 | 需要几何重建 |
| 基于深度的视角合成 | 深度图 + 重投影 | 实时可行 | 遮挡区域填补困难 |
| IBR（图像插值） | 直接图像空间操作 | 简单快速 | 视差大时失效 |

---

## 二、核心原理

### 2.1 光场的数学定义

光场（Light Field）的概念由 Adelson & Bergen (1991) 提出，称为"全光函数"（Plenoptic Function）：

$$L(x, y, z, \theta, \phi, \lambda, t)$$

它描述了在空间中任意一点 $(x,y,z)$、向任意方向 $(\theta, \phi)$ 观察时、波长 $\lambda$ 的光、在时刻 $t$ 的辐射亮度（radiance）。

这是一个 7 维函数，存储它需要天文数字的空间。实际应用中必须降维简化。

**关键观察**：在**自由空间**（没有参与介质、没有吸收）中，沿一条光线传播时辐射亮度不变。因此，一条光线可以用它与两个平行平面的交点完全确定。

### 2.2 双平面参数化（Two-Plane Parameterization）

Levoy & Hanrahan (1996) 和 Gortler et al. (1996) 分别独立提出了相同的参数化方案：使用两个平行平面上的坐标 $(u,v)$ 和 $(s,t)$ 描述一条光线。

```
UV平面（相机平面）      ST平面（图像平面）
        |                        |
   v ↑  |                        |  t ↑
     |  |   相机阵列             |  |   图像像素
   --+--+---→                 --+--+---→
     |  u                        |  s
```

具体来说：
- **UV 平面**：放置相机阵列，$(u,v)$ 是第几个相机
- **ST 平面**：每个相机拍摄的图像，$(s,t)$ 是像素坐标
- **一条光线** $= (u, v, s, t)$：从 UV 平面上 $(u,v)$ 位置的相机，拍摄 ST 像素 $(s,t)$

这将 7D 函数降为 4D：$L(u, v, s, t)$。

**物理直觉**：想象一个相机阵列，每台相机拍摄相同场景。相机 $(u_0, v_0)$ 拍到的像素 $(s, t)$ 和相机 $(u_1, v_1)$ 拍到的像素 $(s, t)$，对应的是空间中完全不同的光线——前者由 $(u_0,v_0) \to (s,t)$ 确定，后者由 $(u_1,v_1) \to (s,t)$ 确定。

### 2.3 光场重建：如何合成新视角？

已知在 UV 网格上若干相机的图像，如何合成 UV 平面上任意位置 $(u^*, v^*)$ 处相机看到的图像？

最简单的方法是**双线性插值**：

给定查询位置 $(u^*, v^*)$，找到 UV 网格中四个最近的相机：
- $(u_0, v_0)$，$(u_1, v_0)$，$(u_0, v_1)$，$(u_1, v_1)$，其中 $u_0 \le u^* \le u_1$，$v_0 \le v^* \le v_1$

定义插值权重：
$$\alpha = \frac{u^* - u_0}{u_1 - u_0}, \quad \beta = \frac{v^* - v_0}{v_1 - v_0}$$

对于合成图像的每个像素 $(s, t)$：
$$L(u^*, v^*, s, t) = (1-\beta) \cdot [(1-\alpha) \cdot L(u_0,v_0,s,t) + \alpha \cdot L(u_1,v_0,s,t)]$$
$$+ \beta \cdot [(1-\alpha) \cdot L(u_0,v_1,s,t) + \alpha \cdot L(u_1,v_1,s,t)]$$

**直觉解释**：这就是在相机阵列中做"位置插值"——新视点位于四个已知相机之间，它看到的每个方向的光近似等于四个相邻相机对同一方向光的加权平均。

**为什么这个近似可行？**  
当相机间距相对于场景深度足够小时，同一像素 $(s,t)$ 在相邻相机中对应的是几乎相同的场景区域（只有微小的视差）。插值结果在视差小的区域非常准确，在视差大的区域（前景物体边缘）会有"重影"（ghosting）。

### 2.4 深度辅助重建（理论，本次实现简化版）

更精确的方法是使用深度信息进行"光线重定向"：

已知新视点 $(u^*, v^*)$ 要渲染像素 $(s, t)$，该像素对应的场景深度为 $d$。则：

在平行于 UV 的深度 $d$ 处，新视点的光线与深度 $d$ 平面交点为：
$$(s', t') = (s + (u^* - u_0) \cdot \frac{f}{d}, \; t + (v^* - v_0) \cdot \frac{f}{d})$$

其中 $f$ 是焦距。然后从参考相机 $(u_0, v_0)$ 中采样 $(s', t')$ 而不是 $(s, t)$。

这可以大幅减少重影，是 Lumigraph 的核心改进。本次实现采用简化的纯双线性插值（不做深度校正），以保证代码在 C++17 标准库内完成，无外部依赖。

---

## 三、实现架构

### 3.1 整体数据流

```
场景定义 (spheres + planes + lights)
    │
    ▼
光场采集阶段：
  for vi in [0, UV_RES):
    for ui in [0, UV_RES):
      Camera cam = getCameraPos(ui, vi)
      for each pixel (s, t) in ST_RES x ST_RES:
        ray = cam.getRay(s, t)
        color = scene.shade(ray)         ← 软光追
        depth = scene.intersect(ray).t   ← 深度记录
    │
    ▼
LightField[vi][ui] = SubImage(ST_RES x ST_RES)
    │
    ▼
输出图像合成阶段：
  ① UV 网格可视化（5×5 缩略图）
  ② 信息面板（参数展示、相机分布图）
  ③ 新视角合成（3个插值视角）
    │
    ▼
PNG 写出（无外部依赖）
```

### 3.2 关键数据结构

**SubImage**：存储单个相机拍摄的图像
```cpp
struct SubImage {
    std::vector<Color> pixels;  // ST_RES * ST_RES 个颜色
    std::vector<float> depth;   // 对应深度值（用于后续深度校正扩展）
    
    Color& at(int s, int t) { return pixels[t * ST_RES + s]; }
    float& depthAt(int s, int t) { return depth[t * ST_RES + s]; }
};
```

**LightField**：整个 4D 光场
```cpp
struct LightField {
    std::vector<std::vector<SubImage>> images; // images[v][u]
    float uvSpacing = 0.25f;  // UV 平面相机间距
    Vec3 baseTarget = {0, 0, -2.5f};  // 所有相机的目标点
    float cameraZ = 2.5f;  // 相机平面的 Z 坐标
};
```

为什么用 `images[v][u]` 而非 `images[u][v]`？  
因为图像的内存访问模式是行优先（row-major）：外层索引 v 对应"行"，内层 u 对应"列"，与 y-x 坐标系一致，有利于后续按行渲染时的缓存命中。

### 3.3 相机布局设计

5×5 = 25 个相机均匀分布在 UV 平面上：

```
相机间距 uvSpacing = 0.25 units
中心相机 (ui=2, vi=2) 在 (0, 0, 2.5)
左上角相机 (0,0) 在 (-0.5, -0.5, 2.5)
右下角相机 (4,4) 在 (+0.5, +0.5, 2.5)
```

相机间距 0.25 units 相对于场景深度（2.5~7.5 units）的比例约为 1:10~1:30，保证了插值区域视差适中——不至于全黑（间距太小），也不会鬼影严重（间距太大）。

### 3.4 职责划分

| 模块 | 职责 | CPU/Memory |
|------|------|------------|
| `Scene` | 几何体 + 着色，提供 `shade()` 和 `intersect()` | 纯 CPU 光追 |
| `LightField` | 4D 数组存储 + `capture()` + `queryView()` | CPU，~25×128×128×3×4B ≈ 1.5MB |
| `Image` | 输出像素缓冲 + 基础绘图 | CPU，900×900×3 ≈ 2.4MB |
| PNG 写出 | deflate-store + zlib + CRC32 | CPU |

---

## 四、关键代码解析

### 4.1 光场采集

```cpp
void capture(const Scene& scene) {
    for (int vi = 0; vi < UV_RES; vi++) {
        for (int ui = 0; ui < UV_RES; ui++) {
            Vec3 camPos = getCameraPos(ui, vi);
            Camera cam(camPos, baseTarget, ST_RES, ST_RES, 55.0f);
            
            for (int t = 0; t < ST_RES; t++) {
                for (int s = 0; s < ST_RES; s++) {
                    Ray ray = cam.getRay(s + 0.5f, t + 0.5f);
                    images[vi][ui].at(s, t) = scene.shade(ray).clamp();
                    // 存储深度，备用（深度校正视角合成）
                    auto hit = scene.intersect(ray);
                    if (hit.hit) {
                        images[vi][ui].depthAt(s, t) = hit.t;
                    }
                }
            }
        }
    }
}
```

**为什么使用 `s + 0.5f`？**  
像素采样时取像素中心而非左上角，避免系统性偏移。如果用 `s + 0.0f`，所有射线都穿过像素左上角，导致图像在亚像素级别有半像素偏移，在不同相机视图之间插值时会引入可见的对齐错误。

### 4.2 相机模型

```cpp
Ray getRay(float px, float py) const {
    float aspect = (float)width / height;
    float tanHalfFov = std::tan(fov * 3.14159265f / 360.0f);
    // 将像素坐标归一化到 [-1, 1] 范围，考虑宽高比
    float u = (2.0f * px / width - 1.0f) * aspect * tanHalfFov;
    // 注意 y 轴翻转：PNG 坐标 y 向下，但世界坐标 y 向上
    float v = (1.0f - 2.0f * py / height) * tanHalfFov;
    Vec3 dir = (forward + right * u + up * v).normalize();
    return Ray(pos, dir);
}
```

**为什么要乘以 `tanHalfFov`？**  
投影矩阵的数学来源：在视锥体中，当 z = 1 时，x 方向的范围是 `[-tan(fov/2), tan(fov/2)]`。直接用归一化坐标不乘这个因子，等效于 FOV = 45° 或 90°（取决于缩放方式），导致视角错误。

**`aspect * tanHalfFov` 的作用**：  
宽度方向的"张开角度"需要乘以宽高比，否则宽屏相机看到的物体会被横向压缩。

### 4.3 双线性插值核心

```cpp
Color queryView(float u_norm, float v_norm, int s, int t) const {
    // 将浮点 UV 坐标限制在合法范围内
    float ui_f = std::max(0.0f, std::min((float)(UV_RES-1), u_norm));
    float vi_f = std::max(0.0f, std::min((float)(UV_RES-1), v_norm));

    // 找到四个最近的网格点
    int ui0 = (int)ui_f, ui1 = std::min(ui0 + 1, UV_RES - 1);
    int vi0 = (int)vi_f, vi1 = std::min(vi0 + 1, UV_RES - 1);
    float uFrac = ui_f - ui0;  // U 方向插值权重
    float vFrac = vi_f - vi0;  // V 方向插值权重

    // 取四个邻近相机在 (s,t) 处的颜色
    auto& c00 = images[vi0][ui0].at(s, t);  // 左上
    auto& c10 = images[vi0][ui1].at(s, t);  // 右上
    auto& c01 = images[vi1][ui0].at(s, t);  // 左下
    auto& c11 = images[vi1][ui1].at(s, t);  // 右下

    // 双线性插值：先在 U 方向插值，再在 V 方向插值
    return Color::lerp4(c00, c10, c01, c11, uFrac, vFrac).clamp();
}
```

**双线性插值的数学本质**：  
它等价于在四个点定义的双曲面上进行采样。对于颜色 $C$：

$$C(u,v) = (1-v)[(1-u)C_{00} + u \cdot C_{10}] + v[(1-u)C_{01} + u \cdot C_{11}]$$

这是一个**双线性**（非线性！）函数——单独对 u 或 v 是线性的，但对两者同时是二次的。不过对于光场中视角变化幅度不大的情况，这种近似已经足够准确。

**为什么不用更高阶的插值？**  
双三次（Bicubic）插值能减少"模糊感"，但在光场渲染中，颜色值跨越深度不连续的边缘时，高阶插值会引入振铃（Ringing）效应，看起来比双线性更差。光场插值的核心挑战不在于平滑度，而在于深度不连续处的处理。

### 4.4 场景着色器

```cpp
Color shade(const Ray& ray, int depth = 0) const {
    if (depth > 3) return Color(0.1f, 0.15f, 0.25f);  // 递归限制

    auto hit = intersect(ray);
    if (!hit.hit) {
        // 天空渐变：y 方向 [-1,1] 映射到蓝色渐变
        float t = 0.5f * (ray.dir.y + 1.0f);
        t = std::max(0.0f, std::min(1.0f, t));
        Color sky0(0.6f, 0.7f, 1.0f);   // 地平线颜色（浅蓝）
        Color sky1(0.15f, 0.2f, 0.5f);  // 天顶颜色（深蓝）
        return sky0.lerp(sky1, t);
    }
    
    // 环境光
    Color finalColor = hit.material.albedo * 0.08f;

    // 对每个光源计算漫反射 + 镜面
    for (const auto& light : lights) {
        Vec3 L = (light - hit.point).normalize();
        float NdotL = std::max(0.0f, N.dot(L));

        if (!shadowRay(hit.point, light)) {
            // Lambert 漫反射
            Color diffuse = hit.material.albedo * NdotL * 0.85f;

            // Blinn-Phong 镜面
            Vec3 H = (L + V).normalize();  // Half vector
            float NdotH = std::max(0.0f, N.dot(H));
            float specPow = std::max(1.0f, (1.0f - hit.material.roughness) * 128.0f);
            float spec = std::pow(NdotH, specPow) * (1.0f - hit.material.roughness);
            Color specular = Color(1, 1, 1) * spec * 0.6f;

            finalColor += (diffuse + specular) * (1.0f / lights.size());
        }
    }

    // 金属球的镜面反射（递归）
    if (hit.material.metallic > 0.1f && depth < 2) {
        Vec3 reflDir = Vec3::reflect(ray.dir * (-1.0f), N).normalize();
        Ray reflRay(hit.point + N * 0.001f, reflDir);
        Color reflColor = shade(reflRay, depth + 1);
        float fresnel = hit.material.metallic;
        // 金属：反射色 = 球本身颜色 × 反射颜色（菲涅尔混合）
        finalColor = finalColor * (1.0f - fresnel * 0.7f) + 
                     reflColor * hit.material.albedo * (fresnel * 0.7f);
    }

    return finalColor;
}
```

**为什么 Blinn-Phong 用 Half vector 而不是 Reflect vector？**  
经典 Phong 模型使用视线反射向量 $R = 2(N \cdot L)N - L$ 计算 $\max(0, R \cdot V)^n$。Blinn 改进版用 half vector $H = \text{normalize}(L + V)$ 计算 $\max(0, N \cdot H)^n$：

1. **更快**：半角向量计算比反射向量少一次向量运算
2. **更准**：在掠射角（grazing angle）时，Blinn-Phong 的高光衰减更物理正确
3. **更稳定**：当 $R \cdot V < 0$（相机在高光背面）时 Phong 会截断，Blinn-Phong 的 $N \cdot H$ 不会出现这个问题

**光照乘以 `1.0f / lights.size()` 的原因**：  
多个光源简单累加会过曝。除以光源数量使得总能量守恒——无论几个光源，场景的整体亮度保持一致。真实物理中应该用每个光的实际强度，但这里用归一化的艺术化近似。

### 4.5 软阴影射线

```cpp
bool shadowRay(const Vec3& from, const Vec3& to) const {
    Vec3 dir = (to - from).normalize();
    float dist = (to - from).length();
    Ray r(from + dir * 0.01f, dir);  // ← 关键：偏移起点
    for (const auto& s : spheres) {
        auto h = s.intersect(r);
        if (h.hit && h.t < dist - 0.01f) return true;  // ← 关键：只计算光源前方的遮挡
    }
    // ... 平面同理
    return false;
}
```

**`from + dir * 0.01f` 的意义（自相交偏移）**：  
如果阴影射线从完全在表面上的点出发，由于浮点精度问题，它可能立即与发射它的表面相交（自阴影），导致所有点都认为自己在阴影中（全黑）。

偏移 0.01 units 将起点略微推离表面，跳过自身几何体。偏移量需要在"足够大以避免自交"和"足够小以不越过细薄几何体"之间权衡。本场景最小几何特征约 0.3 units，0.01 是安全的。

**`h.t < dist - 0.01f` 的意义**：  
阴影射线只在 `[0.01, dist-0.01]` 范围内检测遮挡。减去末端偏移是为了防止光源位置本身（如球形光源）被当成遮挡物。

---

## 五、踩坑实录

### Bug 1：FONT 数组的 designated initializer 编译失败

**症状**：
```
sorry, unimplemented: non-trivial designated initializers not supported
```

编译器报了几十行完全相同的错误，令人困惑。

**错误假设**：  
以为是 C++17 不支持 designated initializers，打算升级到 C++20。

**真实原因**：  
GCC 的 C++17 只支持**简单 designated initializers**（`struct Foo { int a; }; Foo f = {.a = 1};`），不支持数组的 designated initializers（`int arr[] = { [5] = 42 }`）。虽然 C99 的 C 语言标准支持，C++ 要到 C++20 才正式支持，而且 GCC 的 C++20 实现中对"non-trivial"类型（含构造函数的 struct）仍有限制。

我的 FONT 数组使用了 `[' '] = {0x00,...}` 语法，在这个 GCC 版本下完全不支持。

**修复方式**：  
将静态 designated initializer 改为通过 `initFont()` 函数动态初始化：

```cpp
static uint8_t FONT[128][7];
static bool FONT_INITIALIZED = false;

void initFont() {
    if (FONT_INITIALIZED) return;
    FONT_INITIALIZED = true;
    // 先全部清零
    for (int i = 0; i < 128; i++)
        for (int j = 0; j < 7; j++)
            FONT[i][j] = 0;
    // 再逐字符赋值
    uint8_t d0[7] = {0x0E,0x11,0x13,0x15,0x19,0x11,0x0E};
    for(int i=0;i<7;i++) FONT['0'][i] = d0[i];
    // ... 以此类推
}
```

**教训**：  
- C++17 ≠ C99。C 语言的很多"当然支持"的语法在 C++ 中是扩展或尚未标准化的
- 看到大量重复的"sorry, unimplemented"时，第一反应应该是"这个语法不被支持"，而不是"代码逻辑错了"

### Bug 2：unused variable 警告

**症状**：
```
warning: variable 'V' set but not used [-Wunused-but-set-variable]
```

**原因**：  
在 `shade()` 函数中，我在 diffuse 计算块内定义了 `Vec3 V`，但是在反射部分的代码重构时，反射代码块里重新计算了方向，导致之前定义的 `V` 在 diffuse 块后未被使用。

**修复**：  
删除 diffuse/specular 部分中计算 `V` 的那行（`V` 只在反射部分需要，而反射是独立的 if 块，在那里直接用 `ray.dir` 更清晰）。

**教训**：  
`-Wall -Wextra` 的 `-Wunused-but-set-variable` 会捕获"赋值了但从未读取"的变量——这种情况比"声明了但从未赋值"更隐蔽，通常说明代码有逻辑重构后的残留。

### 架构陷阱：光场存储顺序 images[v][u] vs images[u][v]

在代码中，光场使用 `images[vi][ui]` 存储。这个顺序不是随意的。

考虑渲染循环：
```cpp
for (int t = 0; t < ST_RES; t++) {      // 外层：行
    for (int s = 0; s < ST_RES; s++) {  // 内层：列
        Color c = lf.queryView(u, v, s, t);
        out.setPixel(ox + s, oy + t + 12, c);
    }
}
```

`queryView` 中访问的是 `images[vi][ui].at(s, t)`，即 `pixels[t * ST_RES + s]`。外层循环枚举 `t`，内层枚举 `s`，使得内存访问是连续的（步长 1），L1 缓存命中率最高。

如果写成 `images[ui][vi]` 并且渲染循环不变，查询时仍然需要 `images[vi][ui]`，会产生混淆。统一使用"v-first"（行主序）是标准做法。

---

## 六、效果验证与数据

### 量化输出检验

```python
from PIL import Image
import numpy as np

img = Image.open('light_field_output.png')
pixels = np.array(img).astype(float)

print(f"图像尺寸: {img.size}")          # (900, 900)
print(f"像素均值: {pixels.mean():.1f}") # 74.6
print(f"像素标准差: {pixels.std():.1f}")# 52.8
```

**验证结果**：

| 指标 | 值 | 标准 | 状态 |
|------|-----|------|------|
| 文件大小 | 2.4 MB | > 10 KB | ✅ |
| 像素均值 | 74.6 | 10~240 | ✅ |
| 像素标准差 | 52.8 | > 5 | ✅ |
| 输出尺寸 | 900×900 | 900×900 | ✅ |

### 性能数据

```
光场采集：25 个相机 × 128×128 像素 = 409,600 条光线
光追深度：最多 3 次递归（环境光 + 2 次反射）
阴影射线：每像素最多 2 条（对应 2 个光源）
最坏情况射线数：409,600 × (1 + 2 + 2) ≈ 2,048,000 条
实际运行时间：~0.11 秒
吞吐量：约 18,600,000 次/秒（包括相交测试 + 着色）
```

### 光场插值可视化

输出图像布局：

- **上部 5×5 网格**（630×630 px）：25 个原始捕捉视图缩略图，中心相机用金色边框标注
- **右侧信息面板**（260×630 px）：光场参数 + 相机阵列可视化图（25 个圆点，金色为中心）
- **下部 3 个合成视角**（280×220 px × 3）：
  - Novel-1：u=0.7, v=0.7（左上区域，靠近第一个网格点的内插）
  - Novel-2：u=2.0, v=2.0（正中心，恰好落在一个网格点，无插值误差）
  - Novel-3：u=3.3, v=3.3（右下区域，四格插值）

Novel-2 视图与 images[2][2]（中心相机）完全一致，验证了插值在网格点处的一致性。

---

## 七、总结与延伸

### 局限性

1. **存储量随分辨率快速增长**：4D 光场的存储是 O(U × V × S × T)。本次 5×5×128×128×3 = ~1.2M 颜色值约 4.8MB。真实 Lytro 相机的 100×100×376×376 约需要 4.2GB——必须压缩。

2. **视角范围有限**：只能合成 UV 网格覆盖范围内的视角。超出范围需要外推，而外推的光场质量很差。

3. **深度不连续处的重影**：本实现使用纯双线性插值，在前景/背景边界处会出现"双重图像"（ghosting）。深度校正可以大幅减少这个问题，但需要深度图和更复杂的重采样。

4. **不支持动态场景**：光场是预捕捉的，场景变化后必须重新采集全部视图。

### 可优化方向

1. **深度校正插值（Depth-Corrected Light Field Rendering）**：已有深度缓存，下一步可以实现 Levoy-Hanrahan 论文中的深度辅助重采样，消除重影。

2. **压缩光场（Compressed Light Field）**：使用 PCA 或小波变换压缩 4D 数组，能以 10:1 到 50:1 的压缩比保持视觉质量。

3. **神经光场（NeRF / Neural Light Field）**：用 MLP 隐式表达光场 $L(u,v,s,t)$，存储仅需几 MB，且理论上支持无限分辨率。

4. **GPU 加速**：25 个独立视角完全可以并行渲染——将光场采集映射到 CUDA kernel，理论上应有 25× 加速。

5. **相机阵列可变密度**：在视角差异大的区域（遮挡边缘附近）使用更密的相机采样，在平坦区域减少采样，类似于 MIP 映射的思路。

### 与本系列其他文章的关联

- **[PRT（预计算辐射传输）](../prt)（05-03）**：同样是"预计算"思路，但 PRT 预计算的是光照与表面响应的乘积（球谐系数），光场预计算的是观察方向函数；两者都是用离散化+插值换取实时渲染能力
- **[ReSTIR 直接光照](../restir)（05-01）**：都涉及采样与重用——ReSTIR 重用时间域和空间域的采样，光场重用不同相机的预计算结果
- **[Radiance Cascade GI](../radiance-cascade)（05-02）**：辐射级联也是多分辨率分层存储光照信息，与光场的 UV 多分辨率扩展有相似的结构思路

光场渲染是一座连接传统计算机图形学和现代神经渲染的桥梁。理解了双平面参数化的本质，再看 NeRF 的输入设计（也是相机位置 + 方向两组参数）、3DGS 的视角相关着色，会发现它们都在用不同的方式回答同一个问题：**如何高效地表达和重建任意视角下的光照信息？**

---

*代码仓库: [daily-coding-practice/2026/05/05-04-light-field-renderer](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/05/05-04-light-field-renderer)*  
*运行环境: C++17, g++ -O2, ~0.11s*
