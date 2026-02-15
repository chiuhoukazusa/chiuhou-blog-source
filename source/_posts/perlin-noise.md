---
title: 每日编程实践：Perlin 噪声程序化纹理生成
date: 2026-02-10 10:00:00
categories: [每日编程实践]
tags: [图形学, 算法, C++, 程序化生成, 每日编程]
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/perlin_clouds.png
description: 实现经典的 Perlin 噪声算法，用于生成自然的程序化纹理。包括云层、大理石、木纹等多种效果，支持多层噪声叠加（Octave Noise）。
---

# 每日编程实践：Perlin 噪声程序化纹理生成

## 🎯 项目概述

在本项目中，我实现了一个经典的 Perlin 噪声生成器，用于创建自然的程序化纹理。Perlin 噪声由 Ken Perlin 于 1985 年发明，广泛应用于计算机图形学中的地形生成、云层模拟、水波效果等领域。

**项目时间**: 2026-02-10  
**代码语言**: C++  
**代码行数**: 230 行  
**核心算法**: Perlin Noise + Octave Noise

## Perlin 噪声原理

Perlin 噪声通过以下步骤生成：

1. **网格定义**: 将空间划分为均匀的网格
2. **梯度分配**: 为每个网格点生成随机的梯度向量
3. **平滑插值**: 使用 fade 函数平滑插值网格点间的值
4. **多层叠加**: 通过 Octave Noise 添加多层次细节

### 核心数学实现

#### Fade 函数（6t⁵ - 15t⁴ + 10t³）
```cpp
double fade(double t) {
    return t * t * t * (t * (t * 6 - 15) + 10);
}
```

这个函数确保插值在网格边界处平滑过渡，避免出现明显的不连续性。

#### 梯度计算
```cpp
double grad(int hash, double x, double y) {
    int h = hash & 15;
    double u = h < 8 ? x : y;
    double v = h < 4 ? y : (h == 12 || h == 14 ? x : 0);
    return ((h & 1) == 0 ? u : -u) + ((h & 2) == 0 ? v : -v);
}
```

#### Octave Noise（多层叠加）
```cpp
double octaveNoise(double x, double y, int octaves, double persistence) {
    double total = 0, frequency = 1, amplitude = 1, maxValue = 0;
    for (int i = 0; i < octaves; i++) {
        total += noise(x * frequency, y * frequency) * amplitude;
        maxValue += amplitude;
        amplitude *= persistence;
        frequency *= 2;
    }
    return total / maxValue;
}
```

## 生成结果

程序生成三种不同类型的纹理：

### 1. 云层纹理
- **参数**: 6 层 Octave Noise，持久度 0.5
- **效果**: 模拟自然云层的柔和渐变
- **颜色映射**: 白色到灰蓝色的平滑过渡

![云层纹理](/images/perlin_clouds.png)

### 2. 大理石纹理
- **参数**: 4 层 Octave Noise，持久度 0.6
- **特殊处理**: 使用正弦波扰动噪声值
- **效果**: 模拟大理石的自然纹理

![大理石纹理](/images/perlin_marble.png)

### 3. 木纹纹理
- **参数**: 3 层 Octave Noise，持久度 0.5
- **特殊处理**: 径向距离 + 环状图案
- **效果**: 模拟木材的年轮和纹理

![木纹纹理](/images/perlin_wood.png)

## 技术实现细节

### 编译与运行
```bash
# 编译
g++ -std=c++17 -Wall -Wextra -O2 perlin_noise.cpp -o perlin_noise -lm

# 运行
./perlin_noise

# 输出
# output_clouds.ppm - 云层纹理 (512x512)
# output_marble.ppm - 大理石纹理 (512x512)
# output_wood.ppm - 木纹纹理 (512x512)
```

### 代码结构
```cpp
class PerlinNoise {
private:
    std::vector<int> permutation; // 256个随机排列
    
    // 噪声生成核心函数
    double noise(double x, double y);
    
    // 辅助工具函数
    double fade(double t);
    double lerp(double t, double a, double b);
    double grad(int hash, double x, double y);
    
public:
    PerlinNoise();
    double octaveNoise(double x, double y, int octaves, double persistence);
};
```

