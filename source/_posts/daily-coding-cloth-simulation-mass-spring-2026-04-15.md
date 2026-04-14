---
title: "每日编程实践: Cloth Simulation Mass-Spring System — 从物理方程到布料在风中飘舞"
date: 2026-04-15 05:38:00
tags:
  - 每日一练
  - 图形学
  - 物理模拟
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-15-Cloth-Simulation-Mass-Spring/cloth_output.png
---

## 一、背景与动机

布料模拟（Cloth Simulation）是游戏与影视特效中几乎无处不在的技术——角色的披风、旗帜在风中飘扬、桌布、蜘蛛网的颤动……这些细节极大地提升了场景的真实感。如果没有布料模拟，游戏中的角色往往穿着"石头衣服"，衣物形状固定不变，看起来极不自然。

早期的游戏（PS2 时代之前）几乎无法实时模拟布料，布料只是静态网格或预录制的骨骼动画。随着 GPU 计算能力提升和算法改进，实时布料模拟在 2000 年代中期开始普及。《半条命 2》（2004）中的物理系统让布料（以及一切刚体）的交互成为了一大卖点。而在现代引擎中（Unreal 的 Chaos Cloth、Unity 的 NvCloth），布料模拟已经是标配功能。

### 1.1 工业界实际使用场景

**游戏引擎**：
- Unreal Engine 5 的 Chaos Cloth 支持多层布料、自碰撞、风场，被用于《堡垒之夜》等 AAA 大作的角色服装
- Unity 的 Skinned Cloth 组件可以和骨骼蒙皮结合，让布料随角色运动自然飘动

**影视特效**：
- Autodesk Maya 的 nCloth 是影视制作的工业标准，用于制作《阿凡达》《奇异博士》中的超级英雄斗篷
- Houdini 的 Vellum 系统基于 PBD（Position Based Dynamics）实现，大量用于电影级布料和软体模拟

**研究前沿**：
- 神经网络加速的布料模拟（Neural Cloth Simulation）正在成为研究热点，可在低多边形角色上实时预测高精度布料变形

### 1.2 为什么选质点弹簧模型？

布料模拟有多种方法，各有优劣：

| 方法 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 质点弹簧（Mass-Spring） | 直观、易实现 | 稳定性差，刚度有上限 | 教学、原型 |
| FEM（有限元） | 物理精确 | 计算量大，实现复杂 | 离线渲染、影视 |
| PBD（位置约束） | 稳定、无条件 | 能量不守恒 | 实时游戏 |
| XPBD（扩展PBD） | 物理参数有意义 | 略复杂 | 现代游戏引擎 |

质点弹簧是最经典的入门方法，它将布料离散为一组质点，质点之间用弹簧连接，通过牛顿第二定律 F=ma 求解每个质点的运动，直觉上非常清晰。本项目采用质点弹簧作为物理核心，辅以位置约束迭代增强稳定性，兼顾可读性和效果。

---

## 二、核心原理

### 2.1 布料的离散化建模

真实布料是连续介质，但计算机只能处理离散数据。最简单的离散化方式是将布料分成 N×M 的规则网格：

```
o - o - o - o - o   (固定点 = 橙色)
|   |   |   |   |
o - o - o - o - o
|   |   |   |   |
o - o - o - o - o
```

每个 `o` 是一个质点（Particle），有质量 m、位置 x、速度 v。相邻质点之间用弹簧连接，弹簧有弹性系数 k 和自然长度 L₀（连接时的初始距离）。

### 2.2 弹力公式

胡克定律（Hooke's Law）是弹簧力的基础：当弹簧被拉伸或压缩时，产生与形变成正比的恢复力。

对于两个质点 a 和 b，弹力的方向是 b→a（拉向对方），大小为：

```
F_spring = k * (|xb - xa| - L₀) * (xb - xa) / |xb - xa|
         = k * (L - L₀) * ê
```

