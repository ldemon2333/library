# 9.1 标量投影

![[Pasted image 20240913173512.png]]

x在v方向上的投影结果为z。向量z的长度（向量模）就是x在v方向上的标量投影（scalar projection）。

令标量s为向量z的模。

由于z和非零向量v共线，因此z与v的单位向量共线，它们之间的关系为：

![[Pasted image 20240913173749.png]]

x-z垂直于v，因此两者向量内积为0：

![[Pasted image 20240913173828.png]]

用矩阵乘法，（2）可以写成：

![[Pasted image 20240913173851.png]]

将（1）代入（3）得到：

![[Pasted image 20240913173917.png]]

（4）经过整理，得到s的解析式，也就是x在v方向上的标量投影为：

![[Pasted image 20240913174004.png]]

上式可以写成如下几种形式：

![[Pasted image 20240913174033.png]]
![[Pasted image 20240913174056.png]]


# 9.2 向量投影

x在v方向上的向量投影就是：

![[Pasted image 20240913174224.png]]
![[Pasted image 20240913174242.png]]

## 向量张量积
---
看（12），假设v为单位列向量，（12）等价于：

![[Pasted image 20240913174701.png]]

![[Pasted image 20240913174907.png]]

![[Pasted image 20240913175155.png]]

Z 中新坐标还是以行向量表达。

# 10.1
![[Pasted image 20240913175442.png]]
![[Pasted image 20240913175457.png]]

## 行向量：点坐标
---
![[Pasted image 20240913175646.png]]
![[Pasted image 20240913175655.png]]
![[Pasted image 20240913175717.png]]
![[Pasted image 20240913175749.png]]
![[Pasted image 20240913175819.png]]

# 10.2 二次投影+层层叠加
![[Pasted image 20240913180001.png]]

## 层层叠加
---
![[Pasted image 20240913180048.png]]
![[Pasted image 20240913180104.png]]
![[Pasted image 20240913180119.png]]

## 二次投影
---
![[Pasted image 20240913180418.png]]
![[Pasted image 20240913180429.png]]
![[Pasted image 20240913180436.png]]

## 向量投影：张量积
---
![[Pasted image 20240913180701.png]]
![[Pasted image 20240913180904.png]]

## 标准正交基：便于理解
---
![[Pasted image 20240913181058.png]]
![[Pasted image 20240913181150.png]]
![[Pasted image 20240913195228.png]]
![[Pasted image 20240913195325.png]]
![[Pasted image 20240913195457.png]]

# 10.3 二特征数据投影：标准正交基
![[Pasted image 20240913195543.png]]

## 水平方向投影
---
![[Pasted image 20240913195638.png]]
![[Pasted image 20240913195649.png]]
![[Pasted image 20240913195743.png]]

很明显，以A为例，A在横轴投影点P在$\mathbb{R}^2=span(e_1,e_2)$的坐标值为（5，0）。这个结果是怎么得到?

![[Pasted image 20240913200008.png]]
![[Pasted image 20240913200034.png]]
![[Pasted image 20240913200100.png]]
![[Pasted image 20240913200110.png]]


## 竖直方向投影
---
![[Pasted image 20240913200433.png]]
![[Pasted image 20240913200448.png]]
![[Pasted image 20240913200512.png]]
![[Pasted image 20240913200535.png]]

## 叠加
---
![[Pasted image 20240914101414.png]]
![[Pasted image 20240914101426.png]]
![[Pasted image 20240914101449.png]]
![[Pasted image 20240914101505.png]]

# 10.4 二特征数据投影：规范正交基
## 第一个规范正交基
---
![[Pasted image 20240914101630.png]]
![[Pasted image 20240914101709.png]]
![[Pasted image 20240914101756.png]]
![[Pasted image 20240914101824.png]]
![[Pasted image 20240914101852.png]]
![[Pasted image 20240914103142.png]]
![[Pasted image 20240914103203.png]]
![[Pasted image 20240914103218.png]]
![[Pasted image 20240914103246.png]]
![[Pasted image 20240914103306.png]]

## 第二个规范正交基
---
![[Pasted image 20240914103405.png]]
![[Pasted image 20240914103423.png]]
![[Pasted image 20240914103437.png]]

## 第三个规范正交基
---
![[Pasted image 20240914103531.png]]
![[Pasted image 20240914103539.png]]

## 旋转角度连续变化
---
![[Pasted image 20240914103633.png]]
![[Pasted image 20240914103719.png]]

# 10.5 四特征数据投影：标准正交基
## 一次投影：标量投影
---
![[Pasted image 20240914103915.png]]

## 二次投影
---
![[Pasted image 20240914103947.png]]

## 向平面投影
---
![[Pasted image 20240914104048.png]]
![[Pasted image 20240914104055.png]]
![[Pasted image 20240914104122.png]]
![[Pasted image 20240914104128.png]]

## 层层叠加：还原原始矩阵
---
![[Pasted image 20240914104220.png]]
![[Pasted image 20240914104238.png]]
![[Pasted image 20240914104245.png]]

# 10.6 四维数据投影：规范正交基
![[Pasted image 20240914104329.png]]
![[Pasted image 20240914104359.png]]

## V中的像
---
![[Pasted image 20240914104430.png]]
![[Pasted image 20240914104436.png]]

## 第1列向量$v_1$
---
![[Pasted image 20240914104612.png]]
![[Pasted image 20240914104620.png]]
![[Pasted image 20240914104702.png]]
![[Pasted image 20240914104720.png]]
![[Pasted image 20240914104751.png]]
![[Pasted image 20240914104807.png]]
![[Pasted image 20240914104837.png]]
![[Pasted image 20240914104851.png]]

## 第2列向量$v_2$
---
![[Pasted image 20240914104927.png]]
![[Pasted image 20240914104952.png]]

## 层层叠加
---
![[Pasted image 20240914105037.png]]
![[Pasted image 20240914105102.png]]
![[Pasted image 20240914105123.png]]

# 10.7 数据正交化
## 成对特征散点图
---
![[Pasted image 20240914105535.png]]
![[Pasted image 20240914105629.png]]

## 两个格拉姆矩阵
---
![[Pasted image 20240914105714.png]]
![[Pasted image 20240914105741.png]]
![[Pasted image 20240914105807.png]]
![[Pasted image 20240914105818.png]]

## V因X而生
---
![[Pasted image 20240914105925.png]]

## 对角化
---
![[Pasted image 20240914110015.png]]
![[Pasted image 20240914110039.png]]

## 回看规范正交基 V：双标图
---
![[Pasted image 20240914110202.png]]
![[Pasted image 20240914110251.png]]
![[Pasted image 20240914110258.png]]
![[Pasted image 20240914110401.png]]

## 反向正交投影
---
![[Pasted image 20240914110501.png]]
![[Pasted image 20240914110508.png]]
![[Pasted image 20240914110534.png]]
![[Pasted image 20240914110629.png]]