### 参数配置
| 纹理类型 | Octaves | Persistence | Frequency 倍数 | 特殊处理 |
|----------|---------|-------------|----------------|----------|
| 云层     | 6       | 0.5         | 8x             | -        |
| 大理石   | 4       | 0.6         | 10x            | sin() 扰动 |
| 木纹     | 3       | 0.5         | 5x             | 径向距离 |

## 扩展方向

1. **3D Perlin Noise**
   - 支持三维空间噪声生成
   - 用于体渲染、3D 地形

2. **Simplex Noise**
   - 更高效的计算方法
   - 消除网格方向性

3. **更多纹理类型**
   - 岩石、地形、水波
   - 程序化动画纹理

4. **GPU 实现**
   - 使用 OpenGL/Compute Shader
   - 实时生成和编辑

## 🔧 完整的代码实现

### PerlinNoise 类定义
```cpp
// Perlin Noise 生成器完整实现
class PerlinNoise {
private:
    std::vector<int> permutation;
    
    double fade(double t) {
        return t * t * t * (t * (t * 6 - 15) + 10);
    }
    
    double lerp(double t, double a, double b) {
        return a + t * (b - a);
    }
    
    double grad(int hash, double x, double y) {
        int h = hash & 15;
        double u = h < 8 ? x : y;
        double v = h < 4 ? y : (h == 12 || h == 14 ? x : 0);
        return ((h & 1) == 0 ? u : -u) + ((h & 2) == 0 ? v : -v);
    }
    
public:
    PerlinNoise(unsigned int seed = 0) {
        permutation.resize(512);
        std::vector<int> p(256);
        for (int i = 0; i < 256; i++) p[i] = i;
        
        std::default_random_engine engine(seed);
        std::shuffle(p.begin(), p.end(), engine);
        
        for (int i = 0; i < 256; i++) {
            permutation[i] = p[i];
            permutation[256 + i] = p[i];
        }
    }
    
    double noise(double x, double y) {
        int X = (int)floor(x) & 255;
        int Y = (int)floor(y) & 255;
        x -= floor(x);
        y -= floor(y);
        
        double u = fade(x);
        double v = fade(y);
        
        int A = permutation[X] + Y;
        int AA = permutation[A];
        int AB = permutation[A + 1];
        int B = permutation[X + 1] + Y;
        int BA = permutation[B];
        int BB = permutation[B + 1];
        
        return lerp(v,
            lerp(u, grad(permutation[AA], x, y), 
                 grad(permutation[BA], x - 1, y)),
            lerp(u, grad(permutation[AB], x, y - 1), 
                 grad(permutation[BB], x - 1, y - 1))
        );
    }
    
    double octaveNoise(double x, double y, int octaves, double persistence) {
        double total = 0, frequency = 1, amplitude = 1, maxValue = 0;
        for (int i = 0; i < octaves; i++) {
            total += noise(x * frequency, y * frequency) * amplitude;
            maxValue += amplitude;
            amplitude *= persistence;
            frequency *= 2;
        }
        return total / maxValue;
    }
};
```

### 📸 图像生成功能

