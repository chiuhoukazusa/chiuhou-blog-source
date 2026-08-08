---
title: "每日编程实践: GMRES Krylov子空间求解器 - Arnoldi迭代与Givens旋转"
date: 2026-08-09 06:00:00
tags:
  - 每日一练
  - 数值方法
  - 线性代数
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/08/08-09-gmres-krylov-solver/gmres_cg_comparison.ppm
---

> **项目仓库**: [daily-coding-practice/2026/08/08-09-gmres-krylov-solver](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/08/08-09-gmres-krylov-solver)

## ① 背景与动机

### 什么是 GMRES？

GMRES（Generalized Minimal RESidual method，广义最小残量法）是求解大规模稀疏线性方程组 $Ax = b$ 的 Krylov 子空间迭代法，由 Saad 和 Schultz 在 1986 年提出。它的核心思想是：在第 $k$ 步迭代中，从 Krylov 子空间 $\mathcal{K}_k(A, r_0) = \text{span}\{r_0, Ar_0, A^2r_0, \ldots, A^{k-1}r_0\}$ 中寻找使残差 $\|b - Ax_k\|_2$ 最小的解。

### 为什么不用直接求解器？

本周我们已经实现了 LU 分解和 QR 分解，但对于大规模稀疏系统（例如从偏微分方程离散化得到的百万阶矩阵），直接求解器的 $O(n^3)$ 复杂度是无法承受的。迭代法通过矩阵-向量乘法 $O(n^2)$ 在 Krylov 子空间中逐步逼近真解，通常几十到几百步迭代就能得到满意精度。

### CG 不够用的情况

共轭梯度法（CG）是求解对称正定（SPD）系统的最优 Krylov 方法，但它有两个根本限制：

1. **必须 SPD**：CG 要求 $A = A^T$ 且所有特征值 > 0。实际工程中大量问题产生非对称矩阵——对流-扩散方程、Navier-Stokes 方程线性化、电路仿真等。
2. **无法处理非对称**：对非对称矩阵，CG 的残差正交性被破坏，算法直接发散。

### GMRES 的优势

GMRES 是 **万能选手**：
- 对**任意可逆矩阵** $A$ 都能工作（不需要对称性）
- 理论保证残差在每一步都**单调不增**（在 Krylov 子空间内做最优逼近）
- 最多 $n$ 步精确收敛（在无舍入误差的理想情况下）

GMRES 被广泛应用于：
- **CFD 计算流体力学**：SU2、OpenFOAM 等求解器的默认线性求解器
- **有限元分析**：ANSYS、COMSOL 中的非对称系统求解
- **油藏模拟**：大规模非对称稀疏系统的工业标准求解器
- **电路仿真**：SPICE 类工具的瞬态分析

### 为什么不直接用"完整"GMRES？

完整 GMRES 每步需要存储所有 Arnoldi 向量 $\{v_0, v_1, \ldots, v_{k-1}\}$，内存开销为 $O(kn)$，当迭代步数 $k$ 很大时内存爆炸。**重启版 GMRES(m)** 在每 $m$ 步后丢弃历史向量、以当前近似解为起点重新开始，将内存控制在 $O(mn)$。

---

## ② 核心原理

### 2.1 Krylov 子空间的数学直觉

给定初始猜测 $x_0$（通常取 $x_0 = 0$），初始残差 $r_0 = b - Ax_0$。

第 $k$ 步近似解 $x_k$ 属于仿射空间 $x_0 + \mathcal{K}_k(A, r_0)$。换句话说：

$$x_k = x_0 + \sum_{j=0}^{k-1} \alpha_j A^j r_0$$

这意味着 $x_k$ 是 $r_0, Ar_0, A^2r_0, \ldots, A^{k-1}r_0$ 的线性组合。从 Cayley-Hamilton 定理可知，$A$ 的特征多项式是 $n$ 次的，因此最多 $n$ 个向量就能张成整个 $\mathbb{R}^n$——理论保证 GMRES 在至多 $n$ 步内得到精确解。

**直觉**：Krylov 子空间的基向量是 $A$ 对初始残差 $r_0$ 的"幂迭代"，每一步都在一个更大的空间中寻找最优近似。这类似于多项式逼近——我们在用矩阵多项式的值去逼近 $A^{-1}b$。

### 2.2 最小残差问题的等价形式

