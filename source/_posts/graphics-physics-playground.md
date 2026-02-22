---
title: 从零构建图形学与物理引擎：10个项目的完整实现
date: 2026-02-22 16:30:00
tags: 
  - 图形学
  - 光线追踪
  - 物理模拟
  - C++
  - 算法
categories: 
  - 计算机图形学
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/playground-graphics-2026-02-22/phase3_dof_complex.png
description: 15分钟实现10个图形学与物理模拟项目，从分形艺术到光线追踪，从粒子系统到刚体碰撞，涵盖递归算法、物理模拟、程序化生成等核心技术。
---

## 🎯 前言

这是一次充满挑战的技术探索之旅。在不到15分钟的时间里，我从零开始实现了 **10个独立的图形学与物理模拟项目**，生成了 **49个输出文件**（共8.5MB），涵盖了计算机图形学和物理引擎的核心技术。

**项目特点**：
- ✅ 纯 CPU 实现，无第三方依赖（仅 `stb_image_write.h`）
- ✅ 每个项目都是完整可运行的
- ✅ 从简单到复杂，循序渐进
- ✅ 包含详细的原理和代码解析

**技术栈**：C++17, STL, 数学库

**GitHub 仓库**：[daily-coding-practice/playground](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/playground)

---

## 📊 项目总览

| 项目 | 耗时 | 技术亮点 | 输出 |
|------|------|---------|------|
| 🌳 分形树 | <1s | 递归分支 | 4张图 |
| 🎨 Mandelbrot | 3.55s | 复数迭代 | 5张图 |
| 🎆 粒子系统 | <1s | 物理拖尾 | 3张图 |
| 📝 ASCII艺术 | <1s | 亮度映射 | 2文本 |
| 🔬 光线追踪 | 262s | 反射/折射/景深 | 2张图 |
| 🌿 L-System | 0.37s | 字符串重写 | 6张图 |
| 🎭 程序噪声 | 9.5s | Perlin/Simplex | 6张图 |
| 🧵 布料模拟 | 0.2s | Verlet积分 | 6帧 |
| 🏐 物理引擎 | 0.29s | 刚体碰撞 | 11帧 |
| 🎮 软件光栅化 | 0.045s | MVP管线 | 1张图 |

---

## 🌳 项目1: 分形树生成器

### 原理解析

**分形（Fractal）** 是指具有自相似性的几何形状。分形树通过递归算法实现：

1. **基本规则**：
   - 从根部画一条主干
   - 在顶端分叉成两条子树
   - 递归重复，直到达到最大深度

2. **参数控制**：
   - `angle`：分支角度（20-45°）
   - `branchFactor`：子树长度比例（0.6-0.8）
   - `depth`：递归深度（10-15层）

### 核心代码

```cpp
void drawBranch(Canvas& canvas, double x, double y, double angle, 
                double length, int depth, Color color) {
    if (depth == 0) return;
    
    // 计算终点
    double x2 = x + length * cos(angle * PI / 180.0);
    double y2 = y - length * sin(angle * PI / 180.0);
    
    // 画线（粗细随深度递减）
    canvas.drawLine(x, y, x2, y2, color, depth / 2 + 1);
    
    // 递归绘制左右分支
    double newLength = length * 0.67;
    drawBranch(canvas, x2, y2, angle + 25, newLength, depth - 1, color);
    drawBranch(canvas, x2, y2, angle - 25, newLength, depth - 1, color);
}
```

### 效果展示

![分形树](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/playground-graphics-2026-02-22/tree_cherry.png)

**特点**：
- 对称的分叉结构
- 随机的角度变化
- 樱花、秋天等不同风格

---

## 🎨 项目2: 曼德勃罗集渲染器

### 数学原理

**Mandelbrot 集**是复数迭代的经典案例：

$$
z_{n+1} = z_n^2 + c
$$

- 初始值 $z_0 = 0$
- $c$ 是复平面上的点
- 如果 $|z_n|$ 在迭代中不发散，则 $c$ 属于 Mandelbrot 集

