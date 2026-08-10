---
title: "每日编程实践: SVD 奇异值分解"
date: 2026-08-11 05:45:00
tags:
  - 每日一练
  - 数值计算
  - 线性代数
  - C++
categories:
  - 编程实践
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/08/08-11-svd/output.png
---

## ① 背景与动机

### 奇异值分解是什么

奇异值分解（Singular Value Decomposition，简称 SVD）是线性代数中最重要的矩阵分解之一。它将任意矩阵 A（不必是方阵、不必对称、不必满秩）分解为三个矩阵的乘积：

$$A = U \Sigma V^T$$

其中 U 和 V 是正交矩阵，Σ 是对角矩阵（对角线上的元素称为奇异值）。这看起来简单，但其威力巨大——它告诉你矩阵的"本质结构"。

### SVD 为什么重要

**没有 SVD 的世界会遇到什么问题？**

1. **求解线性系统**：超定方程组（方程数多于未知数）没有精确解，只能用最小二乘近似。没有 SVD 时，法方程 $A^TAx = A^Tb$ 在矩阵病态时会严重放大数值误差。SVD 通过舍弃接近零的奇异值，天然实现正则化。

2. **降维与压缩**：主成分分析（PCA）本质上就是 SVD。图像压缩（如 JPEG 的核心思想）依赖 SVD 截断——保留最大的 k 个奇异值，丢弃小奇异值对应的成分，用 k × (m+n+1) 个数字近似 m×n 个数字。

3. **伪逆计算**：Moore-Penrose 伪逆 $A^+ = V\Sigma^+ U^T$，对非方阵和奇异矩阵都有定义，是解最小范数最小二乘问题的通用工具。

### 工业界实际使用场景

- **推荐系统**（Netflix Prize）：用户-电影评分矩阵做 SVD，取前 k 维做协同过滤
- **3D 图形学**：顶点坐标矩阵的 SVD 用于刚性变换估计、ICP 点云配准
- **自然语言处理**：词-文档矩阵的 SVD 实现潜在语义分析（LSA）
- **控制系统**：Hankel 矩阵的 SVD 用于系统辨识和模型降阶
- **信号处理**：ESPRIT 和 MUSIC 算法用 SVD 分离信号子空间和噪声子空间

### 为什么今天选择实现 SVD

在之前的每日编程实践中，我们已经实现了 LU 分解（08-07）、QR 分解（08-08）和 Jacobi 特征值分解（08-10）。SVD 是这一系列的"终极关卡"——它比前面所有的分解都更通用：LU 要求方阵、QR 要求方阵、特征值分解要求对称矩阵，而 SVD 对任意矩阵都有效。掌握 SVD，意味着你可以处理任何线性代数问题。

---

## ② 核心原理

### SVD 的几何直觉

把矩阵 A 看作一个线性变换：将输入向量 x 映射到输出向量 Ax。SVD 将这个变换分解为三步：

1. **V^T**（旋转/反射）：将输入旋转到一个"自然"坐标系
2. **Σ**（缩放）：沿着坐标轴方向按奇异值缩放
3. **U**（旋转/反射）：将结果旋转回原来的输出空间

$$A = \underbrace{U}_{\text{输出旋转}} \cdot \underbrace{\Sigma}_{\text{缩放}} \cdot \underbrace{V^T}_{\text{输入旋转}}$$

单位球在 A 的作用下变成一个椭球——奇异值就是椭球各个半轴的长度，U 的列给出了这些半轴在输出空间的方向。

### 数学推导

#### 从 A^T A 出发

考虑 $A^T A$（n×n 矩阵）。它是一个**对称半正定**矩阵（因为 $x^T A^T A x = \|Ax\|^2 \ge 0$）。对称半正定矩阵有完整的正交特征向量和**非负**特征值。

设 $A^T A$ 的特征分解为：
$$A^T A = V \Lambda V^T$$

其中 Λ 是对角阵，对角线是特征值 $\lambda_i \ge 0$。奇异值 $\sigma_i = \sqrt{\lambda_i}$。

> **直觉**：$A^T A$ 衡量的是"经过 A 变换后的长度平方"——$\|Av\|^2 = v^T A^T A v$。如果 v 恰好是 $A^T A$ 的特征向量，那么 $\|Av\|^2 = \lambda\|v\|^2$。所以特征值 λ 就是"在这个方向上，A 拉伸了多少（的平方）"。

#### 从 $A A^T$ 出发

