# 1.神经元模型的范围
神经元模型种类繁多，从精确的生物物理模型(即Hodgkin-Huxley模型)到极其简单的人工神经元，它们遍及现代深度学习的所有方面。

Hodgkin-Huxley神经元模型虽然生物物理模型可以高精度地再现电生理结果，但其复杂性使其目前难以使用。

人工神经元模型另一方面是人工神经元。输入乘以它们对应的权重，并传递给激活函数这种简化使深度学习研究人员在计算机视觉、自然语言处理和许多其他机器学习领域的任务中取得了令人难以置信的成就。

Leaky Integrate-and-Fire神经元模型（LIF）。它需要加权输入的和，很像人工神经元。但它不是将其直接传递给激活函数，而是随着时间的推移将输入与泄露集成，很像RC电路。如果积分值超过阈值，则LIF神经元将发射电压峰。LIF神经元抽象出输出脉冲的形状和轮廓；它被简单地视为一个离散事件。因此，信息不是存储在脉冲中，而是存储在脉冲的时间（或频率）中。简单的脉冲神经元已经对神经代码、记忆、网络动力学以及最近的深度学习产生了很多见解。LIF神经元处于生物学合理性和实用性之间的最佳位置。

![[Pasted image 20241009162917.png]]

不同版本的LIF模型都有自己的动态特性和用例。snnTorch目前支持以下LIF神经元：
- Lapicque的RC模型：snntorch.Lapicque
- 一阶模型：snntorch.Leaky
- 基于突触电导的神经元模型：snntorch.Synaptic
- 递归一阶模型：snntorch.RLeaky
- 基于递归突触电导的神经元模型：snntorch.RSynaptic
- Alpha神经元模型：snntorch.Alpha

# 2.LIF神经元模型

## 2.1 脉冲神经元
在我们的大脑中，一个神经元可能与1000 - 10000个其他神经元相连。如果一个神经元脉冲，所有下坡神经元都可能感受到。但是，是什么决定了神经元是否会出现峰值呢？过去一个世纪的实验表明，如果神经元在输入时受到足够的刺激，那么它可能会变得兴奋，并发出自己的脉冲。

这种刺激从何而来？它可以来自：

- 感觉外围；
- 一种侵入性电极人工刺激神经元；
- 或者在大多数情况下，来自其他突触神经元

![[Pasted image 20241009163222.png]]

考虑到这些尖峰电位是非常短的电活动爆发，不太可能所有输入尖峰电位都精确一致地到达神经元体。这表明存在时间动态维持输入峰值，有点像延迟。

## 2.2被动膜
像所有细胞一样，神经元被一层薄膜包围。这层膜是一个脂质双分子层，它将神经元内的传导盐水溶液与细胞外介质隔离开来。在电的作用下，由绝缘体隔开的两种导电溶液起电容器的作用。

这层膜的另一个功能是控制进出细胞的物质（例如钠离子）。细胞膜通常不能被离子渗透，从而阻止他们进出神经元体。但是细胞膜上有一些特殊的通道，可以通过向神经元注入电流而打开。这种电荷运动由电阻器模拟。

![[Pasted image 20241009163805.png]]

下面的代码块将从零开始推导LIF神经元的行为。

### LIF神经元模型的动力学模型
假设任意的时变电流$I_{\rm in}(t)$被注入神经元中，可能是通过电刺激，也可能是来自其他神经元。电路中的总电流是守恒的，因此：
$$I_{\rm in}(t) = I_{R} + I_{C}$$
根据欧姆定律，神经元内外测得的膜电位$U_{\rm mem}$与通过电阻的电流成正比：
$$I_{R}(t) = \frac{U_{\rm mem}(t)}{R}$$
电容是神经元上存储的电荷$Q$和$U_{\rm mem}(t)$之间的比例常数:
$$Q = CU_{\rm mem}(t)$$
电荷变化率给出电容电流：
$$\frac{dQ}{dt}=I_C(t) = C\frac{dU_{\rm mem}(t)}{dt}$$
因此：
$$I_{\rm in}(t) = \frac{U_{\rm mem}(t)}{R} + C\frac{dU_{\rm mem}(t)}{dt}$$
$$\implies RC \frac{dU_{\rm mem}(t)}{dt} = -U_{\rm mem}(t) + RI_{\rm in}(t)$$
等式右边的单位是伏特。在等式的左边，$\frac{dU_{\rm mem}(t)}{dt}$的单位是伏特每秒。为使其与左侧(即电压)相等，$RC$的单位是秒。我们称$\tau = RC$为电路的时间常数:
$$ \tau \frac{dU_{\rm mem}(t)}{dt} = -U_{\rm mem}(t) + RI_{\rm in}(t)$$
因此，被动膜被描述一个线性微分方程。

