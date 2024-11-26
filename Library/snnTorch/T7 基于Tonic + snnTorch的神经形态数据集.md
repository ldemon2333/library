你将：
1. 学习如何使用Tonic加载神经形态数据集
2. 利用缓存来加速数据加载
3. 使用Neuromorphic-MNIST数据集训练CSNN


# 1. 使用Tonic加载神经形态数据集
让我们首先加载MNIST数据集的神经形态版本，称为N-MNIST。我们可以查看一些原始事件，以了解我们正在处理的内容。
```text
import tonic

dataset = tonic.datasets.NMNIST(save_to='./data', train=True)
events, target = dataset[0]
print(events)
```

每行对应一个事件，由四个参数组成：(*x-coordinate, y-coordinate, timestamp, polarity*)。
- x 和 y 坐标对应34 x 34 网格中的地址
- 事件的时间戳以微秒为单位记录
- 极性指的是是否发生了峰上(+1)或峰下(-1)；即，亮度的增加或减少

如果我们将这些事件累积起来，并将这些事件的集合绘制为图像，则如下所示:

![[Pasted image 20241013115317.png]]

## 1.1 变换
然而，神经网络不将事件列表作为输入。原始数据必须转换为适当的表示，如张量。在将数据输入网络之前，我们可以选择一组转换应用于数据。神经形态相机传感器的时间分辨率为微妙，当转换为密集表示时，最终会成为一个非常大的张量。这就是为什么我们使用ToFrame transformation将事件分成更少帧的原因，它降低了时间精度，但也允许我们以密集的格式处理它。

* `time_window=1000` integrates events into 1000$~\mu$s bins
* 去噪删除了孤立的、一次性的事件。如果在`filter_time`微秒内一个像素的邻域内没有发生事件，则事件被过滤。较小的`filter_time`将过滤更多的事件。

```text
import tonic.transforms as transforms

sensor_size = tonic.datasets.NMNIST.sensor_size

# Denoise removes isolated, one-off events
# time_window
frame_transform = transforms.Compose([transforms.Denoise(filter_time=10000), 
                                      transforms.ToFrame(sensor_size=sensor_size, 
                                                         time_window=1000)
                                     ])

trainset = tonic.datasets.NMNIST(save_to='./data', transform=frame_transform, train=True)
testset = tonic.datasets.NMNIST(save_to='./data', transform=frame_transform, train=False)

def load_sample_simple():
    for i in range(100):
        events, target = trainset[i]
```
## 1.2 快速数据加载
原始数据以一种读取速度较慢的格式存储。为了加速数据加载，我们可以利用磁盘缓存和批处理。这意味着一旦从原始数据集中加载文件，它们就会被写入磁盘。

因为事件记录具有不同的长度，我们将提供一个排序函数`tonic.collation.PadTensors()`，该函数将填充较短的记录，以确保批处理中的所有样本具有相同的尺寸。

```text
from torch.utils.data import DataLoader
from tonic import DiskCachedDataset

cached_trainset = DiskCachedDataset(trainset, cache_path='./cache/nmnist/train')
cached_dataloader = DataLoader(cached_trainset)

batch_size = 128
trainloader = DataLoader(cached_trainset, batch_size=batch_size, collate_fn=tonic.collation.PadTensors())

def load_sample_batched():
    events, target = next(iter(cached_dataloader))
```

通过使用磁盘缓存和支持多线程和批处理的PyTorch数据加载器，我们显著减少了加载时间。

如果你有大量可用的RAM，你可以通过缓存到主内存而不是磁盘来进一步加快数据加载:

```text
from tonic import MemoryCachedDataset

cached_trainset = MemoryCachedDataset(trainset)
```

# 2. 使用事件创建的框架训练我们的网络
现在让我们在N-MNIST分类任务上实际训练一个网络。我们从定义缓存包装器和数据加载器开始。与此同时，我们还将对训练数据进行一些增强。我们从缓存数据集中接收到的样本是帧，因此我们可以使用PyTorch Vision来应用我们想要的任何随机变换。
```text
import torch
import torchvision

transform = tonic.transforms.Compose([torch.from_numpy,
                                      torchvision.transforms.RandomRotation([-10,10])])

cached_trainset = DiskCachedDataset(trainset, transform=transform, cache_path='./cache/nmnist/train')

# no augmentations for the testset
cached_testset = DiskCachedDataset(testset, cache_path='./cache/nmnist/test')

batch_size = 128
trainloader = DataLoader(cached_trainset, batch_size=batch_size, collate_fn=tonic.collation.PadTensors(batch_first=False), shuffle=True)
testloader = DataLoader(cached_testset, batch_size=batch_size, collate_fn=tonic.collation.PadTensors(batch_first=False))
```

