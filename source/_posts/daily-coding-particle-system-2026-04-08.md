---
title: "每日编程实践: Particle System Simulation — 烟花粒子系统"
date: 2026-04-08 05:30:00
tags:
  - 每日一练
  - 图形学
  - 粒子系统
  - C++
  - 物理模拟
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-08-Particle-System-Simulation/particle_output.png
---

# 每日编程实践 Day 50: Particle System Simulation — 烟花粒子系统渲染

## ① 背景与动机

烟花爆炸是最直观的粒子系统应用场景之一。游戏、电影 VFX、实时渲染引擎，几乎所有需要表现爆炸、烟雾、火焰、魔法特效的场景，底层都是粒子系统。

**没有粒子系统时的痛点**：

若要在屏幕上呈现"烟花爆炸"的视觉效果，最朴素的做法是手动绘制一张预先制作的精灵图（Sprite），或者播放一段预渲染的视频。这样的问题是：

1. **缺乏真实感**——爆炸的每次形态都一样，玩家立刻能识别出"假感"
2. **无法与场景交互**——粒子无法受到风场、重力场、碰撞体影响
3. **资源占用大**——视频帧序列的存储成本远高于运行时模拟
4. **参数化调整困难**——改一个颜色就要重新制作素材

粒子系统通过"定义少量参数 + 运行时物理模拟"的方式，用极低的存储成本换来高度多样化的视觉输出。

**工业界实际使用场景**：

- **Unreal Engine Niagara**：UE5 的粒子系统，支持 GPU 粒子（百万量级），用于烟雾、火焰、魔法特效
- **Unity VFX Graph**：基于 GPU Compute Shader，支持复杂的力场和碰撞
- **影视 VFX**：《蜘蛛侠》《复仇者联盟》中的爆炸特效大量使用 Houdini 粒子系统
- **实时游戏**：《原神》的元素爆发特效、《战神》的霜焰特效，都是精心设计的粒子系统

今天的项目在 CPU 软渲染环境下实现一个完整的烟花粒子系统，覆盖：
- 粒子物理模拟（重力 + 阻力）
- 粒子生命周期管理
- HDR 渲染 + Reinhard 色调映射
- 时序帧合成（将多时间步叠加到单张图）
- 软光栅 Splat 绘制（带 Gaussian 衰减的圆形笔刷）

---

## ② 核心原理

### 粒子物理模型

每个粒子是一个质点，状态由以下变量描述：

```
position  p = (x, y)     — 二维坐标
velocity  v = (vx, vy)   — 速度向量
life      l ∈ [0, 1]     — 寿命（0表示消亡）
color     c = (r, g, b)  — HDR 颜色
```

**受力分析**：

烟花粒子主要受到两个力：

1. **重力**（Gravity）：向下的恒定加速度

   ```
   a_gravity = (0, g)   其中 g > 0（屏幕 y 轴向下）
   ```

   直觉：重力把所有粒子往下拉，让爆炸后的粒子形成向下弯曲的抛物线轨迹，而不是无限直线飞出。

2. **空气阻力**（Drag）：与速度反向的力，正比于速度

   最简单的模型是线性阻力（Stokes drag），适用于低雷诺数：

   ```
   F_drag = -k * v
   ```

   在离散模拟中，我们用速度乘一个衰减系数 d ∈ (0, 1) 来近似：

   ```
   v_new = v_old * d
   ```

   直觉：空气阻力让粒子越飞越慢。没有阻力时，粒子会永远保持初速度，爆炸像精准的炮弹射出去。加了阻力，粒子先快后慢，形成自然的"减速飘落"感。

**欧拉积分**（Explicit Euler）：

最简单的数值积分方法，每步：

```
v(t+dt) = v(t) * drag + gravity * dt
p(t+dt) = p(t) + v(t+dt) * dt
```

注意这里先更新速度，再用新速度更新位置（Semi-implicit Euler，稳定性优于纯显式欧拉）。

为什么用欧拉而不用 Runge-Kutta？对于粒子特效，我们追求的是"看起来像烟花"，而不是精确的物理模拟。欧拉足够，RK4 浪费计算。

### 粒子寿命与颜色衰减

每个粒子有一个 `decay` 率（每步减少的寿命量），寿命从 1.0 衰减到 0：

```
life(t+dt) = life(t) - decay
```

颜色随寿命的平方衰减，而不是线性衰减：

```
render_color = base_color * life²
```

**为什么用平方而不是线性？**

