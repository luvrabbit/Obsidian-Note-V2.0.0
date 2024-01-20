1. 支持vim：原生支持，但是很不爽,还需要支持自动切换输入法，等待后续研究
2. 支持同步：git插件，进行尝试:成功
3. 支持方便的图片插入和画图
4. 支持链接
5. 支持搜索
# (STM32)最小系统包括什么
| 电源 | $VDD$, $V_{BAT}$, $VDDA$, $VREF+$<br>$VSS$,            $VSSA$, $VREF-$ |
| ---- | ---- |
|  |  |
## 电源
## 数字电源VDD/VSS

![[Pasted image 20240120193215.png|650]]
VDD 设计需求：
1. 1.8 ~ 3.6 V
2. 封装上每一个VDD需要接一个100nF陶瓷去耦(滤波)电容
3. 单个封装（MCU）需要接一个（min. 4.7 tpy. 10 uF）陶瓷/钽电容

![[Pasted image 20240120195540.png]]
![[Pasted image 20240120200001.png]]
VSS直接接地
# 备用电源$V_{BAT}$

![[Pasted image 20240120201424.png]]
$V_{BAT}$设计需求
1. 1.65 ~ 3.6 V
2. 无外部电池需要接100nF滤波到VDD
![[Pasted image 20240120202944.png]]

## 模拟电源VDDA/VSSA

![[Pasted image 20240120204116.png]]
VDDA需求
1. 两个去耦电容：一个100nF陶瓷电容+ 一个1 uF钽电容

![[Pasted image 20240120204903.png|500]]

![[Pasted image 20240120204933.png|500]]

## 模拟参考电源$V_{REF}$

![[Pasted image 20240120231534.png]]
$V_{}