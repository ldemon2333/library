# Abstract
Previous work showed that simple continuous-valued deep CNNs can be converted into accurate spiking equivalents. These networks did not include
SNNs can potentially offer an efficient way of doing inference because the neurons in the networks are sparsely actived and computations are event-driven.

离散激活，事件驱动计算

Previous work showed that simple continuous-valued deep Convolutional CNNs can be converted into accurate spiking equivalents. These networks did not include certain common operations such as max-pooling, softmax, batch-normalization and Inception-modules.

# 1. Intro
Deep SNNs can be queried for results already after the first output spike is produced, unlike ANNs where the result is available only after all layers have been completely processed.

A more straightforward approach is to take the parameters of a pre-trained ANN and to map them to an equivalent accurate SNN.

One reason is that SNN implementations of many operators that are crucial for improved ANN error rate, such as max-pooling layers, softmax activation functions, and batch-normalization, are non-existent.

# 2 Methods
## 2.1 Theory for Conversion of ANNs into SNNs
The basic principle of converting ANNs into SNNs is that firing rates of spiking neurons should match the graded activations of analog neurons.

SNN 中神经元的脉冲发放频率（firing rate）应该尽量匹配 ANN 中模拟神经元的连续输出值（graded activations）。



## 2.2 Spiking Implementations of ANN Operators
Tile an image to a sequence of n_dt images as input to SNN simulations
- This processing on tile-up images seems inefficient
- however, it is only a software simulation on continous current flow injecting to spiking neurons for n_dt length
- which is ultra power efficient on Neuromorphic hardware
因为是时间步模拟，一个神经元在一个时间步内最多只能发一次脉冲，所以一个神经元**理论上每 Δt 时间最多发一次脉冲**。  因此，最大的可能发放率是

这段内容讨论了 **从 ANN 转换为 SNN 时的一种核心误差来源：由于脉冲神经元（SNN）的“reset-to-zero”机制导致的信息损失**。下面我们逐段解释并总结其关键含义：

---

### 🧠 1. **脉冲发放率（spike rate）近似 ANN 激活**

> > the spike rates are proportional to the ANN activations ai1a_i^1，but reduced by an additive approximation error term...

- 这意味着 SNN 的发放频率（firing rate）理论上应与 ANN 的激活值 ai1a_i^1 成正比。
    
- 但由于离散时间模拟和 reset-to-zero 的机制，**实际发放频率会低于期望值**，这是因为存在：
    
    - **加性误差项（additive error）**
        
    - **乘性误差项（multiplicative error）**（仅在 reset-to-zero 情况下）
        

---

### 🔋 2. **reset-to-zero 的误差机制**

> > 这个误差主要来源于每次脉冲发放时，膜电位被清零（reset to zero），导致之前积累但未达到阈值的“残余电量”被丢弃。

公式部分解释如下：

#### 电压公式：

- 给定输入为常数 zi1z_i^1，每 ni1n_i^1 个时钟周期发射一次脉冲：
    
    ϵi1=Vi1(ni1)−Vthr=ni1⋅zi1−Vthr≥0\epsilon_i^1 = V_i^1(n_i^1) - V_{thr} = n_i^1 \cdot z_i^1 - V_{thr} \ge 0
    
    其中：
    
    - Vi1(ni1)V_i^1(n_i^1) 是膜电位在 ni1n_i^1 步后的值
        
    - VthrV_{thr} 是阈值
        
    - ϵi1\epsilon_i^1 是每次脉冲发射时 **超过阈值但被清除的残余电压**
        

这个残余在 reset 到 0 时 **直接被抛弃**，于是 **平均发放率就变低了**，进而引起信息损失。

---

### 🏞 3. **影响大小和积累效应**

> > For shallow networks and small datasets such as MNIST, this error seems to be a minor problem...

- 在浅层网络 + 简单任务（如 MNIST）中误差影响不大；
    
- 但在 **深层网络中会积累误差**，导致分类准确率下降；
    
- 因为越往后层传播，误差越可能叠加放大。
    

---

### 🧪 4. **控制误差的两个策略**

#### ✅（1）更大的阈值、更小的输入 → 更准，但计算更慢