线性衰减（`color * life`）在 life = 0.5 时亮度已经减半，粒子消失感太早太均匀，缺乏视觉冲击力。

平方衰减（`color * life²`）的曲线更"凸"：粒子在 life 较高时保持明亮，只有在最后阶段才迅速变暗，更接近真实烟花的"持亮→突然暗淡"效果。

数学直觉：若 life ∈ [0,1]，那么 life² < life（对所有 life ∈ (0,1)），说明平方衰减比线性更慢到一半亮度，更快在最后熄灭。

### HDR 渲染与色调映射

传统 LDR（Low Dynamic Range）渲染中，颜色值在 [0, 1] 范围内。这导致一个问题：当多个粒子重叠时，颜色会"溢出截断"（clamp），丢失亮部细节。

我们使用 HDR 渲染：粒子的亮度值可以超过 1.0（比如 1.5~3.5），多个粒子叠加时像素累积亮度可以非常高。

最终输出时，用 **Reinhard 色调映射**将 HDR 值映射回 [0, 1]：

```
v_out = v_hdr / (1 + v_hdr)
```

**公式直觉**：

- 当 `v_hdr` 很小时，`v_out ≈ v_hdr`（暗部保真）
- 当 `v_hdr → ∞` 时，`v_out → 1`（亮部不会硬截断，而是平滑压缩）
- `v_hdr = 1` 时，`v_out = 0.5`（中灰点对应 HDR 亮度 1.0）

为什么不直接 clamp？Clamp 会让多个粒子重叠的核心区域变成一片死白，完全丢失层次感。Reinhard 让核心保持高亮，但仍有细节。

### Gamma 校正

色调映射后还需要 Gamma 校正：

```
v_gamma = pow(v_linear, 1/2.2)
```

显示器的物理响应曲线是幂函数（gamma ≈ 2.2），即显示器输入信号 0.5 对应的实际亮度约为 0.5^2.2 ≈ 0.218（而非 0.5）。

不做 Gamma 校正，图像会整体偏暗，暗部细节丢失。

### 软光栅 Splat 绘制

每个粒子不是一个像素点，而是一个带 Gaussian 衰减的圆形"笔触"（Splat）：

```
for each pixel (x, y) in radius:
    dist = sqrt((fx-x)² + (fy-y)²)
    if dist < radius:
        falloff = (1 - dist/radius)²
        pixel[x][y] += color * falloff
```

`falloff` 的二次衰减让粒子中心最亮，边缘平滑过渡到 0，避免硬边锯齿。

**为什么用加法混合（Additive Blending）？**

烟花粒子是自发光体，多个粒子重叠时亮度应该叠加，而不是覆盖。加法混合天然满足这一点：粒子密集的核心区域亮度高，稀疏的边缘暗，完美模拟真实烟花的发光特性。

### HSV 颜色空间

粒子颜色在 HSV（Hue-Saturation-Value）空间生成，然后转换到 RGB 输出。

为什么用 HSV？直接在 RGB 空间选颜色，很难保证"同一次爆炸的粒子颜色和谐"。HSV 中，固定 Hue（色相），随机 Saturation 和 Value，天然得到同色调的颜色变体，就像烟花同一朵的粒子颜色相近但有细微变化。

HSV → RGB 转换公式：

```
h' = floor(H/60) mod 6
f  = H/60 - floor(H/60)
p  = V*(1-S), q = V*(1-f*S), t = V*(1-(1-f)*S)

h'=0: (V,t,p), h'=1: (q,V,p), h'=2: (p,V,t)
h'=3: (p,q,V), h'=4: (t,p,V), h'=5: (V,p,q)
```

---

## ③ 实现架构

整个渲染器分为以下几个模块：

```
main()
  ├── Image（帧缓冲 + PNG 写入）
  │     ├── pixels[y*w+x]  — HDR 颜色累积
  │     ├── splat()        — 软光栅笔刷绘制
  │     └── savePNG()      — Reinhard+Gamma → 8bit → PNG
  │
  ├── Particle（粒子状态）
  │     ├── pos, vel       — 位置速度
  │     ├── color, life    — 颜色和寿命
  │     └── decay, size    — 衰减率和尺寸
  │
  ├── Explosion（爆炸参数）
  │     ├── center, numParticles
  │     ├── hue, speed
  │     └── gravity, drag
  │
  ├── spawnExplosion()     — 生成一组粒子
  └── simulate()           — 物理步进 + 周期性渲染到 Image
```

