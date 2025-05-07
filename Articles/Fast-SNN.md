脉冲神经网络 (SNN) 凭借其事件驱动的表征，在计算和能效方面较传统人工神经网络 (ANN) 更具优势。SNN 还将 ANN 中的权重乘法替换为加法，从而提高能效并降低计算强度。然而，由于离散脉冲函数的存在，训练深度 SNN 仍然是一项挑战。一种常用的规避这一挑战的方法是将 ANN 转换为 SNN。然而，由于量化误差和累积误差的存在，通常需要大量的时间步长（高推理延迟）才能实现高性能，这抵消了 SNN 的优势。为此，本文提出了一种快速 SNN，它可以在低延迟下实现高性能。我们展示了 SNN 中时间量化与 ANN 中空间量化之间的等效映射，并在此基础上将量化误差的最小化转移到量化 ANN 训练中。通过最小化量化误差，我们证明了序列误差是累积误差的主要原因，并通过引入带符号中频神经元模型和逐层微调机制解决了这一问题。我们的方法在图像分类、目标检测和语义分割等各种计算机视觉任务上实现了最佳性能和低延迟。代码可访问：https://github.com/yangfan-hu/Fast-SNN。

# 1 Intro
However, due to the sparsity of spike trains, directly training SNNs with BPTT is inefficient in both computation and memory with prevalent computing devicecs. Furthermore, the surrogate gradients would cause the vanishing or exploding gradient problem for deep networks, making direct training methods less effective for tasks of high complexity.

In contrast, rate-coded ANN-to-SNN conversion algorithms employ the same training procedures as ANNs, which benefit from the efficient computation of ANN training algorithms. Nevertheless, all existing methods suffer from quantization error and accumulating error, resulting in performance degradation during conversion, especially when the latency is short.

# 2 Related work
## 2.1 Direct Training with Surrogate Gradients
With these emerging techniques, direct training methods can build SNNs low latency and high accuracy. 

However, due to the sparsity of spike trains, directly training SNNs with BPTT is inefficient in both computation and memory with prevalent computing devices. An SNN with T time steps propagates T times iteratively during forward and backward propagation. Compared with a full-precision ANN, it consumes more memory and requires about T times more computation time.

For these reasons, we focus on ANN-to-SNN conversion methods for building SNNs.

## 2.2 ANN2SNN conversion


## 3.1 Activation Equivalence between ANNs and SNNs