> > A larger VthrV_{thr} and smaller inputs improve the approximation at the expense of longer integration times.

这样能使每次脉冲发放尽可能逼近理想比例，但需要更多时间积累电压。

#### ✅（2）Diehl 等人的归一化策略

> > By guaranteeing that the ANN activations are too low to drive a neuron above VthrV_{thr} in a single step...

即强制 ANN 的激活值经过缩放后不能一步就超过阈值，推导出一个匹配策略：

zi1=Vthr⋅ai1⇒ϵi1=ni1zi1−Vthrz_i^1 = V_{thr} \cdot a_i^1 \Rightarrow \epsilon_i^1 = n_i^1 z_i^1 - V_{thr}

这样残余电量（误差）就变得非常小。

---

### 🧮 5. **提高精度的另一方式：缩小仿真时间步长**

> > Another obvious way ... is to reduce the simulation time step

虽然能改善误差，但计算成本会增加，因为每个样本需要更多时间步。

---

### 🔚 总结一句话：

> **reset-to-zero 机制在 SNN 中会丢失每次放电后的残余电压，导致脉冲频率偏低、信息丢失，尤其在深层网络中误差会累积；解决办法包括激活归一化、提高阈值或增加时间步数。**


### 2.1.2 Firing Rates in Higher Layers


## 2.2 Spiking Implementations of ANN Operators

### 2.2.1 Converting Biases

### 2.2.2 Parameter Normalization
One source of approximation errors is that in time-stepped simulations of SNNs, the neurons are restricted to a firing rate range of $[0, r_{max}]$, whereas ANNs typically do not have such constraints.

#### 2.2.2.1 Normalization with biases
![[Pasted image 20250421210804.png]]

这样归一化后，无论输入什么样的训练样本，该层所有神经元的激活值都不会超过1，便于控制其对应的 SNN 发放率不超过上限。

#### 2.2.2 Robust normalization
The drawback is that this procedure is prone to be influenced by singular outlier samples that lead to very high activations, while for the majority of the remaining samples, the firing rates will remain considerably below the maximum rate.

**为什么单纯用最大激活值（max activation）来做归一化可能会导致 SNN 性能下降**。
- 存在离群点
- 观察到的最大激活值比第 99.9 百分位数还高了 **三倍以上**。  这说明最大值只是极少数样本或通道产生的，**它远不能代表一般样本的激活水平**。
- **激活值分布有很大方差，存在极端的 outlier**；

因为对大多数输入样本来说，它们激活出的最大值根本达不到那个“全局最大值”归一化的尺度。  也就是说，这些样本会被 **过度缩小**（激活值除以过大的 max），导致响应信号太弱。

结果就是：这层神经元发放（spike）太少，不能有效“驱动”后续层的神经元——整张网络的活动都偏弱，导致分类性能下降。


这段话提出了一种 **更鲁棒的（robust）权重归一化方法**，用来替代传统的“最大值归一化”（max-norm），从而改善 SNN 的发放情况和分类性能。下面我们逐句解析：

---

### 📌 原文：

> We propose a more robust alternative where we set λl to the p-th percentile of the total activity distribution of layer l.

**解释：**  
作者提出：我们不再使用第 ll 层的最大激活值作为归一化因子 λl\lambda^l，而是用它的 **第 p 个百分位数（p-th percentile）**。

比如：

- 如果 p = 99.9，那就表示我们把那一层所有训练样本中所有神经元激活值排序后，取排在前 99.9% 的那个数作为 $\lambda^l$。

这样做的目的就是避免被极端的大值（outliers）所主导。

---

### 📌 原文：

> This choice discards extreme outliers, and increases SNN firing rates for a larger fraction of samples.

**解释：**  
使用百分位值能自动**忽略激活值分布中极端高的 outliers**，  
这样大部分样本的激活值不会被过度缩小，**可以触发更多脉冲（spikes）**，让 SNN 整体更加活跃。

---

### 📌 原文：

> The potential drawback is that a small percentage of neurons will saturate, so choosing the normalization scale involves a trade-off between saturation and insufficient firing.

**解释：**  
这种做法的副作用是：仍然会有一小部分神经元激活值超过了 λl\lambda^l，会导致 **饱和（saturation）**，也就是在 SNN 中发放率达到最大（上限）。  
因此，这是一种**权衡（trade-off）**：