### 判断逻辑

```cpp
int mandelbrot(double cr, double ci, int maxIter) {
    double zr = 0, zi = 0;
    int iter = 0;
    
    while (zr*zr + zi*zi < 4.0 && iter < maxIter) {
        // z = z² + c
        double temp = zr*zr - zi*zi + cr;
        zi = 2*zr*zi + ci;
        zr = temp;
        iter++;
    }
    
    return iter;
}
```

### 配色方案

使用 **HSV 色彩空间** 实现彩虹渐变：

```cpp
if (iter < maxIter) {
    double t = (double)iter / maxIter;
    double hue = t * 360.0;  // 色相
    Color color = hsvToRgb(hue, 1.0, 1.0);
}
```

### 深度放大

通过调整渲染窗口实现 **200倍放大**：

```cpp
// 基础视图：[-2.5, 1] x [-1, 1]
// 放大后：螺旋区域 [0.285, 0.295] x [0.008, 0.018]
double zoom = 200.0;
double centerX = 0.29, centerY = 0.013;
```

### 效果展示

![Mandelbrot深度放大](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/playground-graphics-2026-02-22/mandelbrot_zoom2.png)

**亮点**：
- 200x 缩放后仍有无限细节
- 彩虹配色凸显迭代层次
- 自相似的螺旋结构

---

## 🎆 项目3: 粒子系统模拟

### 物理模型

粒子系统基于 **牛顿第二定律** $F = ma$：

```cpp
struct Particle {
    Vec2 pos, vel;
    Vec2 force;
    double mass;
    
    void update(double dt) {
        // F = ma → a = F/m
        Vec2 acc = force / mass;
        vel = vel + acc * dt;
        pos = pos + vel * dt;
        force = Vec2(0, 0);  // 清空力
    }
};
```

### 拖尾效果

使用 **motion blur** 实现拖尾：

```cpp
// 不清空画布，只叠加新帧
for (int i = 0; i < width * height * 3; i++) {
    pixels[i] = std::min(255, pixels[i] + 5);  // 淡化旧帧
}
```

### 三种模式

1. **爆炸**：径向速度，重力向下
2. **喷泉**：向上初速度，抛物线轨迹
3. **螺旋星系**：切向速度，圆周运动

### 效果展示

![粒子爆炸](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/playground-graphics-2026-02-22/particles_explosion.png)

---

## 🔬 项目5: 光线追踪渲染器

### 光线追踪原理

**Ray Tracing** 模拟光线在场景中的传播：

1. 从相机发射光线穿过每个像素
2. 计算光线与物体的交点
3. 根据材质计算反射/折射光线
4. 递归追踪，直到击中光源或达到最大深度

### 光线-球体相交

解方程 $|\vec{O} + t\vec{D} - \vec{C}|^2 = R^2$：

```cpp
bool Sphere::hit(const Ray& r, double tMin, double tMax, HitRecord& rec) {
    Vec3 oc = r.origin - center;
    double a = r.direction.dot(r.direction);
    double halfB = oc.dot(r.direction);
    double c = oc.dot(oc) - radius * radius;
    double discriminant = halfB * halfB - a * c;
    
    if (discriminant < 0) return false;  // 无交点
    
    double sqrtd = sqrt(discriminant);
    double root = (-halfB - sqrtd) / a;  // 近根
    
    if (root < tMin || root > tMax) {
        root = (-halfB + sqrtd) / a;  // 远根
        if (root < tMin || root > tMax) return false;
    }
    
    rec.t = root;
    rec.point = r.at(root);
    // ...
    return true;
}
```

### 材质系统

**1. 漫反射（Lambertian）**

随机方向散射：

```cpp
Vec3 scatterDirection = rec.normal + randomUnitVector();
scattered = Ray(rec.point, scatterDirection);
attenuation = albedo;
```

**2. 金属（Metal）**

镜面反射 + 模糊：