也可以考虑 $A A^T$（m×m 矩阵）。它同样是对称半正定的。可以证明 $A^T A$ 和 $A A^T$ 的非零特征值完全一样：

$$A^T A v_i = \lambda_i v_i \implies A A^T (A v_i) = \lambda_i (A v_i)$$

> **直觉**：如果你把一个向量先通过 A（变到输出空间）再通过 A^T（变回来），和先通过 A^T 再通过 A，得到的"方向保持"是相同的——只是中间向量的维度不同。

#### 补全 U

对于非零奇异值对应的列：
$$u_i = \frac{A v_i}{\sigma_i}$$

这保证了 $A V = U \Sigma$。验证：两边同时右乘 $\Sigma^{-1}$（对非零奇异值），得到 $U = A V \Sigma^{-1}$。

对于零奇异值对应的 U 列：这些列张成了 A 的**零空间的正交补**。在标准算法中，通过 Gram-Schmidt 正交化来补全。

### 为什么选择 Two-Step 方法

实现 SVD 有许多种方法：

| 方法 | 精度 | 复杂度 | 适用场景 |
|------|------|--------|---------|
| **Two-Step（ATA + Jacobi）** | 好 | O(n³) | 小型矩阵，教育目的 |
| Golub-Reinsch（双对角化 + QR） | 很好 | O(n³) | LAPACK 使用，工业标准 |
| 分治算法（Divide & Conquer） | 很好 | O(n³) 但常数更小 | 大型矩阵 |
| Jacobi SVD（直接双边 Jacobi） | 极高 | O(n³) 但常数大 | 需要最高精度 |

我选择 Two-Step 方法的理由：
1. **复用已有成果**：可以借助昨天（08-10）的 Jacobi 特征值分解代码
2. **实现清晰**：逻辑分为两步，容易理解和调试
3. **足够好的精度**：对中小矩阵，精度可达机器精度级别
4. **调试意义明确**：每一步的输出都可以独立验证

Two-Step 的局限是：对病态矩阵，$A^T A$ 的条件数是 A 的条件数的**平方**，这会放大数值误差。对于数值要求极高的场景（如矩阵条件数 > 10⁸），应该使用 Golub-Reinsch 或 Jacobi SVD。

---

## ③ 实现架构

### 整体数据流

```
输入矩阵 A (m×n)
       │
       ├─→ [Step 1] 计算 A^T A (n×n)
       │       │
       │       └─→ Jacobi 特征值分解
       │               │
       │               ├─→ 特征值 λ_i → 奇异值 σ_i = √λ_i
       │               └─→ 特征向量 → V (右奇异向量)
       │
       └─→ [Step 2] 计算 U
               │
               ├─→ 对非零 σ_i: u_i = A v_i / σ_i
               ├─→ 对零 σ_i: 填充为零
               └─→ 如果 m > n: Gram-Schmidt 补全到 m×m
       │
       └─→ 输出: U (m×m), Σ (对角, p=min(m,n)), V (n×n)
```

### 关键数据结构

```cpp
using Matrix = std::vector<std::vector<double>>;
const double EPS = 1e-14;
```

- **Matrix**：使用二维 vector，简单直接，适合中小矩阵
- **EPS**：机器精度的判断阈值，用于 Jacobi 收敛判定和零奇异值检测

### 职责划分

| 组件 | 职责 |
|------|------|
| `jacobiEigen()` | 对称矩阵特征值分解，返回特征值和特征向量（降序） |
| `svd()` | 主入口：ATA 计算 → Jacobi → S 和 V → U 计算 → U 补全 |
| `reconstructionError()` | 验证 A ≈ UΣV^T 的精度 |
| `orthogonalityError()` | 验证 U 和 V 的正交性 |
| `svdPropertyError()` | 验证核心性质 AV = UΣ |
| `eigenvalueMatchCheck()` | 验证 σ² 是否匹配 AA^T 的特征值 |

---

## ④ 关键代码解析

### Jacobi 特征值分解（核心）

Jacobi 方法是 SVD 的基石。它的思想是：**反复做平面旋转，每次消去最大的非对角元**。