如果函数的导数与原始函数具有相同的形式，即$\frac{dU_{\rm mem}(t)}{dt} \propto U_{\rm mem}(t)$，这意味着解是具有时间常数$\tau$的指数函数。

假设神经元从某个值$U_{0}$开始，没有进一步的输入，即$I_{\rm in}(t)=0$。线性微分方程的解为：
$$U_{\rm mem}(t) = U_0e^{-\frac{t}{\tau}}$$
一般的推导如下所示。
![[Pasted image 20241009171243.png]]
### 前向欧拉法求解LIF神经元模型
我们设法找到了LIF神经元的解析解，但尚不清楚这在神经网络中如何有用。这次，我们使用前向欧拉方法求解之前的线性常微分方程（ODE）。这种方法可能看起来费力，但它为我们提供了一个离散的、循环的LIF神经元表示。一旦我们得到这个解决方案，它就可以直接应用于神经网络。与前面一样，描述RC电路的线性ODE为：
$$\tau \frac{dU(t)}{dt} = -U(t) + RI_{\rm in}(t)$$
首先，我们来解这个导数，不需要求极限$\Delta t \rightarrow 0$:
$$\tau \frac{U(t+\Delta t)-U(t)}{\Delta t} = -U(t) + RI_{\rm in}(t)$$
对于足够小的$\Delta t$，这是连续时间积分的一个足够好的近似。按以下时间步骤隔离膜：
$$U(t+\Delta t) = U(t) + \frac{\Delta t}{\tau}\big(-U(t) + RI_{\rm in}(t)\big)$$
下面的函数表示了这个等式
```python
def leaky_integrate_neuron(U, time_step=1e-3, I=0, R=5e7, C=1e-10):
	tau = R*C
	U = U + (time_step/tau)*(-U + I*R)
	return U
```
默认值设置为$R=50 M\Omega$和$C=100pF$（即$\tau=5ms$）。这些对于生物神经元来说是相当现实的。

现在循环这个函数，每次迭代一个时间步长。

假设没有注入输入电流，膜电位初始化为$U=0.9 V$，即$I_{\rm in}=0 A$。

这个模拟是以毫秒精度执行的$\Delta t=1\times 10^{-3}$s。

# 3.Lapicque的LIF神经元模型
神经膜和RC电流之间的这种相似性是由Louis Lapicque在1907年观察到的。他用一个短暂的电脉冲刺激青蛙的神经纤维，发现神经元膜可以近似为一个有泄露的电容器。我们通过以他的名字名字命名snnTorch中的基本LIF神经元模型以表示敬意。

Lapicque模型中的大多数概念可以推广到其他LIF神经元模型。现在使用snnTorch来模拟这个神经元。

## 3.1 无刺激的Lapicque
使用下面的代码实例化Lapicque的神经元。
```python
time_step = 1e-3
R = 5
C = 1e-3
# leaky integrate and fire neuron, tau=5e-3
lif1 = snn.Lapicque(R=R, C=C, time_step=time_step)
```

神经元模型现在存储在lif1中。要使用这个神经元：

**Inputs**
* `spk_in`: $I_{\rm in}$的每个元素依次作为输入传递(目前为0) 
* `mem`: the membrane potential, previously $U[t]$, is also passed as input. Initialize it arbitrarily as $U[0] = 0.9~V$.
**Outputs**
* `spk_out`: output spike $S_{\rm out}[t+\Delta t]$ at the next time step ('1' if there is a spike; '0' if there is no spike)
* `mem`: membrane potential $U_{\rm mem}[t+\Delta t]$ at the next time step

这些都需要是torch.Tensor类型

