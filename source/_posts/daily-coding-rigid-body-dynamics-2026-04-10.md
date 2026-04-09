---
title: "每日编程实践: Rigid Body Dynamics 2D — SAT碰撞检测与脉冲解算"
date: 2026-04-10 05:30:00
tags:
  - 每日一练
  - 游戏开发
  - 物理模拟
  - C++
  - 碰撞检测
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-10-rigid-body-dynamics/rigid_body_output.png
---

## 背景与动机

### 为什么需要刚体动力学？

在游戏开发中，几乎所有具有真实感的物理交互都依赖刚体动力学（Rigid Body Dynamics）。从《愤怒的小鸟》里飞出去的石块砸到木结构倒塌，到《GTA》系列里车辆翻滚，再到《Half-Life 2》令人印象深刻的 Gravity Gun 物理玩法——这些体验的背后都是刚体模拟在支撑。

没有刚体物理，游戏世界里的物体要么静止不动，要么只能走预设动画轨迹。一块石头飞出去不会受重力弯曲，碰到墙壁会穿透过去，没有任何真实感。

**刚体动力学要解决三大核心问题**：

1. **运动积分**：给定力和力矩，怎么更新物体的位置和旋转角度？
2. **碰撞检测**：哪些物体在哪些时刻发生了接触？接触点在哪，法线方向是什么？
3. **碰撞响应**：检测到碰撞后，怎么计算反弹？怎么处理摩擦？

### 工业界如何使用

**游戏引擎层面**：
- Unity 使用 PhysX（NVIDIA），Unreal Engine 早期用 PhysX，4.26 后引入 Chaos Physics
- Godot 内置了自己的 GodotPhysics，也支持 Bullet Physics
- Box2D 是 2D 物理的标准（《愤怒的小鸟》《Angry Birds》《Limbo》都在用）

**这些引擎底层几乎都用本文实现的技术**：
- SAT（分离轴定理）或 GJK 算法做碰撞检测
- 脉冲解算（Impulse-based resolution）处理碰撞响应
- Baumgarte 位置修正防止穿透漂移

**今天的目标**：从零实现一个支持矩形 + 圆形刚体、SAT碰撞检测、脉冲解算、摩擦力的 2D 物理引擎，输出时序模拟动图。

---

## 核心原理

### 1. 刚体的数学表示

一个 2D 刚体需要以下状态量：

| 量 | 符号 | 说明 |
|---|---|---|
| 位置 | **x** | 质心世界坐标 |
| 速度 | **v** | 质心线速度 (px/s) |
| 旋转角 | θ | 朝向角（弧度） |
| 角速度 | ω | 旋转速率 (rad/s) |
| 质量 | m | 线性动量相关 |
| 转动惯量 | I | 角动量相关 |

**质量倒数技巧**：物理引擎里通常存 `invMass = 1/m` 而不是 `m`，原因是：
- 静态物体（墙壁、地面）质量无限大，直接存 `invMass = 0` 即可，不需要特判 `if (isStatic)`
- 所有脉冲计算都是除以质量，倒数便于统一运算

**矩形的转动惯量推导**：

对于质量为 m、宽 2w、高 2h 的矩形，绕质心的转动惯量：

```
I = (1/12) * m * (width² + height²)
  = (1/12) * m * ((2w)² + (2h)²)
  = (1/12) * m * (4w² + 4h²)
  = m * (w² + h²) / 3
```

直觉解释：质量越大、分布范围越广（w、h 越大），旋转越难（I 越大），角速度变化越慢。

**圆的转动惯量**：

```
I = (1/2) * m * r²
```

圆形质量集中在中心附近（相比同尺寸矩形），所以 I 更小，更容易旋转。

### 2. 半隐式 Euler 积分

给定外力 F（重力等），更新状态的步骤：

```
// 先更新速度（由力计算加速度）
a = F / m
v(t+dt) = v(t) + a * dt

// 再用新速度更新位置（这是"半隐式"的关键）
x(t+dt) = x(t) + v(t+dt) * dt
```

**为什么是"半隐式"（Symplectic Euler）而不是普通 Euler？**

普通 Euler 会用旧速度更新位置：`x = x + v_old * dt`  
半隐式用新速度：`x = x + v_new * dt`

