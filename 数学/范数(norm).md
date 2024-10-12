# WHY
学习数值分析的迭代法解矩阵时，需要判断收敛条件，也就需要判断两个向量（矩阵）的接近程度，于是引入范数这个概念。
# HOW
https://www.stat.uchicago.edu/~lekheng/courses/302/notes2.pdf
推导过程见上述pdf
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
称作矩阵A的条件数，下面讲阐述条件数是矩阵$A$对空间中的向量的最大伸缩比例比上最小伸缩比例
借鉴自这个pdf的第三页[CS210_lect07.pdf (iitd.ac.in)](https://www.cse.iitd.ac.in/~dheerajb/CS210_lect07.pdf)
$$
\begin{align}
&\lVert A \rVert 是最大伸缩比例 \\
&所以现在需要说明的是\lVert A^{-1} \rVert 代表对向量的最小伸缩比例 \\
&那么设m = \min \frac{\lVert Ax \rVert }{\lVert x \rVert } = \min 
\end{align}
$$