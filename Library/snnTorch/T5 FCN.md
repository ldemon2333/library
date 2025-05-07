- 了解脉冲神经元是如何作为递归网络实现的
- 了解时间上的反向传播，以及snn中的相关挑战，如峰值的不可微性
- 在静态MNIST数据集上训练一个全连接网络

在本教程的最后，将实现一个基本的监督学习算法。我们将使用原始静态MNIST数据集，并使用梯度下降训练多层全连接脉冲神经网络来进行图像分类。

# 1.snn的循环表示
一个Leaky IF神经元的递归表示为：
$$U[t+1] = \underbrace{\beta U[t]}_\text{decay} + \underbrace{WX[t+1]}_\text{input} - \underbrace{R[t]}_\text{reset} \tag{1}$$
其中输入突触电流被解释为$I_{\rm in}[t] = WX[t]$，而$X[t]$可以是任意的脉冲输入、阶跃时变电压或无权阶跃时变电流。脉冲用下面公式表示，如果膜电位超过阈值，就会发出一个脉冲：
$$S[t] = \begin{cases} 1, &\text{if}~U[t] > U_{\rm thr} \\

0, &\text{otherwise}\end{cases} \tag{2}$$
这种离散递归形式的脉冲神经元模型几乎完美地利用了训练递归神经网络(rnn)和基于序列的模型的发展。这用膜电位衰减的隐式递归连接来说明，并与显示递归进行区分。
![[Pasted image 20241011202552.png]]

传统的rnn将 $β$ 视为一个可学习的参数。这对于snn也是可能的，尽管默认情况下，它们被视为超参数。这用超参数搜索取代了梯度消失和梯度爆炸问题。

# 2.尖峰的不可微性
## 2.1 使用反向传播算法进行训练
用（2）表示$S$和$U$之间关系的另一种方法是：
$$S[t] = \Theta(U[t] - U_{\rm thr}) \tag{3}$$
其中$\Theta(\cdot)$是Heaviside阶跃函数：
![[Pasted image 20241011202815.png]]

训练这种形式的网络带来了一些挑战
![[Pasted image 20241011203003.png]]
目标是使用相对于权重的损失梯度来训练网络，以便更新权重以最小化损失。
$$\frac{\partial \mathcal{L}}{\partial W} =

\frac{\partial \mathcal{L}}{\partial S}

\underbrace{\frac{\partial S}{\partial U}}_{\{0, \infty\}}

\frac{\partial U}{\partial I}\

\frac{\partial I}{\partial W}\ \tag{4}$$
从 $(1)$, $\partial I/\partial W=X$, 以及 $\partial U/\partial I=1$. 虽然我们还没有定义损失函数，我们可以假设 $\partial \mathcal{L}/\partial S$ 具有解析解，其形式类似于交叉熵或均方误差损失。

然而，我们要处理的术语是$\partial S/\partial U$。Heaviside阶跃函数（3）的导数是Dirac Delta函数，除了阈值$U_{\rm thr} = \theta$ 趋于无穷外，其他地方的值都为0，这意味着梯度几乎总是0到0（或饱和），并且无法进行学习。这被称为死神经元问题。

## 2.2 克服死神经元问题
解决死神经元问题的最常见方法是保持Heaviside函数在正向传递时的状态不变，但将导数项 $\partial S/\partial U$替换为在反向传递时不会终止学习过程的项，即 $\partial \tilde{S}/\partial U$ 。这可能听起来很奇怪，但事实证明，神经网络对这种近似是相当鲁棒的。这通常被称为代理梯度方法。

使用梯度代理有多种选择，现在我们应用了一个简单的近似，snnTorch中的默认方法(从 v0.6.0开始)是使用arcTangent函数平滑单位阶跃函数，所使用的反向传播导数是：
$$ \frac{\partial \tilde{S}}{\partial U} \leftarrow \frac{1}{\pi}\frac{1}{(1+[U\pi]^2)} \tag{5}$$
也可以用$S$本身来近似

如果 $S$ 没有峰值，那么峰值梯度项为 0 。如果 $S$ 峰值，则梯度项为 1 。这看起来只是像ReLU函数的梯度移位到阈值。这种方法被称为Spike-Operator方法，在下面的论文中有更详细的描述:

>Jason K. Eshraghian, Max Ward, Xinxin Wang, Gregor Lenz, Girish Dwivedi, Mohammed Bennamoun, Doo Seok Jeong, and Wei D. Lu. "Training Spiking Neural Networks Using Lessons From Deep Learning". arXiv, 2021.

脉冲运算符（Spike-Operator）将梯度计算分成两个块：一个是神经元发出脉冲的地方，另一个是静默的地方：
- 静默：如果神经元处于静默状态，那么可以通过将细胞膜缩放0来获得脉冲响应
$$ S=U\times 0 \rightarrow \partial \tilde{S}/\partial U=0$$
- 脉冲：如果神经元是脉冲，那么假设$U\approx U_{thr}$，脉冲响应可以通过将膜缩放1得到：
$$ S=U\times 1 \rightarrow \partial \tilde{S}/\partial U=1$$

