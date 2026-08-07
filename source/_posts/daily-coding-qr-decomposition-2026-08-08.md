---
title: "每日编程实践: QR分解 - CGS vs MGS vs Householder 全方位对比"
date: 2026-08-08 06:15:00
tags:
  - 每日一练
  - 数值方法
  - 线性代数
  - C++
categories:
  - 编程实践
---

> **项目仓库**: [daily-coding-practice/2026/08/08-08-qr-decomposition](https://github.com/chiuhoukazusa/daily-coding-practice/tree/main/2026/08/08-08-qr-decomposition)

## ① 背景与动机

### 为什么需要 QR 分解？

在线性代数的工具箱中，QR 分解是处理最小二乘问题、特征值计算和线性方程组求解的基石之一。将一个矩阵 A 分解为正交矩阵 Q 和上三角矩阵 R 的乘积，听起来简单，但实现起来有讲究。

**实际场景**：

1. **最小二乘拟合**：当你有一堆数据点想拟合一条曲线时（Ax = b 过约束不可解），最小二乘法找的是 min||Ax - b||。直接解 AT A x = AT b 的条件数变成原来的平方（condition number 的平方），数值上极其危险。QR 分解通过 Q^T b = R x 避免了这个平方效应——稳定性天差地别。

2. **特征值计算**：QR 算法（Francis 1961）是现代特征值求解器的核心。通过反复 QR 分解 + 逆序乘（A_{k+1} = R_k Q_k），矩阵逐渐收敛到上三角，对角元就是特征值。MATLAB 的 `eig()`、numpy 的 `linalg.eig()`，底层都在用 QR 算法。

3. **正方系统的稳定求解**：对于 Ax = b，虽然 LU 分解更快，但 QR 不受主元选择（pivoting）失效的影响——对于奇异或近似奇异的矩阵，QR 的数值稳定性比 LU 更好。

4. **计算机视觉中的位姿估计**：PnP、Homography 矩阵分解常用 SVD，而 SVD 的实现第一步就是双对角化，需要 Householder 反射——这正是 QR 分解的核心组件。

### 三种实现方式的价值

这次我们不是实现一种 QR，而是实现三种并量化对比：

- **Classical Gram-Schmidt (CGS)**：教科书上的标准算法，最直观
- **Modified Gram-Schmidt (MGS)**：CGS 的简单修正，数值稳定性的巨大提升
- **Householder 反射**：专业级算法的标杆，数值稳定性最佳

通过对比这三者，能深刻理解"同样的数学等价变换，浮点算术实现方式的不同会带来天壤之别"——这是数值方法最核心的思维方式。

---

## ② 核心原理

### QR 分解的数学定义

给定矩阵 A ∈ R^(m×n)，其中 m ≥ n（方阵或"瘦高"矩阵），求：

- Q ∈ R^(m×m) 为正交矩阵：Q^T Q = Q Q^T = I
- R ∈ R^(m×n) 为上三角矩阵（前 n 行构成 n×n 上三角，其余为零行）

满足：**A = Q R**

如果只需要"经济型"分解（m > n 时），我们可以只在列空间展开，此时 Q ∈ R^(m×n) 满足 Q^T Q = I（列正交但不方阵正交），R ∈ R^(n×n) 为方上三角。

### 算法一：Classical Gram-Schmidt (CGS)

#### 直觉

Gram-Schmidt 正交化的直觉很简单：从 A 的列向量出发，逐步构造一组正交基。每处理一个向量 a_j 时，**减去它在所有已构造的正交向量上的投影**。就像一个迭代的"剥洋葱"过程——从原始列向量中剥离所有已经知道的"方向"分量，剩下的就是新的正交方向。

#### 推导

假设已构造 {q₁, q₂, ..., q_{k-1}}（单位正交），现在处理 a_k：

1. 初始化：v = a_k
2. 对 j = 1 到 k-1：rⱼₖ = qⱼ^T · a_k，然后 v = v - rⱼₖ · qⱼ  
   （这里的关键：投影系数是 qⱼ^T a_k，用的是**原始 a_k**，不是更新后的 v）
3. rₖₖ = ||v||，qₖ = v / rₖₖ

**为什么叫"Classical"？** 第 2 步中，每个投影系数 rⱼₖ = qⱼ^T a_k 都基于原始向量 a_k 计算。这导致一个问题：当向量近乎线性相关时，a_k 在前几个 qⱼ 方向的分量非常大，减去这些分量后剩下的 v 长度很小，精度损失严重。

**直觉解释**：想象你用棍子量 100 米和 1 厘米——先量 100 米（大部分信息），量完还剩下 1 厘米。但浮点表示中，100 和 100.01 之间的差异只有 2-3 位有效数字。CGS 先"消耗"了大部分量，剩下的残余中有效信息极少 → 正交性差。

### 算法二：Modified Gram-Schmidt (MGS)

#### 直觉

MGS 的关键洞察：**不要对原始 a_k 投影，而是对"已经减过部分分量"的中间结果进行投影**。

#### 推导

1. 初始化：v^(0) = a_k
2. 对 j = 1 到 k-1：  
   rⱼₖ = qⱼ^T · **v^(j-1)**（注意：用的是已更新的向量，不是原始 a_k！）  
   v^(j) = v^(j-1) - rⱼₖ · qⱼ
3. rₖₖ = ||v^(k-1)||，qₖ = v^(k-1) / rₖₖ

**为什么更好？** 用 v^(j-1)（已经被前面的 qⱼ 处理过的）来计算投影系数，而不是原始 a_k。这相当于重新计算当前的残余分量，避免了经典方法中"用旧信息做投影"的数值问题。

**直觉**：继续"量尺寸"的比喻——MGS 每砍掉一个方向的分量后，立刻重新量一下还剩多少。这样每一步都是"精确地量当前剩余物"，而不会因为累计舍入而丢失小分量的精度。

**数学上** CGS 和 MGS 是等价的（精确算术下给出相同结果），但在浮点算术下，MGS 的正交性误差显著小于 CGS。

### 算法三：Householder 反射

#### 直觉

Gram-Schmidt 方法的本质是"逐个向量通过投影进行正交化"。Householder 方法则完全不同——它通过**对称反射变换**，将整个列的"下半段"一次性映射为零。

把向量看作空间的坐标点。对于向量 x = (x₁, x₂, ..., x_m)^T，我们想找一个反射面 H（超平面镜像），使得 x 被反射到 e₁ 方向：

H · x = ||x|| · e₁（即只有第一个分量非零）

这个变换只涉及一个向量反射，计算代价低且数值稳定。

#### 推导

给定向量 x ∈ R^m，构造 Householder 向量：

1. 计算范数：α = -sign(x₁) · ||x||  
   （取负号是为了避免 catastrophic cancellation——当 x₁ ≈ ||x|| 时直接用差会丢失精度）

2. 反射方向：u = x - α · e₁（x 到目标 e₁ 方向的差异）

3. 归一化：v = u / ||u||

4. 反射矩阵：H = I - 2 v v^T

当一个向量 w 被 H 反射时：Hw = w - 2（v^T w）v

**几何直觉**：v 是反射面的单位法向量。w 在 v 方向的分量是 (v^T w)v，反射就是翻转这个分量——就好像光照射到镜面（以 v 为法向量），入射角 = 反射角。`I - 2vv^T` 就是"保留与镜面平行的分量，翻转法线分量"的数学表达。

#### QR 分解步骤

对于矩阵 A(1) = A：

1. 取第一列 a₁，构造反射 H₁ 使其变为 r₁₁·e₁
2. 应用到整个矩阵：A(2) = H₁ · A(1)，第一列变成 (r₁₁, 0, 0, ..., 0)^T
3. 对 A(2) 的右下 (m-1)×(n-1) 子矩阵重复
4. 最终：H_{n}···H₁ · A = R → Q = H₁ ··· H_{n}

**每次只消除一列的"下三角"部分**。经过 n 次反射变换，整个矩阵变为上三角 R，而累计变换的逆就是正交矩阵 Q。

### 三种算法的比较总结

| 属性 | CGS | MGS | Householder |
|------|-----|-----|-------------|
| 正交性误差 ||Q^T Q - I|| 大（对病态矩阵可能 > 1） | 中等（通常 10⁻⁸ ~ 10⁻⁵） | 极小（接近机器精度 ~10⁻¹⁶） |
| 计算复杂度 | 2mn² | 2mn² | 2mn²（2m²n 若需要完整 Q） |
| 实现复杂度 | 最简单 | 稍简单 | 中等 |
| 病态矩阵鲁棒性 | 差 | 中等 | 优秀 |
| 并行性 | 差（逐列串行） | 差（逐列串行） | 较好（分块版本可并行） |

**核心洞察**：三种方法在精确算术下等价，但在浮点算术下差别巨大。Householder 是业界标准（LAPACK 的 `dgeqrf` 就用 Householder），MGS 在需要增量更新（如 GMRES 的 Arnoldi 过程）时常用，CGS 仅适用于教学演示。

---

## ③ 实现架构

### 数据流概览

```
输入矩阵 A (m×n)
      │
      ├──→ [CGS 管线] → Q_cgs, R_cgs
      ├──→ [MGS 管线] → Q_mgs, R_mgs
      └──→ [Householder 管线] → Q_hh, R_hh
                                        │
                                        ├── 重构误差：||A - QR||_F
                                        ├── 正交性误差：||Q^T Q - I||_F
                                        ├── 上三角误差：max_{i>j} |R(i,j)|
                                        ├── 线性系统求解误差
                                        └── Q^T A = R 验证
```

### 核心数据结构

```cpp
struct Matrix {
    int rows, cols;
    std::vector<double> data;  // 行主序存储 data[i * cols + j]

    // 索引访问
    double& operator()(int i, int j) { return data[i * cols + j]; }
    const double& operator()(int i, int j) const { return data[i * cols + j]; }
};
```

选择行主序是因为 C/C++ 中更自然，而且逐行访问在 QR 分解中比逐列访问更常见（Householder 反射需要计算 v^T w）。

### 三个核心函数签名

```cpp
// Classical Gram-Schmidt
void cgs_qr(const Matrix& A, Matrix& Q, Matrix& R);

// Modified Gram-Schmidt
void mgs_qr(const Matrix& A, Matrix& Q, Matrix& R);

// Householder Reflection
void householder_qr(const Matrix& A, Matrix& Q, Matrix& R);
```

统一的接口使得替换算法非常简单，也方便交叉验证和基准测试。

### 验证框架

不使用视觉输出（无 PPM/PNG），所有验证都是**数学量化验证**：

通过 `test_one()` 函数统一调用三种算法，并对每种输出统一的量化指标。测试矩阵覆盖：

1. **5×3 随机矩阵**：基本功能验证
2. **4×4 方阵随机矩阵**：方阵情况
3. **6×6 Hilbert 矩阵**：典型病态矩阵（条件数随维度指数增长）
4. **8×5 Vandermonde 矩阵**：中等条件数
5. **20×3 Tall-Skinny**：m >> n 瘦高矩阵
6. **线性系统求解**：验证 QR 求解 Ax = b
7. **Q^T A = R 恒等式验证**

---

## ④ 关键代码解析

### 4.1 CGS 实现

```cpp
void cgs_qr(const Matrix& A, Matrix& Q, Matrix& R) {
    int m = A.rows, n = A.cols;
    Q = Matrix(m, n);  // Q 初始化为零
    R = Matrix(n, n);  // R 初始化为零

    for (int k = 0; k < n; k++) {
        // 第 k 列正交化
        // 步骤 1：计算投影系数（基于原始列 a_k）
        for (int j = 0; j < k; j++) {
            R(j, k) = 0;
            for (int i = 0; i < m; i++)
                R(j, k) += Q(i, j) * A(i, k);  // ⚠️ 用的是原始 A(:,k)
        }

        // 步骤 2：构造 v = a_k - sum(r_jk * q_j)
        std::vector<double> v(m);
        for (int i = 0; i < m; i++) {
            v[i] = A(i, k);
            for (int j = 0; j < k; j++)
                v[i] -= R(j, k) * Q(i, j);
        }

        // 步骤 3：归一化
        double norm = 0;
        for (int i = 0; i < m; i++) norm += v[i] * v[i];
        R(k, k) = sqrt(norm);

        for (int i = 0; i < m; i++)
            Q(i, k) = v[i] / R(k, k);
    }
}
```

**关键细节**：第 13-15 行中 `R(j,k)` 的计算用的是 `A(i,k)`（原始列），而不是更新后的 v。**这就是 "Classical" Gram-Schmidt 和 "Modified" 的关键区别。** 如果用 v（已被前面的 qⱼ 处理过），就是 MGS。

### 4.2 MGS 实现

```cpp
void mgs_qr(const Matrix& A, Matrix& Q, Matrix& R) {
    int m = A.rows, n = A.cols;
    Q = Matrix(m, n);
    R = Matrix(n, n);

    // 先复制 A 到工作向量
    std::vector<std::vector<double> > v(n);
    for (int k = 0; k < n; k++) {
        v[k].resize(m);
        for (int i = 0; i < m; i++) v[k][i] = A(i, k);
    }

    for (int k = 0; k < n; k++) {
        R(k, k) = 0;
        for (int i = 0; i < m; i++)
            R(k, k) += v[k][i] * v[k][i];
        R(k, k) = sqrt(R(k, k));

        // 归一化为 q_k
        for (int i = 0; i < m; i++)
            Q(i, k) = v[k][i] / R(k, k);

        // 从剩余列中消除 q_k 的分量
        for (int j = k + 1; j < n; j++) {
            R(k, j) = 0;
            for (int i = 0; i < m; i++)
                R(k, j) += Q(i, k) * v[j][i];  // ⚠️ 用的是当前的 v[j]，不是原始 A(:,j)

            for (int i = 0; i < m; i++)
                v[j][i] -= R(k, j) * Q(i, k);
        }
    }
}
```

**关键变化**：第 24 行中 `R(k,j) = Q(:,k)^T · v[j]`，v[j] 已经被前面的正交化步骤修改过了。所以 MGS 是"逐步精炼"——每次只消去一个 q 方向的分量，然后立即更新残余，而不是一次性消去所有方向。

**MGS 和 CGS 的微妙差别**：
- CGS：所有投影系数一次性用原始列计算
- MGS：逐步消去（每处理完一个 q，立即更新剩余所有列）
- 这个"立即更新"保证了正交投影都是用最新、最准确的信息做的 → 数值稳定性大幅提升

### 4.3 Householder 反射实现

```cpp
// 计算 Householder 反射向量 v（不存储为完整矩阵）
double householder_vector(const double* x, int m, double* v, double& beta) {
    // x    : 输入向量
    // v    : 输出 Householder 向量（只有被消除的部分非零）
    // beta : 标量因子，H = I - beta * v * v^T

    double sigma = 0;
    for (int i = 1; i < m; i++) sigma += x[i] * x[i];  // 只计算 x[1:] 的部分
    v[0] = 1.0;

    if (sigma == 0.0) {
        beta = 0.0;  // x 已经是 e1 方向
        return x[0];
    }

    double mu = sqrt(x[0] * x[0] + sigma);
    double x0 = x[0];
    double alpha = (x0 <= 0) ? mu : -mu;  // 防止 catastrophic cancellation

    // v = x - alpha * e1（反射方向），归一化隐含在 beta 中
    for (int i = 1; i < m; i++) v[i] = x[i] / (x0 - alpha);
    beta = - (x0 - alpha) / alpha;

    return alpha;  // 返回 R 的对角元
}

// 应用 Householder 反射 H = I - beta * v * v^T 到矩阵 A 的列
void apply_householder(int m, int n, const double* v, double beta, double* A, int ldA) {
    // A 是列主序逻辑存储的，ldA 是 leading dimension
    for (int j = 0; j < n; j++) {
        // 计算内积：tau = v^T * A(:,j)
        double tau = 0;
        for (int i = 0; i < m; i++) tau += v[i] * A[i + j * ldA];

        // A(:,j) = A(:,j) - beta * tau * v
        double btau = beta * tau;
        for (int i = 0; i < m; i++)
            A[i + j * ldA] -= btau * v[i];
    }
}
```

**设计要点**：

1. **不显式存储 Q**：Q 通常是 H₁ · H₂ · ... · Hₙ 的乘积。如果不需要独立的 Q 矩阵（例如只需求解最小二乘），可以直接按反射后的 R 和 Householder 向量存储，节省存储空间。这就是 LAPACK `dgeqrf` 的实现方式。

2. **beta 和 v[0]=1 的隐式存储**：Householder 向量 v 的每个元素只存储 1 和剩下的分量，beta 单独存储。这种紧凑格式使得 n 次反射的存储从 O(m²) 降到 O(mn)。

3. **catastrophic cancellation 的避免**：`alpha = -sign(x₀) * ||x||`。当 x₀ 为正时，取 -||x|| 可以避免 x₀ ≈ ||x|| 时 x₀ - ||x|| ≈ 0 导致的精度损失。

### 4.4 完整的 Householder QR

```cpp
void householder_qr(const Matrix& A, Matrix& Q, Matrix& R) {
    int m = A.rows, n = A.cols;

    // 复制 A 到 R
    R = Matrix(n, n);
    Matrix Ac(m, n);
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            Ac(i, j) = A(i, j);

    // 初始化 Q = I
    Q = Matrix(m, m);
    for (int i = 0; i < m; i++) Q(i, i) = 1.0;

    std::vector<double> v(m), w(m);

    for (int k = 0; k < n && k < m - 1; k++) {
        // 构造 Householder 反射以消除第 k 列的下三角元素
        double sigma = 0;
        for (int i = k; i < m; i++) sigma += Ac(i, k) * Ac(i, k);
        sigma = sqrt(sigma);

        if (sigma < 1e-15) continue;  // 已零化

        double alpha = (Ac(k, k) > 0) ? -sigma : sigma;

        // 构造反射向量 v
        for (int i = 0; i < m; i++) v[i] = 0;
        v[k] = Ac(k, k) - alpha;
        for (int i = k + 1; i < m; i++) v[i] = Ac(i, k);

        double vnorm = 0;
        for (int i = k; i < m; i++) vnorm += v[i] * v[i];
        vnorm = sqrt(vnorm);
        for (int i = k; i < m; i++) v[i] /= vnorm;

        // 应用到 Ac（相当于 H_k * H_{k-1} * ... * H_1 * A）
        for (int j = k; j < n; j++) {
            double tau = 0;
            for (int i = k; i < m; i++) tau += v[i] * Ac(i, j);
            for (int i = k; i < m; i++) Ac(i, j) -= 2.0 * tau * v[i];
        }

        // 应用到 Q（累积反射：Q_final = H_1 * ... * H_k）
        for (int j = 0; j < m; j++) {
            double tau = 0;
            for (int i = k; i < m; i++) tau += v[i] * Q(i, j);
            for (int i = k; i < m; i++) Q(i, j) -= 2.0 * tau * v[i];
            //  ⚠️ 注意 Q 是在右边累积的：最终的 Q = H_1 * H_2 * ... * H_n
            //  每一步 Q = H_k * Q（相当于对 Q 的每一列做反射）
        }

        Ac(k, k) = alpha;
        for (int i = k + 1; i < m; i++) Ac(i, k) = 0;
    }

    // 取 R 的上三角部分
    for (int i = 0; i < n; i++)
        for (int j = i; j < n; j++)
            R(i, j) = (i < m) ? Ac(i, j) : 0;
}
```

**代码解读**：

- **第 19 行**：`Q(i,i) = 1.0` 初始化 Q 为单位矩阵。Q 的累积通过"在右边乘 H_k"实现。
- **第 33 行**：`alpha = -sign(A(k,k)) * ||A(k:end,k)||` 避免数值问题（见上文）。
- **第 42-44 行**：反射向量归一化。v[k] = Ac(k,k) - alpha，其他分量为原始值除以 vnorm。
- **第 47-51 行**：将 H_k 应用到 Ac。H_k = I - 2vv^T，所以 H_k * A_col = A_col - 2*(v^T A_col)*v。
- **第 54-58 行**：更新 Q 矩阵。每次操作 Q = H_k * Q（对 Q 的每一列应用反射变换）。

### 4.5 量化验证函数

```cpp
// 重构误差：||A - QR||_F
double recon_error(const Matrix& A, const Matrix& Q, const Matrix& R) {
    double err = 0;
    for (int i = 0; i < A.rows; i++) {
        for (int j = 0; j < A.cols; j++) {
            double qr = 0;
            for (int k = 0; k < A.cols; k++)
                qr += Q(i, k) * R(k, j);
            double diff = A(i, j) - qr;
            err += diff * diff;
        }
    }
    return sqrt(err);
}
```

**解释**：计算 Frobenius 范数 ||A - QR||_F，这是验证分解正确性的最基本指标。对所有 (i,j)，累加 (A_ij - (QR)_ij)² 然后开方。

```cpp
// 正交性误差：||Q^T Q - I||_F
double ortho_error(const Matrix& Q) {
    int n = Q.cols;
    double err = 0;
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            double sum = 0;
            for (int k = 0; k < Q.rows; k++)
                sum += Q(k, i) * Q(k, j);
            double target = (i == j) ? 1.0 : 0.0;
            double diff = sum - target;
            err += diff * diff;
        }
    }
    return sqrt(err);
}
```

**解释**：Q^T Q 应该是单位矩阵。Frobenius 范数衡量与 I 的偏差。这是区分三种算法的关键指标——CGS 对病态矩阵的正交性误差可达 10⁻³ 甚至更大，而 Householder 接近机器精度 ~10⁻¹⁶。

```cpp
// 上三角误差：R 下三角元素的最大绝对值
double upper_tri_error(const Matrix& R) {
    double max_err = 0;
    for (int i = 1; i < R.rows; i++)
        for (int j = 0; j < i; j++)
            max_err = std::max(max_err, std::abs(R(i, j)));
    return max_err;
}
```

**解释**：R 的所有下三角元素（i > j）理论上应为零。检查实现中是否有累积误差导致非零值。

---

## ⑤ 踩坑实录

### 坑 1：Householder 反射的 Q 矩阵构建方向

**症状**：Q 矩阵构造后，后续验证中 Q^T A ≠ R。

**错误假设**：我当时以为反射矩阵的累积顺序和 A 的消除方向一致——即 Q_k * ... * Q_1 * A = R，Q = Q_1 ... Q_k。

**真实原因**：如果 H_k * ... * H_1 * A = R，那么 A = (H_1 * ... * H_k) * R，所以 Q = H_1 * ... * H_k。但如果 Q 的累积方式是在**右边**乘反射矩阵（Q = Q * H_k），而不是左边，顺序会反。

**修复**：Q 初始化 I，每一步做 Q = H_k * Q（注意是左乘，不是右乘）。由于 H 是对称的，H = I - 2 v v^T，所以 `Q(i,j) -= 2*tau*v[i]` 是正确的左乘操作。关键在于：每步用 H_k 左乘当前的 Q，最终 Q = H_n * ... * H_1，而我们需要的是 Q = H_1 * ... * H_n → 实际上 H 的乘积是顺序无关的...并不对，顺序很重要！

最终我通过直接验证 Q^T·A = R 和 A - Q·R = 0 来确认 Q 构造正确，而不是靠推理方向的对称性。

### 坑 2：Hilbert 矩阵导致 CGS 完全失效

**症状**：CGS 分解 6×6 Hilbert 矩阵后，Q^T Q ≠ I，正交性误差高达 10⁻² ~ 10⁻¹。

**错误假设**：我以为数值误差只是"精度稍差"（10⁻⁸ 级别）。

**真实原因**：Hilbert 矩阵的条件数（6×6 时 cond ≈ 1.5×10⁷，8×8 时 cond ≈ 1.5×10¹⁰）使得列向量近乎线性相关。CGS 的投影系数（qⱼ^T a_k）虽然理论上很大，但浮点表示中 a_k 的方向已经被前几个向量"支配"，残余方向的精度极其有限。这就导致了正交性迅速退化。

**修复**：这不是有"bug"，而是 CGS 的固有数学局限。解决方案是换用 MGS 或 Householder。这也恰好成为最佳的教学案例——证明"选择正确的数值算法"有多重要。

### 坑 3：R 矩阵大小不一致

**症状**：对于 m > n 的瘦高矩阵（如 20×3），R 矩阵的行列数容易搞混。

**错误假设**：我一开始将 R 初始化为 m×n 矩阵，但因为只用到上三角部分的前 n 行，后面的全为零行浪费存储。

**真实原因**：经济型 QR 分解中，R 是 n×n 的上三角方阵。这在数学上等价，但代码中必须一致——验证循环的边界（A.cols vs m vs n）需要仔细对齐。

**修复**：统一用 R ∈ R^(n×n)。重构验证时，Q*R 的乘积只涉及 R 的前 n 行。

### 坑 4：MGS 中的工作向量更新顺序

**症状**：MGS 的正交性改善不明显，和 CGS 差别不大。

**错误假设**：我以为做了"立即更新"就够了，于是每处理一个 q_k 后，只更新了"当前列之后的剩余列"。

**真实原因**：MGS 的正确循环结构是——对每个 k，先归一化 q_k，然后**遍历 j = k+1 到 n-1**，计算 r_{kj} = q_k^T v_j，并用 r_{kj} 更新 v_j。这个"遍历—计算—更新"必须在同一个 k 循环中完成，不能分割。

**修复**：确保内层 j 循环（更新剩余列）和外层 k 循环（逐个正交化）正确嵌套。

---

## ⑥ 效果验证与数据

### 运行结果

```
============================================
  QR Decomposition - 量化验证
  Classical GS vs Modified GS vs Householder
============================================

Test 1: 5x3 Random Matrix
A =
  [0]  0.374540  0.950714  0.731994
  [1]  0.598658  0.156019  0.155995
  [2]  0.058084  0.866176  0.601115
  [3]  0.708073  0.020584  0.969910
  [4]  0.832443  0.212339  0.181825

  Classical Gram-Schmidt         recon=1.92e-15  ortho=9.26e-16  upper_tri=0.00e+00
  Modified Gram-Schmidt          recon=1.92e-15  ortho=9.26e-16  upper_tri=0.00e+00
  Householder                    recon=3.92e-15  ortho=1.10e-15  upper_tri=0.00e+00
```

**分析**：对 5×3 随机矩阵（无病态元素），三种方法的表现几乎完美。CGS/MGS 的重构误差和正交性误差都在 10⁻¹⁵ 量级，接近双精度机器精度。这里 CGS 和 MGS 退化到几乎一样，因为矩阵列向量之间有足够的"正交余量"——条件数很低。

```
Test 3: 6×6 Hilbert Matrix (ill-conditioned)
  Classical Gram-Schmidt         recon=8.29e-16  ortho=1.22e-01  upper_tri=1.29e-15
  Modified Gram-Schmidt          recon=2.02e-15  ortho=8.18e-06  upper_tri=5.08e-14
  Householder                    recon=3.66e-16  ortho=3.06e-16  upper_tri=3.35e-15
```

**关键数据解读**：

- **CGS**：重构误差 8.29e-16（仍然很小！），但正交性误差暴涨到 0.122。说明 A = QR 仍然成立，但 Q 的列已经不是正交的了。这就像你拿到了错误的尺子（Q 不准），然后量出来的"距离"（R）也被相应扭曲了，但两者相互抵消使得乘积仍等于 A。这种"自洽但不正确"的分解在实际应用中会导致灾难——例如用 QR 解最小二乘时，误差会成倍放大。

- **MGS**：正交性误差 8.18e-06，比 CGS 好了 4 个数量级。这证明了"逐步精炼"策略在病态矩阵下的价值——每一步都用最新残余计算投影系数，减少了累积误差。

- **Householder**：正交性误差 3.06e-16，完美接近机器精度。Householder 通过反射变换（而不是迭代投影）自然保持了正交性，因为每个 Householder 反射本身就是完美的正交变换。

```
Test 5: 20x3 Tall-Skinny (m >> n)
  Classical Gram-Schmidt         recon=1.94e-15  ortho=9.61e-17  upper_tri=0.00e+00
  Modified Gram-Schmidt          recon=1.94e-15  ortho=9.61e-17  upper_tri=0.00e+00
  Householder                    recon=1.53e-14  ortho=1.01e-15  upper_tri=0.00e+00
```

对于随机瘦高矩阵（高度 20、宽度 3），列向量在 20 维空间中有广阔的正交空间，条件数很低，所以三种方法再次一致。注意这里 Householder 的重构误差略大（1.53e-14），原因是 Q 矩阵的累积涉及多次矩阵乘法。

```
Test 6: Linear System via QR
  Solve error = 2.78e-15

Test 7: Verify Q^T A = R
  max|Q^T A - R| = 8.88e-16

============================================
  SUMMARY
============================================
  T1 (reconstruction):  ✅
  T2 (MGS <= CGS):      ✅
  T3 (HH <= MGS):       ✅
  T4 (upper triangular):✅
  T5 (ill-cond rank):   ✅
  T6 (linear solve):    ✅
  T7 (Q^T A = R):       ✅

  Overall: ✅ ALL PASSED
```

### 量化指标汇总

| 验证项 | CGS | MGS | Householder | 阈值 | 结果 |
|--------|-----|-----|-------------|------|------|
| 重构误差 (5×3) | 1.92e-15 | 1.92e-15 | 3.92e-15 | 1e-10 | ✅ |
| 正交性误差 (5×3) | 9.26e-16 | 9.26e-16 | 1.10e-15 | - | ✅ |
| Hilbert 正交性 | 1.22e-01 | 8.18e-06 | 3.06e-16 | - | ✅ (排名) |
| 上三角误差 | 0.00e+00 | 0.00e+00 | 0.00e+00 | 1e-13 | ✅ |
| 线性求解 | - | - | 2.78e-15 | 1e-10 | ✅ |
| Q^T A = R | - | - | 8.88e-16 | 1e-12 | ✅ |

---

## ⑦ 总结与延伸

### 核心收获

这次实践展示了数值方法中最核心的理念：**"数学上等价并不等于数值上等价"**。CGS、MGS、Householder 在精确算术下给出完全相同的分解结果，但浮点算术中对病态矩阵的表现天差地别：

1. **CGS → MGS → Householder，正交性误差递减约 4-5 个数量级**（10⁻¹ → 10⁻⁶ → 10⁻¹⁶），每一步都是跳跃式的改进
2. **重构误差 ||A - QR|| 对三种方法都很小**（都 ≤ 10⁻¹⁵），但这不意味着质量好——"自洽但正交性差"的分解在最小二乘应用中会传播误差
3. **Hilbert 矩阵是完美的病态测试集**：6×6 就能让 CGS 完全失效（正交性误差 > 0.1）

**选算法的第一定律**：
- 如果矩阵可能接近奇异 → 用 Householder（几乎没有额外代价，稳定性最好）
- 如果矩阵确定条件数很低（< 10³） → MGS 就够了
- CGS → 仅用于教学演示，不要在生产代码中使用

### 技术局限性

- **Householder 对宽矩阵（m < n）需要特殊处理**：当行数少于列数时，QR 分解需要转向 LQ 分解（先对 A^T 做 QR）
- **MGS 对极度病态矩阵也会失效**：cond(A) > 10⁸ 时，MGS 的正交性误差可能达到 10⁻² 级别
- **存储 vs 计算权衡**：Householder 如果需要显式 Q 矩阵，存储和计算量从 O(mn²) 变为 O(m²n)；LAPACK 的紧凑表示（`dgeqrf`）只存反射向量
- **并行化困难**：Gram-Schmidt 整体是串行过程；Householder 可以通过分块（Block Householder）实现 BLAS-3 级并行，这是生产级实现的方式

### 可优化的方向

1. **Column Pivoting**：当矩阵列向量线性依赖度高时，可以在每一步选择范数最大的剩余列做主元（QR with Column Pivoting, QRP），进一步提升稳定性
2. **Block Householder**：将多个反射合并为矩阵-矩阵乘法（GEMM），利用 BLAS-3（Level-3 BLAS）加速
3. **Givens 旋转**：另一种正交变换方式，通过平面旋转消除元素。适合稀疏矩阵或需要在线更新的场景
4. **Rank-Revealing QR**：结合列主元和条件数估计，可以同时检测矩阵的数值秩

### 与本系列的关联

- **08-05 GMRES**（待开发）：GMRES 的 Arnoldi 迭代天然使用 MGS（增量正交化），正是本项目的直接应用
- **08-04 Conjugate Gradient**：CG 也需要正交基构造，但利用的是对称正定矩阵的特殊结构
- **08-07 LU Decomposition**：LU 和 QR 都是矩阵分解，但 LU 更快（无正交化开销），QR 更稳定（不放大条件数）

QR 分解是通往 SVD（奇异值分解）、特征值计算、最小二乘的大门。理解了三种 QR 方法的本质差异，就理解了数值方法的核心思维方式：**好的数值算法不只是数学公式的翻译，而是对浮点算术特性的深刻洞察。**