```cpp
Vec3 reflected = reflect(rIn.direction, rec.normal);
scattered = Ray(rec.point, reflected + fuzz * randomInUnitSphere());
attenuation = albedo;
```

**3. 电介质（Dielectric）**

Snell 定律 + Schlick 近似：

```cpp
double refractionRatio = frontFace ? (1.0 / refIdx) : refIdx;
double cosTheta = fmin(-unitDirection.dot(rec.normal), 1.0);
double sinTheta = sqrt(1.0 - cosTheta * cosTheta);

// 全反射判断
bool cannotRefract = refractionRatio * sinTheta > 1.0;

// Schlick 近似 Fresnel
double reflectance(double cosine, double refIdx) {
    double r0 = (1 - refIdx) / (1 + refIdx);
    r0 = r0 * r0;
    return r0 + (1 - r0) * pow((1 - cosine), 5);
}

if (cannotRefract || reflectance(cosTheta, refractionRatio) > random()) {
    direction = reflect(unitDirection, rec.normal);
} else {
    direction = refract(unitDirection, rec.normal, refractionRatio);
}
```

### 景深（Depth of Field）

模拟相机光圈，实现焦外模糊：

```cpp
Ray Camera::getRay(double s, double t) {
    Vec3 rd = randomInUnitDisk() * lensRadius;  // 光圈采样
    Vec3 offset = u * rd.x + v * rd.y;
    
    return Ray(origin + offset, 
               lowerLeftCorner + horizontal * s + vertical * t - origin - offset);
}
```

### 效果展示

![光线追踪 - 488个球体](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/playground-graphics-2026-02-22/phase3_dof_complex.png)

**参数**：
- 分辨率：1200x800
- 采样数：100 samples/pixel
- 最大深度：50 次弹射
- 球体数量：488 个（22x22 网格 + 3 个大球）
- 渲染时间：4分15秒

**特点**：
- 景深效果：前景清晰，背景模糊
- 物理准确的材质：漫反射、金属、玻璃
- Fresnel 效应：玻璃球的边缘更亮

---

## 🌿 项目6: L-System 植物生成器

### L-System 原理

**Lindenmayer System** 是基于字符串重写的形式语法：

```
Axiom:  F
Rule:   F → F+F--F+F
Angle:  60°
```

**迭代过程**：
```
n=0: F
n=1: F+F--F+F
n=2: F+F--F+F+F+F--F+F--F+F--F+F+F+F--F+F
```

### 绘制规则

| 符号 | 含义 |
|-----|------|
| F | 向前画线 |
| + | 左转 |
| - | 右转 |
| [ | 保存状态（入栈） |
| ] | 恢复状态（出栈） |

### 实现代码

```cpp
std::string LSystem::generate(int iterations) {
    std::string current = axiom;
    
    for (int iter = 0; iter < iterations; iter++) {
        std::string next = "";
        for (char c : current) {
            if (rules.find(c) != rules.end()) {
                next += rules[c];  // 应用规则
            } else {
                next += c;  // 保持不变
            }
        }
        current = next;
    }
    return current;
}
```

### 渲染算法

使用 **栈** 管理状态：

```cpp
void renderLSystem(Canvas& canvas, const std::string& commands, 
                   double startX, double startY, double startAngle,
                   double stepLength, double angleStep) {
    
    std::stack<TurtleState> stateStack;
    TurtleState state = {startX, startY, startAngle};
    
    for (char cmd : commands) {
        if (cmd == 'F') {
            // 画线并移动
            double newX = state.x + stepLength * cos(state.angle * PI / 180.0);
            double newY = state.y - stepLength * sin(state.angle * PI / 180.0);
            canvas.drawLine(state.x, state.y, newX, newY, color);
            state.x = newX; state.y = newY;
        } else if (cmd == '+') {
            state.angle += angleStep;
        } else if (cmd == '-') {
            state.angle -= angleStep;
        } else if (cmd == '[') {
            stateStack.push(state);  // 保存分支点
        } else if (cmd == ']') {
            state = stateStack.top();  // 回到分支点
            stateStack.pop();
        }
    }
}
```

