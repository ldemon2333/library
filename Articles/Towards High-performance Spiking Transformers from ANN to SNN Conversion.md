This is due to unique nonlinear modules such as layernorm and GELU in Transformers that differ from the ReLU function in CNNs. 这些模块需要同一层内的神经元之间进行交互，并表现出非线性特性，因此很难通过单个神经元的线性分段量化来实现精确的转换。

The main challenge lies in handling non-linear modules.
- 我们分析了 Transformer 中非线性模块转换的挑战，并提出了一种名为“期望补偿模块”的新颖解决方案。该模块利用前几个时间步的信息来计算当前时间步的预期输出。该模块克服了传统方法的局限性，同时最大程度地减少了功耗。
- 为了克服 Transformer 转换过程中准确率提升缓慢的问题，我们提出了一种多阈值神经元和相应的并行参数归一化方法，从而大幅降低了功耗要求并显著减少了延迟。

# 2 Related Works


# 3 Preliminaries
## 3.1 ANN-SNN conversion theory
![[Pasted image 20250506145902.png]]


![[Pasted image 20250506154830.png]]