- 若 λ 太大 → 多数神经元发放太少（insufficient firing）；
    
- 若 λ 太小 → 少数神经元饱和（saturation）；
    
- 我们要找一个两者之间的平衡点。
    

---

### 📌 原文：

> In the following, we refer to the percentile p as the “normalization scale,” and note that the “max-norm” method is recovered as the special case p = 100.

**解释：**  
文中将这个百分位值 pp 称为“**归一化尺度（normalization scale）**”。  
当 p=100p = 100 时，也就是用最大值做归一化（就是传统的 max-norm）。

---

### 📌 原文：

> Typical values for p that perform well are in the range [99.0, 99.999].

**解释：**  
实验中表现良好的归一化尺度通常选在 99.0 到 99.999 之间。  
也就是说，虽然仍然很接近最大值，但它已经排除了极端 outlier 的影响。

---

### 📌 原文：

> In general, saturation of a small fraction of neurons leads to a lower degradation of the network classification error rate compared to the case of having spike rates that are too low.

**解释：**  
相比之下，如果是“**少数神经元饱和**”，对分类准确率的影响远小于“**大多数神经元发放太少**”。  
所以宁愿让少数神经元饱和，也要避免整体太“安静”。

---

### 📌 原文：

> This method can be combined with batch-normalization (BN) used during ANN training (Ioffe and Szegedy, 2015), which normalizes the activations in each layer and therefore produces fewer extreme outliers.

**解释：**  
这个归一化方法可以与 **批归一化（Batch Normalization）** 结合使用。  
BN 本身就可以减少激活值的波动和 outliers，因此结合使用可以进一步提升稳定性与性能。

---

BN 是在 channel 维度进行的。

同样，对于卷积层，我们可以在卷积层之后和非线性激活函数之前应用批量规范化。 当卷积有多个输出通道时，我们需要对这些通道的“每个”输出执行批量规范化，每个通道都有自己的拉伸（scale）和偏移（shift）参数，这两个参数都是标量。 假设我们的小批量包含m个样本，并且对于每个通道，卷积的输出具有高度p和宽度q。 那么对于卷积层，我们在每个输出通道的m⋅p⋅q个元素上同时执行每个批量规范化。 因此，在计算平均值和方差时，我们会收集所有空间位置的值，然后在给定通道内应用相同的均值和方差，以便在每个空间位置对值进行规范化。

|方法|操作维度|特点与用途|
|---|---|---|
|**BatchNorm**|每个通道（B, H, W）|常用于 CNN，训练加速，受 batch size 影响|
|**LayerNorm**|每个样本的所有维度（C, H, W）|常用于 Transformer，不依赖 batch size|
|**InstanceNorm**|每个样本每个通道（H, W）|常用于风格迁移，不跨样本统计|
|**GroupNorm**|每个样本的某几组通道（分组）|兼顾 BatchNorm 和 InstanceNorm 的优点|

### 2.2.4 Analog Input to First Hidden Layer
这段话讨论了在将传统人工神经网络（ANN）转换为脉冲神经网络（SNN）时，**输入编码方式**对性能的影响，并提出了一种更稳定、更高效的替代方法。下面是逐句解释和背景说明：

---

## 🔍 背景

由于**事件驱动（event-based）数据集稀缺**，比如：

- DVS128（Hu et al., 2016）
    
- N-MNIST（Rueckauer and Delbruck, 2016）
    

所以很多研究只能使用普通的帧式图像数据集（frame-based datasets），如：

- MNIST（LeCun et al., 1998）
    
- CIFAR（Krizhevsky, 2009）
    

来测试从 ANN 转换得到的 SNN 的分类效果。

---

## 🚨 传统方法的问题（用 Poisson 编码）

> Previous methods [...] transform the analog input activations [...] into Poisson firing rates. But this transformation introduces variability [...] and impairs its performance.

### 🧠 什么是 Poisson 编码？

Poisson 编码会把输入图像的像素值（如灰度值或 RGB 值）变成**脉冲发放率**，并在每个时间步以某个概率随机发放脉冲，比如：