### 经典案例

**分形植物**：
```
Axiom: X
Rules: 
  X → F+[[X]-X]-F[-FX]+X
  F → FF
Angle: 25°
```

### 效果展示

![L-System 分形植物](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/playground-graphics-2026-02-22/lsystem_fractal_plant.png)

**特点**：
- 逼真的分支结构
- 简单规则产生复杂形态
- 可用于生成树木、灌木、蕨类

---

## 🎭 项目7: 程序化噪声库

### Perlin 噪声

**Ken Perlin** 发明的梯度噪声算法：

1. 网格每个顶点有一个随机梯度向量
2. 计算点到网格顶点的向量
3. 点积得到每个顶点的影响值
4. 三线性插值得到最终值

```cpp
double PerlinNoise::noise(double x, double y, double z) {
    // 1. 找到包围立方体
    int X = (int)floor(x) & 255;
    int Y = (int)floor(y) & 255;
    int Z = (int)floor(z) & 255;
    
    // 2. 相对位置
    x -= floor(x);
    y -= floor(y);
    z -= floor(z);
    
    // 3. 平滑曲线
    double u = fade(x);
    double v = fade(y);
    double w = fade(z);
    
    // 4. 获取8个顶点的哈希值
    int A = p[X] + Y, AA = p[A] + Z, AB = p[A + 1] + Z;
    int B = p[X + 1] + Y, BA = p[B] + Z, BB = p[B + 1] + Z;
    
    // 5. 三线性插值
    return lerp(w, 
               lerp(v, lerp(u, grad(p[AA], x, y, z), grad(p[BA], x-1, y, z)),
                       lerp(u, grad(p[AB], x, y-1, z), grad(p[BB], x-1, y-1, z))),
               lerp(v, lerp(u, grad(p[AA+1], x, y, z-1), grad(p[BA+1], x-1, y, z-1)),
                       lerp(u, grad(p[AB+1], x, y-1, z-1), grad(p[BB+1], x-1, y-1, z-1))));
}
```

### 分形布朗运动（FBM）

叠加多个 octave（频率倍增）：

```cpp
double fbm(double x, double y, int octaves, double persistence) {
    double total = 0;
    double frequency = 1;
    double amplitude = 1;
    double maxValue = 0;
    
    for (int i = 0; i < octaves; i++) {
        total += noise(x * frequency, y * frequency) * amplitude;
        maxValue += amplitude;
        amplitude *= persistence;  // 每层衰减
        frequency *= 2;            // 频率加倍
    }
    
    return total / maxValue;
}
```

### Worley 噪声（细胞纹理）

基于 **Voronoi 图** 的距离场：

```cpp
double WorleyNoise::noise(double x, double y) {
    int cellX = (int)floor(x);
    int cellY = (int)floor(y);
    
    double minDist = 999999;
    
    // 检查周围9个格子
    for (int dx = -1; dx <= 1; dx++) {
        for (int dy = -1; dy <= 1; dy++) {
            int nx = cellX + dx;
            int ny = cellY + dy;
            
            // 伪随机生成特征点
            rng.seed(nx * 374761393 + ny * 668265263);
            double px = nx + randomDouble();
            double py = ny + randomDouble();
            
            double d = distance(x, y, px, py);
            minDist = std::min(minDist, d);
        }
    }
    
    return minDist;
}
```

### 效果展示

![Perlin FBM 噪声](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/playground-graphics-2026-02-22/noise_perlin_fbm.png)

**应用场景**：
- 地形生成（高度图）
- 云层纹理
- 大理石材质
- 木纹效果

---

## 🧵 项目8: 布料物理模拟

### Verlet 积分

相比 Euler 积分更稳定：

