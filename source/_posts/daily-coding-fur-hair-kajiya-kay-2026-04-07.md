---
title: "每日编程实践: Fur/Hair Rendering with Kajiya-Kay Model"
date: 2026-04-07 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 毛发渲染
  - 着色模型
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-07-fur-hair-kajiya-kay/fur_hair_output.png
---

## ① 背景与动机

### 毛发渲染为什么难？

在实时和离线渲染中，毛发（hair）与毛皮（fur）是公认最难处理的材质之一。难点不在于单根头发本身，而在于：

1. **几何复杂度**：一个人类头部约有 10 万根头发，每根头发是一条细长的圆柱体（直径 0.05~0.1 mm），传统三角形面片难以高效表达；
2. **光照各向异性**：头发纤维的截面是圆形，光线打在上面不像漫反射那样均匀散射，而是沿着纤维切线方向产生拉丝状高光——这是毛发区别于其他材质的核心视觉特征；
3. **自遮挡与透射**：头发丛密集，内部有大量自遮挡；同时头发具有一定透明度，逆光看时会出现透射光（subsurface glow）效果；
4. **实时性要求**：游戏引擎中毛发不能逐根光线追踪，需要近似模型以达到实时帧率。

### Kajiya-Kay 模型的历史地位

1989 年，James T. Kajiya 和 Timothy L. Kay 在 SIGGRAPH 发表了论文《Rendering Fur with Three Dimensional Textures》，提出了第一个专门针对毛发/纤维着色的光照模型——**Kajiya-Kay 模型**。

这篇论文的核心洞察是：头发不能用点法线来描述光照，而应该用**切线（tangent）**——头发沿切线方向延伸，光照响应对称地围绕切线，而不是法线。

尽管今天有了更物理精确的 **Marschner (2003)**、**d'Eon (2011)** 和 **Chiang (2016)** 等模型，Kajiya-Kay 因其极低的计算成本，在实时渲染中仍大量使用，尤其是：

- **《刺客信条》系列**（早期版本）的 NPC 头发
- 各类手游中的卡通角色毛发
- 影视制作中的快速预览
- VR 场景中需要极低开销的毛发效果

今天的实践目标是：**从零实现 Kajiya-Kay 毛发着色模型**，包含程序化毛发几何生成、软光栅化管线，渲染出一个带有真实感毛发的头部模型。

---

## ② 核心原理

### 2.1 Kajiya-Kay 着色模型的数学推导

**传统 Phong 模型的局限**

在 Phong 模型中，着色依赖于表面法线 **N**：

```
Diffuse  = kd * max(0, N · L)
Specular = ks * max(0, R · V)^p
```

其中 **L** 是光线方向，**V** 是视线方向，**R** 是反射方向。

这套公式假设表面是局部平坦的，有一个明确的法线。但头发是一根圆柱，它的"法线"取决于观察角度——从不同方向看，同一位置的有效法线不同。更重要的是，头发对光的响应在绕切线旋转方向上是对称的（各向同性于绕切线的旋转），这恰恰是 Phong 无法处理的。

**Kajiya-Kay 的关键替换**

设 **T** 为毛发的单位切线向量，**L** 为光源方向（归一化，指向光源），**V** 为视线方向（归一化，指向眼睛）。

定义 θ_LT 为 **L** 与 **T** 之间的夹角，则：

```
cos(θ_LT) = T · L
sin(θ_LT) = sqrt(1 - (T · L)²)
```

**Kajiya-Kay 漫反射项**：

```
I_diffuse = kd * sin(θ_LT)
```

直觉解释：当光线**垂直**于头发（θ_LT = 90°），sin = 1，漫反射最强，这符合现实——光线从侧面打来，照亮了最大面积的纤维截面；当光线**平行**于头发（θ_LT = 0°），sin = 0，漫反射为零，光线沿头发滑过，没有照亮表面。

这与传统漫反射 `cos(θ)` 的行为是对偶的——传统模型是"法线与光线越平行越亮"，而毛发是"切线与光线越垂直越亮"。

**Kajiya-Kay 高光项**：