```python3
# Leaky neuron model, overriding the backward pass with a custom function
class LeakySurrogate(nn.Module):
  def __init__(self, beta, threshold=1.0):
      super(LeakySurrogate, self).__init__()

      # initialize decay rate beta and threshold
      self.beta = beta
      self.threshold = threshold
      self.spike_op = self.SpikeOperator.apply
  
  # the forward function is called each time we call Leaky
  def forward(self, input_, mem):
    spk = self.spike_op((mem-self.threshold))  # call the Heaviside function
    reset = (spk * self.threshold).detach() # removes spike_op gradient from reset
    mem = self.beta * mem + input_ - reset # Eq (1)
    return spk, mem

  # Forward pass: Heaviside function
  # Backward pass: Override Dirac Delta with the Spike itself
  @staticmethod
  class SpikeOperator(torch.autograd.Function):
      @staticmethod
      def forward(ctx, mem):
          spk = (mem > 0).float() # Heaviside on the forward pass: Eq(2)
          ctx.save_for_backward(spk)  # store the spike for use in the backward pass
          return spk

      @staticmethod
      def backward(ctx, grad_output):
          (spk,) = ctx.saved_tensors  # retrieve the spike 
          grad = grad_output * spk # scale the gradient by the spike: 1/0
          return grad
```

请注意，重置机制与计算图是分离的，因为代理梯度只应用于$\partial S/\partial U$，而不是$\partial R/\partial U$。

上面的神经元实例化如下：
```text
lif1 = LeakySurrogate(beta=0.9)
```
这个神经元可以使用for循环来模拟，就像在之前的教程中一样，而PyTorch的自动微分机制在背景中跟踪梯度

或者，同样的事情可以通过调用snn.Leaky

事实上，每次你从snnTorch中调用任何神经元模型时，都会默认对其应用Spike算子代理梯度：
```text
lif1 = snn.Leaky(beta=0.9)
```

# 3.BPTT(Backprop Through Time)
公式（4）只计算单个时间步的梯度（称为下图中的即使影响），但反向传播（BPTT）算法计算从到所有后代的梯度，并将它们相加。

权重$W$在每个时间步长应用，因此假设在每个时间步长也计算损失。权重对当前损失和历史损失的影响必须相加，以定义全局梯度：
$$\frac{\partial \mathcal{L}}{\partial W}=\sum_t \frac{\partial\mathcal{L}[t]}{\partial W} =

\sum_t \sum_{s\leq t} \frac{\partial\mathcal{L}[t]}{\partial W[s]}\frac{\partial W[s]}{\partial W} \tag{5} $$
（5）的意义在于确保因果关系：通过约束$s\leq t$，我们只考虑$W$对损失的直接影响和先前影响的贡献。循环系统限制在所有步骤中共享权重$W[0]=W[1] =~... ~ = W$。因此，$W[s]$的变化将对所有$W$产生相同的影响，这意味着$\partial W[s]/\partial W=1$：
$$\frac{\partial \mathcal{L}}{\partial W}=

\sum_t \sum_{s\leq t} \frac{\partial\mathcal{L}[t]}{\partial W[s]} \tag{6} $$
例如，隔离$s = t-1$的先验影响；这意味着向后传递必须在时间上后退一步。$W[t-1]$对损失的影响为：
$$\frac{\partial \mathcal{L}[t]}{\partial W[t-1]} =

\frac{\partial \mathcal{L}[t]}{\partial S[t]}

\underbrace{\frac{\partial \tilde{S}[t]}{\partial U[t]}}_{Eq.~(5)}

\underbrace{\frac{\partial U[t]}{\partial U[t-1]}}_\beta

\underbrace{\frac{\partial U[t-1]}{\partial I[t-1]}}_1

\underbrace{\frac{\partial I[t-1]}{\partial W[t-1]}}_{X[t-1]} \tag{7}$$
我们已经处理了（4）的所有项，除了$\partial U[t]/\partial U[t-1]$。从（1）开始，时间导数项的$\beta$。如果我们真的想这样做，我们现在已经知道了足够多的知识，可以辛苦地手动计算每个时间步的每个权重的导数，对于单个神经元来说，它看起来像这样:
![[Pasted image 20241011210501.png]]
The reset mechanism has been omitted from the above figure. In snnTorch, reset is included in the forward-pass, but detached from the backward pass.


# 4.设置Loss/输出解码
在传统的非脉冲神经网络中，有监督的多类分类问题将具有最高激活值的神经元视为预测类别。

在脉冲神经网络中，有几个选项来解释输出的脉冲。最常见的方法是:

- 速率编码:将具有最高放电率(或脉冲计数)的神经元作为预测类别
- 延迟编码:将最先触发的神经元作为预测类别

这和之前讲编码方式的教程很像。不同的是，在这里，我们解释(解码)输出峰值，而不是将原始输入数据编码/转换为峰值。

