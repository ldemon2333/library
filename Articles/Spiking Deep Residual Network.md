We propose a shortcut conversion model to appropriately scale continuous-valued activations to match firing rates in SNN, and  a compensation mechanism to reduce the error caused by discretisation.

# 1. Intro
At current stage, how to train a SNN can be categorised into four classes. The first class seeks to make SNN differentiable with some approximations and apply gradient descent. The second class takes inspiration from biological neurons and utilises synaptic plasticity rules. The third class views SNN as a stochastic process and learns with probabilistic inference. However, the three kinds of approaches are not yet well-developed and cannot deal with deep architectures.

这段话指出了目前**训练脉冲神经网络（SNN, Spiking Neural Networks）**的挑战，特别是因为脉冲机制的**不连续性**，导致无法像传统神经网络那样直接应用反向传播算法（backpropagation）。下面我们来逐类解释这四种训练方法及其各自的优缺点：

---

# Related Work
### 🧠 1. **可微分近似 + 梯度下降（Approximate gradients）**

**思路**：  
用某种平滑函数来近似脉冲神经元的发放函数（如 spike 函数的阶跃），从而使得网络在数学上“可导”，以便使用常规的**梯度下降法（SGD）**进行训练。

**方法示例**：

- surrogate gradient（代理梯度）
    
- straight-through estimator（STE）
    

**优点**：

- 可以在端到端的神经网络中应用训练工具（如 PyTorch、TensorFlow）。
    
- 模拟生物特性但保留训练便利。
    

**缺点**：

- 近似并不准确，训练过程可能不稳定。
    
- 难以扩展到深层结构。
    

---

### 🌿 2. **突触可塑性规则（Synaptic Plasticity）**

**思路**：  
模拟生物神经元中的局部学习机制，比如**STDP（Spike-Timing Dependent Plasticity）**，即权重的更新依赖于突触前后的发放时间差。

**优点**：

- 训练过程生物学合理，适合无监督学习。
    
- 适合在神经形态硬件上实现（能耗低）。
    

**缺点**：

- 很难获得精确的全局最优结果。
    
- 不适合复杂任务，如图像分类。
    

---

### 🎲 3. **概率推理（Probabilistic Inference）**

**思路**：  
把 SNN 看作一个**随机过程**（如泊松过程、玻尔兹曼分布），采用**贝叶斯推理**等方式训练网络，比如：

- 贝叶斯神经网络
    
- 马尔科夫链蒙特卡洛（MCMC）
    

**优点**：

- 理论基础扎实，可以解释神经元的随机性。
    
- 有助于构建不确定性估计模型。
    

**缺点**：

- 推理开销大，收敛慢。
    
- 难以扩展到大规模深层网络。
    

---

### 🔁 4. **ANN → SNN 转换（ANN-to-SNN Conversion）**

**思路**：  
先用反向传播训练一个传统的人工神经网络（ANN），然后将其参数（尤其是权重）**映射到等价的 SNN**。常用方法包括：

- 时间编码转换
    
- ReLU 与发放率之间的转换
    

**优点**：

- 利用成熟的 ANN 工具，可训练更深的网络。
    
- 实现精度较高的分类性能（在浅层结构上）。
    

**缺点**：

- 深层网络转换时精度**迅速下降**（如超过 30 层）。
    
- 一些操作（如池化、归一化）难以精确还原为脉冲机制。
    

---

### ✅ 总结：

|方法类别|优点|缺点|
|---|---|---|
|近似梯度|可用反向传播、易实现|难训练深层网络，近似精度受限|
|生物启发可塑性规则|生物真实、能耗低|难用于监督训练、性能弱|
|概率推理|理论扎实、支持不确定性建模|推理复杂、效率低|
|ANN → SNN 转换|精度较高、支持深层结构（有限）|难精确转换，深层性能下降严重|

Previous conversion methods are not applicable to residual network for the structural difference between residual structure and conventional linear structure. 

We design a shortcut conversion model to jointly normalise synaptic weights in spiking residual network.


Conversion methods. 
Perez-Carrasco 等人是最早尝试将 CNN 转为 SNN 的研究者之一，他们的方法大致如下：

1. 使用 **LIF 模型**作为目标 SNN 的神经元。
    
2. 将 CNN 中的权重 **根据 LIF 神经元的参数进行缩放**。
    