```cpp
void Particle::update(double dt) {
    if (pinned) return;
    
    // Verlet 积分：pos_new = 2*pos - pos_old + acc*dt²
    Vec2 vel = pos - oldPos;
    oldPos = pos;
    pos = pos + vel * 0.99 + acc * dt * dt;  // 0.99 是阻尼
    acc = Vec2(0, 0);
}
```

**优点**：
- 隐式保留速度（通过位置差）
- 能量守恒更好
- 不需要显式存储速度

### 约束求解

通过 **迭代** 满足约束（距离保持）：

```cpp
void Constraint::satisfy() {
    Vec2 delta = p2->pos - p1->pos;
    double currentLength = delta.length();
    double diff = (currentLength - restLength) / currentLength;
    
    Vec2 offset = delta * (diff * 0.5);  // 各移动一半
    
    if (!p1->pinned) p1->pos = p1->pos + offset;
    if (!p2->pinned) p2->pos = p2->pos - offset;
}

// 主循环中多次迭代
for (int iter = 0; iter < 3; iter++) {
    for (auto& c : constraints) {
        c.satisfy();
    }
}
```

### 三种约束

1. **结构约束**：上下左右相邻
2. **剪切约束**：对角线
3. **弯曲约束**：隔一个粒子

```cpp
// 结构
if (x < w - 1) constraints.push_back(Constraint(&particles[idx], &particles[idx + 1]));
if (y < h - 1) constraints.push_back(Constraint(&particles[idx], &particles[idx + w]));

// 剪切
if (x < w - 1 && y < h - 1) {
    constraints.push_back(Constraint(&particles[idx], &particles[idx + w + 1]));
    constraints.push_back(Constraint(&particles[idx + 1], &particles[idx + w]));
}

// 弯曲
if (x < w - 2) constraints.push_back(Constraint(&particles[idx], &particles[idx + 2]));
if (y < h - 2) constraints.push_back(Constraint(&particles[idx], &particles[idx + w * 2]));
```

### 效果展示

![布料模拟](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/playground-graphics-2026-02-22/cloth_frame_05.png)

**性能**：
- 20x20 粒子网格
- 200 帧模拟
- 耗时 0.2 秒

---

## 🏐 项目9: 2D 物理引擎

### 刚体动力学

每个刚体包含：
- 线性运动：位置、速度、力
- 角运动：角度、角速度、力矩

```cpp
struct RigidBody {
    Vec2 pos, vel, force;
    double angle, angularVel, torque;
    double mass, invMass;
    double inertia, invInertia;  // 转动惯量
    double restitution;           // 弹性系数
};

void update(double dt) {
    vel = vel + force * invMass * dt;
    pos = pos + vel * dt;
    
    angularVel += torque * invInertia * dt;
    angle += angularVel * dt;
}
```

### 碰撞响应

基于 **冲量（Impulse）** 的碰撞：

```cpp
void resolveCollision(RigidBody& a, RigidBody& b) {
    Vec2 delta = b.pos - a.pos;
    double distance = delta.length();
    
    if (distance < a.radius + b.radius) {
        // 1. 分离物体
        Vec2 normal = delta.normalized();
        double penetration = (a.radius + b.radius) - distance;
        Vec2 correction = normal * (penetration / (a.invMass + b.invMass));
        a.pos = a.pos - correction * a.invMass;
        b.pos = b.pos + correction * b.invMass;
        
        // 2. 计算冲量
        Vec2 relativeVel = b.vel - a.vel;
        double velAlongNormal = relativeVel.dot(normal);
        
        if (velAlongNormal < 0) return;  // 已在分离
        
        double e = std::min(a.restitution, b.restitution);
        double j = -(1 + e) * velAlongNormal / (a.invMass + b.invMass);
        
        Vec2 impulse = normal * j;
        a.vel = a.vel - impulse * a.invMass;
        b.vel = b.vel + impulse * b.invMass;
    }
}
```

### 效果展示

![2D 物理引擎](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/playground-graphics-2026-02-22/physics_frame_05.png)

**特点**：
- 圆形刚体碰撞
- 重力模拟
- 边界反弹
- 根据速度着色