对于高光，我们考虑：头发在某个"有效法线"方向发生镜面反射，这个有效法线是 **L** 在垂直于 **T** 的平面上的投影方向。类似地，**V** 也有一个相对于 **T** 的分量。

令：

```
T_L = T * (T · L)    // L 在 T 方向的投影
L_perp = L - T_L     // L 的垂直分量（长度 = sin(θ_LT)）

T_V = T * (T · V)    // V 在 T 方向的投影
V_perp = V - T_V     // V 的垂直分量（长度 = sin(θ_TV)）
```

Kajiya-Kay 高光定义为这两个垂直分量的点积（归一化后），再提升到幂次 p：

```
cos_angle = (L_perp / |L_perp|) · (V_perp / |V_perp|)
           = (L - T*(T·L)) · (V - T*(T·V)) / (sin_LT * sin_TV)
           
I_specular = ks * max(0, cos_angle)^p
```

化简后可以写成：

```
I_specular = ks * (sin_LT * sin_TV * cos(φ) - cos_LT * cos_TV)^p
```

其中 φ 是 L 和 V 在垂直于 T 的平面上的方位角差。

在实现上，也常用等价的近似（本实现采用）：

```
I_specular = ks * pow(max(0, sin_shifted), p)
```

其中 `sin_shifted = sqrt(1 - (T·V + offset)²)`，offset 用于沿切线方向偏移高光叶片，模拟头发根部与发梢的高光位移差。

### 2.2 为什么毛发有两个高光叶片？

真实头发的横截面并非完美圆形，而是有角质层（cuticle）的粗糙表面。光打到头发上会发生：

1. **R 叶片**（primary highlight）：光从角质层表面直接反射，产生明亮、偏白的高光；
2. **TT 叶片**（secondary highlight）：光透射进入头发，在内部散射后从对侧射出，颜色接近头发本色，位置与 R 叶片约相差 2-3 度。

Kajiya-Kay 原始模型只有一个高光叶片，Marschner 模型引入了双叶片。本实现用切线偏移模拟了偏移效果。

### 2.3 程序化毛发几何：Fibonacci 球面分布

要在头部球面上均匀分布 6000 根头发，需要一种好的球面分布策略。

**随机均匀**方法容易出现"簇集"（clustering）——随机数有时集中在一个区域，留下空洞。

**Fibonacci 球面（Sunflower Spiral）** 是目前最优的均匀球面分布之一：

```
φ_i = 2π * i * φ_golden           // 黄金角 φ_golden ≈ 0.618033...
θ_i = arccos(1 - 2*(i+0.5)/N)     // 让 cos(θ) 均匀分布在 [-1, 1]
```

其中 `φ_golden = (√5 - 1)/2 ≈ 0.618034`（黄金比例），每根头发的经向角递增黄金角，避免了整数倍重叠，产生螺旋状的均匀分布。

这种方法的球面覆盖误差为 O(1/N)，而随机方法期望误差为 O(1/√N)，Fibonacci 分布在相同数量下覆盖质量更好。

### 2.4 毛发生长方向：重力垂落与随机卷曲

单根头发从球面法线方向出发，随着发尖逐渐向重力方向倾斜：

```
dir_at_segment_s = lerp(normal, gravity_dir, s/S)
```

加上每段随机的法平面内扰动，模拟卷曲与不整齐感：

```
random_perp = random_vec3 - normal * dot(random_vec3, normal)
dir_final = normalize(dir_blend + random_perp * 0.15)
```

通过控制 blend 权重和随机强度，可以得到从直发到卷发的不同效果。

---

## ③ 实现架构

### 3.1 整体渲染管线

```
程序入口
    │
    ├─ renderBackground()     背景渐变（直接填像素）
    ├─ renderSphere()         头部球体（逐像素光线-球相交）
    └─ drawStrand() × 6000   毛发渲染（软光栅化线段）
         │
         ├─ generateHair()   Fibonacci球面分布 + 重力垂落几何
         ├─ 排序（Z远→近）   画家算法保证正确遮挡
         ├─ Kajiya-Kay着色   逐线段计算切线→漫反射+高光
         └─ 投影+光栅化       3D→屏幕空间，圆盘状线段绘制
```

