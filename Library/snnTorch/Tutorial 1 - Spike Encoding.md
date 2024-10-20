	Learn how to use snnTorch to：
- 转换数据集into spiking datasets
- 如何可视化spiking datasets
- how to generate random spike trains.

# 介绍
当视网膜将光子转化为脉冲时，我们所看到的就是光。当挥发分子转化为脉冲时，我们闻到的就是气味。当神经末梢将触觉压力变成脉冲时，我们就会感觉到触觉。大脑处理的是神经元传导的全局电流的脉冲。

如果我们的最终目标是构建一个脉冲神经网络(SNN)，那么在输入端使用脉冲也是由意义的。虽然使用非脉冲输入非常常见，但编码数据的优势来自三个“S”：**脉冲（Spikes）、稀疏性（Sparsity）和静息抑制（Static Suppression）**。

- **脉冲**：生物神经元通过脉冲进行处理和交流，脉冲是大约100 mV振幅的电脉冲。许多神经元的计算模型将电压突发简化为离散的单比特事件:“1”或“0”。在硬件中，这比高精度值表示要简单得多。
- **稀疏性**：神经元大部分时间处于静息状态，在任何给定的时间都将大多数激活状态静音为零。不仅稀疏向量)/张量(具有大量的零)存储起来很便宜，而且我们需要将稀疏激活与突触权重相乘。如果大多数值都乘以“0”，那么我们不需要从内存中读取许多网络参数。这意味着神经形态硬件可以非常高效。
- **静息抑制**（又称**事件驱动处理**）：感知模块只在有新信息需要处理时才处理信息。图像中的每个像素都会响应亮度的变化，因此图像的大部分被遮挡。传统的信号处理要求所有通道/像素都遵循全局采样/快门速率，这降低了感知发生的频率。事件驱动处理现在只通过阻塞不变的输入来提高稀疏性和功耗效率，但它通常可以实现更快的处理速度。

![[Pasted image 20241008203754.png]]

在本教程中，我们将假设我们有一些非尖峰输入数据(即 MNIST 数据集) ，并且我们希望使用一些不同的技术将其编码为峰值。

# 2.脉冲编码
SNN用来探索时变数据，但是MNIST不是时变数据，这里有两种方式将MNIST使用SNN：
1. 在每个时间步重复传入相同的训练样本。这就像转变MNIST为一个静态不变的视频。每个X的元素都归一化到0到1之间的高精度值。$X_{ij}\in [0,1]$

![[Pasted image 20241008210143.png]]

2. 将输入转换为序列长度为num_steps的尖峰序列，每个特征/像素变为一个离散值$X_{i,j}\in\{0,1\}$。In this case, MNIST is converted into a time-varying sequence of spikes that features a relation to the original image.

![[Pasted image 20241008211006.png]]

第一种方法非常简单，没有充分利用snn的时间动态性。因此，让我详细考虑一下第二种方法

## 2.1 数据到脉冲的编码(Encode)
三种常见的编码方式如下：

1. 频率编码（Rate coding）使用输入特征来确定脉冲的频率
2. 延迟编码（Latency coding）使用输入特性来确定脉冲的时间
3. 增量调制（Delta modulation）使用输入特征的时间变化来产生脉冲信号

How do these differ?

1. _Rate coding_ uses input features to determine spiking **frequency**
    
2. _Latency coding_ uses input features to determine spike **timing**
    
3. _Delta modulation_ uses the temporal **change** of input features to generate spikes

### 2.1.1 对MNIST频率编码
每个归一化的输入特征 $X_{ij}$ 被用作事件(Spike)在任何给定时间步长发生的概率，返回一个频率编码值 $R_{ij}$。这可以看作是一个伯努利分布$R_{ij} - B(n,p)$，其中试验次数为 $n=1$，成功的概率为 $p=X_{ij}$.

对于MNIST图像，脉冲的概率对应于像素值。白色像素对应100%的脉冲概率，而黑色像素永远不会产生脉冲。
```python
from snntorch import spikegen

# Iterate through minibatches
data = iter(train_loader)
data_it, targets_it = next(data)

# Spiking Data
spike_data = spikegen.rate(data_it, num_steps=num_steps)
```

如果输入落在$[0,1]$ 之外，则不再表示概率。这样的情况会被自动裁剪，以确保特征代表一个概率。
![[Pasted image 20241008212552.png]]

### 2.1.2 Rate Coding 总结
频率编码的想法实际上是相当有争议的。虽然我们相当确信频率编码发生在我们的感觉外围，但我们不相信大脑皮层在全局范围内将信息编码为频率频率。几个令人信服的原因包括：

