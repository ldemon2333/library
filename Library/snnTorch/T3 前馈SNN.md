学会两点
1）如何改进LIF神经元以更加适应深度学习任务
2）实现一个简单的前馈SNN

# 1.Leaky Integrate-and-Fire神经元模型简化

## 1.1 衰减率：$\beta$
在上一篇教程中，我们使用欧拉方法推导出被动式膜模型的如下解：
$$U(t+\Delta t) = (1-\frac{\Delta t}{\tau})U(t) + \frac{\Delta t}{\tau} I_{\rm in}(t)R \tag{1}$$
现在假设没有输入电流，$I_{\rm in}(t)=0 A$
$$U(t+\Delta t) = (1-\frac{\Delta t}{\tau})U(t) \tag{2}$$
设后续的$U$值之比，即$U(t+\Delta t)/U(t)$为膜电位的衰减率，也称为逆时间常数：
$$U(t+\Delta t) = \beta U(t) \tag{3}$$
从$(1)$，可得到：
$$\beta = (1-\frac{\Delta t}{\tau}) \tag{4}$$
为了获得合理的精度，$\Delta t << \tau$

## 1.2 加权输入电流
如果我们假设$t$表示序列时间步长，而不是连续时间，那么我们可以设置$\Delta t = 1$。为了进一步减少超参数的数量，假设$R=1$，从$(4)$开始，这些假设得到：
$$\beta = (1-\frac{1}{C}) \implies (1-\beta)I_{\rm in} = \frac{1}{C}I_{\rm in} \tag{5}$$
输入电流加权为$(1-\beta)$。通过额外假设输入电流对膜电位的瞬时贡献：
$$U[t+1] = \beta U[t] + (1-\beta)I_{\rm in}[t+1] \tag{6}$$
请注意，时间的离散化意味着我们假设每个时间步长$t$都足够短，以至于一个神经元在此间隔内只能发出最大一个脉冲。

在深度学习中，输入的权重因子通常是一个可学习的参数。抛开到目前为止所做的物理上可行的假设，我们将$(1-\beta)$从$(6)$的影响纳入一个可学习的权重$W$中，并相应地将$I_{\rm in}[t]$替换为输入$X[t]$：
$$WX[t] = I_{\rm in}[t] \tag{7}$$
这可以按以下方式解释。$X[t]$ 是输入电压或尖峰，并通过 $W$ 的突触电导缩放以产生对神经元的电流注入。这给了我们以下结果：
$$U[t+1] = \beta U[t] + WX[t+1] \tag{8}$$
在以后的模拟中，$W$和$\beta$的影响是解耦的。$W$是一个可学习的参数，它的更新独立于$\beta$。

## 1.3 脉冲和重置
如果膜超过阈值，神经元就会发出一个输出脉冲：
$$S[t] = \begin{cases} 1, &\text{if}~U[t] > U_{\rm thr} \\

0, &\text{otherwise}\end{cases} \tag{9}$$
如果脉冲被触发，膜电位应该被重置。减法复位机制的模型如下：
$$U[t+1] = \underbrace{\beta U[t]}_\text{decay} + \underbrace{WX[t+1]}_\text{input} - \underbrace{S[t]U_{\rm thr}}_\text{reset} \tag{10}$$
由于$W$是一个可学习的参数，而$U_{thr}$通常被设置为1，这使得$\beta$成为唯一需要指定的超参数。

> 注意: 有些实现可能会做出稍微不同的假设，例如$S[t] \rightarrow S[t+1]$ in $(9)$, 或 $X[t] \rightarrow X[t+1]$ in $(10)$. 上面的推导是在snntorch中使用的，因为我们发现它直观地映射到递归神经网络表示，而没有任何性能变化。

## 1.4 代码实现
在Python中实现这个神经元如下所示:

```python3
def leaky_integrate_and_fire(mem, x, w, beta, threshold=1):
  spk = (mem > threshold) # if membrane exceeds threshold, spk=1, else, 0
  mem = beta * mem + w*x - spk*threshold
  return spk, mem
```