```python
# Initialize membrane, input, and output
mem = torch.ones(1) * 0.9  # U=0.9 at t=0
cur_in = torch.zeros(num_steps, 1)  # I=0 for all t
spk_out = torch.zeros(1)  # initialize output spikes
```
这些值仅针对初始时间步$t=0$。要分析'`mem`随时间的变化，请创建一个列表mem_rec来记录每个时间步长的这些值。
``` python
# A list to store recordings of membrane potential
mem_rec = [mem]
```
现在是时候运行一个模拟了。在每个时间点，都会更新men并保存在men_rec中：
```python
# pass updated value of mem and cur_in[step]=0 at every time step
for step in range(num_steps):
  spk_out, mem = lif1(cur_in[step], mem)
  # Store recordings of membrane potential
  mem_rec.append(mem)

# crunch the list of tensors into one tensor
mem_rec = torch.stack(mem_rec)
plot_mem(mem_rec, "Lapicque's Neuron Model Without Stimulus")
```
在没有任何输入刺激的情况下，膜电位随着时间的推移而衰退。

## 3.2 Lapicque：每步输入
现在施加一个阶跃电流$I_{\rm in}(t)$，在$t=t_0$时输入神经元。给定一阶线性微分方程：
$$ \tau \frac{dU_{\rm mem}}{dt} = -U_{\rm mem} + RI_{\rm in}(t)$$
解析解为：
$$U_{\rm mem}=I_{\rm in}(t)R + [U_0 - I_{\rm in}(t)R]e^{-\frac{t}{\tau}}$$
如果膜电位初始化为$U_{\rm mem}(t=0) = 0 V$，则
$$U_{\rm mem}(t)=I_{\rm in}(t)R [1 - e^{-\frac{t}{\tau}}]$$
基于这种显示的时间依赖形式，我们期望$U_{\rm mem}$向$I_{\rm in}R$呈指数级松弛。让我们通过触发$I_{in}=100mA$在$t_0 = 10ms$的电流脉冲来可视化它的样子。
```python
# Initialize input current pulse
cur_in = torch.cat((torch.zeros(10), torch.ones(190)*0.1), 0)  # input current turns on at t=10

# Initialize membrane, output and recordings
mem = torch.zeros(1)  # membrane potential of 0 at t=0
spk_out = torch.zeros(1)  # neuron needs somewhere to sequentially dump its output spikes
mem_rec = [mem]
```
这一次，将cur_in的新值传递给神经元：
```python
num_steps = 200

# pass updated value of mem and cur_in[step] at every time step
for step in range(num_steps):
  spk_out, mem = lif1(cur_in[step], mem)
  mem_rec.append(mem)

# crunch -list- of tensors into one tensor
mem_rec = torch.stack(mem_rec)

plot_step_current_response(cur_in, mem_rec, 10)
```
对于$t\rightarrow \infty$, 膜电位$U_{\rm mem}$指数松弛为$I_{\rm in}R$:

## 3.3 Lapicque：脉冲输入
现在，如果阶跃输入在$t=30ms$时被裁剪会怎样？
```python
# Initialize current pulse, membrane and outputs
cur_in1 = torch.cat((torch.zeros(10), torch.ones(20)*(0.1), torch.zeros(170)), 0)  # input turns on at t=10, off at t=30
mem = torch.zeros(1)
spk_out = torch.zeros(1)
mem_rec1 = [mem]

# neuron simulation
for step in range(num_steps):
  spk_out, mem = lif1(cur_in1[step], mem)
  mem_rec1.append(mem)
mem_rec1 = torch.stack(mem_rec1)

plot_current_pulse_response(cur_in1, mem_rec1, "Lapicque's Neuron Model With Input Pulse", 
                            vline1=10, vline2=30)
```

$U_{\rm mem}$和阶跃输入一样上升，但现在它随着时间常数$\tau$衰减，就像我们第一个模拟中那样。