```cpp
void jacobiEigen(const Matrix& A, std::vector<double>& evals, Matrix& evecs) {
    int n = (int)A.size();
    Matrix V(n, std::vector<double>(n, 0));
    for (int i = 0; i < n; i++) V[i][i] = 1;  // V 初始为单位矩阵
    Matrix M = A;  // 工作拷贝

    for (int iter = 0; iter < 500; iter++) {
        // 1. 找到最大的非对角元 (p, q)
        int p = 0, q = 1;
        double maxVal = 0;
        for (int i = 0; i < n; i++)
            for (int j = i + 1; j < n; j++) {
                double v = std::abs(M[i][j]);
                if (v > maxVal) { maxVal = v; p = i; q = j; }
            }
        if (maxVal < EPS) break;  // 已收敛

        // 2. 计算旋转角度 θ
        double theta;
        if (std::abs(M[p][p] - M[q][q]) < EPS)
            theta = (M[p][q] > 0) ? M_PI / 4.0 : -M_PI / 4.0;
        else
            theta = 0.5 * std::atan2(2.0 * M[p][q], M[p][p] - M[q][q]);
        double c = std::cos(theta), s = std::sin(theta);
```

> **为什么是 `atan2(2b, a-d)`？**  
> 对于 2×2 的对称矩阵 [[a, b], [b, d]]，要消去 b，需要满足 $b \cos(2\theta) + \frac{a-d}{2} \sin(2\theta) = 0$。移项得 $\tan(2\theta) = \frac{2b}{a-d}$。

```cpp
        // 3. 应用旋转变换 M = J^T * M * J
        // Step 3a: J^T * M（左乘）——只影响行 p 和 q
        std::vector<double> oldRowP = M[p], oldRowQ = M[q];
        for (int j = 0; j < n; j++) {
            M[p][j] = c * oldRowP[j] + s * oldRowQ[j];   // J^T[p][p]*M[p]+J^T[p][q]*M[q]
            M[q][j] = -s * oldRowP[j] + c * oldRowQ[j];  // J^T[q][p]*M[p]+J^T[q][q]*M[q]
        }
        // Step 3b: (J^T*M) * J（右乘）——只影响列 p 和 q
        for (int i = 0; i < n; i++) {
            double old_mip = M[i][p], old_miq = M[i][q];
            M[i][p] = c * old_mip + s * old_miq;   // 乘 J 的列 p: (c, -s)
            M[i][q] = -s * old_mip + c * old_miq;  // 乘 J 的列 q: (s, c)
        }
```

> **关键细节**：行更新必须先保存旧行数据（`oldRowP, oldRowQ`），因为我们在原地修改矩阵。列更新读取的是行更新**之后**的值——对于非 p,q 的行（i ≠ p,q），这些值没有被改动过，所以是正确的。

> **最常见的 Bug**：把 J^T 和 J 的符号搞混。J^T = [[c, s], [-s, c]]，J = [[c, -s], [s, c]]。左乘用 J^T 的符号，右乘用 J 的符号。如果搞反了，矩阵不会被对角化，而是变成另一个非对角形式。

```cpp
        // 4. 累积特征向量 V = V * J
        for (int i = 0; i < n; i++) {
            double vip = V[i][p], viq = V[i][q];
            V[i][p] = c * vip + s * viq;    // J 的列 p: (c, s)
            V[i][q] = -s * vip + c * viq;   // J 的列 q: (-s, c)
        }
    }
```

> **重要**：V 的更新使用 J 的符号（不是 J^T），因为我们要的是 $M_{\text{new}} = J^T M J$ 对应的特征向量，其中 V 是右乘累积：$V_{\text{new}} = V_{\text{old}} \cdot J$。

#### 排序

```cpp
    // 提取特征值并降序排列
    evals.resize(n);
    for (int i = 0; i < n; i++) evals[i] = M[i][i];

    std::vector<int> idx(n);
    for (int i = 0; i < n; i++) idx[i] = i;
    std::sort(idx.begin(), idx.end(),
        [&](int a, int b) { return evals[a] > evals[b]; });

    std::vector<double> sortedEvals(n);
    evecs.assign(n, std::vector<double>(n));
    for (int i = 0; i < n; i++) {
        sortedEvals[i] = evals[idx[i]];
        for (int j = 0; j < n; j++)
            evecs[j][i] = V[j][idx[i]];  // 对应特征向量跟随排序
    }
    evals = sortedEvals;
```

> **为什么降序排列？** SVD 的惯例是将奇异值从大到小排列，这样截断前 k 个就得到最优的 k 秩近似。

### A^T A 的计算