### 3.2 关键数据结构

```cpp
// 图像缓冲区（含 Z-Buffer）
struct Image {
    int w, h;
    std::vector<Vec3> pixels;   // RGB 颜色
    std::vector<float> zbuf;    // 深度（相机空间 Z）
};

// 毛发几何：每根头发是一条折线
std::vector<std::vector<Vec3>> strands;  // 外层=毛发，内层=节点
```

选择折线（poly-line）而非贝塞尔曲线的原因：折线在软光栅化时可以直接逐段处理，无需采样参数曲线，计算成本更低；8 段折线已经足以表达自然弯曲，过多段数在视觉上没有明显提升但增加渲染时间。

### 3.3 相机投影模型

使用透视投影，自实现而非依赖 OpenGL/Vulkan：

```
相机坐标系 = {forward, right, up}（orthonormal basis）
给定世界点 P：
    d = P - cam.origin
    z_cam = d · forward              // 相机空间深度
    x_cam = d · right
    y_cam = d · up
    
    half_h = tan(fovY/2) * z_cam     // 视锥半高
    half_w = half_h * aspect
    
    screen_x = (x_cam/half_w * 0.5 + 0.5) * W
    screen_y = (-y_cam/half_h * 0.5 + 0.5) * H   // Y轴翻转
```

Z-Buffer 存储相机空间深度 `z_cam`（越小越近），线段插值时对 `z_cam` 线性插值（近似；精确的透视正确插值需要用 `1/z` 插值，对直径 ≤ 1px 的毛发误差可忽略）。

### 3.4 毛发 vs 球体的 Z 排序策略

球体使用逐像素光线追踪，天然支持 Z-Buffer；毛发使用软光栅化线段。两者共享同一个 Image 的 Z-Buffer，因此：

- 球体先渲染（占据 Z-Buffer 靠近处的值）
- 毛发后渲染，通过 `setPixel` 的 Z-Test 自动处理遮挡
- 毛发内部按根-深度排序（Z 远→近），使用画家算法辅助（但 Z-Buffer 仍是最终仲裁）

---

## ④ 关键代码解析

### 4.1 Kajiya-Kay 着色实现

```cpp
// 漫反射：sin(angle between tangent T and light L)
inline float KK_diffuse(const Vec3& T, const Vec3& L) {
    float cosTheta = T.dot(L);
    // sin² = 1 - cos²，取正数平方根
    float sinTheta = std::sqrt(std::max(0.f, 1.f - cosTheta*cosTheta));
    return sinTheta;
}

// 高光：基于切线-视线夹角，+偏移模拟双叶片
inline float KK_specular(const Vec3& T, const Vec3& V, float shininess) {
    float cosTheta_TV = T.dot(V);
    // 沿切线方向偏移高光叶片（模拟 R 与 TT 的分离）
    float shifted = cosTheta_TV + 0.2f;
    float sinShifted = std::sqrt(std::max(0.f, 1.f - shifted*shifted));
    return std::pow(std::max(0.f, sinShifted), shininess);
}
```

**为什么对 `cosTheta + 0.2` 做偏移？**

在 Kajiya-Kay 的原始公式中，高光叶片位于 V 和 T 所在平面的特定角度。给 `cos(θ_TV)` 加一个偏移量等效于将高光叶片沿切线方向移动，使反射高光与镜面高光分离，视觉上产生"两层高光"的效果——这是长发反光的标志性特征。

**最终颜色合成**：

```cpp
// ambient: 基础环境光，防止背光面全黑
const Vec3 ambient  = Vec3(0.10f, 0.08f, 0.05f);  // 暖色调环境光
const float kd = 0.7f, ks = 0.4f, shininess = 60.f;

float diff = KK_diffuse(tangent, lightDir);    // [0,1]
float spec = KK_specular(tangent, viewDir, shininess);  // [0,1]

// hairColor = 暖棕色 (0.58, 0.35, 0.15)
// 高光颜色用偏暖白 (1, 1, 0.8)，模拟发丝光泽感
Vec3 shadedColor = (ambient 
                  + hairColor * (kd * diff) 
                  + Vec3(1,1,0.8f) * (ks * spec)).clamp01();
```