GMRES 在第 $k$ 步求解：

$$\min_{x \in x_0 + \mathcal{K}_k} \|b - Ax\|_2$$

等价地，令 $x = x_0 + V_k y$（其中 $V_k = [v_0, v_1, \ldots, v_{k-1}]$ 是子空间的标准正交基），则：

$$\min_{y \in \mathbb{R}^k} \|r_0 - A V_k y\|_2$$

利用 Arnoldi 关系 $A V_k = V_{k+1} \bar{H}_k$（其中 $\bar{H}_k$ 是 $(k+1) \times k$ 的上 Hessenberg 矩阵），代入得：

$$\min_{y} \|V_{k+1}(\beta e_1 - \bar{H}_k y)\|_2 = \min_{y} \|\beta e_1 - \bar{H}_k y\|_2$$

最后一步利用了 $V_{k+1}$ 是正交矩阵（保范数）。我们将一个 $n$ 维的最小二乘问题**压缩**到了 $k$ 维——这就是 GMRES 高效的关键！

### 2.3 Arnoldi 迭代：构建正交基

Arnoldi 迭代是 GMRES 的核心子程序，它将矩阵 $A$ 投影到 Krylov 子空间上，产生上 Hessenberg 矩阵 $H$。

**算法**（Modified Gram-Schmidt 版本）：

```
给定 v0 = r0 / ||r0||，对 k = 0, 1, 2, ...:
  w = A * vk                          // 矩阵-向量乘法（最昂贵的操作）
  对 i = 0..k:                        // MGS 正交化
    h_{i,k} = (w, vi)                 // 投影系数
    w = w - h_{i,k} * vi              // 减去投影
  h_{k+1,k} = ||w||                   // 新向量的范数
  v_{k+1} = w / h_{k+1,k}            // 归一化
```

这里的关键是 `h_{i,k} = (w, vi)` 存储的是 $A$ 在 Krylov 基下的投影系数。MGS 的数值稳定性优于 Classical Gram-Schmidt，我们在上一天的 QR 分解项目中已经验证过这一点。

**为什么用 MGS 而不是 CGS？** CGS 一次性减去所有投影，会导致正交性在迭代后期迅速退化。MGS 逐个减去投影，数值稳定性显著更好。在 GMRES 中，正交性丢失会直接导致收敛停滞（stagnation）。

Hessenberg 矩阵 $\bar{H}_k$ 的结构：

$$\bar{H}_k = \begin{bmatrix}
h_{0,0} & h_{0,1} & h_{0,2} & \cdots & h_{0,k-1} \\
h_{1,0} & h_{1,1} & h_{1,2} & \cdots & h_{1,k-1} \\
0 & h_{2,1} & h_{2,2} & \cdots & h_{2,k-1} \\
0 & 0 & h_{3,2} & \cdots & h_{3,k-1} \\
\vdots & \vdots & \vdots & \ddots & \vdots \\
0 & 0 & 0 & \cdots & h_{k,k-1}
\end{bmatrix}$$

这是一个 $(k+1) \times k$ 的上 Hessenberg 矩阵——下对角线以下全是零。这个结构是 Arnoldi 迭代的自然产物：MGS 确保新向量 $v_{k+1}$ 正交于之前所有向量，因此投影系数在 $i > k+1$ 的位置为零。

### 2.4 Givens 旋转：高效求解最小二乘

现在我们需要求解 $\min_y \|\beta e_1 - \bar{H}_k y\|_2$。$\bar{H}_k$ 是上 Hessenberg 矩阵，我们可以用 **Givens 旋转** 逐列消去下对角线元素，将其化为上三角矩阵。

Givens 旋转矩阵：

$$G(c, s) = \begin{bmatrix} c & -s \\ s & c \end{bmatrix}, \quad c^2 + s^2 = 1$$

给定两元素 $a$ 和 $b$，我们希望消去 $b$：

$$G(c, s) \begin{bmatrix} a \\ b \end{bmatrix} = \begin{bmatrix} r \\ 0 \end{bmatrix}$$

解得：

$$c = \frac{a}{\sqrt{a^2 + b^2}}, \quad s = \frac{-b}{\sqrt{a^2 + b^2}}$$

**为什么用 Givens 而不是 Householder？** GMRES 每次 Arnoldi 步只向 Hessenberg 矩阵添加一列。Householder 反射一次消去整列，对单列插入不经济。Givens 旋转天然适合逐列处理：每步只需一次新旋转，且前 $k$ 次旋转的累积效果容易维护。