要设置 $β$ ，可以使用公式 $(3)$来定义它，也可以直接对它进行硬编码。在这里，为了演示，我们将使用$(3)$，但在未来，它将被硬编码，因为我们更专注于工作的东西，而不是生物精度。

公式$(3)$告诉我们$β$是膜电位在两个后续时间步长的比率。使用方程的连续时间依赖形式来解决这个问题(假设没有电流注入)，这在之前的教程中推导出来的。

$$U(t) = U_0e^{-\frac{t}{\tau}}$$
$U_0$ 是在 $t=0$时的初始膜电位. 假设这个含时方程是按$t, (t+\Delta t), (t+2\Delta t)~...~$,则我们可以使用 
$$\beta = \frac{U_0e^{-\frac{t+\Delta t}{\tau}}}{U_0e^{-\frac{t}{\tau}}} = \frac{U_0e^{-\frac{t + 2\Delta t}{\tau}}}{U_0e^{-\frac{t+\Delta t}{\tau}}} =~~...$$
$$\implies \beta = e^{-\frac{\Delta t}{\tau}} $$
```text
# set neuronal parameters
delta_t = torch.tensor(1e-3)
tau = torch.tensor(5e-3)
beta = torch.exp(-delta_t/tau)

print(f"The decay rate is: {beta:.3f}")
```
运行一个快速的模拟，检查神经元对阶跃电压输入的正确反应：
```text
num_steps = 200

# initialize inputs/outputs + small step current input
x = torch.cat((torch.zeros(10), torch.ones(190)*0.5), 0)
mem = torch.zeros(1)
spk_out = torch.zeros(1)
mem_rec = []
spk_rec = []

# neuron parameters
w = 0.4
beta = 0.819

# neuron simulation
for step in range(num_steps):
  spk, mem = leaky_integrate_and_fire(mem, x[step], w=w, beta=beta)
  mem_rec.append(mem)
  spk_rec.append(spk)

# convert lists to tensors
mem_rec = torch.stack(mem_rec)
spk_rec = torch.stack(spk_rec)

plot_cur_mem_spk(x*w, mem_rec, spk_rec, thr_line=1,ylim_max1=0.5,
                 title="LIF Neuron Model With Weighted Step Voltage")
```

# 2.snntorch中的泄露神经元模型
同样的事情可以通过实例化snn.Leaky的方式来实现。与我们使用snn的方式类似。在之前的教程中使用Lapicque，但是超参数更少：
```text
lif1 = snn.Leaky(beta=0.8)
```
神经元模型现在存储在lif1中，要使用这个神经元：
**Inputs**

* `cur_in`: each element of $W\times X[t]$ is sequentially passed as an input

* `mem`: the previous step membrane potential, $U[t-1]$, is also passed as input.

**Outputs**

* `spk_out`: output spike $S[t]$ ('1' if there is a spike; '0' if there is no spike)

* `mem`: membrane potential $U[t]$ of the present step

这些都需要是torch.Tensor类型。请注意，这里我们假设输入电流在传递到snn.Leaky之前已经被加权。当我们构建一个网络规模的模型时，这将更有意义。此外，公式10被时间向后移了一步，而不失一般性。

将此图与手动导出的泄漏积分和激发神经元进行比较。膜电位重置稍弱：即，它使用*软重置*。这样做是故意的，因为它可以在一些深度学习基准上实现更好的性能。使用的方程是：
$$U[t+1] = \underbrace{\beta U[t]}_\text{decay} + \underbrace{WX[t+1]}_\text{input} - \underbrace{\beta S[t]U_{\rm thr}}_\text{soft reset} \tag{11}$$
# 3.前馈脉冲神经网络
到目前为止，我们只考虑了单个神经元如何响应输入刺激。snntorch可以直接将其扩展到深度神经网络。在本节中，我们将创建一个维度为784-1000-10的3层全连接神经网络。与我们到目前为止的模拟相比，每个神经元现在将整合更多的输入峰值。

