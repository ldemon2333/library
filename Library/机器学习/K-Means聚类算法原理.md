K-Means算法是无监督的聚类算法。

# 1.K-Means原理初探
K-Means算法的思想很简单，对于给定的样本集，按照样本之间的距离大小，将样本集划分为K个簇。让簇内的点尽量紧密的连在一起，而让簇间的距离尽量的大。

如果用数据表达式表示，假设簇划分为$(C_1,C_2,...C_k)$，则我们的目标是最小化平方误差$E$：
$$
E = \sum\limits_{i=1}^k\sum\limits_{x \in C_i} \|x-\mu_i\|_2^2
$$
其中$\mu_i$是簇$C_i$的均值向量，有时也称为质心，表达式为：
$$
\mu_i = \frac{1}{\|C_i\|}\sum\limits_{x \in C_i}x
$$
这是一个NP难的问题，因此只能采用启发式的迭代方法。

K-Means采用的启发式方式很简单，用下面一组图就可以形象的描述。

![[Pasted image 20240925163110.png]]

上图a表达了初始的数据集，假设k=2。在图b中，我们随机选择了两个k类所对应的类别质心，即图中的红色质心和蓝色质心，然后分别求样本中所有点到这两个质心的距离，并标记每个样本的类别为和该样本距离最小的质心的类别，如图c所示，经过计算样本和红色质心和蓝色质心的距离，我们得到了所有样本点的第一轮迭代后的类别。此时我们对我们当前标记为红色和蓝色的点分别求其新的质心，如图4所示，新的红色质心和蓝色质心的位置已经发生了变动。图e和图f重复了我们在图c和图d的过程，即将所有点的类别标记为距离最近的质心的类别并求新的质心。最终我们得到的两个类别如图f。

当然在实际K-Mean算法中，我们一般会多次运行图c和图d，才能达到最终的比较优的类别。

# 2. 传统K-Means算法流程
在上一节我们对K-Means的原理做了初步的探讨，这里做一个总结。

1）对于K-Means算法，首先要注意的是k值的选择，一般来说，我们会根据对数据的先验经验选择一个合适的k值，如果没有什么先验知识，则可以通过交叉验证选择一个合适的k值。

2）在确定了k的个数后，我们需要选择k个初始化的质心，就像上图b中的随机质心。由于我们是启发式方法，k个初始化的质心的位置选择对最后的聚类结果和运行时间都有很大的影响，因此需要选择合适的k个质心，最好这些质心不能太近。

算法：

输入：样本集$D=\{x_1,x_2,...x_m\}$，聚类的簇树k，最大迭代次数N
输出：簇划分$C=\{C_1,C_2,...C_k\}$

1）从数据集$D$中随机选择k个样本作为初始的k个质心向量：$\{\mu_1,\mu_2,...,\mu_k\}$
![[Pasted image 20240925163710.png]]

# 3.K-Means初始化优化K-Means++
![[Pasted image 20240925163817.png]]

# 4.K-Means距离计算优化elkan K-Means
![[Pasted image 20240925163852.png]]

# 5.大样本优化Mini Batch K-Means
在传统的K-Means算法中，要计算所有的样本点到所有的质心的距离。如果样本量非常大，比如达到10万以上，特征有100以上，此时用传统的K-Means算法非常的耗时，就算加上elkan K-Means优化也依旧。在大数据时代，这样的场景越来越多。此时Mini Batch K-Means应运而生。

顾名思义，Mini Batch，也就是用样本集中的一部分的样本来做传统的K-Means，这样可以避免样本量太大时的计算难题，算法收敛速度大大加快。当然此时的代价就是我们的聚类的精确度也会有一些降低。一般来说这个降低的幅度在可以接受的范围之内。

在Mini Batch K-Means中，我们会选择一个合适的批样本大小batch size，我们仅仅用batch size个样本来做K-Means聚类。那么这batch size个样本怎么来的？一般是通过无放回的随机采样得到的。

为了增加算法的准确性，我们一般会多跑几次Mini Batch K-Means算法，用得到不同的随机采样集来得到聚类簇，选择其中最优的聚类簇。

# 6.K-Means 与 KNN
初学者很容易把K-Means和KNN搞混，两者其实差别还是很大的。

K-Means是无监督学习的聚类算法，没有样本输出；而KNN是监督学习的分类算法，有对应的类别输出。KNN基本不需要训练，对测试集里面的点，只需要找到在训练集中最近的k个点，用这最近的k个点的类别来决定测试点的类别。而K-Means则有明显的训练过程，找到k个类别的最佳质心，从而决定样本的簇类别。

当然，两者也有一些相似点，两个算法都包含一个过程，即找出和某一个点最近的点。两者都利用了最近邻(nearest neighbors)的思想。

# 7.小结
K-Means是个简单实用的聚类算法，这里对K-Means的优缺点做一个总结。

K-Means的主要优点有：

1）原理比较简单，实现也是很容易，收敛速度快。

2）聚类效果较优。

3）算法的可解释度比较强。

4）主要需要调参的参数仅仅是簇数k。

K-Means的主要缺点有：

1）K值的选取不好把握

2）对于不是凸的数据集比较难收敛

3）如果各隐含类别的数据不平衡，比如各隐含类别的数据量严重失衡，或者各隐含类别的方差不同，则聚类效果不佳。

4） 采用迭代方法，得到的结果只是局部最优。

5） 对噪音和异常点比较的敏感。