```cpp
    // 计算 A^T * A（只计算上三角，利用对称性）
    Matrix ATA(n, std::vector<double>(n, 0));
    for (int i = 0; i < n; i++) {
        for (int j = i; j < n; j++) {
            double sum = 0;
            for (int k = 0; k < m; k++)
                sum += A[k][i] * A[k][j];   // (A^T)[i][k] * A[k][j] = A[k][i] * A[k][j]
            ATA[i][j] = ATA[j][i] = sum;
        }
    }
```

`A^T A` 的第 (i,j) 个元素 = Σ_k A[k][i] * A[k][j]（A 的列 i 和列 j 的内积）。利用对称性只算一半，节省约一半计算量。

### 奇异值提取和 U 计算

```cpp
    // 奇异值 = sqrt(特征值)
    S.resize(p);
    for (int i = 0; i < p; i++)
        S[i] = (evals[i] > 0) ? std::sqrt(evals[i]) : 0.0;

    // U = A * V * Σ^{-1}
    U.assign(m, std::vector<double>(p, 0));
    for (int i = 0; i < p; i++) {
        if (S[i] > 1e-12) {
            double invS = 1.0 / S[i];
            for (int row = 0; row < m; row++) {
                double sum = 0;
                for (int col = 0; col < n; col++)
                    sum += A[row][col] * V[col][i];
                U[row][i] = sum * invS;
            }
        }
    }
```

> **为什么除以 S[i]？** 根据 SVD 的定义，$AV = U\Sigma$。对于第 i 列：$A v_i = \sigma_i u_i$，所以 $u_i = A v_i / \sigma_i$。

> **零奇异值处理**：当 $\sigma_i \approx 0$ 时，U[:,i] 保持为零向量。这些列稍后在 Gram-Schmidt 阶段被填充。

### U 的补全（Gram-Schmidt 正交化）

当 m > n 时，U 需要从 m×n 扩展到 m×m。额外的列张成 A 的零空间。另外，对应零奇异值的列（在前面步骤中保持为零）也需要填充。

```cpp
    // 对每个列检查是否需要填充
    for (int col = 0; col < m; col++) {
        // 检查当前列是否已有效
        double existingNorm = 0;
        for (int row = 0; row < m; row++)
            existingNorm += Uext[row][col] * Uext[row][col];
        if (existingNorm > 1e-10) continue;

        // 尝试多个初始向量
        for (int trial = 0; trial < m + 10; trial++) {
            std::vector<double> vec(m, 0);
            if (trial < m) vec[trial] = 1.0;  // 先尝试标准基
            else { /* 随机向量作为后备 */ }

            // 3 遍 Modified Gram-Schmidt
            for (int pass = 0; pass < 3; pass++) {
                for (int j = 0; j < col; j++) {
                    double nj = 0;
                    for (int row = 0; row < m; row++)
                        nj += Uext[row][j] * Uext[row][j];
                    if (nj < 1e-10) continue;  // 跳过零列

                    double proj = 0;
                    for (int row = 0; row < m; row++)
                        proj += vec[row] * Uext[row][j];
                    for (int row = 0; row < m; row++)
                        vec[row] -= proj * Uext[row][j];
                }
            }

            double colNorm = std::sqrt(/* norm of vec */);
            if (colNorm > 1e-10) {
                // 归一化并存储
                for (int row = 0; row < m; row++)
                    Uext[row][col] = vec[row] / colNorm;
                break;
            }
        }
    }
```

> **为什么需要 3 遍 Gram-Schmidt？** Classic Gram-Schmidt 在数值上不稳定——正交化第 k 个向量时，累积的舍入误差可能导致它和前 k-1 个向量有残留相关性。重复正交化（reorthogonalization）可以将这种误差降低到机器精度级别。通常 2-3 次就足够了。

> **跳过零列**：当 S[i] ≈ 0 时，U[:,i] 是零向量。如果 Gram-Schmidt 试图用零向量做正交基，会除以零。跳过零列，让后续算法填充这些位置。

---

## ⑤ 踩坑实录

### Bug 1：Jacobi 特征向量符号错误

**症状**：V^T V = I（正交性完美），但 A^T A · V ≠ V · Λ。特征向量完全不满足特征方程。

**错误假设**：以为 V 的更新公式和 M 的更新公式符号相同。

**真实原因**：我最初的代码中 V 更新使用了 J^T 的符号：
```cpp
// 错误代码：
V[i][p] = c * vip - s * viq;   // 这是 J^T 的符号！
V[i][q] = s * vip + c * viq;
```
正确的 V 更新应该使用 J 的符号（V = V · J，不是 V = V · J^T）。

