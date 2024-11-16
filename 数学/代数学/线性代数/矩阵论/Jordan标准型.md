# 快速求矩阵的Jordan标准型

## 求$\lvert \lambda E-A \rvert$，并因式分解
$\lvert \lambda E-A \rvert=(\lambda-\lambda_{1})^{m_{1}}\dots(\lambda-\lambda_{s})^{m_{s}}$
根据特征值一定为最小多项式的根，以及最小多项式整除零化多项式（$\lvert \lambda E-A \rvert$也是零化多项式）
可以对最小多项式进行猜测，而$R(\lambda_{i}E-A)$可以反应$A$对应$\lambda_{i}$有几个特征向量，如果秩为$R$，则特征向量个数个数为$n-R$也是零空间的维数，如果$\lambda_{i}$的代数重数（$m_{i}$)和特征向量的个数不同意，那么说明需要求广义特征向量，也即对应的Jordan块阶数不为1，具体见下图


# 三阶求法示例
![[Pasted image 20241116204900.png]]