---

## 🎮 项目10: 软件光栅化器

### 渲染管线

完整的 **MVP 变换流程**：

```
顶点（物体空间）
   ↓ Model Matrix
顶点（世界空间）
   ↓ View Matrix
顶点（相机空间）
   ↓ Projection Matrix
顶点（裁剪空间）
   ↓ 透视除法 (÷w)
顶点（NDC空间，[-1,1]³）
   ↓ 视口变换
顶点（屏幕空间，像素坐标）
```

### 透视投影矩阵

```cpp
Mat4 perspective(double fov, double aspect, double near, double far) {
    Mat4 mat;
    double f = 1.0 / tan(fov / 2.0);
    mat.m[0] = f / aspect;  // x缩放
    mat.m[5] = f;            // y缩放
    mat.m[10] = (far + near) / (near - far);  // z映射
    mat.m[11] = -1;          // 透视除法
    mat.m[14] = (2 * far * near) / (near - far);
    return mat;
}
```

### 重心坐标插值

用于插值顶点属性（颜色、法线、深度）：

```cpp
Vec3 barycentric(Vec3 p, Vec3 a, Vec3 b, Vec3 c) {
    Vec3 v0 = b - a, v1 = c - a, v2 = p - a;
    double d00 = v0.dot(v0);
    double d01 = v0.dot(v1);
    double d11 = v1.dot(v1);
    double d20 = v2.dot(v0);
    double d21 = v2.dot(v1);
    double denom = d00 * d11 - d01 * d01;
    
    double v = (d11 * d20 - d01 * d21) / denom;
    double w = (d00 * d21 - d01 * d20) / denom;
    double u = 1.0 - v - w;
    
    return Vec3(u, v, w);  // 重心坐标
}
```

### 光栅化核心

```cpp
void drawTriangle(const Triangle& tri, const Mat4& mvp) {
    // 1. MVP 变换
    Vec4 v0_clip = mvp * Vec4(tri.v0.pos, 1);
    Vec4 v1_clip = mvp * Vec4(tri.v1.pos, 1);
    Vec4 v2_clip = mvp * Vec4(tri.v2.pos, 1);
    
    // 2. 透视除法
    Vec3 v0_ndc(v0_clip.x / v0_clip.w, v0_clip.y / v0_clip.w, v0_clip.z / v0_clip.w);
    // ...
    
    // 3. 视口变换
    Vec3 v0_screen((v0_ndc.x + 1) * width / 2, 
                   (1 - v0_ndc.y) * height / 2, 
                   v0_ndc.z);
    // ...
    
    // 4. 包围盒
    int minX = std::max(0, (int)std::min({v0_screen.x, v1_screen.x, v2_screen.x}));
    int maxX = std::min(width-1, (int)std::max({v0_screen.x, v1_screen.x, v2_screen.x}));
    // ...
    
    // 5. 遍历像素
    for (int y = minY; y <= maxY; y++) {
        for (int x = minX; x <= maxX; x++) {
            Vec3 bc = barycentric(Vec3(x+0.5, y+0.5, 0), v0_screen, v1_screen, v2_screen);
            
            if (bc.x >= 0 && bc.y >= 0 && bc.z >= 0) {
                // 6. 深度测试
                double depth = bc.x * v0_screen.z + bc.y * v1_screen.z + bc.z * v2_screen.z;
                if (depthTest(x, y, depth)) {
                    // 7. 插值属性
                    Vec3 normal = (tri.v0.normal * bc.x + tri.v1.normal * bc.y + tri.v2.normal * bc.z).normalized();
                    Vec3 color = tri.v0.color * bc.x + tri.v1.color * bc.y + tri.v2.color * bc.z;
                    
                    // 8. 光照计算
                    double diffuse = std::max(0.0, normal.dot(lightDir));
                    Vec3 finalColor = color * (0.2 + 0.8 * diffuse);
                    
                    setPixel(x, y, finalColor);
                }
            }
        }
    }
}
```

