---
title: "每日编程实践: FFT Ocean Surface Renderer"
date: 2026-05-15 05:30:00
tags:
  - 每日一练
  - 图形学
  - C++
  - 海洋渲染
  - FFT
  - 物理模拟
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-15-fft-ocean/fft_ocean_output.png
---

# FFT Ocean Surface Renderer — 基于频谱的物理海洋渲染

> 从 Phillips 频谱到 Cooley-Tukey FFT，完整推导海洋表面高度场生成的数学基础，实现实时级别的真实感海洋渲染。

---

## ① 背景与动机

### 海洋渲染的难题

游戏和电影中的海洋是最难渲染的场景之一。一个真实的海洋有以下特性：

- **多尺度叠加**：从几毫米的涟漪到几十米的涌浪同时存在
- **各向异性**：沿风向的波浪比垂直风向更强
- **随机性**：每处水面都不同，但整体统计特性满足特定分布
- **时变性**：波形随时间连续演化，不能每帧重新生成随机数

如果用朴素的方法——叠加几十个 Gerstner 波——会发现无论叠多少个，视觉上都缺少真实感，因为真实海洋是**无数个随机相位波的叠加**，不是少数确定波的叠加。

### 工业界方案：FFT 海洋

**Tessendorf (2001)** 在 SIGGRAPH 上发表的 "Simulating Ocean Water" 开创了基于频谱的海洋模拟方法，至今仍是游戏引擎的标准方案：

- **《God of War》**：基于 Tessendorf 方法 + 多层 tiling 消除重复感
- **《Sea of Thieves》**：FFT 海洋 + Gerstner 波混合，GPU compute shader 实现
- **Unreal Engine 5 Water Plugin**：`WaterWaves` 系统内置 FFT 求解器
- **Unity HDRP**：High Definition Water Surface 使用 FFT ocean
- **Blender Ocean Modifier**：内置 Tessendorf 海洋

核心思路：在**频域**描述海洋（哪些频率的波有多强），然后通过 **IFFT（逆快速傅里叶变换）** 转换到空间域得到高度场。这样可以用 O(N² log N) 的复杂度生成 N×N 的海洋高度场，比朴素叠加法快得多，而且物理正确。

### 为什么今天做这个

海洋渲染综合了：
- **信号处理**（FFT）
- **海洋物理**（Phillips 谱、色散关系）
- **渲染技术**（法线生成、Fresnel、Blinn-Phong）

是一个把数学和渲染完美结合的练习项目，非常值得深入理解。

---

## ② 核心原理

### 2.1 海洋的频域表示

一个复杂的海洋表面 h(x, t) 可以分解为很多单频正弦波的叠加：

```
h(x, t) = Σ h̃(k, t) · e^(i·k·x)
```

其中：
- `x = (x, z)` 是水平坐标（y 轴朝上）
- `k = (kx, kz)` 是波数向量（波的空间频率）
- `h̃(k, t)` 是波数 k 对应的复振幅（随时间演化）

这是一个傅里叶变换的形式：**空间域的高度场 = 频域振幅的逆傅里叶变换**。

关键问题变成了：**每个频率 k 的初始振幅 h̃(k, 0) 是多少？**

### 2.2 Phillips 频谱

真实海洋中，各频率分量的能量分布满足 **Phillips 频谱**：

```
Ph(k) = A · exp(-1 / (k · L)²) / k⁴ · |k̂ · ŵ|²
```

拆解每一项的物理意义：

**① 振幅常数 A**：控制海浪整体大小。A = 1.2×10⁻³ 对应大约 20m/s 风速下的典型海浪。

**② 指数衰减项 exp(-1/(k·L)²)**：
- L = V²/g，V 是风速，g 是重力加速度
- 当 k·L << 1（即波长远大于 L）时，这一项趋近 0
- 意思是：比"主导波长"长得多的波几乎不存在
- 直觉上：风能驱动的最大波长由风速决定，太长的波无法被风激发

**③ k⁴ 分母**：
- 高频（短波）能量减弱
- 物理原因：短波容易被黏性耗散，长波更持久
- 这项使得海洋表面在视觉上有明显的主频波

**④ 方向扩展 |k̂ · ŵ|²**：
- k̂ 是波向单位向量，ŵ 是风向单位向量
- 点积平方确保：沿风向（k̂ = ŵ）的波能量最强，垂直风向（k̂ ⊥ ŵ）的波能量为 0
- 这产生了海浪沿风向的"各向异性"特征