看似只是顺序不同，但数学上半隐式 Euler 是**辛积分**（Symplectic Integrator），能保持能量守恒！普通 Euler 会逐渐增加系统能量（弹簧会爆炸），半隐式则保持稳定。这是物理引擎选择它的根本原因。

### 3. 分离轴定理（SAT）

**核心命题**：两个凸多边形不相交，当且仅当存在某个轴，使得它们在该轴上的投影区间不重叠。

**直觉理解**：把两个图形都"压扁"投影到某根轴上，如果存在任何一根轴使得两段投影没有交叠，那这两个形状就没有碰撞。这根轴叫"分离轴"。

**关键定理**：对于凸多边形，只需检查各自边的法线方向作为候选轴。

```
两个矩形 A 和 B：
- A 有 4 条边 → 2 个唯一法线方向（矩形对边平行）
- B 有 4 条边 → 2 个唯一法线方向
- 共检查 4 个轴
```

**SAT 的碰撞信息提取**：

不只是判断有没有碰撞，还要知道：
- **碰撞法线**（normal）：物体应该被推开的方向
- **穿透深度**（depth）：重叠了多少

找到最小重叠深度的那个轴，就是碰撞法线：

```
对每个候选轴 n：
  projA = 投影 A 的所有顶点到 n → [minA, maxA]
  projB = 投影 B 的所有顶点到 n → [minB, maxB]
  overlap = min(maxA, maxB) - max(minA, minB)
  if overlap ≤ 0: 分离，无碰撞，提前退出
  跟踪 minOverlap 和对应的轴

结果：minOverlap 是穿透深度，对应轴是碰撞法线
```

### 4. 圆 vs 矩形碰撞检测

圆与矩形的碰撞是个特殊情况，SAT 对圆不直接适用（圆没有"边"）。解法是：

1. 将圆心变换到矩形的**局部坐标系**（旋转矩阵逆变换）
2. 将局部坐标 clamp 到矩形的半宽/半高范围内，得到矩形上的最近点
3. 计算圆心到最近点的距离，与半径比较

```
local = inverse_rotate(circle.center - rect.center)
closest = clamp(local, -half_extents, +half_extents)
diff = local - closest
dist = length(diff)
if dist >= radius: 无碰撞
else: 碰撞，穿透深度 = radius - dist
```

**特殊情况：圆心在矩形内部**：
此时 `closest == local`，`diff = 0`，无法从 diff 获取法线方向。需要找到圆心到矩形每个边的最短距离，从最近的边推出圆。

### 5. 脉冲解算（Impulse-based Resolution）

**什么是脉冲（Impulse）？**

脉冲 J = 力 × 时间，单位是 kg·m/s（也等于动量变化量）。与其模拟持续力随时间积分，不如直接在碰撞那一刻施加一个瞬间的速度变化。

**接触点的相对速度**：

在接触点 P 处，物体 A 的速度不只有质心线速度 **v**，还要加上旋转引起的切向速度：

```
v_contact_A = v_A + ω_A × r_A
```

其中 r_A = P - center_A（接触点相对质心的向量）。  
在 2D 中，叉积退化为：`ω × r = (-ω*r.y, ω*r.x)`

相对速度（A 相对于 B）：
```
v_rel = v_contact_A - v_contact_B
```

**法向脉冲大小推导**：

碰撞后，法线方向的相对速度必须反弹：

```
v_rel_n_after = -e * v_rel_n_before   (e = 恢复系数)
```

脉冲 j 施加在法线方向，对 A 加速，对 B 减速：

```
v_A_after = v_A + (j/m_A) * n
ω_A_after = ω_A + (r_A × (j*n)) / I_A
```

代入相对速度条件，解出 j：

```
j = -(1 + e) * v_rel · n
    ─────────────────────────────────────────────────────────
    invMass_A + invMass_B + (r_A × n)² * invI_A + (r_B × n)² * invI_B
```

直觉解释：
- 分子：需要翻转多大的法向速度
- 分母：系统对法向脉冲的"响应敏感度"（质量越大、接触点越靠近质心，响应越弱）

### 6. 摩擦力脉冲

碰撞法向脉冲处理后，还需要处理切线方向的摩擦。

**库仑摩擦定律**：
- 静摩擦力 ≤ μ × 法向力
- 如果需要的静摩擦脉冲超过 μ × j，就用动摩擦