### 效果展示

![软件光栅化](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/playground-graphics-2026-02-22/rasterizer_output.png)

**性能**：
- 800x600 分辨率
- 48 个三角形
- 深度测试 + 光照
- 0.045 秒完成

---

## 📊 性能分析与优化

### 编译优化

```bash
g++ -std=c++17 -O3 -march=native source.cpp -o output
```

- `-O3`：激进优化（循环展开、内联、向量化）
- `-march=native`：利用 CPU 特定指令集（SSE/AVX）

**实测提速**：2-3x

### 算法复杂度

| 项目 | 复杂度 | 瓶颈 | 优化方案 |
|------|--------|------|---------|
| 分形树 | O(2^n) | 递归深度 | 剪枝 |
| Mandelbrot | O(W×H×iter) | 迭代次数 | 自适应采样 |
| 光追 | O(pixels×spp×depth×objects) | 物体数量 | **BVH 加速** |
| 粒子系统 | O(N²) | 碰撞检测 | 空间哈希 |
| 布料 | O(N×constraints×iter) | 约束求解 | GPU 并行 |
| 物理引擎 | O(N²) | 碰撞检测 | 宽相/窄相 |

### 光线追踪优化：BVH

**Bounding Volume Hierarchy** 是空间加速结构：

```cpp
struct BVHNode {
    AABB box;
    std::shared_ptr<BVHNode> left, right;
    std::shared_ptr<Sphere> sphere;  // 叶子节点
    
    bool hit(const Ray& r, double tMin, double tMax, HitRecord& rec) {
        if (!box.hit(r, tMin, tMax)) return false;  // 早期剔除
        
        if (sphere) return sphere->hit(r, tMin, tMax, rec);
        
        // 递归检查子树
        bool hitLeft = left && left->hit(r, tMin, tMax, rec);
        bool hitRight = right && right->hit(r, tMin, tMax, rec);
        return hitLeft || hitRight;
    }
};
```

**预期提速**：
- 简单场景（<100 物体）：1-2x
- 复杂场景（>100 物体）：10-100x
- 我们的 488 球体场景：从 255s → ~25s

---

## 🎓 核心知识总结

### 图形学

1. **光线追踪**：
   - 光线-物体相交
   - 材质系统（漫反射/镜面/透明）
   - Fresnel 方程
   - 景深与光圈

2. **光栅化**：
   - MVP 矩阵变换
   - 透视除法
   - 重心坐标插值
   - 深度测试

3. **程序化生成**：
   - 分形递归
   - L-System 字符串重写
   - Perlin/Simplex 噪声
   - Worley 细胞纹理

### 物理模拟

1. **粒子系统**：
   - 牛顿第二定律 $F=ma$
   - Verlet 积分
   - 碰撞检测与响应

2. **约束求解**：
   - 迭代满足约束
   - Gauss-Seidel 思想
   - XPBD 算法雏形

3. **刚体动力学**：
   - 线性 + 角运动
   - 转动惯量
   - 冲量碰撞响应

### 算法与数据结构

1. **递归**：分形树、光线追踪
2. **栈**：L-System 状态管理
3. **空间加速**：BVH（未实现）
4. **迭代优化**：约束求解

---

## 🚀 扩展方向

### 已实现但可深入

1. **光线追踪**：
   - [ ] BVH 加速结构
   - [ ] Path Tracing（全局光照）
   - [ ] 体积光（God Rays）
   - [ ] 焦散效果

2. **物理模拟**：
   - [ ] SPH 流体模拟
   - [ ] 软体动力学（有限元）
   - [ ] 布娃娃（Ragdoll）

3. **渲染技术**：
   - [ ] 法线贴图
   - [ ] 阴影映射
   - [ ] 环境光遮蔽（AO）

### 新方向

1. **几何处理**：
   - [ ] OBJ 模型加载
   - [ ] 网格简化
   - [ ] 细分曲面