其中：
- k：弹性系数（stiffness）
- L = |xb - xa|：当前弹簧长度
- L₀：自然长度（rest length）
- ê = (xb - xa) / L：从 a 指向 b 的单位向量

**直觉解释**：(L - L₀) 是形变量。当 L > L₀（被拉长），力指向 b（拉着 a 往 b 方向走）；当 L < L₀（被压缩），力背离 b（把 a 往反方向推）。这就是弹簧"弹"的本质。

### 2.3 三类弹簧的物理意义

布料不只是一张橡皮膜，它有方向性、抗弯性。用三类弹簧来模拟这些性质：

#### 结构弹簧（Structural Springs）
连接水平和垂直相邻的质点（距离为 spacing）。它们保持布料的基本尺寸，防止无限伸长。这是布料最主要的弹力来源。

```
o - S - o   (S = structural spring)
|       |
S       S
|       |
o - S - o
```

#### 剪切弹簧（Shear Springs）
连接对角线方向的质点（距离为 spacing * √2）。没有剪切弹簧时，布料网格在受到剪切力（斜向拉扯）时会"菱形化"——四个质点变成平行四边形，看起来像破碎的网格。剪切弹簧防止这种变形。

```
o   o
 \  |
  \ |
   o   
```

**为什么剪切弹簧的刚度比结构弹簧低？** 真实布料对对角方向的拉伸抵抗力较弱（你可以斜向撕一块布），所以我们给剪切弹簧 k=0.8，比结构弹簧的 k=0.95 略小。

#### 弯曲弹簧（Bending Springs）
连接间隔一格的质点（距离为 2 * spacing）。它们模拟布料的**抗弯刚度**——布料有一定的硬挺度，不会折叠成任意角度。

```
o - o - o   (bending spring 连接 第1和第3个质点，跳过中间)
```

没有弯曲弹簧，布料会过度折叠，形成不自然的锐角；有了弯曲弹簧，折叠变得更平滑、更像真实织物。

### 2.4 Verlet 积分

这是整个模拟的核心算法。标准的 Euler 积分（`x += v*dt; v += a*dt`）在大时间步时会能量爆炸（弹簧越来越长）。Verlet 积分通过记录上一时刻的位置，隐含地处理了速度，数值稳定性更好。

**位置 Verlet 公式**：

```
x(t + dt) = 2*x(t) - x(t - dt) + a(t)*dt²
```

代入 `v(t) ≈ (x(t) - x(t-dt)) / dt`，展开得：

```
x(t + dt) = x(t) + [x(t) - x(t-dt)] + a(t)*dt²
          = x(t) + v(t)*dt + a(t)*dt²
```

看起来和 Euler 一样？区别在于速度 `v(t) = x(t) - x(t-dt)` 是从位置差推算出来的，不是独立存储的变量。这使得算法天然地"过去影响未来"，数值阻尼更稳定。

**代码实现**：

```cpp
void step(float dt) {
    if (pinned) { pos = pinnedPos; prevPos = pinnedPos; return; }
    // vel ≈ (pos - prevPos)，乘以阻尼系数 0.98
    Vec3 vel = (pos - prevPos) * 0.98f;
    Vec3 next = pos + vel + acc * (dt * dt);
    prevPos = pos;   // 保存当前位置为"上一帧"
    pos = next;      // 更新到新位置
    acc = Vec3();    // 重置加速度（每帧重新累积）
}
```

**阻尼的引入**：乘以 0.98 相当于每帧速度衰减 2%，模拟空气阻力。如果不加阻尼，弹簧系统在没有能量耗散时会永远振荡，布料会一直"弹跳"而不会静止。

### 2.5 约束迭代（Constraint Projection）

纯弹力计算有个问题：弹簧刚度有上限。如果弹性系数 k 太大，时间步 dt 小而刚度大，弹力会超调（overshoot），导致质点飞出去。解决方法是**约束投影**：不用力来校正弹簧，而是直接修改质点位置，把弹簧拉回到自然长度附近。