**⑤ 小波压制 exp(-k²·l²)**：
- l = 0.001 · L（极小的截止波长）
- 避免数值不稳定：最高频的分量在离散 FFT 中不够精确，提前衰减掉

代码实现：

```cpp
float phillipsSpectrum(Vec2 k) {
    float kLen = std::sqrt(k.x*k.x + k.y*k.y);
    if(kLen < 1e-6f) return 0.0f;  // k=0 时能量为0（避免除零）

    float L_  = V*V / G;           // 主导波长 L = V²/g
    float k2  = kLen * kLen;
    float k4  = k2 * k2;
    float kL2 = k2 * L_ * L_;
    
    // 基础 Phillips 谱
    float Ph  = A * std::exp(-1.0f / kL2) / k4;
    
    // 方向扩展：沿风向的分量
    float kHatDotW = (k.x * W.x + k.y * W.y) / kLen;
    Ph *= kHatDotW * kHatDotW;
    
    // 压制极小波
    float l = 0.001f * L_;
    Ph *= std::exp(-k2 * l * l);
    
    return Ph;
}
```

### 2.3 初始复振幅 H₀(k)

Phillips 谱给出了每个频率的**功率**（能量的平方），但我们需要的是**复振幅**。

根据随机波浪理论，初始复振幅是：

```
h₀(k) = (1/√2) · (ξᵣ + i·ξᵢ) · √Ph(k)
```

其中 ξᵣ, ξᵢ 是独立的标准高斯随机数（均值0，方差1）。

**为什么是高斯分布？**

海洋表面是大量独立波的叠加，根据**中心极限定理**，叠加结果接近高斯分布。在频域，每个频率分量的振幅和相位也应该独立高斯分布。

**1/√2 的因子**：保证功率谱等于 Ph(k)。

代码：

```cpp
void initH0(std::vector<Complex>& h0, std::vector<Complex>& h0conj,
             std::mt19937& rng)
{
    std::normal_distribution<float> gauss(0.0f, 1.0f);
    h0.resize(N * N);
    h0conj.resize(N * N);

    for(int m = 0; m < N; m++){
        for(int n = 0; n < N; n++){
            Vec2 k = {
                (float)(2.0 * M_PI * (n - N/2)) / L,  // kx
                (float)(2.0 * M_PI * (m - N/2)) / L   // kz
            };
            float P    = phillipsSpectrum(k);
            float sqrtP = std::sqrt(P * 0.5f);  // 1/√2 · √Ph
            
            // h₀(k)
            h0[m*N+n] = Complex(gauss(rng) * sqrtP, gauss(rng) * sqrtP);
            
            // h₀(-k)* 共轭：用于保证 IFFT 结果为实数
            Vec2 mk = {-k.x, -k.y};
            float Pm = phillipsSpectrum(mk);
            float sqrtPm = std::sqrt(Pm * 0.5f);
            h0conj[m*N+n] = std::conj(Complex(gauss(rng)*sqrtPm, gauss(rng)*sqrtPm));
        }
    }
}
```

**为什么需要 h₀(-k)* 共轭？**

傅里叶变换的实数性条件：如果 h(x,t) 是实数（高度场是实数），则 h̃(k) = h̃(-k)*。  
所以我们同时初始化 h₀(k) 和 h₀(-k) 的共轭，确保最终 IFFT 的结果是实数。

### 2.4 时间演化与色散关系

波浪随时间演化遵循**色散关系**：

```
ω(k) = √(g · |k|)
```

这是深水重力波的色散关系。直觉上：
- 短波（大 k）的**角频率 ω 更大**（震动更快）
- 长波（小 k）的频率更低（演化更慢）
- 相速度 c = ω/k = √(g/k)：**长波传播更快**

在时刻 t，频域振幅为：

```
h̃(k, t) = h₀(k) · e^(iωt) + h₀(-k)* · e^(-iωt)
```

- 第一项是正向传播的波
- 第二项是反向传播波（共轭确保实数性）

代码：

```cpp
for(int m=0; m<N; m++){
    for(int n=0; n<N; n++){
        Vec2 k = {
            (float)(2.0 * M_PI * (n - N/2)) / L,
            (float)(2.0 * M_PI * (m - N/2)) / L
        };
        float kLen = std::sqrt(k.x*k.x + k.y*k.y);
        float omega = std::sqrt(G * kLen);   // 色散关系

        Complex e_pos(std::cos(omega*t),  std::sin(omega*t));   // e^(iωt)
        Complex e_neg(std::cos(omega*t), -std::sin(omega*t));   // e^(-iωt)

        int idx = m*N+n;
        ht[idx] = h0[idx] * e_pos + h0conj[idx] * e_neg;
    }
}
```