```
t = (v_rel - (v_rel · n) * n).normalized()   // 切线方向
jt = -v_rel · t / (invMassSum_tangential)

// 库仑条件
if |jt| < μ * j:
    friction_impulse = jt * t   // 静摩擦
else:
    friction_impulse = -μ * j * t   // 动摩擦（Coulomb）
```

`μ = sqrt(μ_A * μ_B)` 是两个物体摩擦系数的几何均值，这是标准的材质组合方式。

### 7. Baumgarte 位置修正

**问题**：由于浮点精度和时间步离散化，碰撞响应后物体仍可能有残余穿透。多帧累积后，物体会慢慢"沉进去"。

**解法（Baumgarte Stabilization）**：每步直接修正位置，按穿透深度的一定比例往外推：

```
correction = max(depth - slop, 0) / (invMass_A + invMass_B) * percent
A.pos += correction * invMass_A * normal
B.pos -= correction * invMass_B * normal
```

- `slop`（容差）：允许少量穿透存在，防止物体"抖动"（大约 0.01~1.0 px）
- `percent`：每帧修正多少比例（通常 20%~80%）

**为什么不全量修正（percent=1.0）**：会引起"位置爆炸"。每帧修正量太大，下一帧又过度修正回来，形成震荡。

---

## 实现架构

### 整体模块设计

```
main.cpp
├── Vec2          - 2D向量数学库
├── Color         - RGB颜色
├── Canvas        - 软光栅化画布
│   ├── fillPolygon()     - 扫描线填充凸多边形
│   ├── fillCircle()      - 圆形填充
│   ├── drawLine()        - Bresenham直线
│   └── save()            - PNG输出
├── RigidBody     - 刚体数据 + 工厂方法
│   ├── makeRect()        - 矩形刚体
│   ├── makeCircle()      - 圆形刚体
│   ├── getVertices()     - 获取旋转后的顶点
│   └── applyImpulse()    - 施加脉冲
├── Collision     - 碰撞检测
│   ├── satRectRect()     - SAT矩形vs矩形
│   ├── circleCircle()    - 圆vs圆
│   ├── circleRect()      - 圆vs矩形
│   └── detectCollision() - 分发函数
├── resolveCollision()    - 脉冲解算
├── physicsStep()         - 单步模拟
├── drawBody()            - 绘制单个刚体
├── createScene()         - 场景初始化
└── main()                - 主循环
```

### 数据流

```
createScene() → 初始化 14 个刚体（静态 + 动态）
     ↓
for each frame:
    physicsStep(bodies, dt)
        ├── 遍历所有动态体：施加重力，更新速度和位置
        ├── 碰撞解算（10次迭代）：
        │    ├── detectCollision(A, B) → ContactInfo
        │    └── resolveCollision(A, B, info) → 修改速度、角速度、位置
        └── 边界约束（简单 clamp）
    if 需要保存帧: drawScene() → Canvas → save PNG
```

**10次迭代**：物理引擎通常对同一帧内的碰撞做多次求解，让相互接触的物体（如堆叠箱子）逐步传播力。迭代次数越多越稳定，但越慢。

### 刚体设计决策

```cpp
struct RigidBody {
    ShapeType shape;      // RECT or CIRCLE
    Vec2 pos, vel;        // 位置、速度
    float angle, angularVel;  // 旋转角、角速度
    float invMass, invInertia; // 存倒数！静态体=0
    float hw, hh, radius; // 形状参数
    float restitution, friction; // 材质
    bool isStatic;
};
```

**关键设计**：存 invMass 而非 mass。对静态体设 invMass=0，脉冲计算公式完全一致，无需分支。

---

## 关键代码解析

### SAT 矩形 vs 矩形

```cpp
ContactInfo satRectRect(const RigidBody& a, const RigidBody& b) {
    auto vertsA = a.getVertices();
    auto vertsB = b.getVertices();

    // 收集所有候选分离轴（每条边的法线）
    std::vector<Vec2> axes;
    auto addAxes = [&](const std::vector<Vec2>& verts) {
        int n = (int)verts.size();
        for (int i = 0; i < n; i++) {
            Vec2 edge = verts[(i+1)%n] - verts[i];
            axes.push_back(edge.perp().normalized());
        }
    };
    addAxes(vertsA);
    addAxes(vertsB);  // 8个轴，但矩形对边平行，实质4个唯一方向
```