**修复方式**：将 V 更新改为：
```cpp
V[i][p] = c * vip + s * viq;    // J 的列 p
V[i][q] = -s * vip + c * viq;   // J 的列 q
```

**教训**：在 Jacobi 迭代中，M 做相似变换 M = J^T M J（使用 J^T 左乘、J 右乘），而特征向量矩阵 V 只右乘 J（V = V J），因为 V 的列是特征向量基的线性组合系数，每次只累积旋转。混淆 J 和 J^T 的符号会导致 V 变成"反方向的旋转"。

### Bug 2：U 扩展列不正交

**症状**：U 的正交性检查显示某些列的 U^T U 非对角元高达 0.5 甚至 1.0。

**错误假设**：Gram-Schmidt 只需要一次正交化就足够。

**真实原因**：对于秩亏矩阵（某些 σ_i = 0），对应的 U 列为零。原来只处理 m > n 的扩展列（i ≥ p），而忽略了 i < p 但 σ_i ≈ 0 的列。这些零列不会被 Gram-Schmidt 跳过，导致后续正交化时除以零或得到退化结果。

**修复方式**：
1. 对所有列检查是否为零列，不仅限于扩展列
2. 增加 3 遍 reorthogonalization
3. 每次 Gram-Schmidt 前检查目标列的二范数，跳过零列

### Bug 3：秩亏矩阵的 AV=US 验证失败

**症状**：对于秩为 1 的 4×3 矩阵，AV=US 的误差高达 1e-7。

**错误假设**：期望对所有 k，A v_k = σ_k u_k 精确相等。

**真实原因**：当 σ_k = 0 时，U[:,k] 被 Gram-Schmidt 填充为非零向量，而 A v_k 理论上也是零（或接近零的数值噪声）。此时 U[:,k] · 0 = 0 ≠ A v_k（虽然 A v_k 很小但不等于 0）。

**修复方式**：将 AV=US 的验证分为两种情况：
- σ_k > 阈值：检查 A v_k ≈ σ_k u_k
- σ_k ≈ 0：检查 A v_k ≈ 0（验证零空间性质）

### Bug 4：rank 检测阈值过小

**症状**：秩为 2 的 6×4 矩阵被误判为满秩（rank=4）。

**错误假设**：用 1e-8 作为零奇异值的阈值是个好选择。

**真实原因**：A^T A 的特征值 λ_i 是 σ_i²。Jacobi 算法的收敛容限是 EPS=1e-14，所以最小的"零"特征值大约也是 1e-14 量级。σ = √λ ≈ 1e-7，刚好大于我的 1e-8 阈值。

**修复方式**：将零检测阈值提高到 1e-6，并且对秩亏矩阵使用放宽的验证容限（1e-6 而非 1e-10）。

---

## ⑥ 效果验证与数据

### 测试矩阵设计

为了全面验证 SVD 实现的正确性，我设计了 5 个测试用例，覆盖了所有形态组合：

| 测试用例 | 维度 | 特点 | 预期秩 |
|---------|------|------|--------|
| 对称正定 3×3 | 3×3 方阵 | 良态，满秩 | 3 |
| 等差数列 6×4 | 6×4 高矩阵 | 近似秩亏（实际秩=2） | 2 |
| 随机 3×5 | 3×5 宽矩阵 | 满秩 | 3 |
| 随机 8×6 | 8×6 | 满秩 | 6 |
| 秩-1 矩阵 | 4×3 | 精确秩亏 | 1 |

### 量化验证结果

完整代码执行后输出的验证数据（已转换为表格）：

#### 测试 1：对称正定 3×3
```
奇异值: σ₁=9.348494, σ₂=3.730159, σ₃=1.921347
重建误差: 1.39 × 10⁻¹⁶  ✅
U 正交性: 4.44 × 10⁻¹⁶  ✅
V 正交性: 5.55 × 10⁻¹⁷  ✅
AV=US:    8.88 × 10⁻¹⁶  ✅
S² vs AAT: 1.42 × 10⁻¹⁴  ✅
```

#### 测试 2：秩亏 6×4（rank=2）
```
奇异值: σ₁=69.95, σ₂=2.62, σ₃≈0, σ₄≈0
重建误差: 4.94 × 10⁻⁹   ✅（秩亏容限 1e-6）
U 正交性: 2.96 × 10⁻¹⁴  ✅
V 正交性: 4.44 × 10⁻¹⁶  ✅
AV=US:    2.21 × 10⁻⁷   ✅
S² vs AAT: 2.73 × 10⁻¹²  ✅
```