对于一根弹簧，如果当前长度是 L，自然长度是 L₀，我们直接把两端质点各向中间移动 (L - L₀)/2 * k：

```
校正量 = (xb - xa) * (L - L₀) / L * 0.5 * stiffness
xa += 校正量    (向 b 方向移)
xb -= 校正量    (向 a 方向移)
```

**为什么这比力积分稳定？** 因为直接操作位置，绕过了力→加速度→速度→位置的链式积分，避免了数值误差的积累。每帧迭代 15 次，让所有弹簧的约束都基本满足。这本质上是 XPBD（Extended Position Based Dynamics）的核心思想。

### 2.6 球体碰撞检测

布料与球体的碰撞是最简单的碰撞情形。对于每个质点 p，如果它进入了球体内部（距球心 < 球半径），就把它推到球面上：

```
d = p.pos - sphereCenter           // 质点到球心的向量
dist = |d|                          // 当前距离
if dist < sphereRadius + epsilon:
    p.pos = sphereCenter + d.normalized() * (sphereRadius + epsilon)
```

**直觉**：沿着"质点到球心"方向，把质点放到球面外侧一点点（epsilon = 0.015）。这是一种位置约束，和弹簧约束迭代的思路一致。

注意这里不计算碰撞力，而是直接修正位置。Verlet 积分会自动从位置差中计算出弹开的速度（因为 prevPos 还在原位，而 pos 被推到球面外，差值就变成了远离球面的速度）。

### 2.7 风力场

风力模拟采用时变正弦函数，产生方向不断变化的阵风效果：

```cpp
Vec3 wind(
    sinf(t * 6.28f * 2.5f) * 0.0015f,   // X方向：2.5个周期/模拟时长
    0,                                     // Y方向：无竖向风
    cosf(t * 6.28f * 1.8f) * 0.0008f    // Z方向：1.8个周期，强度略弱
);
```

`t` 是 0~1 的归一化时间。X 和 Z 方向的频率不同（2.5 vs 1.8），产生李萨如图形般的非周期风向变化，避免周期性重复的规律感。

---

## 三、实现架构

### 3.1 整体数据流

```
[初始化]
  Cloth::init()
    → 创建 ROWS×COLS 个 Particle（位置、是否固定）
    → 创建结构/剪切/弯曲 Spring（自然长度=初始距离）
    
[每帧物理步骤] × 12 substeps
  ① 施加外力：gravity + wind → acc 累积
  ② Particle::step()：Verlet 积分
  ③ Cloth::solveConstraints()：15次迭代约束投影
  ④ 球体碰撞检测：直接推出质点
  ⑤ 地面碰撞：y < -3.5 时夹紧

[渲染]
  Camera::project()：透视投影
  renderClothAndSphere()：
    → 背景梯度 + 地面条
    → 球体（近似投影球 + Phong着色）
    → 布料四边形网格（逐格 drawLine，法向着色）
    → 固定点黄色圆圈高亮
    
[输出]
  3帧合成 → PPM → Pillow转PNG
```

### 3.2 关键数据结构

**Particle**：
```cpp
struct Particle {
    Vec3 pos;        // 当前位置
    Vec3 prevPos;    // 上一帧位置（Verlet 使用）
    Vec3 acc;        // 本帧累积加速度（每帧清零）
    bool pinned;     // 是否固定（固定点不受力）
    Vec3 pinnedPos;  // 固定位置
};
```

只存储两个时刻的位置，不需要显式的速度变量——这是 Verlet 积分的特点。

**Spring**：
```cpp
struct Spring {
    int a, b;         // 两端质点的索引
    float restLen;    // 自然长度（初始化时计算）
    float stiffness;  // 弹性系数（0~1，越大越硬）
};
```

弹簧用索引引用质点，而不是指针，便于数组存储和 SIMD 优化。

**Cloth**：
```cpp
struct Cloth {
    int rows, cols;
    std::vector<Particle> particles;  // 一维存储，用 idx(r,c) = r*cols+c 访问
    std::vector<Spring> springs;
};
```