x=0.7⇒70%概率发一个脉冲

这种方法**引入了随机性**（spike variability），虽然常用，但缺点是：

- 引入**不稳定性**（尤其在激活较低时）；
    
- 降低了 SNN 的可预测性和性能；
    
- 神经元可能因为激活太低而“几乎不发火”，导致**欠采样（undersampling）**问题。
    

---

## 💡 本文提出的新方法

> Here, we interpret the analog input activations as constant currents.

### ✅ 新方法思路：

将输入图像像素值（analog）**看作持续的电流输入**（constant current），而不是发放率。 也就是说：

- 不再“用像素值生成脉冲”，
    
- 而是“用像素值作为恒定电流源，直接驱动后续神经元的膜电位”。

![[Pasted image 20250421214051.png]]

## 这个电荷的意义：

- 这个 $z_i^1$​ 是一个常数，不变；
    
- 在 SNN 模拟中，每个时间步都会把这个值加到膜电位上；
    
- 神经元如果膜电位超过阈值，就发一个脉冲；
    
- **因此 SNN 的发放行为从第一隐藏层才开始**，而不是输入层。

### 2.2.5 Spiking Softmax
Previous approachs for ANN-to-SNN conversion did not convert softmax layers, but simply predicted the output class corresponding to the neuron that spiked most during the presentation of the stimulus. However, this approach fails when all neurons in the final layer receive negative inputs, and thus never spike.

这段话讨论了 **在 SNN 中实现 softmax 层** 的三种不同方法，它们都试图在保持脉冲神经网络（SNN）特性的同时，实现传统 ANN 中的 softmax 功能。

下面我们逐段解释：

---

## 🧠 背景：softmax 层在 SNN 中的挑战

在传统 ANN 中，softmax 层用于输出一个**概率分布**，通常作为分类器的最后一层。但在 SNN 中，由于输出是以**脉冲的形式**表达的，如何实现类似功能变得困难。

---

## ✅ 第一种方法（来自 Nessler et al. 2009）：

> output spikes are triggered by an external Poisson generator with fixed firing rate.

### 🔧 机制：

- 每个输出神经元都有一个**外部 Poisson 发放器**；
    
- 神经元本身不会主动发放脉冲；
    
- 而是**累积膜电位**（integrate only）；
    
- 当外部 Poisson 触发器“抽中”某个神经元时，才检查是否发脉冲；
    
- 这时通过 softmax 竞争机制：所有神经元的膜电位进行 softmax 运算，选中谁就谁发脉冲。
    

### ⚠️ 缺点：

- 依赖于外部固定发放率（一个额外超参数）；
    
- 增加了系统复杂性；
    
- 带来**人为的随机性**。
    

---

## ✅ 第二种方法（本文主推）：

> use the resulting values in range of [0, 1] as rate parameters in a Poisson process for each neuron.

### 🔧 机制：

- 每个神经元将膜电位做一次 softmax（无需外部信号）；
    
- 得到每个神经元的概率 pi∈[0,1]p_i \in [0, 1]；
    
- 把这个 pip_i 作为神经元自己的 **Poisson 发放率**；
    
- 即，膜电位高的神经元更容易发脉冲；
    
- 最后根据哪一个神经元**累计发脉冲最多**来判定分类结果。
    

### ✅ 优点：

- 不依赖外部“时钟”；
    
- 没有额外超参数；
    
- 更自然、更符合 SNN 的原生发放机制。
    

---

## ✅ 第三种方法（审稿人建议）：

> simply infer the classification output from the softmax computed on the membrane potentials...

### 🔧 机制：

- 既然是最后一层，那干脆**直接在膜电位上做 softmax**；
    
- 不再生成脉冲；
    
- 直接用 softmax 的输出作为分类依据。
    

### ✅ 优点：

- **最简单最快**；
    
- 不再引入任何随机性（不再需要 Poisson）；
    
- 减少误差，提高准确率；
    
- 非常适用于“不强调纯脉冲实现”的系统。
    

---

## 📌 总结比较：