一个小批量现在有了尺寸(时间步长，批量大小，通道，高度，宽度)。将时间步数设置为小批量中最长记录的步数，所有其他样本将用零填充以匹配它。

```text
event_tensor, target = next(iter(trainloader))
print(event_tensor.shape)
```
输出结果为：
```text
torch.Size([310, 128, 2, 34, 34])
```
## 2.1 定义我们的网络
我们将使用snnTorch + PyTorch来构建CSNN，就像在前一个教程中一样。使用的卷积网络架构是:12C5-MP2-32C5-MP2-800FC10

- 12C5是一个 5×5 带有12个滤波器的卷积核
- MP2是一个 2×2 最大池化函数
- 800FC10是一个全连接层，它将800个神经元映射到10个输出

```text
import snntorch as snn
from snntorch import surrogate
from snntorch import functional as SF
from snntorch import spikeplot as splt
from snntorch import utils
import torch.nn as nn

device = torch.device("cuda") if torch.cuda.is_available() else torch.device("cpu")

# neuron and simulation parameters
spike_grad = surrogate.atan()
beta = 0.5

#  Initialize Network
net = nn.Sequential(nn.Conv2d(2, 12, 5),
                    snn.Leaky(beta=beta, spike_grad=spike_grad, init_hidden=True),
                    nn.MaxPool2d(2),
                    nn.Conv2d(12, 32, 5),
                    snn.Leaky(beta=beta, spike_grad=spike_grad, init_hidden=True),
                    nn.MaxPool2d(2),
                    nn.Flatten(),
                    nn.Linear(32*5*5, 10),
                    snn.Leaky(beta=beta, spike_grad=spike_grad, init_hidden=True, output=True)
                    ).to(device)

# this time, we won't return membrane as we don't need it 

def forward_pass(net, data):  
  spk_rec = []
  utils.reset(net)  # resets hidden states for all LIF neurons in net

  for step in range(data.size(0)):  # data.size(0) = number of time steps
      spk_out, mem_out = net(data[step])
      spk_rec.append(spk_out)
  
  return torch.stack(spk_rec)
```

## 2.2 训练
在上一个教程中，交叉熵损失被应用于总脉冲数，以最大化来自正确类的脉冲数。

另一个来自`snn.Functional`模块用于指定正确和错误类别的目标峰值数量。下面的方法使用了_均方误差峰值计数损失_，目的是在80%的情况下从正确的类别中提取峰值，在20%的情况下从不正确的类别中提取峰值。鼓励不正确的神经元放电可以激励避免死去的神经元。

```text
optimizer = torch.optim.Adam(net.parameters(), lr=2e-2, betas=(0.9, 0.999))
loss_fn = SF.mse_count_loss(correct_rate=0.8, incorrect_rate=0.2)
```
训练神经形态数据是昂贵的，因为它需要连续迭代许多时间步骤(在N-MNIST数据集中大约300个时间步骤)。下面的模拟将需要一些时间，因此我们将坚持训练50次迭代(大约是整个迭代的十分之一)。如果你有更多的时间，请随意更改`num_ters`。由于我们在每次迭代中都打印结果，因此结果将非常嘈杂，并且需要一段时间才能看到任何形式的改进。

在我们自己的实验中，我们花了大约20次迭代才看到任何改进，在50次迭代后，成功破解了约60%的准确率。

```text
num_epochs = 1
num_iters = 50

loss_hist = []
acc_hist = []

# training loop
for epoch in range(num_epochs):
    for i, (data, targets) in enumerate(iter(trainloader)):
        data = data.to(device)
        targets = targets.to(device)

        net.train()
        spk_rec = forward_pass(net, data)
        loss_val = loss_fn(spk_rec, targets)

        # Gradient calculation + weight update
        optimizer.zero_grad()
        loss_val.backward()
        optimizer.step()

        # Store loss history for future plotting
        loss_hist.append(loss_val.item())
 
        print(f"Epoch {epoch}, Iteration {i} \nTrain Loss: {loss_val.item():.2f}")

        acc = SF.accuracy_rate(spk_rec, targets) 
        acc_hist.append(acc)
        print(f"Accuracy: {acc * 100:.2f}%\n")

        # This will end training after 50 iterations by default
        if i == num_iters:
          break
```