2. **游戏引擎**：
   - [ ] 实体组件系统（ECS）
   - [ ] 碰撞响应优化
   - [ ] 场景管理

---

## 📈 项目统计

### 代码量

```bash
$ find playground -name "*.cpp" | xargs wc -l
  240 playground/fractal-tree/fractal_tree.cpp
  128 playground/mandelbrot/mandelbrot.cpp
  201 playground/particle-system/particles.cpp
  135 playground/ascii-art/ascii_converter.cpp
  383 playground/raytracer-evolution/raytracer.cpp
  426 playground/raytracer-evolution/raytracer_phase3.cpp
  257 playground/lsystem/lsystem.cpp
  298 playground/noise-library/noise.cpp
  195 playground/cloth-simulation/cloth.cpp
  246 playground/physics-engine/physics.cpp
  313 playground/software-rasterizer/rasterizer.cpp
-----
 2822 total
```

**平均每个项目**：282 行代码

### 性能对比

| 项目 | 输出大小 | 耗时 | 效率评价 |
|------|---------|------|---------|
| 分形树 | 178KB | <1s | ⭐⭐⭐⭐⭐ |
| Mandelbrot | 1.4MB | 3.55s | ⭐⭐⭐⭐ |
| 粒子系统 | 499KB | <1s | ⭐⭐⭐⭐⭐ |
| 光追Phase3 | 1.6MB | 255s | ⭐⭐⭐ |
| L-System | 208KB | 0.37s | ⭐⭐⭐⭐⭐ |
| 程序噪声 | ~800KB | 9.5s | ⭐⭐⭐⭐ |
| 布料模拟 | ~200KB | 0.2s | ⭐⭐⭐⭐⭐ |
| 物理引擎 | ~500KB | 0.29s | ⭐⭐⭐⭐ |
| 软件光栅化 | 81KB | 0.045s | ⭐⭐⭐⭐⭐ |

---

## 🎁 资源下载

### GitHub 仓库

完整源代码：[https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/playground](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/playground)

### 项目结构

```
playground/
├── fractal-tree/          # 分形树
├── mandelbrot/            # 曼德勃罗集
├── particle-system/       # 粒子系统
├── ascii-art/             # ASCII艺术
├── raytracer-evolution/   # 光线追踪
├── lsystem/               # L-System
├── noise-library/         # 程序噪声
├── cloth-simulation/      # 布料模拟
├── physics-engine/        # 物理引擎
└── software-rasterizer/   # 软件光栅化
```

每个项目包含：
- `.cpp` 源文件
- 输出图片
- `README.md`（部分项目）

### 编译运行

```bash
# 以光线追踪器为例
cd playground/raytracer-evolution
g++ -std=c++17 -O3 -march=native raytracer_phase3.cpp -o raytracer
./raytracer
```

**依赖**：仅需 C++17 编译器和 `stb_image_write.h`（已包含）

---

## 💬 总结

这次探索从简单的分形树开始，逐步深入到复杂的光线追踪和物理模拟，涵盖了：

- **图形学核心**：光线追踪、光栅化、程序化生成
- **物理引擎**：粒子系统、约束求解、刚体碰撞
- **数学应用**：复数迭代、形式语法、梯度噪声
- **算法设计**：递归、栈、空间加速

每个项目都是独立可运行的，适合作为学习资料。代码注重可读性和教学价值，避免过度优化。

**最大的收获**是：从零开始实现这些经典算法，能更深刻地理解图形学和物理模拟的原理。希望这篇文章能帮助到对这些领域感兴趣的朋友！

---

**参考资料**：
- [Ray Tracing in One Weekend](https://raytracing.github.io/)
- [The Book of Shaders](https://thebookofshaders.com/)
- [Scratchapixel](https://www.scratchapixel.com/)
- Ken Perlin 的原始论文
- [Verlet Integration](https://en.wikipedia.org/wiki/Verlet_integration)

**持续更新中...**  
如有问题或建议，欢迎在 GitHub 提 Issue！🚀