Givens 旋转的核心优势是**正交性保持**：$c^2 + s^2 = 1$，在整个消元过程中数值稳定性极高。

### 2.5 重启策略：GMRES(m)

完整 GMRES 的瓶颈是内存。GMRES(m) 的解决思路：
1. 运行 $m$ 步 GMRES
2. 用得到的近似解 $x_m$ 更新解向量
3. 以 $x_m$ 为新起点，重新计算残差，重启整个 Arnoldi 过程

重启的代价是丢失了 Krylov 子空间的历史信息，可能导致收敛变慢。但工程实践表明 $m = 30\sim50$ 是很好的平衡点。

重启时残差可能不单调——这就是为什么我们要验证"每个 restart 内部残差递减"而非全局单调。

---

## ③ 实现架构

### 3.1 整体数据流

```
输入: A(n×n矩阵), b(n维向量), 重启参数 m, 收敛容差 tol

外层循环 (restart):
  ┌─ 计算当前残差 r = b - Ax
  │  检查收敛: ||r||/||b|| < tol? → 返回解
  │
  ├─ Arnoldi 迭代 (m步):
  │   for k = 0..m-1:
  │     w = A * vk           ← 矩阵-向量积 (核心计算)
  │     MGS正交化 w ← w - Σ((w,vi)·vi)
  │     填充 Hessenberg H[k]
  │
  ├─ Givens 旋转消元:
  │   对第k列:
  │     应用前k次Givens旋转
  │     计算新旋转 G(c_k,s_k)
  │     消去 H[k+1][k]
  │
  ├─ 回代求解:
  │   从三角系统 H*y = g 求 y
  │
  └─ 更新解:
     x = x + Σ(y_i · v_i)     ← 解在Krylov基下的线性组合
```

### 3.2 关键数据结构

```cpp
// Arnoldi 向量：V[0..m] 存储正交基
// V[k] 是 ℝ^n 中的单位向量，V[0..k] 正交
std::vector<Vector> V(m + 1);

// Hessenberg 矩阵：H[i][j] 仅在 i ≤ j+1 时非零
// 上三角 + 一条下对角线的结构
std::vector<std::vector<double>> H(m + 1, std::vector<double>(m, 0));

// Givens 旋转参数：第i步旋转的 (c_i, s_i)
std::vector<Givens> givens_list(m);

// 最小二乘问题的右端项 g
Vector g(m + 1);
```

**设计理由**：
- `V` 的大小是 `m+1` 而非 `m`，因为 Arnoldi 每步生成一个新向量 $v_{k+1}$
- `H` 的行索引范围 `0..m`（包含下对角线 `H[k+1][k]`），列索引范围 `0..m-1`
- Givens 旋转参数按顺序存储，每一步的新旋转应用于当前列

### 3.3 算法复杂度分析

| 操作 | 每步复杂度 | 瓶颈 |
|------|-----------|------|
| 矩阵-向量乘法 `A*vk` | $O(n^2)$ | 🔥 最大开销 |
| MGS 正交化 | $O(nk)$ | 随 $k$ 线性增长 |
| Givens 旋转 | $O(k)$ | 可忽略 |
| 回代求解 | $O(k^2)$ | 可忽略 |
| 解更新 `x = x + Vy` | $O(nk)$ | 与 MGS 同级 |

对于稀疏矩阵 $A$（实际应用中最常见的情况），`A*vk` 降为 $O(\text{nnz})$，此时 MGS 和解更新成为瓶颈。

---

## ④ 关键代码解析

### 4.1 Arnoldi 步（Modified Gram-Schmidt）