**数据流**：

1. 定义 5 个 `Explosion` 描述符（中心位置、粒子数、颜色色相、初速度等）
2. 对每个爆炸，`spawnExplosion()` 生成粒子数组
3. `simulate()` 循环：
   - 每 `renderEvery` 步将当前所有粒子 splat 到帧缓冲（时序叠加）
   - 每步更新粒子的速度（重力+阻力）和位置，减少寿命
4. 所有爆炸处理完后，`savePNG()` 将 HDR 帧缓冲色调映射后写入 PNG

**时序帧合成的关键**：

`simulate()` 不是渲染最终帧，而是在模拟过程中多次渲染，把粒子轨迹叠加到同一张图。这样最终图像同时显示了粒子爆炸的整个过程——相当于一张长曝光照片，捕捉了烟花从中心绽放到粒子飘散的完整轨迹。

**CPU 侧职责** vs **可 GPU 化的部分**：

| 当前（CPU）| 可 GPU 化 |
|-----------|----------|
| 粒子物理更新（串行循环）| Compute Shader，每粒子一线程 |
| Splat 绘制（CPU 像素写入）| Fragment Shader / Raster |
| Tonemapping（串行）| Post-process Fragment Shader |
| HSV→RGB 颜色转换 | Shader 内置函数 |
| 随机数生成（mt19937）| GPU 上用 Hash 函数替代 |

本项目是软渲染实现，验证算法正确性，工业实现会全部移到 GPU。

**内存布局分析**：

每个粒子存储 `Particle` 结构体（约 48 字节）：

```
pos  (2 float = 8 bytes)
vel  (2 float = 8 bytes)
color(3 float = 12 bytes)
life (1 float = 4 bytes)
decay(1 float = 4 bytes)
size (1 float = 4 bytes)
+ padding ≈ 8 bytes
```

5 组爆炸，每组约 500 粒子，总计约 2500 粒子 × 48 bytes ≈ 120KB，远在 CPU 缓存可容纳范围内，无需担心内存瓶颈。

帧缓冲（`Image::pixels`）大小：800 × 600 × 3 float = 5.76MB，稍大于 L3 Cache 典型大小（3~8MB），这意味着 splat 的随机写入存在 cache miss。GPU VRAM 没有这个问题（VRAM 带宽远高于系统内存）。

---

## ④ 关键代码解析

### 图像的 Splat 函数

```cpp
void splat(float fx, float fy, const Color& c, float radius = 1.5f) {
    int x0 = static_cast<int>(fx - radius);
    int x1 = static_cast<int>(fx + radius) + 1;
    int y0 = static_cast<int>(fy - radius);
    int y1 = static_cast<int>(fy + radius) + 1;
    for (int y = y0; y <= y1; ++y) {
        for (int x = x0; x <= x1; ++x) {
            float dx = fx - x, dy = fy - y;
            float dist = std::sqrt(dx * dx + dy * dy);
            if (dist < radius) {
                float falloff = 1.0f - dist / radius;
                addPixel(x, y, c * (falloff * falloff));
            }
        }
    }
}
```

**关键设计**：
- `radius` 控制笔刷大小，粒子的 `size * life` 动态缩小，模拟粒子随生命消亡而变小
- `falloff * falloff` 是二次衰减，比线性更平滑（边缘梯度更缓）
- `addPixel()` 而不是 `setPixel()`：加法混合，HDR 累积
- 边界检查放在 `addPixel()` 内部，splat 不做越界判断，简洁但需确保 addPixel 安全

### 粒子生成（spawnExplosion）

```cpp
std::vector<Particle> spawnExplosion(const Explosion& e) {
    std::vector<Particle> ps;
    ps.reserve(e.numParticles);

    for (int i = 0; i < e.numParticles; ++i) {
        float angle = randf(0.0f, 2.0f * 3.14159265f);  // 全向随机角度
        float speed = randf(0.3f, 1.0f) * e.speed;       // 速度随机缩放

        // 速度随机：0.3~1.0 倍，而不是固定速度
        // 为什么？固定速度让所有粒子同时到达同一个圆弧，
        // 看起来像"气球膨胀"而不是爆炸。速度随机才有"炸开"的感觉。

        float decay = randf(0.008f, 0.018f);  // 寿命衰减率随机
        float hue   = e.hue + randf(-20.0f, 20.0f);  // ±20° 色相抖动
        float sat   = randf(0.7f, 1.0f);
        float val   = randf(1.5f, 3.5f);  // HDR 亮度 > 1.0

        Particle p;
        p.pos   = e.center;
        p.vel   = {std::cos(angle) * speed, std::sin(angle) * speed};
        p.color = hsvToRgb(hue, sat, val);
        p.life  = 1.0f;
        p.decay = decay;
        p.size  = randf(1.0f, 2.5f);
        ps.push_back(p);
    }
    return ps;
}
```

