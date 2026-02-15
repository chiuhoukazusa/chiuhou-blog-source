---
title: Bresenham直线绘制算法实践 - 图形学每日挑战
date: 2026-02-14 10:00:00
categories: [图形学, C++编程, 每日挑战]
tags: [图形学, 算法, Bresenham, C++, 像素绘制]
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/bresenham-2026-02-14/bresenham_output.png
---

## 每日编程挑战：Bresenham直线算法实现

今天完成了图形学中的一个经典算法——**Bresenham直线绘制算法**。这是早期的计算机图形显示中发明的算法，使用整数运算避免了浮点运算的性能开销。

### 项目概述
- **实现语言**：C++17
- **算法核心**：Bresenham整数直线算法
- **输出格式**：800x600像素PNG图像
- **绘制内容**：10条不同方向的直线

### 算法原理

Bresenham算法的核心思想是使用整数误差项来决定像素点的选择。算法步骤如下：

1. 计算dx = |x₂ - x₁|, dy = |y₂ - y₁|
2. 判断步进方向：sx = (x₁ < x₂ ? 1 : -1), sy = (y₁ < y₂ ? 1 : -1)
3. 初始化误差项：err = dx - dy
4. 循环绘制：
   - 如果err² > -dy，则err -= dy，x₁ += sx
   - 如果err² < dx，则err += dx，y₁ += sy

```cpp
void draw_line(vector<vector<int>>& pixels, int x1, int y1, int x2, int y2) {
    int dx = abs(x2 - x1);
    int dy = abs(y2 - y1);
    int sx = (x1 < x2) ? 1 : -1;
    int sy = (y1 < y2) ? 1 : -1;
    int err = dx - dy;
    
    while (true) {
        if (x1 >= 0 && y1 >= 0 && 
            x1 < static_cast<int>(pixels[0].size()) && 
            y1 < static_cast<int>(pixels.size())) {
            pixels[y1][x1] = 1;
        }
        
        if (x1 == x2 && y1 == y2) break;
        
        int e2 = 2 * err;
        if (e2 > -dy) {
            err -= dy;
            x1 += sx;
        }
        if (e2 < dx) {
            err += dx;
            y1 += sy;
        }
    }
}
```

### 效果展示

![](https://github.com/chiuhoukazusa/blog_img/raw/main/bresenham-2026-02-14/bresenham_output.png)

**绘制的10条直线包括**：
1. 顶部和底部水平边界
2. 左侧和右侧垂直边界  
3. 主对角线和反对角线
4. 几个任意方向的斜线

### 迭代过程
1. **初始编译**：编译成功但存在类型转换警告
2. **修复**：使用`static_cast<int>()`显式转换size_t类型
3. **最终版本**：0警告，运行正确，输出符合预期

### 技术要点

**1. 边界检查**
```cpp
if (x1 >= 0 && y1 >= 0 && 
    x1 < static_cast<int>(pixels[0].size()) && 
    y1 < static_cast<int>(pixels.size())) {
    // 安全绘制
}
```

**2. PPM图像格式**
采用P3格式存储图像：
```
P3
800 600
255
0 0 0 0 0 0 ... (RGB值)
```

**3. 性能优化**
- 全部使用整数运算，避免浮点运算
- 预分配像素数组，避免动态分配

### 时间线
- 10:02 - 项目规划：选择Bresenham算法
- 10:07 - 代码编写完成
- 10:13 - 编译修复警告
- 10:15 - 程序运行成功
- 10:19 - GitHub代码推送
- 10:20 - 博客图床部署

### 学习收获
1. **加深算法理解**：通过手写实现，更深入理解整数误差项的作用机制
2. **类型安全**：在C++开发中注意有符号/无符号整数比较
3. **图形基础**：理解像素级图形绘制的原理

### GitHub链接
完整代码和输出：[GitHub仓库](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/02/14-bresenham-line)

---

*本文是"每日编程挑战"系列的一部分，每天完成一个可运行的小项目。*

**标签**：`#图形学` `#Bresenham算法` `#C++` `#像素绘制`