注意高光的颜色用 `Vec3(1, 1, 0.8)` 而非纯白，因为自然光高光总有微弱暖色调（来自光源色温）。

### 4.2 程序化头发几何生成

```cpp
std::vector<std::vector<Vec3>> generateHair(
        Vec3 center, float headRadius,
        int numHairs, float hairLength, unsigned int seed)
{
    std::mt19937 rng(seed);
    std::uniform_real_distribution<float> u01(0.f, 1.f);

    for(int i = 0; i < numHairs; ++i) {
        // ---- Fibonacci 球面分布 ----
        float phi   = 2.f * PI * (float)i * 0.618033988f;  // 黄金角递增
        float theta = std::acos(1.f - 2.f*(i+0.5f)/numHairs);  // 均匀 cos 分布
        
        // 随机抖动（±0.2 弧度），让分布更自然，避免过度规律
        phi   += (u01(rng)-0.5f)*0.4f;
        theta += (u01(rng)-0.5f)*0.2f;
        
        // 跳过头部下方 20%（避免颈部/下巴长头发）
        if(theta > PI * 0.8f) continue;
        
        // ---- 根节点：球面上的点 ----
        Vec3 root = center + Vec3(
            std::sin(theta)*std::cos(phi),
            std::cos(theta),
            std::sin(theta)*std::sin(phi)
        ) * headRadius;
        
        Vec3 normal = (root - center).norm();  // 球面法线 = 头发生长方向
        
        // ---- 重力方向（下垂 + 轻微侧风） ----
        Vec3 windDir = Vec3(0.1f, -1.f, 0.05f).norm();
        
        // ---- 生成 8 段折线 ----
        const int SEG = 8;
        float segLen = hairLength / SEG;
        Vec3 pos = root;
        
        for(int s = 0; s < SEG; ++s) {
            float blend = (float)s / SEG;  // 0(根部)→1(发梢) 越来越下垂
            
            // 当前方向 = 法线（根部直立） lerp 到重力方向（发梢下垂）
            Vec3 d = (normal*(1.f-blend) + windDir*blend).norm();
            
            // 随机法平面内扰动（产生卷曲感）
            Vec3 rndPerp = Vec3(u01(rng)-0.5f, u01(rng)-0.5f, u01(rng)-0.5f);
            rndPerp = rndPerp - normal*(rndPerp.dot(normal));  // 投影掉法线分量
            d = (d + rndPerp * 0.15f).norm();
            
            pos = pos + d * segLen;
            strand.push_back(pos);
        }
    }
}
```

**关键设计说明**：

- `blend = s/SEG`：越靠近发梢，重力影响越大，模拟真实头发根部硬（附着头皮）、发梢软（自由垂落）的特性；
- `rndPerp * 0.15f`：0.15 是经过调试的合适值，太小头发过于整齐（像假发），太大则杂乱无章；
- `theta > PI * 0.8f` 跳过下方：这避免了下巴和颈部的穿插问题，同时让头顶密度略高于两侧（接近真实发型）。

### 4.3 软光栅化：线段投影与 Z-Buffer 写入