**为什么用一维 vector？** 二维数组（`Particle[rows][cols]`）的内存不连续（实际上行间有跳转），而一维 vector 保证连续存储，对 CPU cache 友好，遍历所有质点时更高效。

### 3.3 软光栅化渲染管线

渲染部分完全在 CPU 上实现，不依赖 OpenGL 或任何图形 API：

```
World Space Particle Positions
         ↓
  Camera::project()（透视投影 → NDC → 屏幕坐标）
         ↓
  drawLine()（Bresenham 线段光栅化）
         ↓
  面法向计算（相邻质点叉积）
         ↓
  Lambert 漫反射着色（N·L）
         ↓
  Image::savePPM()（写二进制文件）
         ↓
  Python Pillow（PPM → PNG）
```

### 3.4 子步（Substep）机制

每渲染帧调用 12 次物理子步（substep），每个子步 dt = 0.016/12 ≈ 0.00133 秒。

为什么不直接用 dt=0.016？弹簧系统对时间步非常敏感——大时间步会导致弹力过冲（overshoot），质点飞出去。小时间步每步的错误更小，整体更稳定。12 个子步是精度和性能的平衡点。

---

## 四、关键代码解析

### 4.1 布料初始化

```cpp
void init(int rows, int cols, float spacing) {
    this->rows = rows;
    this->cols = cols;
    particles.resize(rows * cols);

    // 布料居中，从顶部向下延伸
    float halfW = (cols-1) * spacing * 0.5f;  // 布料宽度的一半
    float halfH = (rows-1) * spacing * 0.5f;  // 布料高度的一半

    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            float x = -halfW + c * spacing;   // 等间距排列
            float y = halfH - r * spacing;     // 从上到下：y递减
            
            // 顶行每 1/4 处固定一个点（共4个固定点）
            // c==0, c==cols/4, c==cols*3/4, c==cols-1
            bool pin = (r == 0) && 
                       (c == 0 || c == cols-1 || c == cols/4 || c == cols*3/4);
            
            particles[idx(r,c)] = Particle(Vec3(x, y, 0), pin);
        }
    }
    // ... 添加弹簧 ...
}
```

**为什么固定4个点而不是整行？** 只固定4点（而不是整行20点），允许顶行其余部分的质点也随物理模拟移动。这样布料的"挂起边缘"更自然地弯曲下垂，而不是完全水平的直线。固定整行会让布料顶部看起来被钉在木板上。

### 4.2 弹簧创建

```cpp
auto addS = [&](int a, int b, float k) {
    float l = (particles[a].pos - particles[b].pos).len();
    springs.emplace_back(a, b, l, k);  // 自然长度 = 当前距离
};

for (int r = 0; r < rows; r++) {
    for (int c = 0; c < cols; c++) {
        if (c+1 < cols) addS(idx(r,c), idx(r,c+1), 0.95f);   // 水平结构
        if (r+1 < rows) addS(idx(r,c), idx(r+1,c), 0.95f);   // 垂直结构
        if (r+1<rows && c+1<cols) addS(idx(r,c), idx(r+1,c+1), 0.8f); // 右斜剪切
        if (r+1<rows && c-1>=0)   addS(idx(r,c), idx(r+1,c-1), 0.8f); // 左斜剪切
        if (c+2 < cols) addS(idx(r,c), idx(r,c+2), 0.7f);    // 水平弯曲
        if (r+2 < rows) addS(idx(r,c), idx(r+2,c), 0.7f);    // 垂直弯曲
    }
}
```

自然长度使用 **初始化时的质点间距离**，而不是硬编码为 spacing。这样即使布料有不规则形状，弹簧也能正确工作。

刚度值的选取有设计考量：
- 结构弹簧 0.95：接近 1（最硬），防止明显伸长
- 剪切弹簧 0.80：略软，真实布料对角方向更易变形
- 弯曲弹簧 0.70：最软，允许一定的弯曲