**关键参数解读**：

`val = randf(1.5f, 3.5f)` — 为什么 HDR 亮度设为 1.5~3.5？

在 Reinhard 映射下，`v = 1.5` 映射到 `1.5/2.5 = 0.6`，`v = 3.5` 映射到 `3.5/4.5 ≈ 0.78`。设置 HDR 亮度是为了让粒子在叠加后（中心区域可能累积 10+ 个粒子的亮度），通过 Reinhard 压缩得到高亮但不死白的核心。

`decay = randf(0.008f, 0.018f)` — 粒子寿命在 `1/0.018 ≈ 56` 到 `1/0.008 ≈ 125` 步之间消亡。模拟步数是 200，所以有些粒子很早消失，有些能存在较长时间，增加轨迹的层次感。

### 物理步进与时序渲染（simulate）

```cpp
static void simulate(std::vector<Particle>& particles,
                     Image& img,
                     int steps, float dt, int renderEvery,
                     float gravity, float drag)
{
    for (int step = 0; step < steps; ++step) {
        // 每 renderEvery 步渲染一次
        if (step % renderEvery == 0) {
            for (const auto& p : particles) {
                if (p.life <= 0) continue;
                Color c = p.color * (p.life * p.life);  // 平方寿命衰减
                img.splat(p.pos.x, p.pos.y, c, p.size * p.life);
            }
        }
        // 物理步进
        for (auto& p : particles) {
            if (p.life <= 0) continue;
            p.vel.y += gravity * dt;   // 重力加速
            p.vel.x *= drag;           // X 方向阻力
            p.vel.y *= drag;           // Y 方向阻力
            p.pos.x += p.vel.x * dt;
            p.pos.y += p.vel.y * dt;
            p.life  -= p.decay;
        }
    }
}
```

**renderEvery = 4** 的意义：每 4 步渲染一次（共 200 步 / 4 = 50 次快照叠加）。这相当于快门开启了 50 个瞬间，把 50 张快照叠在一起。`renderEvery` 越小，轨迹越密，图像越亮；越大，轨迹越稀，更接近运动模糊效果。

**为什么先渲染再步进？**

若先步进再渲染，则第 0 步（爆炸瞬间）不会被渲染到，丢失了核心爆炸点的高亮。先渲染确保捕捉到粒子从出生点开始的完整轨迹。

### PNG 写入（无外部库的纯 C++ 实现）

```cpp
static std::vector<uint8_t> zlibStore(const std::vector<uint8_t>& raw) {
    std::vector<uint8_t> out;
    out.push_back(0x78); // CMF: deflate method, 32KB window
    out.push_back(0x01); // FLG: no dict, checksum

    size_t pos = 0, remaining = raw.size();
    while (remaining > 0) {
        size_t block = std::min(remaining, (size_t)65535);
        bool last = (block == remaining);
        out.push_back(last ? 0x01 : 0x00); // BFINAL=1 if last, BTYPE=00 (store)
        uint16_t len16 = (uint16_t)block;
        uint16_t nlen  = ~len16;
        // LEN + NLEN (little-endian)
        out.push_back(len16 & 0xFF); out.push_back(len16 >> 8);
        out.push_back(nlen  & 0xFF); out.push_back(nlen  >> 8);
        for (size_t i = 0; i < block; ++i) out.push_back(raw[pos + i]);
        pos += block; remaining -= block;
    }
    // Adler-32 checksum
    uint32_t s1 = 1, s2 = 0;
    for (uint8_t b : raw) { s1 = (s1+b)%65521; s2 = (s2+s1)%65521; }
    uint32_t adler = (s2 << 16) | s1;
    out.push_back((adler>>24)&0xFF); out.push_back((adler>>16)&0xFF);
    out.push_back((adler>>8)&0xFF);  out.push_back(adler&0xFF);
    return out;
}
```

**PNG 格式关键要点**：