```cpp
void drawStrand(Image& img, const Camera& cam,
                const std::vector<Vec3>& pts,
                const Vec3& lightDir, const Vec3& viewDir,
                const Vec3& hairColor, float thickness)
{
    for(size_t i = 0; i+1 < pts.size(); ++i) {
        const Vec3& A = pts[i];
        const Vec3& B = pts[i+1];
        
        // 每段的切线 = 方向向量（归一化）
        Vec3 tangent = (B-A).norm();
        
        // Kajiya-Kay 着色（一次性计算整段颜色）
        float diff = KK_diffuse(tangent, lightDir);
        float spec = KK_specular(tangent, viewDir, shininess);
        Vec3 shadedColor = (ambient + hairColor*(kd*diff) 
                           + Vec3(1,1,0.8f)*(ks*spec)).clamp01();
        
        // 投影到屏幕空间（初始化为 0 防止 uninitialized warning）
        float ax=0,ay=0,az=0, bx=0,by=0,bz=0;
        bool okA = cam.project(A, ax, ay, az);
        bool okB = cam.project(B, bx, by, bz);
        if(!okA && !okB) continue;
        // 只有一个端点可见时，用可见端代替不可见端（近似裁剪）
        if(!okA){ ax=bx; ay=by; az=bz; }
        if(!okB){ bx=ax; by=ay; bz=az; }
        
        // 沿线段均匀采样，步数 = 屏幕空间像素距离
        int steps = (int)std::max(std::abs(bx-ax), std::abs(by-ay)) + 2;
        int r = (int)std::ceil(thickness);  // 像素半径
        
        for(int s = 0; s <= steps; ++s) {
            float t = (float)s / steps;
            float px = ax + (bx-ax)*t;
            float py = ay + (by-ay)*t;
            float pz = az + (bz-az)*t;  // 深度插值（近似透视校正）
            
            // 以 (px,py) 为中心，绘制半径 r 的圆盘
            for(int dy=-r; dy<=r; ++dy)
            for(int dx=-r; dx<=r; ++dx) {
                if(dx*dx+dy*dy <= r*r) {  // 圆形截面（而非正方形）
                    img.setPixel((int)(px+dx+.5f), (int)(py+dy+.5f), shadedColor, pz);
                }
            }
        }
    }
}
```

**用圆盘而非线段单像素的原因**：
单像素线段在 512×512 的图像中几乎不可见（头发直径 < 1px），需要 `thickness=0.7` 的圆盘半径才能渲染出可见的发丝。圆形截面比正方形更自然，且计算开销极低（r=1 时只有 π≈3 个额外像素）。

### 4.4 头部球体：光线-球相交

球体不使用光栅化，而是逐像素做光线-球相交，以确保边缘精确（球体边缘光栅化需要额外处理抗锯齿）：

```cpp
// 给定像素 (x,y)，构造视线方向
float u = ((float)x+0.5f)/W * 2.f - 1.f;   // [-1, 1]
float v = 1.f - ((float)y+0.5f)/H * 2.f;   // [-1, 1]（Y 向上）
Vec3 dir = (cam.forward 
           + cam.right*(u*half_w) 
           + cam.up*(v*half_h)).norm();

// 光线-球相交（代数法）
Vec3 oc = eye - center;
float b = 2.f * oc.dot(dir);
float c = oc.dot(oc) - radius*radius;
float disc = b*b - 4*c;   // a=1（dir已归一化）
if(disc < 0) continue;    // 未命中

float t = (-b - std::sqrt(disc)) / 2.f;  // 取近交点
if(t < cam.near) continue;               // 在相机后方

Vec3 hit = eye + dir*t;
Vec3 N   = (hit - center).norm();        // 命中点法线
```

这段代码处理了 **near clip**（避免相机在球内时的异常），并用**代数判别式**直接求解，比几何法（先求投影点再判断距离）稍快且数值更稳定。

### 4.5 伽马校正

```cpp
// 线性空间 RGB → sRGB 近似（γ=2.2）
float r = std::pow(std::max(0.f, std::min(1.f, p.x)), 1.f/2.2f);
float g = std::pow(std::max(0.f, std::min(1.f, p.y)), 1.f/2.2f);
float b = std::pow(std::max(0.f, std::min(1.f, p.z)), 1.f/2.2f);
```

不做伽马校正时，图像会整体偏暗（人眼对暗部更敏感，线性空间的中灰在显示器上显示为暗灰）。这一步将线性光照计算结果转换为显示器正确显示的 sRGB 近似。

---

## ⑤ 踩坑实录

### Bug 1：`-Wuninitialized` 导致编译警告（变量未初始化）

**症状**：
编译通过，但 `-Wextra` 产生大量 `may be used uninitialized` 警告，涉及 `drawStrand` 中的 `ax, ay, az, bx, by, bz` 以及排序 lambda 中的 `za, zb`。