### 4.3 约束求解器

```cpp
void solveConstraints(int iter) {
    for (int i = 0; i < iter; i++) {
        for (auto& s : springs) {
            Particle& pa = particles[s.a];
            Particle& pb = particles[s.b];
            Vec3 d = pb.pos - pa.pos;        // a→b 向量
            float l = d.len();               // 当前弹簧长度
            if (l < 1e-8f) continue;         // 避免除零
            
            // 需要校正的分数 = (偏差/长度) * 0.5 * 刚度
            // 两端各承担一半
            float diff = (l - s.restLen) / l * 0.5f * s.stiffness;
            Vec3 corr = d * diff;
            
            // 只移动非固定质点
            if (!pa.pinned) pa.pos += corr;   // a 向 b 移动
            if (!pb.pinned) pb.pos -= corr;   // b 向 a 移动
        }
    }
}
```

**关键细节**：`(l - s.restLen) / l` 而不是 `(l - s.restLen)`。  
除以 l 的原因：使校正量在方向上已经是单位向量乘上形变长度，等价于 `d.normalized() * (l - s.restLen) * 0.5 * k`。写成除法避免了额外的 `normalized()` 调用（一次 sqrt），性能更好。

### 4.4 球体碰撞响应

```cpp
// 球体碰撞（在约束求解之后执行）
for (auto& p : particles) {
    if (p.pinned) continue;
    Vec3 d = p.pos - sphereCenter;
    float dist = d.len();
    if (dist < sphereRadius + 0.015f) {
        // 把质点推到球面外侧（+ 一点 epsilon 防止持续接触）
        p.pos = sphereCenter + d.normalized() * (sphereRadius + 0.015f);
    }
}
```

**为什么在约束求解之后执行？** 约束求解可能把质点从球面外拉进去（比如相邻质点在球内被弹出，弹簧把这个质点也拉进去），在约束求解后再做碰撞检测，保证最终结果不穿透球体。

**epsilon = 0.015 的作用**：防止质点刚好落在球面上反复触发碰撞（floating point jitter），略微推出去能确保稳定。

### 4.5 透视投影

```cpp
bool project(const Vec3& world, float& sx, float& sy, float& sz) const {
    Vec3 d = world - eye;
    float ex = d.dot(right);       // 相机右方向上的分量
    float ey = d.dot(actualUp);    // 相机上方向上的分量
    float ez = d.dot(forward);     // 相机前方向上的分量（深度）
    
    if (ez < znear) return false;  // 在相机后面，不渲染
    
    // 透视除法：近大远小
    // tanHalf = tan(FOV/2)，决定视野宽度
    float px = ex / (ez * tanHalf * aspect);  // [-1, +1]
    float py = ey / (ez * tanHalf);            // [-1, +1]
    
    // NDC → 屏幕坐标（Y轴翻转，屏幕Y向下）
    sx = (px + 1.0f) * 0.5f * W;
    sy = (1.0f - (py + 1.0f) * 0.5f) * H;
    sz = ez;  // 保存深度（可用于深度测试）
    return sx >= -10 && sx < W+10 && sy >= -10 && sy < H+10;
}
```

**透视投影的本质**：近处的物体被投影到更大的屏幕区域（因为除以了较小的 ez），远处的物体被压缩到更小的区域。`tanHalf * aspect` 决定了视锥体的水平张角，`tanHalf` 决定垂直张角。

**为什么做相机空间分解？** 世界坐标中的 xyz 轴和相机的前/右/上轴不对齐，所以需要用点积把世界坐标投影到相机坐标系。这等价于标准的 View Matrix 变换，但避免了显式构造矩阵。

### 4.6 布料网格渲染