PNG 的图像数据（IDAT chunk）使用 zlib 压缩。这里使用的是 DEFLATE Store（无压缩，BTYPE=00），即原样存储数据，用 zlib 头/尾包裹。好处是实现极简，无需压缩算法，坏处是文件较大（1.4MB，真实 DEFLATE 可压缩到数百 KB）。

每行图像数据前有一个 Filter byte（固定为 0 = None），PNG 规范要求每行开头有 filter 字节指明行过滤方式。

CRC32 和 Adler-32 都需要手动计算：

- **CRC32**：用于每个 PNG chunk 的完整性校验
- **Adler-32**：用于 zlib 数据流的完整性校验，比 CRC32 计算更快

### 色调映射与 Gamma 校正

```cpp
auto tonemapChannel = [](float v) -> uint8_t {
    // Reinhard tone mapping
    v = v / (1.0f + v);
    // Clamp to [0,1]
    v = std::max(0.0f, std::min(1.0f, v));
    // Gamma 2.2 correction
    v = std::pow(v, 1.0f / 2.2f);
    return static_cast<uint8_t>(v * 255.0f + 0.5f);
};
```

注意 `+ 0.5f` 的四舍五入。直接截断（`(uint8_t)(v * 255.0f)`）对于中间值有系统性偏暗 0.5 的误差，四舍五入消除偏差。

---

## ⑤ 踩坑实录

### Bug 1：PNG 文件无法被任何查看器打开

**症状**：生成的 .png 文件大小非零，但 Preview、浏览器、PIL 都报"无效图片文件"。

**错误假设**：以为是颜色数据写入顺序有问题（RGB/BGR）。

**真实原因**：zlib NLEN 字段写入时用了大端序（big-endian），但 DEFLATE 规范要求 LEN 和 NLEN 都是**小端序**（little-endian）。PNG 文件格式中 zlib 的 DEFLATE 块头：

```
BFINAL+BTYPE | LEN_lo | LEN_hi | NLEN_lo | NLEN_hi
```

初版代码写的是：
```cpp
out.push_back((len16 >> 8) & 0xFF);  // 错！大端
out.push_back( len16       & 0xFF);
```

修复为：
```cpp
out.push_back( len16       & 0xFF);  // 小端，低字节在前
out.push_back((len16 >> 8) & 0xFF);
```

**教训**：PNG/zlib 规范中，chunk 长度、CRC、Adler-32 用大端序，但 DEFLATE 内部的 LEN/NLEN 字段用小端序。两种字节序混用，不仔细读 RFC 1950/1951 容易踩坑。

### Bug 2：图像整体偏暗，所有爆炸核心都是灰的

**症状**：运行成功，图像生成，但视觉效果像是把亮度调低了 4 倍，没有烟花的发光感。

**错误假设**：以为是颜色混合 bug，double-check 了 splat 函数。

**真实原因**：漏掉了 Gamma 校正。Reinhard 映射后的值仍是线性光照空间的值，直接写入 PNG（假定是 sRGB 空间）会导致图像偏暗。添加 `pow(v, 1/2.2)` 后，暗部被提亮，图像恢复正常的视觉亮度。

**修复**：在 tonemapChannel 中加入 Gamma 校正步骤（见上节代码）。

**教训**：渲染管线的最后一步"线性空间 → sRGB"经常被忘记，表现为"图像明明有内容但看起来很暗"。

### Bug 3：粒子核心全白，没有层次感

**症状**：爆炸核心是一片死白，周围粒子正常，没有发光中心渐变到暗边的层次感。

**错误假设**：以为是 HDR 值太高导致 clamp。

**真实原因**：初版 tonemapChannel 用的是简单 clamp（`min(v, 1.0f)`），而不是 Reinhard。多个粒子叠加后核心亮度可达 15.0+，clamp 直接截断为 1.0，没有层次。

**修复**：将 `v = min(v, 1.0f)` 替换为 `v = v/(1+v)`（Reinhard），核心亮度被平滑压缩，层次感恢复。

### Bug 4：烟花发射轨迹看起来太直太硬

**症状**：上升轨迹是完美的直线，和爆炸后的粒子风格不一致，显得突兀。

**错误假设**：以为需要给轨迹加曲线。

**真实原因**：发射轨迹只是一条直线插值（`lerp(start, end, t)`），没有亮度变化，感觉是"画上去"而非"发射出去"。

**修复**：给轨迹粒子加亮度渐变（接近爆炸点越亮），并用较小 radius 绘制，模拟细线发射轨迹：