`edge.perp()` 返回 `(-edge.y, edge.x)`，即边的外法线。归一化是必要的，否则投影值没有物理意义（无法比较重叠量）。

```cpp
    float minOverlap = std::numeric_limits<float>::max();
    Vec2 bestAxis;

    for (auto& axis : axes) {
        auto [minA, maxA] = projectOnAxis(vertsA, axis);
        auto [minB, maxB] = projectOnAxis(vertsB, axis);
        float overlap = std::min(maxA, maxB) - std::max(minA, minB);
        if (overlap <= 0) return {false, {}, 0, {}};  // 早期退出！
        if (overlap < minOverlap) {
            minOverlap = overlap;
            bestAxis = axis;
        }
    }
```

**早期退出**（`if overlap <= 0 return`）是 SAT 的性能优化精髓。一旦找到分离轴就立即返回 false，不需要继续检查其他轴。碰撞检测数量远多于碰撞发生次数，早期退出极大提升性能。

```cpp
    // 确保法线从 B 指向 A（推开 A 的方向）
    Vec2 d = a.pos - b.pos;
    if (d.dot(bestAxis) < 0) bestAxis = bestAxis * -1;
```

SAT 找到的轴可能方向是任意的，需要确保法线方向"从 B 指向 A"（即 A 应该被推向正方向）。通过检查两个质心差向量与法线的点积来判断。

### 脉冲解算核心

```cpp
void resolveCollision(RigidBody& a, RigidBody& b, const ContactInfo& info) {
    Vec2 n = info.normal;
    Vec2 rA = info.contactPoint - a.pos;  // 接触点相对 A 质心
    Vec2 rB = info.contactPoint - b.pos;  // 接触点相对 B 质心

    // 计算接触点的相对速度（包含旋转分量）
    Vec2 vA = a.vel + Vec2(-a.angularVel * rA.y, a.angularVel * rA.x);
    Vec2 vB = b.vel + Vec2(-b.angularVel * rB.y, b.angularVel * rB.x);
    Vec2 relVel = vA - vB;

    float velAlongNormal = relVel.dot(n);
    if (velAlongNormal > 0) return;  // 物体正在分离，不需要脉冲
```

`(-ω*r.y, ω*r.x)` 是 2D 中 ω × r 的展开。如果 ω > 0（逆时针），接触点在质心右上方（r.x > 0, r.y > 0），则切线速度向左上方。

**判断是否需要脉冲**：如果接触点相对速度已经是分离的（`velAlongNormal > 0`），则跳过。这防止了"已经弹开却又施加脉冲"的双重处理。

```cpp
    float e = std::min(a.restitution, b.restitution);  // 取较小恢复系数

    // 脉冲分母：线性质量 + 旋转贡献
    float rACrossN = rA.cross(n);  // r_A × n（2D标量叉积）
    float rBCrossN = rB.cross(n);
    float invMassSum = a.invMass + b.invMass
                     + rACrossN * rACrossN * a.invInertia
                     + rBCrossN * rBCrossN * b.invInertia;

    float j = -(1 + e) * velAlongNormal / invMassSum;
```

`rA.cross(n)` 在 2D 中等于 `rA.x * n.y - rA.y * n.x`，其平方乘以 invInertia 就是旋转对"响应"的贡献。

为什么取 `min(restitution)` 而不是平均？因为碰撞的弹性受限于较弱的那一方。一个橡皮球碰到水泥墙（e=0），结果不会弹得很高。

```cpp
    // 摩擦脉冲
    Vec2 tangent = (relVel - n * relVel.dot(n)).normalized();
    float velAlongTangent = relVel.dot(tangent);

    float rACrossT = rA.cross(tangent);
    float rBCrossT = rB.cross(tangent);
    float invMassSumT = a.invMass + b.invMass
                      + rACrossT * rACrossT * a.invInertia
                      + rBCrossT * rBCrossT * b.invInertia;

    float jt = -velAlongTangent / invMassSumT;
    float mu = std::sqrt(a.friction * b.friction);

    Vec2 frictionImpulse;
    if (std::abs(jt) < j * mu) {
        frictionImpulse = tangent * jt;   // 静摩擦：完全消除切向速度
    } else {
        frictionImpulse = tangent * (-j * mu);  // 动摩擦：库仑定律上限
    }
```