让我们用一半的时间向电路交付大致相同的电荷$Q = I \times t$。这意味着输入电流幅值必须增加一点，时间窗必须减小。
```python
# Increase amplitude of current pulse; half the time.
cur_in2 = torch.cat((torch.zeros(10), torch.ones(10)*0.111, torch.zeros(180)), 0)  # input turns on at t=10, off at t=20
mem = torch.zeros(1)
spk_out = torch.zeros(1)
mem_rec2 = [mem]

# neuron simulation
for step in range(num_steps):
  spk_out, mem = lif1(cur_in2[step], mem)
  mem_rec2.append(mem)
mem_rec2 = torch.stack(mem_rec2)

plot_current_pulse_response(cur_in2, mem_rec2, "Lapicque's Neuron Model With Input Pulse: x1/2 pulse width",
                            vline1=10, vline2=20)
```
让我们在做一次，但使用更快的输入脉冲和更高的振幅：
```python
# Increase amplitude of current pulse; quarter the time.
cur_in3 = torch.cat((torch.zeros(10), torch.ones(5)*0.147, torch.zeros(185)), 0)  # input turns on at t=10, off at t=15
mem = torch.zeros(1)
spk_out = torch.zeros(1)
mem_rec3 = [mem]

# neuron simulation
for step in range(num_steps):
  spk_out, mem = lif1(cur_in3[step], mem)
  mem_rec3.append(mem)
mem_rec3 = torch.stack(mem_rec3)

plot_current_pulse_response(cur_in3, mem_rec3, "Lapicque's Neuron Model With Input Pulse: x1/4 pulse width",
                            vline1=10, vline2=15)
```
现在在同一个图形上比较这三个实验：
```python
compare_plots(cur_in1, cur_in2, cur_in3, mem_rec1, mem_rec2, mem_rec3, 10, 15, 
              20, 30, "Lapicque's Neuron Model With Input Pulse: Varying inputs")
```
随着输入电流脉冲幅值的增大，膜电位的上升时间加快。在输入电流脉冲宽度变得无穷小$T_W \rightarrow 0s$的极限下，膜电位将在几乎为零的上升时间内直线上升:

```python
# Current spike input
cur_in4 = torch.cat((torch.zeros(10), torch.ones(1)*0.5, torch.zeros(189)), 0)  # input only on for 1 time step
mem = torch.zeros(1) 
spk_out = torch.zeros(1)
mem_rec4 = [mem]

# neuron simulation
for step in range(num_steps):
  spk_out, mem = lif1(cur_in4[step], mem)
  mem_rec4.append(mem)
mem_rec4 = torch.stack(mem_rec4)

plot_current_pulse_response(cur_in4, mem_rec4, "Lapicque's Neuron Model With Input Spike", 
                            vline1=10, ylim_max1=0.6)
```

现在的脉冲宽度很短，看起来就像一个尖峰。也就是说，电荷是在无限短的时间内传递的，$I_{\rm in}(t) = Q/t_0$ ，其中$t_0 \rightarrow 0$。更正式地：
$$I_{\rm in}(t) = Q \delta (t-t_0),$$
其中$\delta (t-t_0)$是Dirac-Delta函数。从物理上讲，不可能“立即”到达目标电压。但是将$I_{\rm in}$积分得到的结构在物理上是由意义的，因为我们可以得到交付的电荷。
$$1 = \int^{t_0 + a}_{t_0 - a}\delta(t-t_0)dt$$
这里
$$f(t_0) = \int^{t_0 + a}_{t_0 - a}f(t)\delta(t-t_0)dt$$
Here, $f(t_0) = I_{\rm in}(t_0=10) = 0.5A \implies f(t) = Q = 0.5C$.

希望你们对膜电位在静止时的泄漏，以及对输入电流的积分有一个很好的感觉。这就涵盖了神经元的“漏”和“整合”部分。那激活（Fire）是怎样实现的呢。

## 3.4 Lapcique：激活
到目前为止，我们只看到了神经元对输入峰值的反应。为了让神经元在输出端产生和发射自己尖峰电位，被动膜模型必须与阈值相结合

如果膜电位超过了这个阈值，那么将产生一个电压尖峰，外部的膜模型
![[Pasted image 20241009185209.png]]

修改之前的leaky_integrate_neuron函数，添加一个脉冲响应。
```python
# R=5.1, C=5e-3 for illustrative purposes
def leaky_integrate_and_fire(mem, cur=0, threshold=1, time_step=1e-3, R=5.1, C=5e-3):
  tau_mem = R*C
  spk = (mem > threshold) # if membrane exceeds threshold, spk=1, else, 0
  mem = mem + (time_step/tau_mem)*(-mem + cur*R)
  return mem, spk
```
设置threshod=1，并施加一个阶跃电流以获得该神经元脉冲。
```python
# Small step current input
cur_in = torch.cat((torch.zeros(10), torch.ones(190)*0.2), 0)
mem = torch.zeros(1)
mem_rec = []
spk_rec = []

# neuron simulation
for step in range(num_steps):
  mem, spk = leaky_integrate_and_fire(mem, cur_in[step])
  mem_rec.append(mem)
  spk_rec.append(spk)

# convert lists to tensors
mem_rec = torch.stack(mem_rec)
spk_rec = torch.stack(spk_rec)

plot_cur_mem_spk(cur_in, mem_rec, spk_rec, thr_line=1, vline=109, ylim_max2=1.3, 
                 title="LIF Neuron Model With Uncontrolled Spiking")
```
你会发现上面的输出尖峰信号已经失控了，这是因为我们忘记添加重置机制。实际上，每当神经元放电时，膜电位就会超极化回到静息电位。