```cpp
float brightness = 0.3f + 0.7f * t;  // 越接近爆炸点越亮
Color c = hsvToRgb(e.hue, 0.5f, brightness * 1.5f);
img.splat(x, y, c * 0.5f, 1.0f);    // radius=1.0, 细线
```

---

## ⑥ 效果验证与数据

### 量化验证结果

```
文件大小: 1.4MB（800×600 PNG，无压缩 zlib store）
像素均值: 13.4（在 10~240 区间内 ✅）
像素标准差: 38.7（远大于 5，图像内容丰富 ✅）
```

标准差 38.7 意味着图像中明暗对比显著——黑色背景（值 0）和烟花轨迹（值 100+）之间有大范围的亮度变化。

### 渲染结果

![烟花粒子系统渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/04/04-08-Particle-System-Simulation/particle_output.png)

图像包含 5 组烟花：
- **中央金色**：最大组（600 粒子），视觉中心
- **左上蓝色**：500 粒子，冷色调对比
- **右上青色**：500 粒子
- **左下红色**：400 粒子
- **右下绿色**：450 粒子

每组烟花下方有发射上升轨迹，整体呈现"多颗烟花同时爆炸"的视觉效果。

### 编译性能

```bash
g++ main.cpp -o output -std=c++17 -O2 -Wall -Wextra
# 编译时间: ~0.8 秒
# 运行时间: ~1.2 秒（800×600，5组烟花×~500粒子，200步/组）
```

主要计算瓶颈在 `splat()` 的像素写入——每个粒子需要遍历约 `(2r+1)² ≈ 25` 个像素，总计约 `5 * 500 * 50步 * 25像素 ≈ 3,125,000` 次像素写入。单线程 CPU 在 1 秒内完成，完全可接受。

### 与无 Gamma 校正的对比

没有 Gamma 校正时，`v = 0.5`（中等亮度）显示为 `0.5 * 255 ≈ 127`，但人眼感知的亮度对应线性值 `0.5^2.2 ≈ 0.218`，显示出来偏暗。加了 `pow(v, 1/2.2)` 后，中等亮度对应输出值 `pow(0.5, 1/2.2) * 255 ≈ 188`，图像亮度视觉正常。

---

## ⑦ 总结与延伸

### 本项目实现了什么

一个完整的 CPU 软渲染烟花粒子系统，覆盖了工业粒子系统的核心要素：
- 粒子发射（全向随机速度分布）
- 粒子物理（重力 + 阻力，Semi-implicit Euler 积分）
- 生命周期管理（平方寿命衰减）
- 软光栅绘制（Gaussian falloff Splat）
- HDR 渲染 + Reinhard 色调映射 + Gamma 校正
- 时序帧合成（轨迹叠加，等效长曝光）
- 纯 C++ 手写 PNG 编码（含 CRC32、Adler-32、zlib store）

### 局限性

1. **2D 模拟**：真实烟花是 3D 的，2D 爆炸看起来是一个圆，而真实烟花是球形爆炸，从侧面看是圆，但视角变化时形态不同
2. **无碰撞检测**：粒子可以飞出屏幕边界，也没有与地面/障碍物的交互
3. **无 GPU 加速**：百万级粒子需要 Compute Shader
4. **无 Alpha 混合**：只有加法混合，无法实现烟雾（半透明叠加）
5. **无温度模型**：真实烟花颜色随温度变化（高温白→低温橙→暗红），当前用固定 HDR 亮度

### 可优化的方向

1. **GPU 粒子**：将 simulate() 移到 Compute Shader，百万粒子实时
2. **粒子系统参数化 DSL**：用 JSON/YAML 描述发射器，支持运行时热更新
3. **3D 转 2D 投影**：增加摄像机和透视投影，更真实的烟花视觉
4. **烟雾粒子**：加半透明 Alpha，模拟爆炸后的烟雾扩散
5. **音效同步**：结合 SDL 音频，爆炸点触发音效

### 与本系列其他文章的关联

- **Day 44（Motion Blur）**：运动模糊也使用了时序帧叠加的思路，本项目的"多时间步渲染合成"是更通用的版本
- **Day 43（Depth of Field）**：DOF 的 Bokeh 散斑绘制本质也是软光栅 Splat，与本项目的粒子绘制方法同源
- **Day 38（SSAO）**：SSAO 的半球采样随机性与本项目的粒子速度随机化思路类似，都是重要性采样的不同应用

---

**代码仓库**：[https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-04-08](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-04-08)