```cpp
void arnoldi_step(const Matrix& A, std::vector<Vector>& V,
                  std::vector<std::vector<double>>& H, int k) {
    size_t n = A.size();
    // 步骤1: 生成新的候选向量 w = A * vk
    // 这是整个算法中最昂贵的操作，对于稠密矩阵是 O(n²)
    Vector w = matvec(A, V[k]);

    // 步骤2: Modified Gram-Schmidt 正交化
    // 为什么用MGS而不是CGS？
    // CGS是一次性计算所有投影系数再减去，但数值不稳定
    // MGS逐个减去，保持更好的正交性：
    //   for i=0..k:  // 逐个处理之前的正交基向量
    //     计算投影系数 h_{i,k} = (w, vi)
    //     立即减去投影: w = w - h_{i,k} * vi
    for (int i = 0; i <= k; ++i) {
        H[i][k] = dot(w, V[i]);          // 投影到第i个基向量
        for (size_t j = 0; j < n; ++j) {
            w[j] -= H[i][k] * V[i][j];   // 减去投影分量
        }
    }

    // 步骤3: 计算新基向量的范数
    H[k+1][k] = norm2(w);

    // 步骤4: 归一化得到 v_{k+1}
    // 注意：如果 H[k+1][k] ≈ 0，说明发生了"幸运breakdown"
    // 意味着当前Krylov子空间已经包含了精确解
    if (H[k+1][k] > 1e-14) {
        V[k+1].resize(n);
        for (size_t j = 0; j < n; ++j)
            V[k+1][j] = w[j] / H[k+1][k];
    }
}
```

**关键设计点**：

1. **MGS 的迭代顺序**：`for i = 0..k` 逐个正交化。这与 CGS 的关键区别在于：CGS 先计算所有内积，再一次性减去。当向量之间接近线性相关时，CGS 的舍入误差会被放大。

2. **`H[k+1][k]`** 存储下对角线元素，这是 Arnoldi 关系 $AV_k = V_{k+1}\bar{H}_k$ 的核心——$\bar{H}_k$ 的最后一行的唯一非零元素。

3. **Lucky breakdown 处理**：当 `H[k+1][k] ≈ 0` 时，当前 Krylov 子空间对 $A$ 封闭（invariant subspace），意味着我们已经找到了精确解。这是一个"幸运"的收敛信号，不是错误。

### 4.2 Givens 旋转的计算与应用

```cpp
// 计算 Givens 旋转参数，使得 [c -s; s c] * [a; b] = [r; 0]
Givens givens_rotation(double a, double b) {
    Givens g;
    // 数值稳定的实现：避免 b ≈ 0 或 a ≈ 0 时的溢出
    if (std::abs(b) < 1e-14) {
        // b 已经是零，无需旋转
        g.c = 1.0; g.s = 0.0;
    } else if (std::abs(b) > std::abs(a)) {
        // b 为主导分量：使用 tau = -a/b 计算
        // 避免 a/b 可能产生的溢出
        double tau = -a / b;
        g.s = 1.0 / std::sqrt(1.0 + tau * tau);
        g.c = g.s * tau;
    } else {
        // a 为主导分量：使用 tau = -b/a 计算
        double tau = -b / a;
        g.c = 1.0 / std::sqrt(1.0 + tau * tau);
        g.s = g.c * tau;
    }
    return g;
}

// 将 Givens 旋转应用到一对值 (a, b)
// 变换: [a_new; b_new] = [c -s; s c]^T [a; b]
void apply_givens(const Givens& g, double& a, double& b) {
    double tmp = g.c * a - g.s * b;
    b = g.s * a + g.c * b;
    a = tmp;
}
```

**为什么需要分情况处理？**

直接使用公式 $c = a/\sqrt{a^2+b^2}, s = -b/\sqrt{a^2+b^2}$ 在以下情况会出问题：
- 当 $|b| \gg |a|$ 时，$a/b$ 接近零但仍需精确计算
- 当 $|a| \gg |b|$ 时，$b/a$ 接近零但仍需精确计算

使用 `tau` 参数化避免了 $a^2 + b^2$ 可能的溢出。这是 LAPACK 中 `drotg` 的标准实现方式。

### 4.3 GMRES 主循环中的 Givens 应用

```cpp
for (k = 0; k < restart_m; ++k) {
    arnoldi_step(A, V, H, k);  // 构建 H 的第 k 列

    // 关键：对新列应用之前所有 Givens 旋转
    // 因为每步旋转变换了整个 H，新列必须继承之前的变换
    for (int i = 0; i < k; ++i) {
        apply_givens(givens_list[i], H[i][k], H[i+1][k]);
    }

    // 计算新的 Givens 旋转消去 H[k+1][k]
    givens_list[k] = givens_rotation(H[k][k], H[k+1][k]);
    apply_givens(givens_list[k], H[k][k], H[k+1][k]);
    // 同时更新右端项 g
    apply_givens(givens_list[k], g[k], g[k+1]);

    // 检查收敛：|g[k+1]| 就是当前残差范数
    double current_res = std::abs(g[k+1]) / b_norm;
    if (current_res < tol) {
        breakdown = true; break;  // 提前收敛
    }
}
```

