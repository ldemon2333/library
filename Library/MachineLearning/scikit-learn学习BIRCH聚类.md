# 1.scikit-learn之BIRCH类
在scikit-learn中，BIRCH类实现了原理篇里讲到的基于特征树CF Tree的聚类。因此要使用BIRCH来聚类，关键是对CF Tree结构参数的处理。

在CF Tree中，几个关键的参数为内部节点的最大CF数B， 叶子节点的最大CF数L， 叶节点每个CF的最大样本半径阈值T。这三个参数定了，CF Tree的结构也基本确定了，最后的聚类效果也基本确定。可以说BIRCH的调参就是调试B,L和T。

至于类别数K，此时反而是可选的，不输入K，则BIRCH会对CF Tree里各叶子节点CF中样本的情况自己决定类别数K值，如果输入K值，则BIRCH会CF Tree里各叶子节点CF进行合并，直到类别数为K。

# 2.BIRCH类参数
1）**threshold**：即叶节点每个CF的最大样本半径阈值T，它决定了每个CF里所有样本形成的超球体的半径阈值。一般来说threshol越小，则CF Tree的建立阶段的规模会越大即BIRCH算法第一阶段所花的时间和内存会越多。但是选择多大以达到聚类效果则需要通过调参决定。默认值是0.5.如果样本的方差较大，则一般需要增大这个默认值。

2）**branching_factor**：即CF Tree内部节点的最大CF数B，以及叶子节点的最大CF数L。这里scikit-learn对这两个参数进行了统一取值。也就是说，branching_factor决定了CF Tree里所有节点的最大CF数。默认是50。如果样本量非常大，比如大于10万，则一般需要增大这个默认值。选择多大的branching_factor以达到聚类效果则需要通过和threshold一起调参决定

3）**n_clusters**：即类别数K，在BIRCH算法是可选的，如果类别数非常多，我们也没有先验知识，则一般输入None，此时BIRCH算法第4阶段不会运行。但是如果我们有类别的先验知识，则推荐输入这个可选的类别值。默认是3，即最终聚为3类。

4）**compute_labels**：布尔值，表示是否标示类别输出，默认是True。一般使用默认值挺好，这样可以看到聚类效果。

在评估各个参数组合的聚类效果时，还是推荐使用Calinski-Harabasz Index，Calinski-Harabasz Index在scikit-learn中对应的方法是metrics.calinski_harabaz_score.