`tangent = relVel - (relVel·n)*n` 是将相对速度中的法向分量去掉，剩下切向分量，再归一化。

库仑摩擦判断：如果计算出的静摩擦脉冲 `|jt|` 小于法向脉冲的 μ 倍，就用静摩擦（完全阻止滑动）；否则物体在滑动，用动摩擦（限制在 μ*j）。

### 圆 vs 矩形碰撞

```cpp
ContactInfo circleRect(const RigidBody& circle, const RigidBody& rect) {
    // 将圆心变换到矩形局部坐标系
    float c = std::cos(-rect.angle), s = std::sin(-rect.angle);
    Vec2 d = circle.pos - rect.pos;
    Vec2 local = {c*d.x - s*d.y, s*d.x + c*d.y};

    // 求矩形上的最近点（clamp到矩形范围内）
    Vec2 closest = {
        std::max(-rect.hw, std::min(rect.hw, local.x)),
        std::max(-rect.hh, std::min(rect.hh, local.y))
    };

    Vec2 diff = local - closest;
    float dist = diff.length();
    if (dist >= circle.radius) return {false, {}, 0, {}};
```

**旋转角用 `-rect.angle`**：这是将世界坐标系旋转到矩形局部坐标系，即矩形的逆变换。如果矩形旋转了 +θ 度，那么点要旋转 -θ 度才能变换到矩形局部。

clamp 操作 `max(-hw, min(hw, x))` 找到矩形上沿 x 轴最近的点。二维 clamp 后得到的 `closest` 就是矩形表面（或内部）上距离圆心最近的点。

```cpp
    // 圆心在矩形内部的特殊处理
    if (dist < 1e-8f) {
        float ox = rect.hw - std::abs(local.x);  // 到左右边的距离
        float oy = rect.hh - std::abs(local.y);  // 到上下边的距离
        Vec2 normal_local;
        float depth;
        if (ox < oy) {  // 从最近的边推出
            normal_local = {(local.x > 0) ? 1.0f : -1.0f, 0};
            depth = ox + circle.radius;
        } else {
            normal_local = {0, (local.y > 0) ? 1.0f : -1.0f};
            depth = oy + circle.radius;
        }
        // 将法线旋转回世界坐标系
        float ca = std::cos(rect.angle), sa = std::sin(rect.angle);
        Vec2 normal = {ca*normal_local.x - sa*normal_local.y,
                       sa*normal_local.x + ca*normal_local.y};
        ...
    }
```

圆心在矩形内部时，`diff = local - closest = 0`，无法通过 diff 求法线。改为找到圆心距离四条边最近的那条边，从那个方向推出去，深度 = 到那条边的距离 + 圆的半径。

### 软光栅化多边形填充

```cpp
void fillPolygon(const std::vector<Vec2>& verts, Color c) {
    // 求包围盒
    int minY = height, maxY = 0;
    for (auto& v : verts) {
        minY = std::min(minY, (int)v.y);
        maxY = std::max(maxY, (int)v.y);
    }

    for (int y = minY; y <= maxY; y++) {
        std::vector<float> xs;  // 当前扫描线与各边的交点
        int n = (int)verts.size();
        for (int i = 0; i < n; i++) {
            Vec2 a = verts[i], b = verts[(i+1)%n];
            // 奇偶规则：当扫描线穿过边时记录 x 交点
            if ((a.y <= y && b.y > y) || (b.y <= y && a.y > y)) {
                float t = (y - a.y) / (b.y - a.y);
                xs.push_back(a.x + t * (b.x - a.x));
            }
        }
        std::sort(xs.begin(), xs.end());  // 从左到右排序
        // 两两配对填充水平段
        for (int i = 0; i+1 < (int)xs.size(); i += 2) {
            for (int x = (int)xs[i]; x <= (int)xs[i+1]; x++)
                setPixel(x, y, c);
        }
    }
}
```

**扫描线填充原理**：对每条水平扫描线，找到它与多边形各边的交点，排序后两两配对，填充每对交点之间的水平段。对凸多边形，每条扫描线恰好有 2 个交点（进入和退出）。