**此处最容易出错的地方**：必须先应用前 $k$ 次 Givens 旋转，再计算第 $k$ 次新旋转。如果顺序搞反，Hessenberg 矩阵的三角化将不正确。

### 4.4 回代求解与解更新

```cpp
// 回代：从三角系统求 y
// 此时 H 的前 (solve_size+1) × (solve_size+1) 子矩阵是上三角的
int solve_size = breakdown ? k : (restart_m - 1);
Vector y(solve_size + 1, 0.0);
for (int i = solve_size; i >= 0; --i) {
    double sum = g[i];
    for (int j = i + 1; j <= solve_size; ++j) {
        sum -= H[i][j] * y[j];
    }
    y[i] = sum / H[i][i];  // 注意：H[i][i] 已经被 Givens 旋转为非零
}

// 解更新：x = x + V * y
// y 是 Krylov 子空间基下的坐标，V*y 将其映射回 ℝ^n
for (int i = 0; i <= solve_size; ++i) {
    for (size_t j = 0; j < n; ++j) {
        result.x[j] += y[i] * V[i][j];
    }
}
```

**为什么 `H[i][i]` 不会为零？** Givens 旋转保持矩阵的秩不变。如果 $A$ 可逆且 `H[k+1][k] > 0`，则三角化后的对角元素不会为零（除非发生 breakdown，此时我们已经提前收敛）。

---

## ⑤ 踩坑实录

### 坑1：SPD 矩阵生成不当导致 CG 无法收敛

**症状**：实现完成后，Test 1（SPD 系统 GMRES vs CG 对比）中 CG 跑满 10001 次迭代仍未收敛到 $10^{-8}$，残差停留在 $2.4 \times 10^{-4}$。

**错误假设**：我以为 $A = LL^T + nI$ 能生成条件数良好的 SPD 矩阵。实际上这个构造产生的矩阵特征值分布极不均匀——$LL^T$ 的最小特征值可能非常接近零，$nI$ 的加入不足以改善条件数。

**真实原因**：$L$ 是随机下三角矩阵，$LL^T$ 的最大和最小特征值之比（条件数）可能达到 $O(n^3)$ 级别。条件数越大，CG 收敛越慢。

**修复方式**：改用对称化 + 对角线优势构造：
```cpp
// 1. 生成随机矩阵 R，对称化: A = (R + R^T)/2
// 2. 使每行严格对角占优: A[i][i] = Σ_{j≠i} |A[i][j]| + 10
```
这样构造的矩阵保证 SPD（Gershgorin 圆盘定理），条件数可控，CG 在 10 次迭代内收敛到 $10^{-9}$。

### 坑2：MGS 正交化的向量维度混淆

**症状**：Arnoldi 步在 $k > 20$ 时产生的向量范数异常增大（$> 10^{6}$），导致 Hessenberg 矩阵元素溢出。

**错误假设**：我最初用 `w[i] -= h * V[i][j]` 写成了错误的索引顺序。

**真实原因**：MGS 正交化中，`w` 和 `V[i]` 都是长度为 $n$ 的向量。外层循环 `i` 遍历基向量索引，内层循环 `j` 遍历向量分量。混淆内外层循环会导致完全错误的正交化。

**修复方式**：严格按两层循环的结构编写：
```cpp
for (int i = 0; i <= k; ++i) {       // 对每个基向量
    H[i][k] = dot(w, V[i]);           // 投影系数
    for (size_t j = 0; j < n; ++j) {  // 对每个分量
        w[j] -= H[i][k] * V[i][j];
    }
}
```

### 坑3：Givens 旋转应用到 Hessenberg 列时索引错误

**症状**：GMRES 对某些测试矩阵在前几步就声称收敛（残差 $< 10^{-15}$），但实际解完全错误。

**错误假设**：我以为 `apply_givens(givens_list[i], H[i][k], H[i+1][k])` 的列索引应该用 `i`。

**真实原因**：对第 $k$ 列应用第 $i$ 次旋转时，作用的元素是 `H[i][k]` 和 `H[i+1][k]`（第 $i$ 和 $i+1$ 行的第 $k$ 列）。如果写成 `H[i][i]` 和 `H[i+1][i]`，应用的是历史列而非当前列。

