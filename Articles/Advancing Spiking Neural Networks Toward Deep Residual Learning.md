# Abstract
Residual learning and shortcuts have been evidenced as an important approach for training deep neural networks, but rarely did previous work assessed their applicability to the specifics of SNNs. In this article, we first identify that this negligence leads to impeded information flow and the accompanying degradation problem in a spiking version of vanilla ResNet. 

直接训练 SNN-ResNet


# 1.Introduction
Conversion and surrogate gradient-based 

The basic idea of the former is  that the activation values in a ReLU-based ANN can be approximated by the average firing rages of an SNN under a rate-coding scheme. However, the conversion routine also suffers from inherent defects. An accuracy gap will be caused by the constraints on ANN models and a long simulation with hundreds or thousands of timesteps is required to complete an inference, which leads to extra delay and energe consumption.

The latter method utilizes a surrogate gradient function. One problem of the directly trained SNNs lies in the limited scale of models.

# 2 Preliminaries and motivation
Degradation Problem in SNNs


# 3 Spiking Residual blocks
In this section, we will try to explain why the interblock LIF obstructs the applicability of vanilla ResNet. The advantages of our model are threefold as follows.
1) 畅通的推理流程可以避免无效的残差表示和工作负载不平衡。
2) 实现块动态等距，这是避免梯度消失或爆炸问题的重要指标。
3) 有意保留的主要节能特性，适合在神经形态硬件上进一步实现。
![[Pasted image 20250426194529.png]]