在神经元中实现这种重置机制：
```python
# LIF w/Reset mechanism
def leaky_integrate_and_fire(mem, cur=0, threshold=1, time_step=1e-3, R=5.1, C=5e-3):
  tau_mem = R*C
  spk = (mem > threshold)
  mem = mem + (time_step/tau_mem)*(-mem + cur*R) - spk*threshold  # every time spk=1, subtract the threhsold
  return mem, spk

# Small step current input
cur_in = torch.cat((torch.zeros(10), torch.ones(190)*0.2), 0)
mem = torch.zeros(1)
mem_rec = []
spk_rec = []

# neuron simulation
for step in range(num_steps):
  mem, spk = leaky_integrate_and_fire(mem, cur_in[step])
  mem_rec.append(mem)
  spk_rec.append(spk)

# convert lists to tensors
mem_rec = torch.stack(mem_rec)
spk_rec = torch.stack(spk_rec)

plot_cur_mem_spk(cur_in, mem_rec, spk_rec, thr_line=1, vline=109, ylim_max2=1.3, 
                 title="LIF Neuron Model With Reset")
```
我们现在有了一个功能性的leaky integrate-and-fire神经元模型！

注意，如果$I_{\rm in}=0.2 A$和$R<5 \Omega$，那么$I\times R < 1 V$。如果`threshold=1`，则不会发生脉冲。您可以随时返回，更改值并进行测试。和之前一样，所有代码都是通过从snnTorch中调用内置的Lapicque神经元模型来压缩的：
```python
# Create the same neuron as before using snnTorch
lif2 = snn.Lapicque(R=5.1, C=5e-3, time_step=1e-3)

print(f"Membrane potential time constant: {lif2.R * lif2.C:.3f}s")

# Initialize inputs and outputs
cur_in = torch.cat((torch.zeros(10), torch.ones(190)*0.2), 0)
mem = torch.zeros(1)
spk_out = torch.zeros(1) 
mem_rec = [mem]
spk_rec = [spk_out]

# Simulation run across 100 time steps.
for step in range(num_steps):
  spk_out, mem = lif2(cur_in[step], mem)
  mem_rec.append(mem)
  spk_rec.append(spk_out)

# convert lists to tensors
mem_rec = torch.stack(mem_rec)
spk_rec = torch.stack(spk_rec)

plot_cur_mem_spk(cur_in, mem_rec, spk_rec, thr_line=1, vline=109, ylim_max2=1.3, 
                 title="Lapicque Neuron Model With Step Input")
```

膜电位呈指数增长，然后达到阈值，在这一点上复位。我们可以大致看到这发生在$105ms < t_{\rm spk} < 115ms$之间。出于好奇，让我们看看峰值记录实际上是由什么组成的：
```text
print(spk_rec[105:115].view(-1))
```
没有脉冲的情况用$S_{\rm out}=0$表示，出现脉冲的情况用$S_{\rm out}=1$表示。这里，脉冲出现在$S_{\rm out}[t=109]=1$处。如果你想知道为什么这些条目都被存储为张量，这是因为在未来的教程中，我们将模拟大规模的神经网络。每个条目将包含许多神经元的脉冲响应，张量可以加载到GPU内存中以加快训练过程。