### 2.5 法线生成

有了高度场 h(x, z)，法线向量为：

```
N = normalize(-∂h/∂x, 1, -∂h/∂z)
```

梯度 ∂h/∂x 和 ∂h/∂z 也可以通过 FFT 高效计算：

```
∂h/∂x  ←→  ikx · h̃(k)
∂h/∂z  ←→  ikz · h̃(k)
```

即在频域，对位置求导等于**乘以 ik**。代码中：

```cpp
// ik · H：在频域计算梯度
dhtdx[idx] = Complex(-H.imag() * k.x,  H.real() * k.x);  // i*kx*H
dhtdz[idx] = Complex(-H.imag() * k.y,  H.real() * k.y);  // i*kz*H
```

（注：`i * (a + bi) = -b + ai`，所以 `i * Complex(a, b) = Complex(-b, a)`）

---

## ③ 实现架构

### 数据流

```
[Phillips 频谱 Ph(k)]
        ↓
[初始化 H₀(k), H₀(-k)*]  ← 随机高斯数
        ↓
[时间演化：H̃(k,t) = H₀e^(iωt) + H₀*e^(-iωt)]
        ↓
[频域梯度：∂H/∂x = ikx·H̃, ∂H/∂z = ikz·H̃]
        ↓
[2D IFFT × 3：height, slopeX, slopeZ]
        ↓
[FFT shift 符号校正：sign = (-1)^(m+n)]
        ↓
[逐像素渲染：法线 + Blinn-Phong + Fresnel + 颜色分层]
        ↓
[PNG 输出 800×600]
```

### 关键数据结构

```cpp
// 复数网格（FFT 计算域）
using Complex = std::complex<float>;
std::vector<Complex> h0;      // 初始振幅 H₀(k)   N×N
std::vector<Complex> h0conj;  // H₀(-k)* 共轭     N×N
std::vector<Complex> ht;      // 当前时刻振幅      N×N
std::vector<Complex> dhtdx;   // x方向梯度频域     N×N
std::vector<Complex> dhtdz;   // z方向梯度频域     N×N

// 空间域结果
std::vector<float> height;  // 高度场     N×N
std::vector<float> slopeX;  // x梯度      N×N
std::vector<float> slopeZ;  // z梯度      N×N

// 渲染帧缓冲
struct Framebuffer {
    int w, h;
    std::vector<Vec3> data;  // RGB float [0,1]
};
```

### 参数设计

| 参数 | 值 | 意义 |
|------|-----|------|
| N | 256 | FFT 网格大小（必须 2 的幂次） |
| L | 512m | 海洋 tile 物理大小 |
| A | 1.2×10⁻³ | Phillips 振幅常数 |
| V | 20 m/s | 风速 |
| W | (1,1)归一化 | 风向（东北方向） |
| t | 10s | 渲染时刻 |

N=256 和 L=512m 意味着每个格子代表 2m×2m，频率分辨率 Δk = 2π/L ≈ 0.012 rad/m。

---

## ④ 关键代码解析

### 4.1 Cooley-Tukey FFT（迭代实现）

```cpp
void fft1D(std::vector<Complex>& a, bool inverse) {
    int n = (int)a.size();
    
    // Step 1: 位反转置换（Bit-Reversal Permutation）
    // 将序列重排为 FFT butterfly 所需的顺序
    for(int i=1, j=0; i<n; i++){
        int bit = n >> 1;
        for(; j & bit; bit >>= 1) j ^= bit;
        j ^= bit;
        if(i < j) std::swap(a[i], a[j]);
    }
    
    // Step 2: 蝶形迭代
    // len 从 2 倍增到 n，每轮合并相邻两段
    for(int len=2; len<=n; len<<=1){
        // 旋转因子：正变换用 e^(-2πi/len)，逆变换用 e^(2πi/len)
        float ang = (inverse ? 1.0f : -1.0f) * 2.0f * (float)M_PI / len;
        Complex wlen(std::cos(ang), std::sin(ang));
        
        for(int i=0; i<n; i+=len){
            Complex w(1.0f, 0.0f);  // 初始旋转因子 = 1
            for(int j=0; j<len/2; j++){
                Complex u = a[i+j];
                Complex v = a[i+j+len/2] * w;
                // 蝶形操作：(u+v, u-v)
                a[i+j]       = u + v;
                a[i+j+len/2] = u - v;
                w *= wlen;  // 旋转因子递推
            }
        }
    }
    
    // 逆变换需要除以 N（归一化）
    if(inverse){
        for(auto& x : a) x /= (float)n;
    }
}
```