```cpp
// 云层纹理生成
void generateCloudTexture(int width, int height, const std::string& filename) {
    PerlinNoise perlin(12345);
    std::vector<unsigned char> image(width * height * 3);
    
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            double nx = (double)x / width;
            double ny = (double)y / height;
            double value = perlin.octaveNoise(nx * 8, ny * 8, 6, 0.5);
            value = (value + 1.0) / 2.0;
            
            unsigned char color = (unsigned char)(value * 255);
            int idx = (y * width + x) * 3;
            image[idx] = image[idx + 1] = image[idx + 2] = color;
        }
    }
    
    saveImage(filename, image, width, height);
}

// 大理石纹理生成
void generateMarbleTexture(int width, int height, const std::string& filename) {
    PerlinNoise perlin(54321);
    std::vector<unsigned char> image(width * height * 3);
    
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            double nx = (double)x / width;
            double ny = (double)y / height;
            
            double value = perlin.octaveNoise(nx * 10, ny * 10, 4, 0.6);
            value = sin((nx * 20 + value * 5) * M_PI);
            value = (value + 1.0) / 2.0;
            
            unsigned char base = 230;
            unsigned char vein = (unsigned char)(value * 80);
            unsigned char color = base - vein;
            
            int idx = (y * width + x) * 3;
            image[idx] = color;
            image[idx + 1] = color - 20;
            image[idx + 2] = color - 10;
        }
    }
    
    saveImage(filename, image, width, height);
}

// 木纹纹理生成
void generateWoodTexture(int width, int height, const std::string& filename) {
    PerlinNoise perlin(99999);
    std::vector<unsigned char> image(width * height * 3);
    
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            double nx = (double)x / width - 0.5;
            double ny = (double)y / height - 0.5;
            
            double dist = sqrt(nx * nx + ny * ny);
            double noise = perlin.octaveNoise(nx * 5, ny * 5, 3, 0.5);
            double value = sin((dist + noise * 0.3) * 40) * 0.5 + 0.5;
            
            // 木纹颜色 - 棕色系
            unsigned char r = (unsigned char)(180 * value + 75);
            unsigned char g = (unsigned char)(150 * value + 60);
            unsigned char b = (unsigned char)(120 * value + 45);
            
            int idx = (y * width + x) * 3;
            image[idx] = r;
            image[idx + 1] = g;
            image[idx + 2] = b;
        }
    }
    
    saveImage(filename, image, width, height);
}
```

## 🧪 数学原理解析

### 1️⃣ 网格梯度系统
Perlin 噪声的核心思想是在每个整数网格点定义随机梯度向量，然后在网格内进行平滑插值。

**梯度计算公式**:
- 使用哈希表选择 8 个基础方向
- 双线性插值保证平滑过渡

### 2️⃣ Fade 函数的数学推导
经典的 Perlin fade 函数确保了 C² 连续性：
```math
f(t) = 6t⁵ - 15t⁴ + 10t³
```

这个函数满足:
- f(0) = 0, f(1) = 1
- f'(0) = f'(1) = 0
- f''(0) = f''(1) = 0

这些条件保证了噪声值在网格边界处平滑。

### 3️⃣ Octave Noise 的频率分布
多层噪声叠加使用指数衰减的频率倍数和振幅：

```math
\text{频域} = \{1, 2, 4, 8, 16, ...\}
\text{振幅} = \{1, p, p², p³, p⁴, ...\}
```

其中 p 是持久度参数，控制低频/高频的平衡。

## 📊 性能优化技巧

1. **查表优化**: 将 fade() 函数预计算到查找表中
2. **SIMD 并行**: 使用 AVX/SSE 指令集并行计算多个点的噪声
3. **空间局部性**: 按小块区域计算，利用 CPU 缓存
4. **多线程**: 为图像的每个部分分配独立线程

## 学习收获

1. **深入理解 Perlin 噪声的数学基础**
2. **掌握 Octave Noise 实现多层细节**
3. **学习程序化纹理的生成方法**
4. **实践 C++ 高性能图形算法**

## 项目链接

- **GitHub 项目**: [2026-02-10-perlin-noise](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026-02-10-perlin-noise)
- **其他实现**:
  - [WebGL 实时演示](https://webglfundamentals.org/webgl/lessons/zh_cn/webgl-2d-perlin-noise.html)
  - [JavaScript 实现](https://github.com/josephg/noisejs)

## 总结

通过实现 Perlin 噪声，我不仅掌握了这个经典的图形学算法，还深入理解了程序化生成的核心思想。这个项目为后续的图形学学习和项目开发奠定了坚实基础，展示了通过简单数学原理可以创造出丰富自然效果的能力。

**技术栈**: C++ | 数学 | 图形学 | 程序化生成  
**难度**: ⭐⭐☆☆☆  
**实现时间**: 2小时  
**状态**: ✅ 完成

---

*注：所有代码和生成图像已上传到 GitHub 仓库，可作为学习和参考的完整项目。*