- 功耗：自然优化的效率。完成任何任务都需要多个脉冲，而每个脉冲都会消耗能量。事实上，Olshausen和Field的工作表明，速率编码最多只能解释初级视觉皮层(V1)中15%的神经元的活动。它不太可能是大脑中唯一的机制，因为大脑既资源受限又高效。
- 反应响应时间：我们知道人类的反应时间大约是250ms。如果人类大脑中神经元的平均放电速率为10Hz，那么在我们的反应时间尺度内，我们只能处理大约2个峰值。

那么，如果频率编码对功率效率或延迟不是最优的，我们为什么还要使用它呢？即使我们的大脑不以频率处理数据，我们也相当确定我们的生物传感器可以。通过显示巨大的噪声鲁棒性，可以部分抵消功率/延迟的缺点：即使不能产生一些脉冲信号也没关系，因为它们来自的地方还有很多。

此外，你可能看过Hebbian的工作。如果有大量的脉冲，这可能表明有大量的学习。在某些情况下，训练SNN被证明是具有挑战性的，通过脉冲码鼓励更多的激活脉冲是一个可能的解决方案。

几乎可以肯定，脉冲编码是与大脑中的其他编码方案一起工作的。我们将在接下来的章节中讨论其他的编码机制。

## 2.2 Latency coding 延迟编码
时序编码捕获神经元的精确放电时间信息；单个脉冲比依赖于发射频率的频率码具有更多的含义。虽然这增加了对噪声的敏感性，但也可以将运行SNN算法的硬件的功耗降低几个数量级。

spikegen.latency是snntorch中一个函数，它允许每个输入在整个扫描过程中最多触发一次。

接近1的特征会更早触发，接近0的特征会更晚触发。也就是说，在我们的MNIST情况下，亮像素会更早触发，暗像素会更晚触发。

下面的代码块解释了它的工作原理。如果你忘记了电路理论或概率论，别担心！重要的是：**大的输入意味着快速的峰值；小的输入意味着峰值的延迟**。

### 2.2.1 延迟编码机制的推导
默认情况下，脉冲时序是通过将输入特征处理为RC电路的电流注入 $I_{in}$ 来计算的。这个电流使电容上的电荷（电势）增加 $V(t)$ 。我们假设有一个触发电压 $V_{thr}$ ，一旦达到，就会产生一个脉冲。那么问题就变成了：**对于给定的输入电流（以及等效的输入特征），脉冲产生需要多长时间**？

从基尔霍夫电流定律开始，$I_{in} = I_R + I_C$，其余的推导使我们得到时间和输入之间的对数关系。

![[Pasted image 20241009213235.png]]

下面的函数使用上述结果将强度$X_{ij}\in [0,1]$的特征转换为延迟编码响应$L_{ij}$。
```python
def convert_to_time(data, tau=5, threshold=0.01):
  spike_time = tau * torch.log(data / (data - threshold))
  return spike_time 
```
现在，使用上述函数可视化输入特征强度与其对应的脉冲时间之间的关系。
```python
raw_input = torch.arange(0, 5, 0.05) # tensor from 0 to 5
spike_times = convert_to_time(raw_input)

plt.plot(raw_input, spike_times)
plt.xlabel('Input Value')
plt.ylabel('Spike Time (s)')
plt.show()
```
值越小，峰值出现的时间越晚，具有指数依赖性。

向量spike_times包含触发脉冲的时间，而不是包含尖峰本身（1和0）的稀疏张量。 在运行SNN模拟时，我们需要1/0表示来获得使用峰值的所有优势。

整个过程可以使用spikegen.latency自动完成，我们从data_it中传入MNIST数据集的一个minibatch：
```text
spike_data = spikegen.latency(data_it, num_steps=100, tau=5, threshold=0.01)
```

其中一些参数含义如下：

- tau：电路的RC时间常数。默认情况下，输入特性被视为注入RC电路的恒定电流。更高的tau将导致更慢的发射。
- threshold：膜电位触发阈值。低于此阈值的输入值没有闭合形式的解，因为输入电流不足以驱动膜达到阈值。所有低于阈值的值都被裁剪并分配给最后的时间步长。
## 2.3 增量调制编码
有理论认为，视网膜是适应性的: 它只会处理信息时，有新的东西处理。如果你的视野没有变化，那么你的感光细胞就不太容易被激活。

也就是说: 生物学是事件驱动的，神经元在变化中茁壮成长。

Delta 调制是基于事件驱动的峰值。The `snntorch.delta` 函数接受一个时间序列张量作为输入。它在所有时间步骤中采用每个后续特性之间的差。默认情况下，如果差异为正且大于阈值$V_{thr}$，就会产生一个spike

![[Pasted image 20241008225245.png]]
	