|方法|是否生成脉冲|随机性|是否需要外部机制|是否推荐|
|---|---|---|---|---|
|**Nessler 方法**|✔（依赖外部触发）|高|是（Poisson 触发器）|❌|
|**本文主推方法**|✔（自己发放）|中|否|✅|
|**第三种方法**|❌（只做 softmax）|无|否|✅（若不坚持纯 SNN）|

### 2.2.6 Spiking Max-Pooling Layers
这段话讨论的是如何在**脉冲神经网络（SNN, Spiking Neural Network）**中实现**最大池化（max-pooling）**，以及为什么这比在传统的**人工神经网络（ANN）**中要难得多。

我来逐句为你解释：

---

> **Most successful ANNs use max-pooling to spatially down-sample feature maps.**  
> 大多数成功的人工神经网络都会使用最大池化（max-pooling）来对特征图（feature maps）进行空间下采样。

➡️ Max-pooling 是 CNN 中非常常见的操作，用来减少特征图的尺寸、降低计算量，并保留最显著的特征（最大值）。

---

> **However, this has not been used in SNNs because computing maxima with spiking neurons is non-trivial.**  
> 然而，在 SNN 中并不常用 max-pooling，因为用脉冲神经元计算最大值并不是一件容易的事。

➡️ 在 SNN 中，信息是通过脉冲（spikes）来编码的，不是连续的实数值。比较哪一个神经元“值更大”就变得很难，因为你并没有直接的数值去比较。

---

> **Instead, simple average pooling used in Cao et al. (2015), Diehl et al. (2015), results in weaker ANNs being trained before conversion.**  
> 相反，一些工作（如 Cao 和 Diehl 的研究）使用了简单的平均池化，但这样训练出来的 ANN 相对较弱，转换成 SNN 后性能受限。

➡️ 这些研究中为了避开 max-pooling 的难点，改用 average pooling（平均池化），但这样通常导致性能下降。

---

> **Lateral inhibition, as suggested in Cao et al. (2015), does not fulfill the job properly, because it only selects the winner, but not the actual maximum firing rate.**  
> Cao 提出的侧抑制（lateral inhibition）方法也不能很好地实现 max-pooling，因为它只是选出了一个“赢家”神经元，并没有真正比较脉冲率的最大值。

➡️ “侧抑制”是一种神经网络中常见的机制，通常意味着一个神经元兴奋时会抑制周围神经元。虽然可以选出一个“最先发放脉冲的”，但不能代表那个神经元发放率最高。

---

> **Another suggestion is to use a temporal Winner-Take-All based on time-to-first-spike encoding, in which the first neuron to fire is considered the maximally firing one (Masquelier and Thorpe, 2007; Orchard et al., 2015b).**  
> 另一种方案是基于“首次发放时间”编码的时间型 Winner-Take-All 策略，将最先发放脉冲的神经元视为“发放率最大”的（参考 Masquelier 和 Orchard 的研究）。

➡️ 这是个聪明的方法，通过哪一个神经元“最早发放”来间接判断它最“强”（最可能具有最大输入）。

---

> **Here we propose a simple mechanism for spiking max-pooling, in which output units contain gating functions that only let spikes from the maximally firing neuron pass, while discarding spikes from other neurons.**  
> 在这项工作中，我们提出了一种简单的脉冲 max-pooling 机制：输出单元使用“门控函数”，只允许来自发放率最高的神经元的脉冲通过，其它神经元的脉冲都被丢弃。

➡️ 作者在这里提出了自己的方法：加一个“门控机制”，只有“发放最多”的神经元的脉冲能传给下一层，其它的被过滤掉。

---

> **The gating function is controlled by computing estimates of the pre-synaptic firing rates, e.g., by computing an online or exponentially weighted average of these rates.**  
> 门控函数是根据前突触神经元的发放率估计来控制的，比如可以用在线计算或指数加权平均的方法来估算这些发放率。

➡️ 怎么知道哪个神经元“发放最多”？作者的方法是对所有输入神经元的发放频率做实时统计，比如用滑动平均或指数衰减平均。

---

> **In practice we found several methods to work well, but demonstrate only results using a finite impulse response filter to control the gating function.**  
> 实验中我们尝试了几种方法，发现都能工作；本文只展示了使用有限脉冲响应（FIR）滤波器控制门控函数的结果。

