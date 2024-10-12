# WHY
学习数值分析的die'dai'fa
# HOW

# WHAT

## 矩阵范数
$$
\lVert A \rVert = \sup_{ x \neq 0 } \frac{\lVert Ax \rVert}{\lVert x \rVert}   
$$
其中$\lVert A \rVert$就是矩阵$A$的范数，矩阵范数可以看作是矩阵对向量作用后的长度倍数变化（伸长，缩短）
比如矩阵的二范数对应的就是特征值的最大值，而特征值就是对某一种向量的伸缩长度，具体定义见pdf
$c_{2} \lVert A \rVert_{\alpha} \leq \lVert A \rVert_{\beta} \leq c_{1}\lVert A \rVert_{\alpha}$

## 条件数

$$
\lVert A \rVert \lVert A^{-1} \rVert
$$
称作矩阵A的条件数，下面讲阐述条件数