![[Pasted image 20241010114502.png]]

PyTorch用于形成神经元之间的连接，而snnTorch用于创建神经元。首先，初始化所有层。
```text
# layer parameters
num_inputs = 784
num_hidden = 1000
num_outputs = 10
beta = 0.99

# initialize layers
fc1 = nn.Linear(num_inputs, num_hidden)
lif1 = snn.Leaky(beta=beta)
fc2 = nn.Linear(num_hidden, num_outputs)
lif2 = snn.Leaky(beta=beta)
```
接下来，初始化每个脉冲神经元的隐藏变量和输出。随着网络规模的增加，这变得更加繁琐。静态方法init_leaky()可用于处理此问题。snntorch中的所有神经元都有自己的初始化方法，遵循相同的语法，例如init_lapicque()。在第一次前向传递期间，隐藏状态的形状根据输入数据维度自动初始化。
```text
# Initialize hidden states
mem1 = lif1.init_leaky()
mem2 = lif2.init_leaky()

# record outputs
mem2_rec = []
spk1_rec = []
spk2_rec = []
```
创建一个输入脉冲序列传递到网络。有200个时间步来模拟784个输入神经元，即输入最初的维度是 $200 \times 784$ 。然而，神经网络通常以小批量处理数据。snntorch使用时间优先的维度:

$[time \times batch\_size \times feature\_dimensions]$

因此，将输入沿dim=1进行unsqueeze，以表示一个batch数据。输入张量的维度必须是200 $\times$ 1 $\times$ 784：
```text
spk_in = spikegen.rate_conv(torch.rand((200, 784))).unsqueeze(1)
print(f"Dimensions of spk_in: {spk_in.size()}")
```
现在终于可以运行一个完整的模拟了。

考虑PyTorch和snnTorch如何一起工作的一种直观方法是，PyTorch将神经元路由在一起，而snnTorch将结果加载到脉冲神经元模型中。在编码网络方面，这些脉冲神经元可以被视为时变激活函数。

下面是对发生的事情的顺序描述:

*  The $i^{th}$ input from `spk_in` to the $j^{th}$ neuron is weighted by the parameters initialized in `nn.Linear`: $X_{i} \times W_{ij}$

* This generates the input current term from Equation $(10)$, contributing to $U[t+1]$ of the spiking neuron

* If $U[t+1] > U_{\rm thr}$, 那么从这个神经元触发一个脉冲

* 该脉冲由第二层权重加权，并对所有输入、权重和神经元重复上述过程。

* 如果没有脉冲，那么什么都不会传递给突触后神经元。

到目前为止，与我们的模拟唯一的区别是，我们现在用nn.Linear生成的权重来缩放输入电流。而非手动设置。

```text
# network simulation
for step in range(num_steps):
    cur1 = fc1(spk_in[step]) # post-synaptic current <-- spk_in x weight
    spk1, mem1 = lif1(cur1, mem1) # mem[t+1] <--post-syn current + decayed membrane
    cur2 = fc2(spk1)
    spk2, mem2 = lif2(cur2, mem2)

    mem2_rec.append(mem2)
    spk1_rec.append(spk1)
    spk2_rec.append(spk2)

# convert lists to tensors
mem2_rec = torch.stack(mem2_rec)
spk1_rec = torch.stack(spk1_rec)
spk2_rec = torch.stack(spk2_rec)

plot_snn_spikes(spk_in, spk1_rec, spk2_rec, "Fully Connected Spiking Neural Network")
```

在这个阶段，尖峰没有任何实际意义。输入和权重都是随机初始化的，没有进行任何训练。但峰值应该看起来是从第一层传播到输出。如果你没有看到任何峰值，那么你可能在权重初始化中运气不太好——你可能想尝试重新运行最后四个代码块。