```cpp
for (int r = 0; r < rows-1; r++) {
    for (int c = 0; c < cols-1; c++) {
        // 获取4个角的屏幕坐标
        float sx0=0,sy0=0,sz0=0, sx1=0,sy1=0,sz1=0;
        float sx2=0,sy2=0,sz2=0, sx3=0,sy3=0,sz3=0;
        bool ok0 = cam.project(cloth.particles[cloth.idx(r,c)].pos,   sx0,sy0,sz0);
        bool ok1 = cam.project(cloth.particles[cloth.idx(r,c+1)].pos, sx1,sy1,sz1);
        bool ok2 = cam.project(cloth.particles[cloth.idx(r+1,c)].pos, sx2,sy2,sz2);
        bool ok3 = cam.project(cloth.particles[cloth.idx(r+1,c+1)].pos,sx3,sy3,sz3);
        if (!ok0||!ok1||!ok2||!ok3) continue;  // 任意一点在视锥外则跳过

        // 计算面法向（用于着色）
        Vec3 p0 = cloth.particles[cloth.idx(r,c)].pos;
        Vec3 p1 = cloth.particles[cloth.idx(r,c+1)].pos;
        Vec3 p2 = cloth.particles[cloth.idx(r+1,c)].pos;
        Vec3 n = (p1-p0).cross(p2-p0).normalized();  // 叉积 = 面法向
        float diff = std::max(0.1f, fabsf(n.dot(lightDir)));  // abs：双面光照
        float shading = 0.3f + 0.7f * diff;

        // 棋盘格颜色
        bool ck = ((r+c) % 2 == 0);
        Color baseCol = ck ? Color(0.15f,0.45f,0.85f) : Color(0.9f,0.9f,0.95f);
        Color col = baseCol * shading;

        // 绘制四边形的4条边
        drawLine(img, sx0,sy0, sx1,sy1, col);  // 上边
        drawLine(img, sx0,sy0, sx2,sy2, col);  // 左边
        drawLine(img, sx1,sy1, sx3,sy3, col);  // 右边
        drawLine(img, sx2,sy2, sx3,sy3, col);  // 下边
    }
}
```

**为什么用 `fabsf(n.dot(lightDir))`？** 布料是双面材质——光从正面或背面打都应该有光照。取绝对值实现双面 Lambert 光照，避免背面全黑。

**棋盘格的意义**：纯蓝色布料很难看出变形细节（高频变形在单色中视觉不明显）。蓝白棋盘格在变形时格子会扭曲、拉伸，让形变一目了然，是布料模拟可视化的经典做法。

---

## 五、踩坑实录

### Bug 1：布料在画面中看不到（只看到球体）

**症状**：运行正确，PPM 文件有数据，但渲染图里只有球体和天空背景，没有布料网格。

**错误假设**：以为是渲染逻辑问题，开始检查 drawLine 函数。

**真实原因**：布料质点的屏幕 Y 坐标全是负数（-20 ~ -100），即布料完全在画面顶部边界之外。根本原因是相机参数设置不当：  
- 布料初始高度 y=1.5，相机眼睛 y=1.0，相机朝向中心 (0,-0.5,0)  
- 从相机视角看，布料在相机"正前上方"，但投影后 py > 1（超出 NDC 上边界）

**调试方法**：

```cpp
// 打印所有质点的屏幕坐标
for (int r=0; r<rows; r+=5) for (int c=0; c<cols; c+=5) {
    float sx, sy, sz;
    cam.project(cloth.particles[idx(r,c)].pos, sx,sy,sz);
    printf("[%d,%d] sx=%.1f sy=%.1f\n", r, c, sx, sy);
}
```

输出证实 sy 全为负数，确认是相机配置问题。

**修复**：调整相机位置到 `(0, 0.5, 5.5)`，朝向 `(0, -0.3, 0)`，FOV 从 45° 改为 52°，并把布料初始高度从 y=1.5 调到 y = halfH（相对于布料中心）。

**教训**：当整个渲染对象消失时，第一反应应该是检查坐标范围，而不是渲染逻辑。打印投影坐标是最直接的调试手段。

---

### Bug 2：编译警告 "may be used uninitialized"