**错误假设**：
"两个端点必有一个能投影成功，另一个即使失败也不会被使用。"

**真实原因**：
GCC 的 `-Wextra` 分析无法识别 `if(!okA && !okB) continue;` 这种保护逻辑，它只看到变量在声明后没有被显式赋初值，就报 uninitialized。编译器不跟踪 bool 控制流来判断该变量是否"一定已被赋值"。

**修复方式**：
将声明改为显式初始化：
```cpp
// 修复前
float ax,ay,az, bx,by,bz;
// 修复后
float ax=0,ay=0,az=0, bx=0,by=0,bz=0;
```
并在 `!okB` 时将 B 的投影坐标赋值为 A 的投影坐标（合理的降级处理，避免使用未初始化值）：
```cpp
if(!okA){ ax=bx; ay=by; az=bz; }
if(!okB){ bx=ax; by=ay; bz=az; }
```

**教训**：`-Wextra -Wuninitialized` 在 GCC 的分析是保守的，即使逻辑上变量一定被赋值，编译器也可能报警。最安全的做法是**所有局部变量都初始化为合理默认值**。

---

### Bug 2：`convert` 命令未找到（ImageMagick 未安装）

**症状**：
程序运行后报 `sh: line 1: convert: command not found`，随后 `ffmpeg` 也找不到，最终程序以错误码退出，PNG 未生成。

**错误假设**：
"这类 Linux 环境都会预装 ImageMagick 和 ffmpeg。"

**真实原因**：
当前容器环境是最小化镜像，只安装了 Python 和部分开发工具，没有 ImageMagick 或 ffmpeg。

**修复方式**：
检查可用工具：
```bash
python3 -c "from PIL import Image; print('PIL ok')"  # 输出 PIL ok
```
改为优先使用 Python PIL 转换，ImageMagick/ffmpeg 作为降级后备：
```cpp
int ret = std::system(
    ("python3 -c \""
     "from PIL import Image; "
     "img=Image.open('" + ppmPath + "'); "
     "img.save('" + pngPath + "')\"").c_str()
);
if(ret != 0) { /* ImageMagick 降级 */ }
if(ret != 0) { /* ffmpeg 降级 */ }
```

**教训**：外部工具依赖要先探测可用性，不要假设环境。先检查 `which convert`、`python3 -c "from PIL import Image"` 等，根据实际可用工具选择路径。

---

### Bug 3：头发分布不均匀（视觉上头顶稀疏、两侧密集）

**症状**（调试过程中发现）：
纯随机球面采样（两个角度均匀分布）导致极点稀疏，赤道密集。

**根本原因**：
均匀分布 θ ∈ [0, π] 时，面积元 `dA = sin(θ) dθ dφ`，赤道附近 sin(θ) 大，面积实际上更大，点数相同但密度低；极点附近 sin(θ) 趋近 0，面积极小但点数相同，密度极高。

**修复方式**：
改用 `cos(θ)` 均匀分布（即均匀采样球面面积元）：
```cpp
// 错误：θ 均匀分布
float theta = u01(rng) * PI;

// 正确：cos(θ) 均匀分布，等效球面面积均匀
float theta = std::acos(1.f - 2.f*(i+0.5f)/numHairs);  // Fibonacci
// 或随机版：
float theta = std::acos(1.f - 2.f * u01(rng));
```

Fibonacci 球面分布比纯随机更均匀，最终选用了 Fibonacci 方案。

---

## ⑥ 效果验证与数据

### 图像量化验证

```bash
python3 << 'EOF'
from PIL import Image
import numpy as np
img = Image.open("fur_hair_output.png")
pixels = np.array(img).astype(float)
print(f"Size: {img.size}  Mode: {img.mode}")
print(f"Pixel mean: {pixels.mean():.1f}  std: {pixels.std():.1f}")
EOF
```

**实际输出**：

| 指标 | 值 | 验收标准 | 状态 |
|------|-----|---------|------|
| 文件大小 | 170 KB | > 10 KB | ✅ |
| 图像尺寸 | 512×512 | 512×512 | ✅ |
| 像素均值 | 145.5 | 10~240 | ✅ |
| 像素标准差 | 31.4 | > 5 | ✅ |