如果$I_{\rm in}$增大，那么膜电位会更快地接近阈值$U_{\rm thr}$：
```text
# Initialize inputs and outputs
cur_in = torch.cat((torch.zeros(10), torch.ones(190)*0.3), 0)  # increased current
mem = torch.zeros(1)
spk_out = torch.zeros(1) 
mem_rec = [mem]
spk_rec = [spk_out]

# neuron simulation
for step in range(num_steps):
  spk_out, mem = lif2(cur_in[step], mem)
  mem_rec.append(mem)
  spk_rec.append(spk_out)

# convert lists to tensors
mem_rec = torch.stack(mem_rec)
spk_rec = torch.stack(spk_rec)


plot_cur_mem_spk(cur_in, mem_rec, spk_rec, thr_line=1, ylim_max2=1.3, 
                 title="Lapicque Neuron Model With Periodic Firing")
```
降低阈值也可以引起类似的发射频率的增加。这需要初始化一个新的神经元模型，但其余的代码块与上面完全相同:
```text
# neuron with halved threshold
lif3 = snn.Lapicque(R=5.1, C=5e-3, time_step=1e-3, threshold=0.5)

# Initialize inputs and outputs
cur_in = torch.cat((torch.zeros(10), torch.ones(190)*0.3), 0) 
mem = torch.zeros(1)
spk_out = torch.zeros(1) 
mem_rec = [mem]
spk_rec = [spk_out]

# Neuron simulation
for step in range(num_steps):
  spk_out, mem = lif3(cur_in[step], mem)
  mem_rec.append(mem)
  spk_rec.append(spk_out)

# convert lists to tensors
mem_rec = torch.stack(mem_rec)
spk_rec = torch.stack(spk_rec)

plot_cur_mem_spk(cur_in, mem_rec, spk_rec, thr_line=0.5, ylim_max2=1.3, 
                 title="Lapicque Neuron Model With Lower Threshold")
```
这就是恒流注入的情况。但在深度神经网络和生物大脑中，大多数神经元将与其他神经元连接。它们更有可能接受到尖峰电流，而不是注入恒定电流。

## 3.5 Lapicque：输入脉冲
让我们利用我们在Tutorial 1学习的技能，并使用snntorch.spikegen模块来创建一些随机生成的输入峰值。
```python
# Create a 1-D random spike train. Each element has a probability of 40% of firing.
spk_in = spikegen.rate_conv(torch.ones((num_steps)) * 0.40)
```


## 3.6 Lapicque：重置机制
我们已经从零开始实现了一个重置机制，但让我们再深入一点。这种膜电位的急剧下降促进了脉冲产生的减少，这补充了大脑如何如此高效的理论的一部分。生物学上，这种膜电位的下降被称为“超极化”。在此之后，暂时很难从神经元中引出另一个峰电位。本文使用重置机制来模拟超极化。

有两种实现重置机制的方法。

- 减法重置(默认)：每次产生脉冲时从膜电位中减去阈值;
- 重置为零：每次产生脉冲时，将膜电位强制为零。
- 没有重置：什么都不做，并且让激活变得不受控制。

![[Pasted image 20241009191012.png]]

实例化另一个神经元模型，演示如何在复位机制之间进行交替。

默认情况下，snntorch神经元模型使用reset_mechanism = "subtract"。这可以通过传递参数reset_mechanism = "zero"来显式覆盖。
```text
# Neuron with reset_mechanism set to "zero"
lif4 = snn.Lapicque(R=5.1, C=5e-3, time_step=1e-3, threshold=0.5, reset_mechanism="zero")

# Initialize inputs and outputs
spk_in = spikegen.rate_conv(torch.ones((num_steps)) * 0.40)
mem = torch.ones(1)*0.5
spk_out = torch.zeros(1)
mem_rec0 = [mem]
spk_rec0 = [spk_out]

# Neuron simulation
for step in range(num_steps):
  spk_out, mem = lif4(spk_in[step], mem)
  spk_rec0.append(spk_out)
  mem_rec0.append(mem)

# convert lists to tensors
mem_rec0 = torch.stack(mem_rec0)
spk_rec0 = torch.stack(spk_rec0)

plot_reset_comparison(spk_in, mem_rec, spk_rec, mem_rec0, spk_rec0)
```
密切关注膜电位的演变，特别是在膜电位达到阈值后的瞬间。你可能会注意到，对于“重置为零”，膜电位在每个脉冲后都被强制恢复为零。

那么哪个更好呢?应用"subtract" (reset_mechanism中的默认值)损耗更小，因为它不会忽略膜超出阈值的多少。

另一方面，在专用的神经形态硬件上运行时，使用"zero"应用硬复位可以提高稀疏性并可能降低功耗。这两种方法都可以尝试使用。

这就涵盖了LIF神经元模型的基础知识!

# 总结
在实践中，我们可能不会使用这个神经元模型来训练神经网络。Lapicque LIF模型添加了许多需要调优的超参数: $R$, $C$, $\Delta t$, $U_{\rm thr}$ ，以及重置机制的选择。这一切都有点令人生畏。


