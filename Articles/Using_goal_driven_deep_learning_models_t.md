# 使用目标驱动深度学习模型研究感觉皮层
> recent developments in computational neuroscience have used goal-driven hierarchical convolutional neural networks (HCNNs) to make strides in modeling neural single-unit and population responses in higher visual cortical areas.

[[Using_goal_driven_deep_learning_models_t.pdf#page=1&selection=11,26,15,21|Using_goal_driven_deep_learning_models_t, 页面 1]]


最近计算神经科学已经使用目标驱动分层次卷积神经网络（HCNN）在高级视觉皮层区域的神经单元和群体反应方面取得了进展。关键技术革新，如何使用HCNN方法去挖掘跟深层次的感觉处理过程。

# What should one expect of a model of sensory cortex? 

>  The core problem is that the natural axes of sensory input space (for example, photoreceptor or hair cell potentials) are not well-aligned with the axes along which high-level behaviorally relevant constructs vary.

[[Using_goal_driven_deep_learning_models_t.pdf#page=1&selection=28,39,31,49|Using_goal_driven_deep_learning_models_t, 页面 1]]

核心问题是感觉输入空间的自然轴（例如，光感受器或毛细胞电位）与高级行为相关结构变化的轴不一致。

比如，对于视觉数据，对象翻译，旋转，深度运动，变形，光照变化等会导致原始输入空间（视网膜）发生复杂的非线性变换。反过来，两个在生态上截然不同的物体图像，（不同人的人脸），在像素空间上非常接近。

> Two foundational empirical observations about cortical sensory systems are that they consist of a series of anatomically distinguish- able but connected areas,(Fig. 1b) and that the initial wave of neural activity during the first 100 ms after a stimulus change unfolds as a cascade along that series of areas

[[Using_goal_driven_deep_learning_models_t.pdf#page=1&selection=41,0,52,0|Using_goal_driven_deep_learning_models_t, 页面 1]]

两条基本经验对于感觉皮层系统：
1. 系统由一系列解剖学上区别但是相连的区域构成
2. 神经冲动会在100ms将会展开传播区域，类似与前向传播

![[Pasted image 20240915163714.png]]
图的解释：
	（b) The ventral visual pathway is the most comprehensively studied sensory cascade.猕猴大脑显示。
		PIT：颞下后皮质
		CIT：central
		AIT：前部
		RGC：视网膜神经节细胞
		LGN：外侧膝状体
		DoG：高斯差分模型
		T($\cdot$)：transformation
	（c）HCNNs是多层神经网络，其每一层都由简单操作（例如过滤、阈值、池化和归一化）的线性-非线性 (LN) 组合组成。每层的滤波器组由一组类似于突触强度的权重组成。滤波器组中的每个滤波器都对应一个不同的模板，类似于具有不同频率和方向的 Gabor 小波；该图显示的模型在第 1 层有四个滤波器，在第 2 层有八个滤波器，依此类推。层内的操作局部应用于输入内的空间块，对应于简单的、大小有限的感受野（红色框）。多层的组成导致原始输入刺激的复杂非线性变换。在每一层，视网膜视物减少，有效感受野大小增加。HCNN 是the ventral visual pathway模型的良好候选者。根据定义，它们是图像可计算的，这意味着它们可以为任意输入图像生成响应；它们也是可映射的，这意味着它们可以以组件方式自然地识别，并且具有中可观察的结构；并且，当正确选择它们的参数时，它们具有预测性，这意味着网络内的各层描述了对模型构建领域之外的大量刺激的神经反应模式。

并且个人在每个阶段的处理过程很简单，线性求和，激活阈值，归一化。

但是大脑是如何做到将输入空间高维非线性输入给它解开呢？

大脑所进行的非线性计算空间的计算量是巨大的。一个巨大的挑战是：

> A major challenge in understanding sensory systems is thus systems identification: identifying which transformations the true biological circuits are using.

[[Using_goal_driven_deep_learning_models_t.pdf#page=1&selection=64,51,67,6|Using_goal_driven_deep_learning_models_t, 页面 1]]

对大脑进行系统识别：识别什么样的变换是真正的生物回路。

> While identifying summaries of neural transfer functions (for example, receptive field characterization) can be useful, solving this systems identification problem ultimately involves producing an encoding model: an algorithm that accepts arbitrary stimulus inputs (for example, any pixel map) and outputs a correct prediction of neural responses to that stimulus. Models cannot be limited just to explaining a narrow phenomenon identified on carefully chosen neurons, defined only for highly controlled and simplified stimuli. Operating on arbitrary stimuli and quantitatively predicting the responses of all neurons in an area are two core criteria that any model of a sensory area must meet (see Box 1).

[[Using_goal_driven_deep_learning_models_t.pdf#page=1&selection=67,7,84,2|Using_goal_driven_deep_learning_models_t, 页面 1]]

尽管识别神经传递函数的概要很有用（比如，感受野区域），但解决这个系统识别的本质问题归结于生成一个编码模型：一种接受任意刺激输入（例如，任何像素图）并输出对该刺激的神经反应的正确预测的算法。模型不能仅限于解释在精心选择的神经元上识别的狭窄现象，仅为高度控制和简化的刺激定义。对任意刺激进行操作并定量预测某个区域内所有神经元的反应是任何感官区域模型必须满足的两个核心标准（见方框 1）。

## Box 1 Minimal criteria for a sensory encoding model
三个准则：
1. Stimulus-computability: The model should accept arbitrary stimuli within the stimulus domain of interest
2. Mappability: The components of the model should correspond to experimentally definable components of the neural system 可映射
3. Predictivity: 可预测性

# 分层卷积神经网络

视觉系统神经科学表明大脑产生不变目标识别行为通过分层有组织一系列的皮层区域，称为the ventral visual stram。后来叫做HCNN模型。

HCNN 是多层堆叠而成，其中包含在传感输入中重复的简单神经回路基序；然后这些层串联起来。

# The motifs in a single HCNN layer
单个HCNN层是LN层（线性-非线性层），这些操作包括flitering，一种线性运算，将输入刺激中的局部块与一组模板进行点积运算，激活，逐点非线性，尤其是修正线性阈值(Relu)或 Sigmoid 型函数，池化，一种非线性聚合操作——通常是局部值的平均值或最大值，还有divisive normalizetion（纠正输出值到一个标准范围）。

这些基本操作存在于HCNN的一个层，映射到一个单独的皮层区域。

在HCNNs中，过滤是由卷积权重共享实现的，意味着相同的卷积过滤块被应用在所有空间位置。因为一样的操作被应用在所有地方，输出空间变化完全受到输入空间的变化。但这与实际大脑不符，生理学显示感觉皮层不太会实现权重共享。这只是对大脑视觉系统的近似，至少在中央视觉部是这样的。

# Deep networks through stacking
感受野是随着层在不断变大的。

多层之后，输出空间分量变少，导致卷积不再有意义，因此后面改接上全连接层。

# HCNNs as a parameterized model family
HCNN特征如下：
- 离散架构参数
- 连续卷积核参数

一个关键目标是确定一个单一的 HCNN 参数设置，其层对应于感兴趣的皮质系统内的不同区域，并准确预测这些区域的响应模式

滤波器参数被认为与突触权重相对应，并且它们的学习算法都是在线更新（反向传播）

# Early models of visual cortex in context

## 手工设计参数