➡️ 他们最终选择用一个 FIR 滤波器（类似滑动平均）来估算哪个神经元发放最多，从而控制 max-pooling 的“门”。

---

### 总结：

这段话的核心在于：

- **SNN 中实现 max-pooling 比 ANN 难**，因为 spike 是离散事件，没有“最大值”可以直接比较；
    
- **已有方法如平均池化或侧抑制效果不好**；
    
- **作者提出的新方法**：为 max-pooling 区域加“门控函数”，只放过发放最多的神经元的脉冲，具体通过在线估算发放率来实现。
    

这个机制使得 SNN 也能近似地实现 ANN 中的 max-pooling，性能更好。

`torch.fx.GraphModule` 是 `torch.fx` 中的核心类之一，它继承自 `nn.Module`，并包含一个 `Graph` 对象，代表模型的结构。以下是它的基本用法、常用属性以及遍历方式：

---

## 🧠 一、GraphModule 是什么？

```python
from torch.fx import symbolic_trace

traced = symbolic_trace(model)
```

`traced` 就是一个 `GraphModule`，它包含了模型的图结构，可以用来：

- 查看网络结构；
    
- 修改网络结构；
    
- 导出中间表示（IR）；
    
- 可视化模型。
    

---

## 🧩 二、基本属性

### ✅ `graph`

模型的计算图，类型为 `torch.fx.Graph`。它是最重要的属性。

### ✅ `code`

返回自动生成的 Python 源代码。

### ✅ `named_modules()` / `modules()`

继承自 `nn.Module`，用于遍历子模块。

---

## 🔍 三、遍历 GraphModule 中的计算图

图中的每一个操作节点是 `torch.fx.Node` 类型，可以通过以下方式遍历：

```python
for node in traced.graph.nodes:
    print(f"op: {node.op}, target: {node.target}, name: {node.name}, args: {node.args}")
```

### 📌 每个 Node 的常用属性：

|属性|含义|
|---|---|
|`node.op`|节点类型，取值如 `'call_module'`, `'call_function'`, `'call_method'`, `'placeholder'`, `'output'`|
|`node.target`|调用的对象，比如函数名、模块名等|
|`node.name`|节点名|
|`node.args`|该节点的输入参数|
|`node.kwargs`|关键字参数|
|`node.users`|使用该节点结果的其它节点（字典）|

---

## 🛠 四、简单例子

```python
import torch
import torch.nn as nn
from torch.fx import symbolic_trace

class SimpleModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(4, 2)
        self.relu = nn.ReLU()
    def forward(self, x):
        return self.relu(self.fc1(x))

model = SimpleModel()
traced = symbolic_trace(model)

for node in traced.graph.nodes:
    print(f"{node.name:10} | {node.op:15} | {node.target} | args: {node.args}")
```

### 输出示例：

```
x          | placeholder     | x         | args: ()
fc1        | call_module     | fc1       | args: (x,)
relu       | call_module     | relu      | args: (fc1,)
output     | output          | output    | args: (relu,)
```

---

## ✏️ 五、如何修改 GraphModule？

你可以像操作 AST 一样修改图，比如插入、删除、重排节点。

```python
graph = traced.graph
with graph.inserting_after(some_node):
    new_node = graph.call_function(torch.sigmoid, args=(some_node,))

# 重新生成 GraphModule
graph.lint()  # 检查合法性
graph_module = torch.fx.GraphModule(traced, graph)
```

---

## 🖼 六、可视化 Graph

你可以用 `graph.print_tabular()` 打印表格形式的图：

```python
traced.graph.print_tabular()
```

或者使用 `torchviz`、`graphviz` 等工具绘图。

---

如果你有一个具体的模型想尝试遍历或修改图，我可以直接带你一步步操作。要不要试试？

## 2.3 Counting Operations
To obtain the number of operations in the networks during classification.

In a convolutional layer, the fan-in is given by the size of the 2-dimensional convolution kernel multiplied by the number of channels in the previous layer. In a fully-connected layer, the fan-in simply equals  the number of neurons in the preceding layer. 


# 3 Results