秩亏矩阵的数值精度略低于满秩矩阵（这是 Two-Step 方法通过 A^T A 引入的额外数值误差）。但 1e-6 量级对大多数应用完全足够。

#### 测试 3-5：全部 PASS

所有测试均通过，满秩矩阵达到机器精度（~1e-15），秩亏矩阵达到 1e-6 级别。

### 可视化输出

SVD 的可视化展示了矩阵分解的结构：

![SVD 分解可视化](https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/2026/08/08-11-svd/output.png)

图中从左到右依次是：原始矩阵 A、左奇异向量 U、奇异值矩阵 Σ（对角线）、右奇异向量 V。颜色从蓝色（负值）过渡到白色（零）再到红色（正值），可以直观看到 U 和 V 的列向量结构以及 Σ 的对角性质。

### 关键指标汇总

| 指标 | 满秩矩阵 | 秩亏矩阵 |
|------|---------|---------|
| 重建误差 | ~10⁻¹⁵ | ~10⁻⁹ |
| U/V 正交性 | ~10⁻¹⁶ | ~10⁻¹⁴ |
| AV=US 误差 | ~10⁻¹⁵ | ~10⁻⁷ |
| 奇异值单调性 | ✓ | ✓ |

---

## ⑦ 总结与延伸

### 技术局限性

1. **Two-Step 方法的条件数问题**：A^T A 的条件数是 A 的平方。当 A 的条件数 > 10⁸ 时，A^T A 的条件数 > 10¹⁶，在双精度下可能失去所有有效位。此时应使用 Golub-Reinsch 方法（直接对 A 做双对角化，不显式计算 A^T A）。

2. **O(n³) 的 Jacobi 迭代**：Jacobi 方法每次都扫描全部非对角元，大矩阵会很慢。对于 n > 1000 的矩阵，应考虑分治算法或随机化 SVD。

3. **实现复杂度**：Two-Step 方法比直接调用 LAPACK 的 dgesvd 多了一些手工步骤，容易出错（如本文踩坑记录所示）。

### 可优化方向

1. **性能优化**：
   - 将 A^T A 的计算改为 BLAS 风格的 `dgemm` 调用
   - Jacobi 的"找最大非对角元"可以用循环展开或并行化加速
   - 对于稀疏矩阵，可以利用稀疏结构大幅减少计算量

2. **数值稳定性**：
   - 实现完整的 Golub-Reinsch 算法（双对角化 + 隐式 QR）
   - 添加 Wilkinson 位移加速收敛
   - 使用 divide-and-conquer 策略处理大矩阵

3. **功能扩展**：
   - **截断 SVD**：只取前 k 个奇异值，实现最优低秩近似
   - **增量 SVD**：新数据到来时，增量更新 SVD 而不重新计算
   - **随机化 SVD**：对于超大矩阵，用随机投影近似 SVD

### 与系列其他文章的关联

本文是数值分解系列的第四篇。在之前我们已经实现了：

- **[08-07] LU 分解**：把方阵分解为下三角和上三角，用于求解线性方程组
- **[08-08] QR 分解**：把方阵分解为正交阵和上三角阵，用于最小二乘和特征值
- **[08-10] Jacobi 特征值分解**：对称矩阵的特征值分解，今天直接复用了其代码
- **[今天] SVD**：最通用的分解，适用于任意形状和秩的矩阵

这些分解形成了一个完整的工具链：从方程求解（LU）→ 最小二乘（QR）→ 谱分析（特征值）→ 通用分解（SVD），每一步都在前一步的基础上增加了能力和通用性。

### 关键洞察

实现 SVD 的过程让我深刻体会到：**数值线性代数中，符号的正负之差可以造就 16 位精度的完美结果，也可以让整个算法彻底失效。** Jacobi 方法中 J 和 J^T 的符号混淆导致特征向量完全错误——但 V^T V = I（正交性检查）却完美通过。这提醒我们，单元测试必须设计**针对性**的检查点，不能只依赖通用指标。

---

## 附录：完整代码

完整代码见 GitHub：https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/08/08-11-svd

编译运行：

```bash
g++ main.cpp -o output -std=c++17 -O2
./output
```

输出包括 5 个测试用例的量化验证结果和 SVD 矩阵分解的可视化图像。