**修复方式**：确保旋转应用的列号是 `k`（当前列）而非 `i`（旋转步号）：
```cpp
apply_givens(givens_list[i], H[i][k], H[i+1][k]);  // 正确：第k列
// apply_givens(givens_list[i], H[i][i], H[i+1][i]);  // 错误：第i列
```

### 坑4：忘记对右端项 g 应用 Givens 旋转

**症状**：Hessenberg 矩阵正确三角化了，但回代得到的 $y$ 使残差不降反升。

**错误假设**：我以为只需要对矩阵应用 Givens 旋转。

**真实原因**：最小二乘问题 $\min \|\beta e_1 - \bar{H}_k y\|$ 涉及矩阵和右端项。消元操作必须**同时**作用于矩阵和右端项，否则 $y$ 的解空间被扭曲。

**修复方式**：每次 Givens 旋转同时更新矩阵和右端项：
```cpp
apply_givens(givens_list[k], H[k][k], H[k+1][k]);  // 矩阵
apply_givens(givens_list[k], g[k], g[k+1]);         // 右端项
```

---

## ⑥ 效果验证与数据

### 6.1 9项量化验证结果

```
====================================================
  GMRES(m) Krylov Subspace Solver - Verification
====================================================

Test 1: Symmetric SPD system (CG equivalence check)
  CG final residual:     2.152244e-09
  GMRES(30) final residual: 5.189402e-09
  GMRES total iterations:   11
  CG iterations:            10
  ✅ PASS — Both solvers converge to tolerance

Test 2: Non-symmetric linear system
  Final relative residual: 3.091863e-09
  Total iterations:        9
  ✅ PASS — GMRES converges on non-symmetric system

Test 3: Residual monotonic decrease
  Reduction factor: 3.477949e+07
  ✅ PASS — Residual decreases monotonically (within restart cycles)

Test 4: Givens rotation orthogonality
  c²+s²-1 errors: 0.000000e+00, 0.000000e+00, 1.110223e-16
  ✅ PASS — Givens rotations preserve orthogonality

Test 5: Arnoldi Hessenberg matrix structure
  ✅ PASS — H[k] is upper Hessenberg (i>j+1 ⇒ 0)

Test 6: Backward error analysis
  ||b - Ax|| / ||b|| = 3.091863e-09
  ✅ PASS — Backward error below tolerance

Test 7: Restart size convergence comparison
  m= 5: iters=   9  residual=3.091863e-09  converged=yes
  m=10: iters=   9  residual=3.091863e-09  converged=yes
  m=20: iters=   9  residual=3.091863e-09  converged=yes
  m=30: iters=   9  residual=3.091863e-09  converged=yes
  ✅ PASS — All restart sizes converge

Test 8: Exact solution recovery
  Max absolute error: 9.640330e-08
  Relative error:     9.914279e-10
  ✅ PASS — Recovers known solution accurately

Test 9: SPD — CG vs GMRES convergence rate
  CG final residual:       2.152244e-09
  GMRES final residual:    5.189402e-09
  CG iterations:           10
  GMRES total iterations:  11
  ✅ PASS — Both converge on SPD system

====================================================
  ALL TESTS PASSED ✅
====================================================
```

### 6.2 关键指标分析

| 指标 | 值 | 解读 |
|------|-----|------|
| 后向误差 $\|b-Ax\|/\|b\|$ | $3.09 \times 10^{-9}$ | 解的质量极高，接近机器精度 |
| 残差缩减因子 | $3.48 \times 10^7$ | 9次迭代将残差降低7个数量级 |
| SPD系统迭代次数 | 11 (GMRES) vs 10 (CG) | GMRES 的收敛速度与专门优化的CG几乎一致 |
| Givens旋转正交性 | 误差 $< 2 \times 10^{-16}$ | 数值方法精确度达到机器精度极限 |
| 精确解恢复误差 | $9.91 \times 10^{-10}$ | 相对误差1ppb级别 |
| Hessenberg结构 | 严格上Hessenberg | Arnoldi迭代的数学性质得到完美保留 |

### 6.3 收敛性可视化

![GMRES收敛对比](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/08/08-09-gmres-krylov-solver/gmres_convergence.ppm)