### 3.3.1 Transient Dynamics and Voltage Clamp
The error rate of the converted Inception-V3 was initially far from the error rate of the ANN. One main reason is that neurons undergo a transient phase at the beginning of the simulation because a few neurons have large biases or large input weights. During the first few time steps, the membrane potential of each neuron needs to accumulate input spikes before it can produce any output.

这段话描述的是**脉冲神经网络（Spiking Neural Network, SNN）**中神经元膜电位随时间积累输入尖峰（spikes）并最终产生输出的动态过程，特别是在不同结构下（如 1×1 卷积 vs. 大核卷积）其**响应时间和稳定性**的不同影响。

我来逐句为你解释，并补充一些直觉理解👇：

---

### 📌 **第一句：**

> During the first few time steps, the membrane potential of each neuron needs to accumulate input spikes before it can produce any output.

**翻译：** 在最初的几个时间步里，每个神经元的膜电位需要**积累输入尖峰**才能产生输出。

**理解：** SNN 模型不像传统神经网络那样瞬时计算，而是依赖时间上的尖峰积累。刚开始，神经元还没有足够的输入，它不会立即发放（spike），需要一段时间“充电”。

---

### 📌 **第二句：**

> The firing rates of neurons in the first layer need several time steps to converge to a steady rate...

**翻译：** 第一层神经元的发放频率需要几个时间步才能**收敛到稳定的速率**。

**理解：** 即使是最靠近输入的神经元，它们的放电率也不是立即稳定的。需要时间累积足够的刺激才能形成持续稳定的发放。

---

### 📌 **第三句：**

> ...and this convergence time is increased in higher layers that receive transiently varying input.

**翻译：** 而且，如果是高层神经元（接收的是变化的输入），**它们的收敛时间会更长**。

**理解：** 高层神经元接收的是下层已经处理后的尖峰，若这些输入变化剧烈、呈现瞬态（transient）特征，它们的膜电位更难在短时间内稳定。

---

### 📌 **第四句：**

> The convergence time is decreased in neurons that integrate high-frequency input, but increased in neurons integrating spikes at low frequency.

**翻译：** 如果神经元接收到的是**高频尖峰输入**，它的收敛时间会缩短；如果是**低频输入**，收敛时间则会增加。

**理解：** 高频输入意味着“充电”更快，容易达到阈值发放；低频输入则要等更久。这和生物神经系统的响应机制相似。

---

### 📌 **第五句：**

> Another factor contributing to a large transient response are 1×1 convolution layers.

**翻译：** 还有一个会导致**较大瞬时响应（transient response）**的因素是 **1×1 卷积层**。

**理解：** 1×1 卷积层的结构非常稀疏——它只在每个位置沿**通道维度**做点乘，相当于每个神经元只看一个“垂直向下”的通道列。

---

### 📌 **第六句：**

> In these layers, the synaptic input to a single neuron consists only of a single column through the channel-dimension...

**翻译：** 在这些层中，一个神经元的突触输入仅由前一层的一个通道方向的列组成。

**理解：** 每个神经元输入很少，这让它对某一个偏置或单个突触的变化特别敏感。

---

### 📌 **第七句：**

> ...so that the neuron’s bias or a single strongly deviating synaptic input may determine the output dynamics.

**翻译：** 因此，神经元的输出动态可能被**偏置项或某个突触权重的异常值主导**。

**理解：** 如果某个连接特别强，或偏置特别大，它可能主导整个神经元是否发放、什么时候发放——这在复杂结构中是有风险的。

---

### 📌 **第八句：**

> With larger kernels, more spikes are gathered that can outweigh the influence of e.g., a large bias.

**翻译：** 如果使用**更大的卷积核**，能汇集更多的尖峰输入，从而**稀释一个大偏置或异常权重的影响**。

**理解：** 更大的感受野等于更多的信息来源，使得单个异常值不至于主导整个输出，更稳健。

---

### ✅ 总结核心要点：

|特性|影响|
|---|---|
|初始阶段膜电位积累|输出延迟，不立即发放|
|高频 vs. 低频输入|高频 → 快速收敛；低频 → 慢速收敛|
|高层 vs. 低层|高层输入更变化 → 收敛更慢|
|1×1 卷积核|输入稀疏，对偏置/异常权重敏感|
|更大卷积核|稀释偏置影响，响应更稳定|