3. 使得 SNN 中神经元的 **脉冲发放率 ≈ CNN 中的激活值**，达到等价映射。

One kind of conversion algorithms builds the mapping between ANN and SNN by fiddling the activation function/artificial neuron to approximate the average firing rate of spiking neurons. These methods require training with less common activation functions and are not well compatible with state-of-the-art ANN architectures.

The other kind of conversion algorithms takes the advantage of the non-negativity of ReLU activation to approximate average firing rate.
![[Pasted image 20250426183358.png]]


# Building Spiking ResNet
The conversion of networks comprised of two different types of neurons is based on the postulation that activation in ANN approximates firing rate of spiking neuron.
![[Pasted image 20250422152619.png]]
Sum layer is no longer needed since IF neuron implicitly implements addition during the integration of spikes. Convolution layer is replaced with a layer of synaptic connections that resembles convolutional operation. Weights of convolution layer are mapped to corresponding synapse layer and biases are converted to constant current injected to spiking neurons. 

卷积层被一层类似卷积运算的突触连接层所取代。卷积层的权重被映射到相应的突触层，偏置被转换为注入脉冲神经元的恒定电流。

For average pooling, synaptic weights are fixed to 1/poolsize^2. For max pooling, we use logical comparision to select spikes only from neuron with highest firing rate and inhibit other incoming spikes by setting synaptic weights to zero. An extra layer of IF neuron is added after pooling layer or fully-connected layer to integrate spikes from these types of synaptic connections.
![[Pasted image 20250422155153.png]]

这段讲的是 **如何将 ResNet 的激活值“压缩”到 SNN 中可接受的脉冲发放率范围内**，并**解释了为什么要做归一化、归一化怎么影响整条路径上的权重与输入**。

我们逐句分析：

---

### 🔁 为什么要匹配激活值和发放率范围？

> “Then we let activations in ResNet match the range of firing rates in S-ResNet...”

- 在 SNN 中，一个神经元每个时间步**只能发一个 spike**（也就是最大发放率为 1）；
    
- 而在 ANN 中，ReLU 激活值可能很大（比如 10，100，甚至更多）；
    
- 如果你不做处理，这些高值在 SNN 中就会**丢失信息**，因为 spike 上限为 1。
    

✅ **解决方法**：将 ANN 中的激活值**整体归一化**，使其范围对齐为 [0, 1]。

---

### ❗ 如果不做归一化会怎么样？

> “Otherwise, information carried by high activation values will be lost during the conversion.”

比如：

- 一个 ANN 神经元的激活值是 7；
    
- 转换成 SNN 时，它最多只能发 1 个 spike；
    
- 那相当于把 **7 映射为 1**，但另一个值为 1 的神经元也发一个 spike；
    
- 那么这两者的“信息强度”在 SNN 里就完全一样了，**区分度丧失**，信息丢失。
    

---

### 🧮 如何进行归一化？

> “weights and biases at each layer are normalised by the maximum possible activation that is calculated from the training set.”

- 使用训练集中每一层激活值的 **最大值（或 99.9% 分位数）** 来归一化：

✅ 这样做的目的是让该层的最大输出值 ≈ 1，对应到 SNN 中就是“最大发放率”。

---

### 🔁 注意：归一化影响是传递的！

> “This leads to new weights, biases in the residual block and new input as well due to the change of weight in previous layers”

- 如果第 l−1l-1 层做了归一化，那它的输出（即第 ll 层的输入）也被缩小了；
    
- 那么第 ll 层的激活值也会随之变小；
    
- 所以你**不能只单独归一化某一层**，需要**从输入开始层层考虑整个链条**，包括 shortcut 分支；
    
- 尤其在 ResNet 中，残差块的两个路径（主路径与捷径）要 **一起归一化**，确保激活的加法结果一致。

![[Pasted image 20250422162024.png]]
![[Pasted image 20250422162436.png]]

## 3.3 Compensation of Propagation Error
The normalised activation of ReLU is approximated by the firing rate of spiking neuron. 
![[Pasted image 20250422212829.png]]![[Pasted image 20250422212835.png]]
As is seen from the difference between 7 and 9, the approximation of ReLU activation incurs an amount of deviation at each neuron by the last term in 9.
![[Pasted image 20250422213332.png]]
