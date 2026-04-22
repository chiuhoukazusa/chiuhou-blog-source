---
title: "每日编程实践: L-System 程序化树木渲染"
date: 2026-04-23 05:44:00
tags:
  - 每日一练
  - 图形学
  - 程序化生成
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-23-lsystem-tree/lsystem_tree_output.png
---

## 一、背景与动机

### 植物建模的本质困境

在游戏和影视中，自然场景不可或缺——森林、草地、树木几乎出现在所有户外关卡中。但手工建模植物极为费时：一棵写实的大树可能包含数万根树枝、数十万片叶子。如果每一根枝条都由美术手动摆放，那么任何一款开放世界游戏的制作周期都将延长数倍。

传统手工建模还有一个根本问题：**难以复用**。游戏里需要成百上千种形态不同的树，美术不可能逐一雕刻，也不可能让每棵树都用同一个模型。真实世界的植物拥有无限多样性，却始终遵循着相似的结构规律。这种"规则中的变化"正是程序化生成要捕捉的本质。

### L-System 的诞生与定位

1968年，匈牙利生物学家 Aristid Lindenmayer 在研究藻类细胞分裂时，提出了一套描述生物生长过程的形式文法系统，后来被称为 **Lindenmayer 系统**，简称 **L-System**。他的洞察是：植物生长可以用极少的**重写规则**来完整描述——每根茎干分叉时遵循相同的模式，只是比例和角度略有不同。

L-System 的核心在于**自相似性**：树的整体结构与每个子树枝高度相似。这正是分形几何的本质特征。通过 4-6 次迭代，一条两个字符的初始公理可以展开成数万个字符的字符串，对应着数千根枝条的完整树形。

### 工业界实际应用

L-System 及其变体在现代工业中无处不在：

- **SpeedTree**（游戏和影视界最主流的植被工具）底层大量借鉴 L-System 思路，用参数化规则生成各类树形，《荒野大镖客2》《赛博朋克2077》均使用该方案
- **Houdini 的 Labs 植物工具**直接集成了 L-System 节点，支持可视化编辑生长规则
- **UE5 的 PCG（程序化内容生成）框架**中，植被生成模块同样包含基于符号重写的参数化结构
- **离线渲染领域**（Disney 内部渲染器 Hyperion、Pixar 的 RenderMan）用 L-System 生成高密度植被分布

今天的实现是最原始、最纯粹的 L-System 渲染——用 C++ 从零实现文法解释器和 3D 软光栅化器，不依赖任何图形库，直接输出 PNG 图像。

---

## 二、核心原理

### 2.1 形式文法与字符串重写

L-System 本质上是一种**并行字符串重写系统（Parallel String Rewriting System）**，由以下三部分定义：

```
G = (V, ω, P)
```

- **V**（字母表）：所有合法符号的集合，如 `{F, A, +, -, &, ^, \, /, [, ]}`
- **ω**（公理 Axiom）：初始字符串，如 `FFFFA`
- **P**（产生式 Production Rules）：每个字符的替换规则

与普通语法的关键区别在于 **并行性**：每一次迭代时，字符串中的**所有**符号同时被替换（不是逐个处理）。这模拟了植物所有部位同步生长的生物学现实。

**本项目使用的文法**：

```
公理：FFFFA
规则：A → FF[&+A][&-A][&\A][&/A]FA
```

解读：
- 主干先长 `FF`（两段茎干）
- 然后分出4个方向的侧枝：偏仰+左转 `[&+A]`、偏仰+右转 `[&-A]`、偏仰+前滚 `[&\A]`、偏仰+后滚 `[&/A]`
- 最后主轴继续延伸 `FA`（自相似：A 在下一次迭代中继续展开）

迭代展开过程：

```
n=0: FFFFA
n=1: FFFFFFFF[&+FFFFA][&-FFFFA][&\FFFFA][&/FFFFA]FFFFA
n=2: (指数级展开)...
n=6: ~89,843 个字符
```