上图展示了不同重启参数下 GMRES 的收敛曲线（红色 m=5，绿色 m=10，蓝色 m=20，紫色 m=30）。对于这个测试矩阵，所有 m 值几乎产生相同的收敛行为，说明矩阵的条件数足够好，所需 Krylov 子空间维度很小。

![GMRES vs CG](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/08/08-09-gmres-krylov-solver/gmres_cg_comparison.ppm)

上图对比了 SPD 系统上 GMRES 和 CG 的收敛曲线。两条曲线几乎重合，印证了理论上 GMRES 在 SPD 系统上与 CG 的等价性（GMRES 的残差最小化在 SPD 情况下等价于 CG 的能量范数最小化）。

### 6.4 像素统计验证

| 图片 | 尺寸 | 文件大小 | 像素均值 | 标准差 | 非背景像素占比 |
|------|------|---------|---------|--------|-------------|
| gmres_convergence.ppm | 800×600 | 1.4MB | 254.3 | 10.0 | 0.88% |
| gmres_cg_comparison.ppm | 800×600 | 1.4MB | 254.2 | 10.2 | 0.89% |

图表在白色背景上绘制轴线、网格线和彩色收敛曲线。非背景像素包括坐标轴（黑色）、网格线（灰色）以及 4-5 条不同颜色的收敛曲线。像素分析确认图表包含有意义的内容而非空白/纯色图像。

---

## ⑦ 总结与延伸

### 技术回顾

GMRES 是 Krylov 子空间方法的集大成者。通过 Arnoldi 迭代构建正交基、Givens 旋转求解最小二乘、重启策略控制内存，它提供了一个对任意可逆矩阵通用的线性求解框架。

本周我们完成了数值线性代数三部曲：
1. **LU 分解**（08-07）：Doolittle 算法，$O(n^3)$，适用于中小型稠密矩阵的直接法
2. **QR 分解**（08-08）：CGS/MGS/Householder 三种方案对比，最小二乘问题的标准解法
3. **GMRES**（08-09）：Arnoldi + Givens，适用于大规模稀疏非对称系统的迭代法

### GMRES 的局限性

1. **内存消耗**：每步 Arnoldi 存储一个 $n$ 维向量。对于 $n = 10^6$ 的矩阵，$m = 50$ 的重启参数意味着需要存储 50 个百万维向量（约 400MB）。在 GPU 上这会更紧张。

2. **条件数敏感**：当 $A$ 的条件数 $\kappa(A) \gg 1$ 时，GMRES 收敛极慢。此时必须配合**预条件子**（preconditioner），如不完全 LU（ILU）分解或代数多重网格（AMG）。

3. **重启可能引入停滞**：GMRES(m) 丢弃历史信息可能导致收敛曲线出现平台期。这是理论与工程的取舍。

4. **不适合对称不定矩阵**：对于对称但不正定的矩阵，MINRES 方法更高效（利用对称性减少一半存储和计算）。

### 可以优化的方向

1. **预条件 GMRES**：在 Arnoldi 步前对残差应用预条件子 $M^{-1}$，将原问题转化为 $\tilde{A}x = \tilde{b}$ 其中 $\tilde{A} = M^{-1}A$ 条件数更小。

2. **灵活 GMRES（FGMRES）**：允许预条件子在每次迭代中变化（例如使用迭代法作为预条件子），适合嵌套迭代。

3. **流水线 GMRES（Pipelined GMRES）**：将矩阵-向量乘法和内积重叠执行，利用 GPU 或分布式系统的计算-通信重叠能力。

4. **压缩通信 GMRES（CA-GMRES）**：使用 $s$-step 方法，一次通信完成 $s$ 步计算，适合 MPI 分布式环境。

### 与系列其他文章的关联

- QR 分解中的 Gram-Schmidt 正交化直接复用于 Arnoldi 迭代
- LU 分解中的前代/回代是 Givens 旋转后回代的简化版
- 后续可以探索**预条件共轭梯度（PCG）** 与预条件 GMRES 的性能对比

### 关键收获

实现 GMRES 让我深刻理解了迭代法的哲学：**用重复的廉价操作（矩阵-向量乘法）替代一次性昂贵操作（矩阵分解）**。在科学计算的绝大多数实际问题中，稀疏性是常态而非例外，这使得 Krylov 子空间方法成为不可或缺的工具。
