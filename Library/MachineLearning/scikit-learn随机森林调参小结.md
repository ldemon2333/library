# 1.scikit-learn随机森林类库概述
在scikit-learn中，RF的分类类是RandomForestClassifier，回归类是RandomForestRegressor。当然RF的变种Extra Trees也有， 分类类ExtraTreesClassifier，回归类ExtraTreesRegressor。由于RF和Extra Trees的区别较小，调参方法基本相同，本文只关注于RF的调参。

和GBDT的调参类似，RF需要调参的参数也包括两部分，第一部分是Bagging框架的参数，第二部分是CART决策树的参数。下面我们就对这些参数做一个介绍。

# 2.RF框架参数
首先我们关注与RF的Bagging框架的参数。这里可以和GBDT对比学习。GBDT的框架参数比较多，重要的有最大迭代器个数，步长和子采样比例。

下面我来看看RF重要的Bagging框架的参数，由于RandomForestClassifier和RandomForestRegressor参数绝大部分相同，这里会将它们一起讲，不同点会指出。

1）n_estimators：也就是最大的弱学习器的个数。一般来说n_estimators太小，容易欠拟合，n_estimators太大，计算量会太大，并且n_estimators到一定的数量后，再增大n_estimators获得的模型提升会很小，所以一般选择一个适中的数值。默认是100。

2）oob_score：即是否采用袋外样本来评估模型的好坏。默认是False。个人推荐是True，因为袋外分数反应了一个模型拟合后的泛化能力。

3）criterion：即CART树做划分时对特征的评价标准。分类模型和回归模型的损失函数是不一样的。分类RF对应的CART分类树默认是基尼系数gini，另一个可选择的是信息增益。回归RF对应的CART回归树默认是均方差mse，另一个可以选择的标准是绝对值差mae。一般来说选择默认。

RF重要的框架参数比较少，主要需要关注的是n_estimators，即RF最大的决策树个数。

# 3.RF决策树参数
下面我们再来看RF的决策树参数，它要调参的参数基本和GBDT相同，如下：

1）RF划分时考虑的最大特征数**max_features**：可以使用很多种类型的值，默认是“auto"，意味着划分时最多考虑$\sqrt N$特征；如果是”log2“意味着划分时最多考虑$log_2 N$个特征。如果是整数，代表考虑的特征绝对数。如果是浮点数，代表考虑特征百分比，即考虑(百分比$\times$N)取整后的特征数。

2）决策树最大深度**max_depth**

3）内部节点再划分所需最小样本数**min_samples_split**

4）叶子节点最少样本数**min_samples_leaf**

5）叶子节点最小的样本权重和**min_weight_fraction_leaf**

6）最大叶子节点数**max_leaf_nodes**

7）节点划分最小不纯度**min_impurity_split**