每次迭代，非终结符 `A` 被完整展开一层，产生的字符串长度大约以 **5倍** 的速度增长（每个 A 产生约 5 个新节点）。

### 2.2 龟形图形学（Turtle Graphics）解释器

字符串本身只是抽象的符号序列，需要通过**龟形图形学**将其转换为几何结构。这个名字来自 LOGO 语言中的"海龟"：一只爬行的虚拟龟，接收指令向前爬行或转身，留下轨迹即为枝条。

在 3D 版本中，龟的状态由以下量完整描述：

```
状态 = {位置 p ∈ ℝ³, 方向矩阵 M ∈ SO(3), 长度 l, 像素半径 r}
```

方向矩阵 `M` 是一个 3×3 正交矩阵，三列分别为：
- `col[0]` = 右方向（Right）
- `col[1]` = 前进方向（Heading，即龟鼻子指向）
- `col[2]` = 局部前向（Forward = Heading × Right）

符号映射表：

| 符号 | 含义 | 操作 |
|------|------|------|
| `F`  | 向前画 | 在当前方向延伸枝条，移动到新位置 |
| `+`  | 偏航左转 | 绕 `col[2]` 轴旋转 +angle |
| `-`  | 偏航右转 | 绕 `col[2]` 轴旋转 -angle |
| `&`  | 俯仰前倾 | 绕 `col[0]` 轴旋转 +angle |
| `^`  | 俯仰后仰 | 绕 `col[0]` 轴旋转 -angle |
| `\`  | 滚转左 | 绕 `col[1]` 轴旋转 +angle |
| `/`  | 滚转右 | 绕 `col[1]` 轴旋转 -angle |
| `[`  | 压栈 | 保存当前状态（分支开始） |
| `]`  | 弹栈 | 恢复上一状态（分支结束） |

**关键问题：旋转的参考轴**

与很多教程不同，本实现中旋转操作的参考轴不是**全局固定轴**，而是**龟的局部坐标轴**。这意味着每次旋转后，下次旋转的参考系也跟着转动——这正是真实植物分枝的行为。

### 2.3 Rodrigues 旋转公式

每次旋转调用 Rodrigues 旋转，将方向矩阵的三列绕指定轴各旋转一次：

对向量 **v** 绕单位轴 **k** 旋转角度 θ：

```
v_rot = v·cos(θ) + (k×v)·sin(θ) + k·(k·v)·(1-cos(θ))
```

这个公式的直觉解释：
- `v·cos(θ)`：将原向量缩小，如同投影在旋转后方向的"影子"
- `(k×v)·sin(θ)`：沿垂直平面的旋转分量（叉乘给出垂直方向）
- `k·(k·v)·(1-cos(θ))`：沿旋转轴方向的保留分量（点积给出平行分量）

合并为矩阵形式（Rodrigues 旋转矩阵 R）：

```
R = I·cos(θ) + (1-cos(θ))·k⊗k + sin(θ)·[k]×
```

其中 `k⊗k` 是外积（rank-1矩阵），`[k]×` 是反对称矩阵（叉积的矩阵表示）。

代码实现：

```cpp
Mat3 rotateAround(const Mat3& m, const Vec3& axis, float angle) {
    float c = cosf(angle), s = sinf(angle), mc = 1-c;
    Vec3 a = axis;
    // 展开 Rodrigues 旋转矩阵的9个元素
    float r00 = c + a.x*a.x*mc,       r01 = a.x*a.y*mc - a.z*s, r02 = a.x*a.z*mc + a.y*s;
    float r10 = a.y*a.x*mc + a.z*s,   r11 = c + a.y*a.y*mc,     r12 = a.y*a.z*mc - a.x*s;
    float r20 = a.z*a.x*mc - a.y*s,   r21 = a.z*a.y*mc + a.x*s, r22 = c + a.z*a.z*mc;
    // 将旋转矩阵应用到方向矩阵的每一列
    Mat3 result;
    for (int i=0; i<3; i++) {
        Vec3 v = m.col[i];
        result.col[i] = {
            r00*v.x + r01*v.y + r02*v.z,
            r10*v.x + r11*v.y + r12*v.z,
            r20*v.x + r21*v.y + r22*v.z
        };
    }
    return result;
}
```

为什么选 Rodrigues 而不是欧拉角或四元数？
- **欧拉角**存在万向锁问题（Gimbal Lock），当某个分量达到 90° 时会丢失一个自由度
- **四元数**计算正确，但需要额外转换，且本项目需要方向矩阵的列向量直接用于旋转操作
- **Rodrigues**：直接操作矩阵，没有奇异点，实现紧凑，3D 植物建模的理想选择

### 2.4 枝条的分形维度与迭代参数

L-System 生成的几何是真正的数学分形。枝条的分支比例系数 `lenScale = 0.68f`，即每分支一次，新枝长度为父枝的 68%。经过 n 层递归后，末端枝条长度为：

```
l_n = l_0 × 0.68^n
```

6层后：`l_6 = l_0 × 0.68^6 ≈ l_0 × 0.098`，即原始长度的约10%。

像素半径的缩减系数 `radScale = 0.64f`，6层后半径约为初始的 `0.64^6 ≈ 0.068`，即初始半径 9.0 像素的约0.6像素——足够细到消失于单像素。

---

## 三、实现架构

### 3.1 整体数据流

```
L-System 定义
    ↓ generate(iters)
