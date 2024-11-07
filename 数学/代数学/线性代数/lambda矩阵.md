# $\lambda$矩阵的定义
元素是$\lambda$的多项式矩阵

## 用途
用来研究若当标准型

# $\lambda$矩阵的秩
$\lambda$矩阵中不恒为零的子式的最高阶数为其秩
因为含有变量$\lambda$所以使用用子式来表述是最好的（相比于列空间维数）

# $\lambda$矩阵可逆的定义
$A(\lambda)$可逆定义为$\exists B(\lambda)\to A(\lambda)B(\lambda)=B(\lambda)A(\lambda)=E$
$\leftrightarrow$$|A(\lambda)|$为常数

# $\lambda$矩阵的初等变换
1. 行（列）对换
2. 行（列）乘以一个非零常数
3. 行（列）加上行（列）的$\lambda$多项式倍

于是可以知道$\lambda$矩阵可逆$\leftrightarrow$$\lambda$矩阵可以分解为有限个初等$\lambda$阵的乘积

# $\lambda$矩阵的等价
$A(\lambda)$经过有限次初等行列变换变为$B(\lambda)$则说明这两个矩阵等价记为$A(\lambda)\cong B(\lambda)$
根据初等变换的性质，等价的充要条件是$A(\lambda)=P(\lambda)B(\lambda)Q(\lambda)$
其中$P(\lambda)，Q(\lambda)$可逆

# $\lambda$矩阵的等价标准型
$A(\lambda)_{m\times n}$的秩为$r$，那么
$$
A(\lambda)\cong
\begin{bmatrix}
D(\lambda) & 0 \\
0 & 0
\end{bmatrix}
=I_{r}(\lambda)
$$
$D(\lambda)$为如下矩阵
$$
\begin{bmatrix}
d_{1}(\lambda) & 0 & 0 & \dots & 0 \\
0 & d_{2}(\lambda) & 0 & \dots & 0 \\
0 & 0 & d_{3}(\lambda) & \dots & 0 \\
\vdots & \vdots & \vdots & \vdots & \vdots \\
0 & 0 & 0 & 0 & d_{r(\lambda)}
\end{bmatrix}
$$
其中$d_{i}(\lambda)|d_{i+1}(\lambda)$（后被前整除），每个$d(\lambda)$首项系数为一

# 求$\lambda$矩阵等价标准型的方法

## 初等变换
从定义得知，直接初等变换出$D(\lambda)$矩阵就可以了，但是这种对于高阶矩阵难以操作

## 行列式因子法
行列式因子记为$D_{i}(\lambda),i=1,2,\dots r$
定义为$A(\lambda)$的$i$阶子式的首一最大公因式
由定义得知，初等变化不会改变矩阵的行列式因子
所以矩阵等价的充要条件为两矩阵行列式因子相同

由于$A(\lambda)$与$D(\lambda)$等价，所以其行列式因子相同
$D(\lambda)$的行列式因子分别为$D_{1}(\lambda)=d_{1}(\lambda)$
$D_{2}(\lambda)$