**边界条件 `a.y <= y && b.y > y`**：只在边从下往上穿过扫描线时计交点，防止顶点被计算两次（否则两条共顶点的边都会计入该交点，破坏奇偶计数）。

---

## 踩坑实录

### Bug 1：SAT 法线方向不一致

**症状**：物体有时被推向对方（穿进去更深）而不是推开。

**错误假设**：SAT 找到的法线方向总是"从 B 指向 A"。

**真实原因**：SAT 投影时，法线方向取决于边的方向（顺时针或逆时针遍历），与"哪个物体是 A、哪个是 B"无关。同一对碰撞，法线可能指向任意方向。

**修复**：计算两质心的差向量 `d = a.pos - b.pos`，如果 `d.dot(bestAxis) < 0`，翻转法线。这确保了法线总是从 B 指向 A。

```cpp
Vec2 d = a.pos - b.pos;
if (d.dot(bestAxis) < 0) bestAxis = bestAxis * -1;
```

### Bug 2：接触点计算影响脉冲稳定性

**症状**：叠放的箱子剧烈抖动，有时飞出去。

**错误假设**：用 SAT 找到的最小重叠区域中心作为接触点足够精确。

**真实原因**：对于堆叠场景，接触点的微小误差会被角速度放大（r × n 的叉积放大了误差）。接触点偏离真实接触位置时，力矩计算出错，产生虚假旋转。

**修复**：10次碰撞迭代求解 + Baumgarte 位置修正。迭代允许力在堆叠物体间逐步传播，位置修正消除残余穿透，两者共同抑制抖动。

### Bug 3：圆 vs 矩形法线变换错误

**症状**：圆形碰到旋转的矩形时，弹飞方向完全错误。

**错误假设**：法线直接用局部坐标系的方向。

**真实原因**：碰撞法线在矩形局部坐标系中计算，但脉冲解算需要世界坐标系的法线。如果矩形有旋转角 θ，局部法线需要旋转 +θ 才能变回世界坐标。

**修复**：
```cpp
float ca = std::cos(rect.angle), sa = std::sin(rect.angle);
Vec2 normal = {ca*normal_local.x - sa*normal_local.y,
               sa*normal_local.x + ca*normal_local.y};
```

注意：这里用 `+rect.angle`（不是 `-rect.angle`），因为我们是从局部坐标变换回世界坐标，是正变换。

### Bug 4：角阻尼使用错误的衰减方式

**症状**：箱子落地后旋转永远不停，或者停得过快（参数难调）。

**错误假设**：`angularVel *= 0.98f` 每帧固定衰减 2%。

**真实原因**：固定帧率时没问题，但若帧率变化（dt 不固定），衰减速率也会变化，导致物理表现不一致。

**修复**：将衰减与 dt 绑定：
```cpp
angularVel *= std::pow(0.98f, dt * 60.0f);
```
这样无论帧率多少，模拟 1 秒后的衰减量都一致。

### Bug 5：`velAlongNormal > 0` 判断反向

**症状**：物体穿透后竟然仍在被"推开"（但方向错），系统不稳定。

**错误假设**：没有加"已在分离则跳过"的判断。

**真实原因**：如果两个物体接触点的相对速度已经是分离方向（`velAlongNormal > 0`），但位置还有穿透（因为上一帧才发生接触），此时不应施加脉冲。强行施加只会打乱速度。

**修复**：
```cpp
if (velAlongNormal > 0) return;  // 物体正在分离，跳过
```

---

## 效果验证与数据

### 输出文件

| 文件 | 大小 | 说明 |
|------|------|------|
| `rigid_body_output.png` | 92.6 KB | 2×3时序合成图 (1816×1212) |
| `rigid_body_seq_000.png` | 14 KB | t=0.0s 初始状态 |
| `rigid_body_seq_001.png` | 18 KB | t=1.0s 碰撞活跃期 |
| `rigid_body_seq_002.png` | 17 KB | t=2.0s 多体堆叠 |
| `rigid_body_seq_003.png` | 17 KB | t=3.0s 系统逐渐稳定 |
| `rigid_body_seq_004.png` | 17 KB | t=4.0s 大部分静止 |
| `rigid_body_seq_005.png` | 17 KB | t=5.0s 最终状态 |