L-string (字符序列)
    ↓ buildTree(lstr, ...)
Branch[] + LeafQuad[]
    ↓ sort by viewZ (Painter's Algorithm)
排序后的几何数组
    ↓ 正交投影 + 软光栅化
Framebuffer (uint8_t[H][W][3])
    ↓ stb_image_write
lsystem_tree_output.png
```

### 3.2 关键数据结构

```cpp
// 龟的状态（压栈弹栈的基本单元）
struct TState {
    Vec3 pos;          // 当前世界坐标
    Mat3 orient;       // 方向矩阵（3个正交列向量）
    float length;      // 当前步长（世界单位）
    float pixRadius;   // 枝条渲染半径（像素）
    int generation;    // 递归深度（用于颜色映射）
};

// 枝条段（一对首尾点的线段）
struct Branch {
    Vec3 start, end;   // 世界坐标
    float r0, r1;      // 首尾半径
    Vec3 color;        // 颜色
    float viewZ;       // 深度值（用于 Painter's Algorithm 排序）
};

// 叶片（圆形近似）
struct LeafQuad {
    Vec3 pos;          // 世界坐标
    float size;        // 像素半径
    Vec3 color;
    float viewZ;
};
```

### 3.3 相机系统设计

本项目使用正交投影（Orthographic Projection），而不是透视投影。原因：

1. 树木体积不大，透视畸变会使近处枝条看起来过粗
2. 正交投影实现简单，不需要处理 near/far clip 的 z-除法
3. 类似于印象派画作的"扁平感"正好适合程序化树木的风格化呈现

正交相机用3个正交基向量定义观察空间：

```cpp
struct Camera {
    Vec3 right, up, forward;  // 相机坐标系
    float scale;              // 世界单位到像素的缩放
    int cx, cy;               // 画面中心（像素）
    
    Vec2 project(const Vec3& p) {
        Vec3 d = p - pos;
        float sx = d.dot(right) * scale + cx;
        float sy = -d.dot(up)   * scale + cy;  // 注意Y轴翻转
        return {sx, sy};
    }
    float projZ(const Vec3& p) {
        return -(p - pos).dot(forward);  // 负号：越小越近
    }
};
```

相机参数（轻微斜视角）：

```cpp
float yaw = 15° * PI/180;   // 水平偏转 15 度
float pitch = 5° * PI/180;  // 垂直仰角 5 度
cam.right   = {cos(yaw), 0,          -sin(yaw)};
cam.up      = {sin(yaw)*sin(pitch), cos(pitch), cos(yaw)*sin(pitch)};
cam.forward = {sin(yaw)*cos(pitch), -sin(pitch), cos(yaw)*cos(pitch)};
cam.scale   = 55.0f;  // 每世界单位对应55像素
cam.cx = 450; cam.cy = 702;  // 树根在画面下方 78% 处
```

### 3.4 Painter's Algorithm（画家算法）

3D 场景的遮挡问题通常用 Z-Buffer 解决。但画家算法在某些情况下更适合：

- 树木枝条大量重叠，Z-Buffer 需要对每个像素比较深度
- 画家算法按深度排序后从后往前绘制，后绘制的自然覆盖前面的——与真实绘画中"先画背景再画前景"相同

深度值使用 `viewZ = cam.projZ(midpoint)`，即中点在相机坐标系中的 Z 分量。排序：

```cpp
std::sort(branches.begin(), branches.end(),
    [](const Branch& a, const Branch& b){ return a.viewZ < b.viewZ; });
```

`viewZ` 越小意味着离相机越远（因为我们用了负号），先渲染。

---

## 四、关键代码解析

### 4.1 L-System 生成器

```cpp
struct LSystem {
    std::string axiom;
    char from[16];       // 规则左侧符号
    std::string to[16];  // 规则右侧字符串
    int count = 0;

    void addRule(char f, const std::string& t) { from[count]=f; to[count]=t; count++; }

    std::string generate(int iters) const {
        std::string cur = axiom;
        for (int i=0; i<iters; i++) {
            std::string next;
            next.reserve(cur.size() * 5);  // 预分配5倍容量，避免频繁重分配
            for (char c : cur) {
                bool replaced = false;
                for (int j=0; j<count; j++) {
                    if (from[j]==c) {
                        next += to[j];  // 并行替换：每个字符独立展开
                        replaced=true; break;
                    }
                }
                if (!replaced) next += c;  // 没有规则的字符（如[、]）直接保留
            }
            cur = next;
        }
        return cur;
    }
};
```

关键设计点：
- `reserve(size*5)`：L-System 每次迭代字符串长度大约增长5倍，预分配减少内存重分配次数
- 规则查找是 O(count) 的线性搜索，因为规则数量很少（通常1-4条），不值得用哈希表
- 无规则符号直接保留，这是 L-System 的核心规定：字母表中的控制符号（如`[`）不需要规则

### 4.2 龟形解释器的方向初始化

```cpp
// 树木向上生长 → 初始前进方向 = +Y 世界轴
turtle.orient.col[0] = {1, 0, 0};   // right  = +X
turtle.orient.col[1] = {0, 1, 0};   // heading = +Y (向上)
turtle.orient.col[2] = {0, 0, 1};   // forward = +Z
```

为什么选 +Y 作为初始前进方向而不是 +Z？
- 在世界坐标系中，+Y 通常代表"向上"（Y-up convention）
- 初始前进方向 = 向上，树自然从地面竖直生长
- 旋转命令 `&`（绕局部right轴旋转）会将前进方向"向下偏转"，模拟枝条弯曲下垂

### 4.3 枝条颜色映射

```cpp
// 用 generation（递归深度）映射颜色
float t = (float)turtle.generation / (float)(maxGen+1);
Vec3 col;
if (t < 0.35f) {
    // 主干：暖棕色
    col = Vec3(0.48f, 0.30f, 0.13f) + Vec3(jit(rng)*0.05f, jit(rng)*0.03f, 0);
} else if (t < 0.65f) {
    // 中等枝条：棕绿色过渡
    float s = (t-0.35f)/0.3f;  // 0→1 的插值因子
    col = Vec3(0.35f - s*0.08f,    // 红色分量下降（棕色→绿色）
               0.30f + s*0.10f,    // 绿色分量上升
               0.12f + s*0.05f);   // 蓝色微增
} else {
    // 细枝：绿色（含随机抖动）
    col = Vec3(0.20f, 0.38f + jit(rng)*0.06f, 0.14f + jit(rng)*0.04f);
}
```

这个颜色映射模拟真实树木的规律：
- 主干和粗枝因为树皮覆盖而呈棕色（木质素丰富）
- 细枝因为仍处于生长期，含有大量叶绿素，偏绿
- 随机抖动避免颜色过于均匀

### 4.4 叶片散布策略

```cpp
// 叶片只在足够细的枝条上生成
if (turtle.pixRadius < basePixR * 0.55f) {
    // 越细的枝条叶片越密集
    float leafChance = 1.0f - (turtle.pixRadius / (basePixR * 0.55f)) * 0.4f;
    // leafChance 范围：0.6(接近阈值) ~ 1.0(最细枝条)
    
    if (prob(rng) < leafChance) {
        int nL = 4 + (int)(rng()%5);  // 每段 4-8 片叶子
        for (int i=0; i<nL; i++) {
            float lt = 0.15f + rand * 0.85f;  // 沿枝条随机位置
            Vec3 lp = startPos + heading * (len*lt);
            // 在枝条法平面内随机散布
            lp += right * (lo(rng)*len*0.55f)
               + fwd   * (lo(rng)*len*0.55f)
               + heading*(lo(rng)*len*0.2f);
            // ...
        }
    }
}
```

设计考量：
- 阈值 `0.55f × basePixR`：当枝条半径降低到初始半径的55%时开始长叶，这确保主干和粗枝不会被叶片遮挡（否则看不到树的骨架结构）
- 叶片沿枝条的法平面（right × fwd 方向）散布，而非沿枝条方向——这让叶簇看起来向外蓬发，而非顺着枝条方向排列
- `heading` 方向的抖动幅度较小（`*0.2f`）：叶片主要在横向散布，不在纵向拉伸

### 4.5 厚线段渲染

枝条通过沿线段方向步进并绘制圆形截面来实现：

```cpp
void drawThickLine(float x0, float y0, float x1, float y1,
                   float z0, float z1, float r0, float r1, Vec3 col) {
    float dx = x1-x0, dy = y1-y0;
    float len = sqrtf(dx*dx+dy*dy);
    if (len < 0.001f) return;

    int steps = (int)(len) + 2;  // 步进次数 ≈ 线段像素长度，确保无缝覆盖
    for (int i=0; i<=steps; i++) {
        float t = (float)i / steps;
        float cx = x0 + dx*t, cy = y0 + dy*t;
        float cz = z0 + (z1-z0)*t;
        float cr = r0 + (r1-r0)*t;  // 线性插值半径（锥形）
        
        float bright = 0.75f + 0.25f*(1.0f-t);  // 近端略亮，远端略暗
        int ir = (int)ceilf(cr);
        for (int oy=-ir; oy<=ir; oy++)
            for (int ox=-ir; ox<=ir; ox++) {
                float d = sqrtf((float)(ox*ox+oy*oy));
                if (d <= cr) {
                    float edge = (1.0f - d/cr * 0.4f) * bright;
                    // edge：圆心处=1.0, 边缘处=0.6，模拟边缘暗化
                    putPixel(...);
                }
            }
    }
}
```

这个方法是"圆形印章沿线段盖印"，每步的圆形印章间距约1像素，确保线段完全填充无缝隙。相比扫描线填充三角形，这种方法实现简单，且天然处理了任意粗细。

---

## 五、踩坑实录

### Bug 1：树木倒置（上下颠倒）

**症状**：树冠在底部，树根在顶部，枝条向下展开

**错误假设**：以为需要将初始方向旋转 -90° 绕X轴，才能让龟"向上"爬行

**真实原因**：搞反了旋转方向。L-System 的 `&` 符号（俯仰）本身会将前进方向向下倾斜分支。初始前进方向直接设为 `{0,1,0}`（+Y = 向上）才是正确的，不需要额外旋转。用 `-PI/2` 旋转会导致初始方向变成 -Y（向下）。

**修复**：将 `rotateX(-PI/2)` 改为直接初始化 `orient.col[1] = {0, 1, 0}`

**教训**：坐标系初始化要从最简单的情况验证——先渲染一条竖直线段，确认"向上生长"是正确的，再加入旋转规则。

---

### Bug 2：上部枝条无叶片

**症状**：树冠下半部分叶片茂密，上半部分枝干全部裸露，整体像"戴草帽的树"

**错误假设**：以为用 `turtle.depth >= maxDepth - 2`（即只有足够深的分支才生成叶片）能让叶片覆盖树冠顶部

**真实原因**：L-System 规则中，`A` 总是直接展开（不经过 `[...]`），所以主干延伸的段始终保持 generation=0，永远不会达到 `maxDepth`。上部枝条实际上都是 generation=0 到 generation=1 的段，远低于 `maxDepth - 2` 的阈值。

**修复**：改用枝条**像素半径**作为叶片生成的判断标准——半径 < baseRadius × 55% 时生成叶片。半径随递归自动缩小，不依赖 generation 深度，正确覆盖所有细枝。

**教训**：在分析生成结果异常时，先打印 `turtle.generation` 的实际分布，而不是凭感觉估计。

---

### Bug 3：叶片"穿地"

**症状**：树木下方的枝条延伸到"地面"以下，对应的叶片出现在地面之下，渲染出来像是地下长了叶子

**错误假设**：以为 L-System 的分枝只会向上和两侧展开

**真实原因**：规则 `[&+A]` 中 `&` 是俯仰，在初始方向为 +Y 的情况下，`&` 会让新枝向 -Z 方向倾斜——对于相机视角，部分枝条会向下弯曲触及"地面"（Y=0 平面以下）。

**修复**：将 `cam.cy`（相机中心Y坐标）从 `H/2` 调整到 `H*0.78`（画面下方78%处），给树木更多向上生长的显示空间，同时让地面背景渐变起始点正好与树根对齐。

---

### Bug 4：编译 stb_image_write.h 产生大量警告

**症状**：编译时出现20余条 `-Wmissing-field-initializers` 警告，来自第三方库 stb_image_write.h

**原因**：stb 库的 `stbi__write_context s = { 0 }` 初始化语句在 GCC 中触发了"未初始化结构体字段"警告。这不是代码错误，而是 GCC 的静态分析过于严格。

**修复**：用 pragma 临时屏蔽第三方库的警告：

```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"
#pragma GCC diagnostic pop
```

`push/pop` 对确保只屏蔽 stb 头文件的警告，不影响其他代码的告警检测。

---

### Bug 5：随机抖动导致枝条角度过大

**症状**：某些迭代下，树形严重不对称，部分枝条几乎水平甚至向下弯曲

**原因**：初始版本的角度抖动范围是 `angle + jit * 0.3f`，其中 `jit ∈ [-0.1, 0.1]`（弧度）。基础角度25°≈0.44rad，抖动量 `0.3 * 0.1 = 0.03rad`。但分支角度 `angle = 25°` + 抖动，在某些情况下叠加后达到 60° 以上，导致分枝几乎水平，最终使叶片深入地面。

**修复**：将抖动系数从 `0.3f` 降低到 `0.15f`，同时将分支角度从 30° 降回 25°，保持树形紧凑。

---

## 六、效果验证与数据

### 渲染输出

![L-System 程序化树木渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-23-lsystem-tree/lsystem_tree_output.png)

*图：6次迭代的 L-System 程序化树木，900×900像素，正交投影，带叶片散布，渲染时间 0.20 秒*

### 量化验证数据

验证脚本输出（像素采样）：

```
均值: 160.2  （范围期望：10-240，✅ 通过）
标准差: 58.6  （期望 > 5，✅ 通过）
尺寸: 900×900
文件大小: 92KB（期望 > 10KB，✅ 通过）
上方蓝色均值: 216.3（天空区域，✅ 通过）
下方绿色均值: 125.9（地面区域，✅ 通过）
```

分区域像素统计：
- 顶部1/4（天空）：蓝色通道均值216，符合天空渐变背景
- 中间1/2（树冠）：绿色通道均值183，叶片密集
- 底部1/4（地面）：绿色通道均值126，地面颜色符合预期

### 几何数据统计

| 指标 | 数值 |
|------|------|
| L-string 长度 | 89,843 字符 |
| 枝条段数量 | 27,347 |
| 叶片数量 | 143,933 |
| 迭代次数 | 6 |
| 渲染时间 | 0.20 秒 |
| 编译时间 | <0.5 秒 |
| 输出文件大小 | 92 KB PNG |
| 最大树高（世界单位） | ~4.5 |
| 分支角度 | 25° ± 1.5° |

### 性能分析

渲染瓶颈分析（粗略测量）：

- **buildTree 阶段**（字符串遍历 + 几何生成）：~30ms
- **深度排序**（Painter's Algorithm sort）：~15ms
- **枝条光栅化**（27,347条线段）：~60ms
- **叶片光栅化**（143,933个圆形）：~95ms
- **总计**：~200ms（单线程）

叶片数量是瓶颈，因为每个叶片需要绘制一个半径约6像素的圆形，每圆约 36×π ≈ 113 像素的循环，143,933 个叶片共约 1,625 万次 `putPixel` 调用。

---

## 七、总结与延伸

### 技术局限性

1. **没有碰撞检测**：枝条可能穿透彼此，真实树木不会这样。解决方案：加入枝条间的排斥力（参考 L-System 的开放 L-System 变体）
2. **正交投影缺乏深度感**：远近枝条大小相同，整体偏平。可改用透视投影，增加3D立体感
3. **叶片是球形近似**：真实叶片是扁平的椭圆形，有复杂的叶脉纹理。可改用纹理映射的四边形（Billboard）
4. **没有光照模型**：只有简单的边缘暗化，缺少太阳方向光和阴影。加入 Lambertian 漫射 + 自阴影会大幅提升质感
5. **L-System 没有参数化**：Prusinkiewicz 的参数化 L-System（Parametric L-System）支持每个符号携带数值参数，可以更精细控制每条枝条的属性

### 可优化方向

1. **随机 L-System（Stochastic L-System）**：为同一个符号提供多条规则，每条规则有不同概率被选中，产生不规则自然的树形
2. **上下文敏感 L-System（Context-sensitive L-System）**：规则可以感知左右邻居，模拟信号在植物中的传递（如激素信号控制分枝）
3. **并行化**：叶片光栅化是天然可并行的，用 OpenMP 或 SIMD 可轻松提速4-8倍
4. **树皮纹理**：主干枝条加入 Perlin 噪声的法向量扰动，渲染时产生凹凸感

### 与本系列的关联

今天的 L-System 与之前的项目有多处技术交集：

- **[04-01 SDF Ray Marching]**：SDF 也是隐式几何描述，L-System 是显式符号描述——两种不同的程序化生成范式
- **[04-06 Procedural Terrain]**：均属于程序化内容生成（PCG），地形用噪声函数，植被用L-System
- **[04-17 Marching Cubes]**：若将 L-System 生成的树形体素化，可用 Marching Cubes 提取树木的光滑曲面表示
- **[04-21 Bloom]**：在游戏引擎中，树木受光部分通常会触发 Bloom 效果（嫩叶透光后的发光感），可将今天的输出作为 Bloom 的输入

L-System 是通往程序化生成世界的入口——从这里延伸出去，可以研究更复杂的植物建模（花朵、草甸、珊瑚礁），甚至城市道路生成（城市 L-System），或者分形艺术（Koch 雪花、Sierpinski 三角形的 L-System 表示）。

---

**代码仓库**：[daily-coding-practice/2026/04/04-23-lsystem-tree](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/04/04-23-lsystem-tree)

**编译运行**：
```bash
g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra
./output  # 生成 lsystem_tree_output.png
```