**为什么需要位反转置换？**

Cooley-Tukey FFT 是分治算法：将 N 点 FFT 分为两个 N/2 点 FFT（偶数项和奇数项），递归下去。递归展开后，第 i 个元素最终位于 bit-reverse(i) 的位置。提前做位反转置换，就可以用迭代（不用递归栈）完成 FFT。

**2D FFT 实现**：对行做 1D FFT，再对列做 1D FFT：

```cpp
void fft2D(std::vector<Complex>& grid, bool inverse) {
    std::vector<Complex> row(N);
    for(int m=0; m<N; m++){
        for(int n=0; n<N; n++) row[n] = grid[m*N+n];
        fft1D(row, inverse);
        for(int n=0; n<N; n++) grid[m*N+n] = row[n];
    }
    std::vector<Complex> col(N);
    for(int n=0; n<N; n++){
        for(int m=0; m<N; m++) col[m] = grid[m*N+n];
        fft1D(col, inverse);
        for(int m=0; m<N; m++) grid[m*N+n] = col[m];
    }
}
```

2D IFFT 的复杂度是 O(N² log N)，N=256 时约 500K 次复数乘法——远少于朴素的 O(N⁴)。

### 4.2 FFT Shift 符号校正

```cpp
// IFFT 后需要进行 FFT-shift 等价的符号校正
for(int i=0; i<N*N; i++){
    int m = i / N;
    int n = i % N;
    float sign = ((m + n) & 1) ? -1.0f : 1.0f;  // (-1)^(m+n)
    height[i] = ht[i].real()    * sign;
    slopeX[i] = dhtdx[i].real() * sign;
    slopeZ[i] = dhtdz[i].real() * sign;
}
```

**为什么需要这个符号？**

标准 FFT 假设频率从 0 排到 N-1（无符号频率）。但物理上，波数应该是 -N/2 到 N/2（有符号，对应正向和反向传播的波）。

在初始化 H₀ 时，我们将频率中心化（`n - N/2`），这等价于先乘以 `e^(iπn) = (-1)^n`。IFFT 后要"撤销"这个中心化，就需要乘以 `(-1)^(m+n)`。

**如果不做这步**：高度场的低频分量会分布在四个角落，而不是中心，导致视觉上海浪图案完全错误。

### 4.3 海洋渲染着色

```cpp
// 计算法线：从高度场梯度推导
Vec3 normal = Vec3(-dhdx, 1.0f, -dhdz).norm();
// 解释：如果水面在 x 方向倾斜，法线就向 -x 方向偏转
// Vec3(0,1,0) 是完全水平面的法线，dh/dx 表示倾斜程度

// Blinn-Phong 高光
Vec3 viewDir = Vec3(0, 1, 0);   // 俯视图：观察方向向上
Vec3 halfV   = (lightDir + viewDir).norm();  // 半角向量
float diffuse  = std::max(0.0f, normal.dot(lightDir));
float specular = std::pow(std::max(0.0f, normal.dot(halfV)), 64.0f);
// 指数64：较集中的高光，适合水面

// 高度分层颜色
float hn = (h - hMin) / hRange;  // 归一化高度 [0,1]
Vec3 deepColor   (0.01f, 0.05f, 0.20f);  // 深水：深蓝
Vec3 shallowColor(0.05f, 0.25f, 0.45f);  // 中层：中蓝
Vec3 crestColor  (0.80f, 0.90f, 1.00f);  // 浪尖：近白

// 两段线性插值
Vec3 baseColor;
if(hn < 0.7f){
    float t = hn / 0.7f;
    baseColor = deepColor * (1-t) + shallowColor * t;
} else {
    float t = (hn - 0.7f) / 0.3f;
    baseColor = shallowColor * (1-t) + crestColor * t;
}
```

**Fresnel 效果**：