让我们关注一下速率编码。当输入数据传递到网络时，我们希望正确的神经元类在模拟运行过程中发出最多的峰值。这就对应了最高的平均发射频率。一种实现方法是将正确类别的膜电位增加到 $U>U_{\rm thr}$ ，将错误类别的膜电位增加到 $U<U_{\rm thr}$ 。将目标函数应用于 $U$ ，可以作为从 $S$ 调制峰值行为的代理。

这可以通过对输出神经元的膜电位进行softmax来实现，其中$C$是输出类别的数量：
$$p_i[t] = \frac{e^{U_i[t]}}{\sum_{j=0}^{C}e^{U_j[t]}} \tag{8}$$
The cross-entropy between $p_i$ and the target $y_i \in \{0,1\}^C$, which is a one-hot target vector, is obtained using:
$$\mathcal{L}_{CE}[t] = -\sum_{i=0}^Cy_i{\rm log}(p_i[t]) \tag{9}$$
其实际效果是鼓励提高正确类别的膜电位，而降低不正确类别的膜电位。实际上，这意味着在所有时间步骤中鼓励触发正确的类，而在所有步骤中禁止不正确的类。这可能不是最有效的SNN实现，但它是最简单的。

该目标应用于仿真的每个时间步骤，因此在每个步骤也会产生损失。在模拟结束时，将这些损失相加:

$$\mathcal{L}_{CE} = \sum_t\mathcal{L}_{CE}[t] \tag{10}$$
这只是将损失函数应用于脉冲神经网络的许多可能方法之一。在snnTorch中(在模块`snn.functional`中)可以使用多种方法，这将是未来教程的主题。

在了解了所有的背景理论之后，让我们最后深入研究一下训练全连接脉冲神经网络。

# 5. 训练
```text
# Network Architecture
num_inputs = 28*28
num_hidden = 1000
num_outputs = 10

# Temporal Dynamics
num_steps = 25
beta = 0.95

# Define Network
class Net(nn.Module):
    def __init__(self):
        super().__init__()

        # Initialize layers
        self.fc1 = nn.Linear(num_inputs, num_hidden)
        self.lif1 = snn.Leaky(beta=beta)
        self.fc2 = nn.Linear(num_hidden, num_outputs)
        self.lif2 = snn.Leaky(beta=beta)

    def forward(self, x):

        # Initialize hidden states at t=0
        mem1 = self.lif1.init_leaky()
        mem2 = self.lif2.init_leaky()
        
        # Record the final layer
        spk2_rec = []
        mem2_rec = []

        for step in range(num_steps):
            cur1 = self.fc1(x)
            spk1, mem1 = self.lif1(cur1, mem1)
            cur2 = self.fc2(spk1)
            spk2, mem2 = self.lif2(cur2, mem2)
            spk2_rec.append(spk2)
            mem2_rec.append(mem2)

        return torch.stack(spk2_rec, dim=0), torch.stack(mem2_rec, dim=0)
        
# Load the network onto CUDA if available
net = Net().to(device)
```
只有将输入参数`x`显式传递给`net`时，`forward()`函数中的代码才会被调用。

- `fc1`对MNIST数据集中的所有输入像素应用线性变换;
- `lif1`随着时间的推移对加权输入进行集成，如果满足阈值条件，则发出脉冲;
- `fc2`对`lif1`的输出峰值进行线性变换;
- `lif2`是另一个脉冲神经元层，随着时间的推移整合加权脉冲。

# 6. 训练SNN

## 6.1 Accuracy指标
下面是一个函数，它接收一批数据，对每个神经元的所有峰值进行计数(即模拟时间的速率码)，并将最高计数的索引与实际目标进行比较。如果它们匹配，则网络正确地预测了目标。
```text
# pass data into the network, sum the spikes over time
# and compare the neuron with the highest number of spikes
# with the target

def print_batch_accuracy(data, targets, train=False):
    output, _ = net(data.view(batch_size, -1))
    _, idx = output.sum(dim=0).max(1)
    acc = np.mean((targets == idx).detach().cpu().numpy())

    if train:
        print(f"Train set accuracy for a single minibatch: {acc*100:.2f}%")
    else:
        print(f"Test set accuracy for a single minibatch: {acc*100:.2f}%")

def train_printer(
    data, targets, epoch,
    counter, iter_counter,
        loss_hist, test_loss_hist, test_data, test_targets):
    print(f"Epoch {epoch}, Iteration {iter_counter}")
    print(f"Train Set Loss: {loss_hist[counter]:.2f}")
    print(f"Test Set Loss: {test_loss_hist[counter]:.2f}")
    print_batch_accuracy(data, targets, train=True)
    print_batch_accuracy(test_data, test_targets, train=False)
    print("\n")
```
仅经过一次迭代，损失应该减少，精度应该增加。注意膜电位是如何用于计算交叉熵的。损失和脉冲计数用于测量准确性。



# 总结

现在你知道如何在静态数据集上构建和训练全连接网络了。脉冲神经元也可以适应其他层类型，包括卷积和跳跃连接。有了这些知识，你现在应该能够构建许多不同类型的snn了。