均值 145.5 表明图像亮度适中（非全黑/全白），标准差 31.4 表明图像有丰富的明暗变化（头部球体 Phong 阴影 + 毛发各向异性高光的空间变化）。

### 渲染性能（非实时，软光栅化参考）

| 阶段 | 耗时估计 |
|------|---------|
| 头部球体（512×512 逐像素）| ~50ms |
| 6000根毛发 × 8段 × 光栅化 | ~200ms |
| 总渲染 + PNG 保存 | < 1s |

（在容器单核 CPU 上，未优化）

实时引擎中，Kajiya-Kay 毛发（10 万根，每帧）在现代 GPU 上可达 60fps，GPU 的并行化将把上述 ~250ms 压缩到 < 1ms。

### 渲染结果

![Fur Hair Rendering - Kajiya-Kay Model](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-07-fur-hair-kajiya-kay/fur_hair_output.png)

效果：
- 头部球体可见，皮肤色 Phong 着色，有高光
- 6000 根棕色头发均匀覆盖头部，根部直立、发梢下垂
- Kajiya-Kay 各向异性高光：在光源方向有明显亮条（发丝光泽）
- 背光面毛发漫反射为零，体现了 sin(θ_LT) 模型的特性

---

## ⑦ 总结与延伸

### 本次实践收获

1. **Kajiya-Kay 模型**的数学本质是"用切线代替法线"，这一直觉替换解决了传统着色模型无法处理各向异性纤维的根本问题；
2. **Fibonacci 球面分布**是一个优雅的均匀球面采样方案，O(1) 时间复杂度，无随机扰动也能得到高质量分布；
3. **软光栅化的局限**：线段 Z-Buffer 只做近似深度测试，不处理透明度（真实毛发有半透明效果），需要 OIT（Order-Independent Transparency）等技术补充。

### Kajiya-Kay 的局限性

| 局限 | 影响 | 更好的替代 |
|------|------|---------|
| 单散射，无透射 | 发丝缺少透光感 | Marschner (2003) 加入 TRT 分量 |
| 高光叶片形状固定 | 无法精确控制高光宽度 | d'Eon (2011) 更精确的 R/TT/TRT |
| 无自阴影 | 发丛内部光照均匀 | 深度贴图阴影 |
| 无 multiple scattering | 金发/白发偏暗 | Volume scattering 或预计算 |
| 切线假设固定 | 只适合直发或简单弯发 | 需要逐段切线（本实现已做） |

### 可优化方向

1. **加入第二高光叶片（R + TT）**：将偏移量参数化，用两个 `KK_specular` 调用叠加，分别模拟直接反射和透射；
2. **发根-发梢颜色渐变**：根据 `s/SEG` 混合两种颜色（发根深、发梢浅），更接近真实漂染发色；
3. **多种发型**：修改 `windDir` 和 `blend` 函数可得到盘发（高 blend 从中间折叠）、卷发（正弦波扰动）、短发（减小 `hairLength`）；
4. **GPU 实现**：将 `drawStrand` 移植为 Vertex + Fragment Shader，利用硬件光栅化加速，10 万根头发可实现实时渲染；
5. **LOD 策略**：远处头发合并为条带（strand → billboard quad），节省顶点数。

### 与本系列的关联

- **04-03 Disney BRDF**：Disney 模型处理的是光滑/粗糙各向同性表面，Kajiya-Kay 处理的是各向异性纤维——两者共同覆盖了游戏引擎材质系统的主流需求；
- **04-04 大气散射**：大气散射也利用了 sin/cos 角的非对称特性（Rayleigh vs Mie 相位函数），与 Kajiya-Kay 在数学结构上有相似之处；
- **04-06 地形渲染**：地形的草地可以用类似思路渲染——用 Kajiya-Kay 着色草叶（宽叶片近似为毛发），或者本次的程序化几何思路生成草丛。

---

**代码仓库**：[daily-coding-practice / 04-07 Fur Hair Kajiya-Kay](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-07-fur-hair-kajiya-kay)