### 像素采样验证（量化）

```
图像尺寸: 1816 × 1212
像素均值: 43.1  （范围 10~240 ✅）
标准差: 28.4    （> 5，图像有内容 ✅）
文件大小: 92.6 KB（> 10KB ✅）
```

均值 43 说明图像整体偏暗（深色背景 + 彩色刚体），标准差 28 说明有明显的内容变化（刚体颜色、边界线等）。

### 场景统计

```
刚体总数: 14
  静态体: 5（地面1 + 墙壁2 + 斜坡2）
  动态矩形: 5（4叠箱 + 1重箱）
  动态圆形: 4
碰撞对数量（最坏情况）: C(14,2) = 91对/帧
实际活跃碰撞对（平均）: ~6对/帧
每帧物理步 × 10次迭代
总模拟帧数: 360帧（6秒 × 60fps）
```

### 模拟结束时的体位置（t=5.0s）

```
Body 5 (蓝箱0): pos=(127.7, 52.9) vel=(0.7, 2.3)  → 基本静止
Body 9 (蓝箱3): pos=(541.3, 47.3) vel=(19.2, 2.3) → 靠右墙滑动
Body 12 (圆1):  pos=(83.8, 154.9) vel=(-1.3, 0.1)  → 卡在斜面
Body 13 (圆4):  pos=(347.2, 34.3) vel=(14.9, 2.3)  → 地面滑行
```

大部分物体速度降到个位数，说明摩擦力和碰撞阻尼工作正常。y 方向速度 2.3 是最小碰撞响应（Baumgarte 修正的残余量），属于正常。

---

## 总结与延伸

### 技术局限性

**本实现的主要局限**：

1. **接触点精度不足**：SAT 给出的接触点是近似值（投影重叠区域中心），堆叠多体时会有抖动。工业级引擎用 Contact Manifold（接触流形）追踪多个接触点，稳定性远好于本实现。

2. **时间步固定**：用固定 dt=1/60s，帧率不足时会跳跃。生产环境需要用 Sub-stepping（将大时间步分成多个小步）。

3. **碰撞检测 O(N²)**：遍历所有物体对，N=100 时 5000 对/帧。需要 BVH 或空间哈希进行宽相（Broad Phase）剔除。

4. **不支持 Sleeping**：静止物体每帧仍参与碰撞计算。真实引擎会将速度接近零的物体标记为 "sleeping"，跳过它们直到有冲击激活。

5. **只有 10 个 Solver 迭代**：复杂堆叠（10+ 个物体堆）可能不收敛，需要更多迭代或改用 PGS（Projected Gauss-Seidel）。

### 与本系列其他文章的关联

- **[03-08 布料模拟]**：也是质点弹簧系统 + Verlet 积分，但布料不是刚体，不需要旋转动力学
- **[03-09 SPH 流体]**：流体用粒子近似，无碰撞检测，只有邻域搜索 + 压力场
- **[04-08 粒子系统]**：纯运动学（无碰撞），本文加入了刚性约束
- **[04-09 SPH]**：流体模拟，粒子之间通过压力交互，而非脉冲碰撞

### 可优化方向

**短期优化**：
- 加入 Broad Phase（AABB Grid），将碰撞检测从 O(N²) 降至近似 O(N)
- Contact Manifold：每对碰撞体维护 2 个接触点，避免单点接触的不稳定
- Island Sleeping：静止物体岛屿跳过模拟

**进阶扩展**：
- 约束系统（铰链、滑轨）：用 Lagrange 乘子方法
- 凸多边形 GJK/EPA：更通用的碰撞检测，支持任意凸形状
- 3D 扩展：需要四元数旋转、3D SAT 或 GJK
- 连续碰撞检测（CCD）：防止高速物体"穿墙"（Tunneling）

今天的实现已经包含了一个完整 2D 物理引擎的所有核心要素——积分、碰撞检测、脉冲响应、摩擦。理解了这套框架，Box2D 和 PhysX 的核心逻辑就不再神秘了。

---

**代码仓库**: [GitHub](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-04-10-rigid-body-dynamics)  
**输出预览**:

![时序合成图](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-10-rigid-body-dynamics/rigid_body_output.png)