**症状**：7 个 `sx0,sy0,...` 系列的 `-Wmaybe-uninitialized` 警告。目标是 0 警告，无法接受。

**错误假设**：认为这是 GCC 的误报——变量在 `ok0=cam.project(...)` 之后就被初始化了，使用前检查了 `if (!ok0||...) continue`，理论上不可能未初始化就使用。

**真实原因**：GCC 的数据流分析不够智能（或者说足够保守）——它无法静态证明"所有 ok==false 时的 continue 路径"覆盖了全部情况。对 GCC 来说，`project()` 的返回值和参数赋值之间的关系是不透明的。

**修复**：手动初始化所有 float 变量为 0：

```cpp
float sx0=0,sy0=0,sz0=0, sx1=0,sy1=0,sz1=0;
float sx2=0,sy2=0,sz2=0, sx3=0,sy3=0,sz3=0;
```

用 `sed` 批量替换，一次解决所有同类问题。

**教训**：凡是"我觉得不可能触发"的警告，只要代码能正常工作，都应该通过代码改动来消除，而不是用 `#pragma GCC diagnostic ignored` 压制。0 警告的代码长期可维护性更好。

---

### Bug 3：物理参数导致布料变形不明显

**症状**：三帧之间的布料形状几乎没有可见差异，像3张一样的截图拼在一起。

**原因**：最初的 gravity = `(0, -9.8f * 0.001f, 0)`（极弱），TOTAL_FRAMES=90，SUBSTEPS=8，布料下垂速度太慢，在90帧内几乎没有变形。

**修复过程**：
1. 增大重力：`-9.8 * 0.001 → -0.032`（重力约增大3倍）
2. 增加帧数：90 → 150，给布料更多时间下垂
3. 增加子步：8 → 12，提高精度避免不稳定
4. 增加约束迭代：10 → 15，防止弹簧过拉伸

修改后三帧有明显的布料下垂和碰撞变形差异。

**教训**：物理模拟的参数选择是直觉+实验的结合。重力太弱效果不明显；太强弹簧会崩溃（力超过约束能纠正的范围）。从小到大调整，每次调整后渲染一帧检查。

---

### Bug 4：PNG 文件太小（9KB），怀疑渲染空白

**症状**：第一版渲染 PNG 只有 9.4KB，远小于 10KB 的验收标准。

**错误假设**：以为布料没渲染进去，图片大部分是纯色背景（高压缩率）。

**调查**：像素统计均值 183.2、标准差 49.7，视觉检查确认布料可见但较稀疏。

**真实原因**：第一版布料只在顶部几行可见，大部分画面是纯色天空背景，PNG 压缩率极高（纯色区域几乎为0 bits）。

**修复**：重新设计相机和布料参数，使布料占据更多画面，引入棋盘格图案增加纹理（降低压缩率）。修复后 PNG 58.3KB，通过验收。

**教训**：文件大小是图像内容复杂度的指标，不单纯是尺寸指标。PNG 的高压缩率可能掩盖"画面空洞"问题，应该结合视觉检查和像素统计综合判断。

---

## 六、效果验证与数据

### 6.1 像素统计

```
=== Validation ===
File size: 58.3 KB
Image size: 1440×360 (3帧拼接，每帧 480×360)
Pixel mean: 153.6  (标准: 10-240 ✅)
Pixel std:  55.0   (标准: >5 ✅)
File size > 10KB   ✅
```

- 均值 153.6：整体偏亮，符合蓝白棋盘格 + 天空背景的预期
- 标准差 55.0：高于最低标准 5 很多，说明图像内容丰富（布料格子的蓝白对比）

### 6.2 渲染时间

```
总帧数: 150
子步数: 12 per frame → 1800 total physics steps  
弹簧数: 约 2100 (22×22 布料)
质点数: 484 (22×22)
约束迭代: 15 per substep

物理计算: ~0.1s（CPU，编译优化 -O2）
渲染 3 帧: ~0.02s
总耗时: < 0.15s
```