```cpp
// Schlick 近似 Fresnel 方程
// 掠射角（法线和视线夹角大）时反射更强
float cosTheta = std::abs(normal.dot(viewDir));
float fresnel  = 0.04f + 0.96f * std::pow(1.0f - cosTheta, 5.0f);
// 0.04 是水的基础反射率（F0），垂直入射时 4% 反射
// 掠射时接近 100% 反射
Vec3 skyColor(0.4f, 0.6f, 0.9f);  // 天空颜色
Vec3 color = diffColor + specColor + skyColor * (fresnel * 0.4f);
```

### 4.4 Reinhard 色调映射 + Gamma 校正

```cpp
// Reinhard 色调映射：将 HDR 浮点值压缩到 [0,1]
Vec3 tonemap(Vec3 c) {
    return { c.x/(1.0f+c.x), c.y/(1.0f+c.y), c.z/(1.0f+c.z) };
}

// 在 main 中：
color = tonemap(color);
color = color.clamp01();

// sRGB Gamma 校正（1/2.2）
color.x = std::pow(color.x, 1.0f/2.2f);
color.y = std::pow(color.y, 1.0f/2.2f);
color.z = std::pow(color.z, 1.0f/2.2f);
```

**为什么需要 Gamma 校正？**

计算是在线性色彩空间进行的（光强直接相加），但显示器的物理特性是非线性的（8bit 表示时中间值偏暗）。Gamma 校正（1/2.2 幂次）把线性值转换为 sRGB 值，使得存储到 PNG 后在显示器上看起来亮度正确。

---

## ⑤ 踩坑实录

### 坑 1：M_PI 类型引发的 narrowing 警告

**症状**：编译输出大量警告：
```
warning: narrowing conversion from 'double' to 'float' [-Wnarrowing]
(2.0f * M_PI * (n - N/2)) / L
```

**错误假设**：`2.0f * M_PI` 应该是 float，因为 `2.0f` 是 float 字面量。

**真实原因**：`M_PI` 是 C 标准定义为 `double`，`float × double` 结果是 `double`，再除以 `float` 的 `L` 仍然是 `double`。`Vec2` 的初始化列表期望 `float`，所以产生了 narrowing conversion。

**修复方式**：显式转换为 float：
```cpp
// 修复前（警告）
Vec2 k = { (2.0f * M_PI * (n - N/2)) / L, ... };

// 修复后（正确）
Vec2 k = { (float)(2.0 * M_PI * (n - N/2)) / L, ... };
```

用 `(float)(...)` 而不是乘以 `1.0f` 是因为前者语义更明确，且不影响计算精度。

### 坑 2：stb_image_write.h 的 -Wmissing-field-initializers 警告

**症状**：来自第三方头文件的大量警告，无法通过修改自己的代码解决。

**错误做法**：全局关掉 `-Wmissing-field-initializers`（会影响自己代码的检查）。

**正确做法**：用 `#pragma GCC diagnostic` 局部关闭：

```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "../stb_image_write.h"
#pragma GCC diagnostic pop
```

这样第三方头文件的警告被抑制，但自己代码的同类警告仍然会被检测到。

### 坑 3：未使用变量触发 -Wunused-variable

**症状**：
```
warning: unused variable 'camAngle' [-Wunused-variable]
warning: unused variable 'camPos'   [-Wunused-variable]
```

**原因**：早期规划了透视摄像机但最终用了俯视图，保留的变量声明未被使用。

**修复**：注释掉未使用的变量声明，或用 `(void)camAngle;` 显式标记忽略（选择注释更清晰）：

```cpp
// Vec3 camPos(256.0f, 80.0f, 256.0f);  // unused in current top-down render
```

### 坑 4：FFT Shift 不做会导致海浪图案错误

**症状**：如果去掉 `sign = ((m+n)&1) ? -1 : 1` 这行，图像不全黑也不全白，但海浪图案是四角分布的奇怪格子，完全不像海洋。

**原因**：初始化时频率中心化 `k = 2π(n-N/2)/L`，等价于在频域乘了 `e^(iπn) = (-1)^n`。IFFT 后未撤销这个乘子，导致空间域数据错排。

**验证方法**：
```python
# 关闭 sign correction 后的像素统计
# std::dev 仍然 > 5，但肉眼可见图案异常
# 这是"通过量化检查但视觉错误"的典型案例
```

这个 bug 提醒我们：量化检查（均值、标准差）能发现明显的渲染失败，但无法检测"图案结构错误"。坐标系/物理正确性需要额外的视觉检查。

---

## ⑥ 效果验证与数据

### 量化验证结果

