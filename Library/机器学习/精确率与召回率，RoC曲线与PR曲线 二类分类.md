# 1. TP, FP, TN, FN
true positives，TP：预测为正样本，实际也为正样本的特征数
false positives，FP：预测为正样本，实际为负样本的特征数
true negative，TN：预测为负样本，实际为负样本的特征数
false negative，FN：预测为负样本，实际为正样本的特征数

![[Pasted image 20240924094145.png]]

# 2. 精确率(precision)，召回率(Recall)与特异性(specificity)
精确率（Precision）的定义在上图可以看出，是绿色半圆除以红色绿色组成的圆。严格的数学定义如下：
$$
P=\frac{TP}{TP+FP}
$$
召回率（Recall）的定义是绿色半圆除以左边的长方形：
$$
R=\frac{TP}{TP+FN}
$$
特异性（specificity）是右边长方形去掉红色半圆部分后除以右边的长方形：
$$
S = \frac{TN}{FP+TN}
$$
有时也用一个$F_1$值来综合评估精确率和召回率，它是精确率和召回率的调和均值。当精确率和召回率都高时，$F_1$值也会高。
$$
\frac{2}{F_1} = \frac{1}{P} + \frac{1}{R}
$$
有时候我们对精确率和召回率并不是一视同仁，比如有时候我们更加重视精确率。我们用一个参数$\beta$来度量两者之间的关系。如果$\beta>1$，召回率有更大影响，如果$\beta<1$，精确率有更大影响。自然，当$\beta=1$的时候，精确率和召回率影响力相同，和$F_1$形式一样。
$$
F_\beta = \frac{(1+\beta^2)*P*R}{\beta^2*P + R}
$$
此外还有灵敏度(true positive rate, TPR)，它是所有实际正例中，正确识别的正例比例，它和召回率的表达式一样

另一个是1-特异度(false positive rate, FPR)，它是实际负例中，错误的识别为正例的负例比例。
$$
FPR = \frac{FP}{FP + TN }
$$

# 3. RoC曲线和PR曲线
以TPR为y轴，以FPR为x轴，我们就直接得到了RoC曲线。从FPR和TPR的定义可以理解，TPR越高，FPR越小，我们的模型和算法就越高效。也就是画出来的RoC曲线越靠近左上越好。如下图左图所示，从几何的角度讲，RoC曲线下方的面积越大，则模型越优。所以我们用RoC曲线下的面积，即AUC(Area Under Curve)值来作为算法和模型好坏的标准。

![[Pasted image 20240924100742.png]]

以精确率为y轴，以召回率为x轴，我们就得到了PR曲线。精确率越高，召回率越高，我们的模型和算法就越高效，画出来的PR曲线越靠近右上越好。

```py
from sklearn.metrics import roc_auc_score

# 假设我们有真实标签和预测概率
y_true = [0, 1, 1, 0, 1]
y_scores = [0.1, 0.4, 0.35, 0.8, 0.7]

# 计算AUC
auc = roc_auc_score(y_true, y_scores)
print("ROC AUC Score:", auc)
```

`roc_auc_score` 函数的主要参数有：

1. **y_true**: 真实的标签，通常是一个数组或列表，包含0和1（负类和正类）。
2. **y_score**: 模型的预测概率或决策函数的输出，通常是正类的概率。
3. **average**: 指定在多分类情况下的计算方式，常用的选项有 `'macro'`, `'micro'`, 和 `'weighted'`，默认为 `None`。
4. **multi_class**: 指定多分类的情况下，使用的策略，选项有 `'raise'`, `'ovr'`（一对其余），和 `'multinomial'`（多项式）。
5. **sample_weight**: 可选的样本权重，用于给不同样本设置不同的重要性。