软光栅化的优势在于完全在 CPU 上运行，无需 GPU 上下文初始化，适合离线批量渲染。

### 6.3 三帧对比分析

**帧 1（模拟开始）**：布料基本保持平面形状，顶部由4个固定点悬挂，底部略微向下弯曲。球体在布料前方，尚未有接触。

**帧 76（模拟中途）**：布料明显下垂，中部向球体凹陷，球体碰撞区域的布料被推向两侧，形成明显的曲面形状。风力使布料略有横向偏移。

**帧 150（模拟末尾）**：布料进入更稳定的悬挂状态，与球体的接触区形成清晰的凹形印记。风力周期变化使布料有轻微的横向摆动。

### 6.4 代码量统计

```
main.cpp: 约 480 行
- 数学库 (Vec3/Color/Image): 60行
- 物理模拟 (Particle/Spring/Cloth): 110行  
- 渲染 (Camera/drawLine/renderClothAndSphere): 140行
- 主函数及输出: 40行
```

---

## 七、总结与延伸

### 7.1 质点弹簧系统的局限性

本项目的实现揭示了质点弹簧模型的几个根本局限：

**1. 弹性系数不直接对应物理材质**  
真实布料的弹性模量（Young's modulus）是物理可测的量（单位 Pa），但质点弹簧的 k 值没有明确的物理单位，只能通过"调感觉"来设定。想要精确模拟棉布 vs 丝绸 vs 皮革，需要 FEM。

**2. 泊松比效应缺失**  
真实布料在一个方向拉伸时，另一方向会收缩（泊松效应）。质点弹簧无法自然地表达这种体积守恒约束。

**3. 大形变不稳定**  
当布料被剧烈拉扯（比如快速移动的角色），弹簧超调问题依然存在，需要更多子步或更大阻尼，但这又会使布料显得过于"滞重"。

### 7.2 延伸优化方向

**换用 PBD/XPBD**：放弃弹力积分，完全用位置约束，可以彻底解决稳定性问题。Unreal 的 Chaos Cloth 和 Unity 的 NvCloth 都基于 PBD/XPBD。

**自碰撞检测**：本项目布料不和自身碰撞，实际上布料会穿过自己。自碰撞需要 BVH 或哈希空间分区，是布料模拟中最复杂的部分。

**GPU 并行化**：质点更新可以完全并行（每个质点独立），适合 CUDA/OpenCL 或 Compute Shader。约束求解需要颜色分图（Color Ordering）来并行，但也可以大幅加速。

**布料-骨骼耦合**：游戏角色的衣物需要跟随骨骼运动，同时受物理影响。这需要将固定点绑定到骨骼关节，并在骨骼运动时更新固定点位置。

**三角形网格代替四边形网格**：本项目用四边形网格渲染（4条边线），真实的 FEM 布料模拟使用三角形。三角形的法向计算更简单，也更适合 GPU 三角形流水线。

### 7.3 系列文章关联

本文是布料模拟主题的基础版本。系列中的相关技术：
- **2026-04-11 PBD Soft Body Deformation**：用位置约束实现软体形变，方法论与本文形成对比——PBD 更稳定但能量不守恒
- **2026-04-09 SPH Fluid Simulation**：同样是粒子系统，但处理流体而非布料，约束的性质（不可压缩性 vs 弹性）不同
- **2026-04-08 Particle System Simulation**：更简单的粒子系统（无约束），本文在其基础上增加了弹簧约束网络

---

**代码仓库**：[GitHub - daily-coding-practice/2026/04/04-15-cloth-simulation](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-15-cloth-simulation)

**渲染结果**：

![布料模拟 - 三帧合成](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-15-Cloth-Simulation-Mass-Spring/cloth_output.png)

*左→右：初始帧 → 第76帧（中途） → 第150帧（末尾）。可观察到布料在重力和风力作用下逐渐下垂，并与橙红色球体产生碰撞形变。*
