---
title: A的n次方问题
tags: [数学,线性代数]
categories: 数学
---

## $A^{n}$问题

可选方案：

1. 对应行列成比例？
2. 可以写成E+B？
3. 可以找到规律递推？
4. 对角化

### Type1：对应行列成比例

$$
已知 A = \left[
  \begin{array}{ccc}
    1 & 2 & -1 \\
    -2 & -4 & 2 \\
    3 & 6 & -3
  \end{array}
\right]，求A^{n}
$$

观察发现，$A$可以写成两个向量相乘的形式：

$$
 A = \left[
  \begin{array}{ccc}
    1  \\
    -2  \\
    3  
  \end{array}
\right]
\left[
  \begin{array}{ccc}
    1  & 2 & -1
  \end{array}
\right] = \alpha^{T}\beta
$$

很容易发现 $\beta\alpha^{T} = [6]$

### Type2：寻找规律递推

$$
已知 A = \left[
  \begin{array}{ccc}
    0 & 1 & 0 \\
    1 & 0 & -1 \\
    0 & -1 & 0
  \end{array}
\right]，求A^{n}
$$

### Type3：幂0，二项展开

$$
已知 A = \left[
  \begin{array}{ccc}
    1 & 1 & 0 \\
    0 & 1 & 1 \\
    0 & 0 & 1
  \end{array}
\right]，求A^{n}
$$