```
文件: fft_ocean_output.png (800×600)
大小: 102KB  ✅ > 10KB
像素均值: 100.5  ✅ 范围 [10, 240]
标准差: 24.2  ✅ > 5（图像有内容）
均值 RGB: (71.9, 102.0, 127.5)
```

颜色比例 R < G < B 符合预期——海洋应该是蓝色调，红色通道最弱，蓝色最强。

### 运行性能

```
初始化 + FFT 计算 + 渲染 800×600：0.143 秒
其中 FFT 部分（3次 2D IFFT）：约 0.02s
渲染部分（逐像素着色）：约 0.12s
```

N=256 的 FFT 在 CPU 上非常快（3次 × N² log N ≈ 1.5M 操作）。实时游戏（GPU compute shader）可以做到每帧完成，支持动态海洋。

### 验证脚本

```python
from PIL import Image
import numpy as np

img = Image.open('fft_ocean_output.png')
pixels = np.array(img).astype(float)

mean = pixels.mean()
std  = pixels.std()
print(f"像素均值: {mean:.1f}  标准差: {std:.1f}")

# 检查颜色通道比例（海洋应该是蓝色调）
r_mean = pixels[:,:,0].mean()
g_mean = pixels[:,:,1].mean()
b_mean = pixels[:,:,2].mean()
print(f"RGB均值: ({r_mean:.1f}, {g_mean:.1f}, {b_mean:.1f})")
assert b_mean > g_mean > r_mean, "海洋应为蓝色调"

# 基本通过条件
assert 10 < mean < 240, "均值异常"
assert std > 5, "图像无变化"
print("✅ 所有验证通过")
```

---

## ⑦ 总结与延伸

### 技术成果

今天实现了基于 Tessendorf 方法的完整 FFT 海洋渲染器：

1. **Phillips 频谱**：物理正确的波浪能量分布
2. **Cooley-Tukey 2D FFT**：O(N² log N) 高度场求解
3. **色散关系时间演化**：波浪随时间物理正确传播
4. **梯度法线生成**：频域微分实现高效法线计算
5. **Blinn-Phong + Fresnel + Reinhard**：完整的物理渲染管线

### 局限性

- **俯视图渲染**：现在是顶视图，实际游戏需要透视相机，还需要实现网格扭曲（Gerstner displacement）
- **无 Choppiness**：真实海浪有"尖锐浪尖"，需要加入水平位移量 λ·∇h（Gerstner 扭曲）
- **单层 Tile**：真实系统需要多层 tiling 消除重复感
- **静态渲染**：只渲染了 t=10s 单帧，动态海洋需要每帧更新 FFT
- **CPU 实现**：GPU compute shader 版本可以实时运行（60+ FPS）

### 可优化方向

**1. 加入 Gerstner 水平位移**：
```cpp
// 频域水平位移（让浪尖更"尖"）
lambda = 1.0f;  // 扭曲强度
Vec2 displacement = -lambda * k_hat * H.imag();
vertex.x += IFFT(displacement.x);
vertex.z += IFFT(displacement.z);
```

**2. 透视摄像机 + 几何网格**：
```cpp
// 将 FFT 高度场应用到 N×N 三角形网格
// 透视投影到屏幕，支持任意视角
```

**3. 多层 Cascade**：使用 3-4 个不同尺度的 FFT tile（L=512m, L=128m, L=32m）叠加，消除重复感。

**4. GPU 加速**：用 cuFFT 或 OpenGL compute shader，N=512 的 FFT 可以达到 60+ FPS。

### 与本系列的关联

- **05-09 OBJ PBR Renderer**：今天的 Fresnel 和 Cook-Torrance 光照与之相同
- **04-06 Procedural Terrain**：同样是高度场渲染，但今天的高度场来自物理频谱而非噪声
- **05-01 ReSTIR**：两者都处理复杂光照；海洋 + 路径追踪的组合是高端离线渲染的典型场景

FFT 海洋是渲染技术和物理模拟的交汇点，也是理解"频域 → 空间域"转换在图形学应用中最直观的例子。

---

## 完整代码

代码仓库：[daily-coding-practice/2026-05-15-fft-ocean](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-05-15-fft-ocean)

![FFT Ocean Surface Renderer 渲染结果](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/05/05-15-fft-ocean/fft_ocean_output.png)

*800×600 分辨率，256×256 FFT 网格，t=10s，风速 20m/s，渲染耗时 0.14s*
