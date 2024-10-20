 tensors和 NumPy 数组通常可以共享相同的底层内存，这样就不需要复制数据了。CPU 和 NumPy 数组上的张量可以共享它们的底层内存位置，更改其中一个将更改另一个。

tensors也针对自动微分进行了优化

Over 100 tensor operations, including arithmetic, linear algebra, matrix manipulation (transposing, indexing, slicing), sampling and more are comprehensively described [here](https://pytorch.org/docs/stable/torch.html).

跨设备复制大tensors在时间和内存方面是昂贵的！

torch.cat 与 torch.stack区别
cat不改变现有的维度，stack形成更高维的张量


In-place 操作将结果存储到操作数中的操作称为就地操作。它们由 _ 后缀表示。例如: x.copy _ (y) ，x.t _ () ，将改变 x。
就地操作可以节省一些内存，但是在计算导数时可能会出现问题，因为会立即丢失历史记录。因此，不鼓励使用它们。



# Datasets & DataLoaders
`torch.utils.data.DataLoader` and `torch.utils.data.Dataset`
We load the [FashionMNIST Dataset](https://pytorch.org/vision/stable/datasets.html#fashion-mnist) with the following parameters:

- `root` is the path where the train/test data is stored,
    
- `train` specifies training or test dataset,
    
- `download=True` downloads the data from the internet if it’s not available at `root`.
    
- `transform` and `target_transform` specify the feature and label transformations

```python
import os
import pandas as pd
from torchvision.io import read_image

class CustomImageDataset(Dataset):
    def __init__(self, annotations_file, img_dir, transform=None, target_transform=None):
        self.img_labels = pd.read_csv(annotations_file)
        self.img_dir = img_dir
        self.transform = transform
        self.target_transform = target_transform

    def __len__(self):
        return len(self.img_labels)

    def __getitem__(self, idx):
        img_path = os.path.join(self.img_dir, self.img_labels.iloc[idx, 0])
        image = read_image(img_path)
        label = self.img_labels.iloc[idx, 1]
        if self.transform:
            image = self.transform(image)
        if self.target_transform:
            label = self.target_transform(label)
        return image, label
```

在 Python 的 NumPy 和 PyTorch 库中，`squeeze()` 是一个用于去除数组或张量中维度为 1 的维度的方法。其主要作用是简化数据结构，使其更容易处理。

# torch.utils.data

At the heart of PyTorch data loading utility is the [`torch.utils.data.DataLoader`](https://pytorch.org/docs/stable/data.html#torch.utils.data.DataLoader "torch.utils.data.DataLoader") class. It represents a Python iterable over a dataset, with support for

- [map-style and iterable-style datasets](https://pytorch.org/docs/stable/data.html#dataset-types),
    
- [customizing data loading order](https://pytorch.org/docs/stable/data.html#data-loading-order-and-sampler),
    
- [automatic batching](https://pytorch.org/docs/stable/data.html#loading-batched-and-non-batched-data),
    
- [single- and multi-process data loading](https://pytorch.org/docs/stable/data.html#single-and-multi-process-data-loading),
    
- [automatic memory pinning](https://pytorch.org/docs/stable/data.html#memory-pinning).

```
DataLoader(dataset, batch_size=1, shuffle=False, sampler=None,
           batch_sampler=None, num_workers=0, collate_fn=None,
           pin_memory=False, drop_last=False, timeout=0,
           worker_init_fn=None, *, prefetch_factor=2,
           persistent_workers=False)
```
## Dataset Types

The most important argument of [`DataLoader`](https://pytorch.org/docs/stable/data.html#torch.utils.data.DataLoader "torch.utils.data.DataLoader") constructor is [`dataset`](https://pytorch.org/docs/stable/utils.html#module-torch.utils.data.dataset "torch.utils.data.dataset"), which indicates a dataset object to load data from. PyTorch supports two different types of datasets:

- [map-style datasets](https://pytorch.org/docs/stable/data.html#map-style-datasets),
    
- [iterable-style datasets](https://pytorch.org/docs/stable/data.html#iterable-style-datasets).



# Build the Neural Network  
torch.nn.Flatten(_start_dim=1_, _end_dim=-1_)从第二个维度开始展开

# Automatic Differentiation with torch.autograd
- We can only perform gradient calculations using `backward` once on a given graph, for performance reasons. If we need to do several `backward` calls on the same graph, we need to pass `retain_graph=True` to the `backward` call.

There are reasons you might want to disable gradient tracking:

- To mark some parameters in your neural network as **frozen parameters**.
    
- To **speed up computations** when you are only doing forward pass, because computations on tensors that do not track gradients would be more efficient.

**DAGs are dynamic in PyTorch** 每次.backward()时，会建立一张动态图。An important thing to note is that the graph is recreated from scratch; after each `.backward()` call, autograd starts populating a new graph. This is exactly what allows you to use control flow statements in your model; you can change the shape, size and operations at every iteration if needed.

## Tensor Gradients and Jacobian Products
In many cases, we have a scalar loss function, and we need to compute the gradient with respect to some parameters. However, there are cases when the output function is an arbitrary tensor. In this case, PyTorch allows you to compute so-called **Jacobian product**, and not the actual gradient.
![[Pasted image 20241014160652.png]]

# Optimizing Model Parameters
Common loss functions include [nn.MSELoss](https://pytorch.org/docs/stable/generated/torch.nn.MSELoss.html#torch.nn.MSELoss) (Mean Square Error) for regression tasks, and [nn.NLLLoss](https://pytorch.org/docs/stable/generated/torch.nn.NLLLoss.html#torch.nn.NLLLoss) (Negative Log Likelihood) for classification. [nn.CrossEntropyLoss](https://pytorch.org/docs/stable/generated/torch.nn.CrossEntropyLoss.html#torch.nn.CrossEntropyLoss) combines `nn.LogSoftmax` and `nn.NLLLoss`.

```
optimizer = torch.optim.SGD(model.parameters(), lr=learning_rate)
```
Inside the training loop, optimization happens in three steps:
- call optimizer.zero_grad() to reset the gradients of model parameters
- loss.backward()
- optimizer.step()

# Save and Load the Model
## Saving and Loading Model Weights[](https://pytorch.org/tutorials/beginner/basics/saveloadrun_tutorial.html#saving-and-loading-model-weights)

PyTorch models store the learned parameters in an internal state dictionary, called `state_dict`. These can be persisted via the `torch.save` method:
```
model = models.vgg16(weights='IMAGENET1K_V1')
torch.save(model.state_dict(), 'model_weights.pth')
```
Using `weights_only=True` is considered a best practice when loading weights.
```
model = models.vgg16() # we do not specify ``weights``, i.e. create untrained model
model.load_state_dict(torch.load('model_weights.pth', weights_only=True))
model.eval()
```
一定要在推断之前调用model.eval()方法，将dropout和batch normalization 层设为评估模式，不这样做将会产生不一致的推理结构。

## Saving and Loading Models with Shapes
在加载模型权重时，我们需要先实例化model class，我们可以保存模型结构：
```
torch.save(model, 'model.pth')
```
然后加载模型权重：
```
model = torch.load('model.pth', weights_only=False),
```


# Autograd 机制
## How autograd encodes the history
Autograd 是一种反向自动微分系统。从概念上讲，autograd 会记录一个图表，该图表记录了执行操作时创建数据的所有操作，从而为您提供一个有向无环图，其叶子是输入张量，根是输出张量。通过从根到叶子跟踪此图，您可以使用链式法则自动计算梯度。

在内部，autograd 将此图表示为 Function 对象（实际上是表达式）的图，可以使用 apply() 来计算评估该图的结果。在计算正向传递时，autograd 同时执行所请求的计算并构建表示计算梯度的函数的图（每个 torch.Tensor 的 .grad_fn 属性是此图的入口点）。当正向传递完成后，我们在反向传递中评估此图以计算梯度。

需要注意的一件重要事情是，每次迭代时都会从头开始重新创建图形，这正是允许使用任意 Python 控制流语句的原因，这些语句可以在每次迭代时更改图形的整体形状和大小。在启动训练之前，您不必对所有可能的路径进行编码 - 您运行的内容就是您要区分的内容。

## 保存tensors
有些操作需要在前向传播期间保存中间结果，以便执行后向传递。比如，函数$x\rightarrow x^2$保存输入$x$去计算梯度。

在自定义Python函数时，可以使用`save_for_behind()`在正向传播过程中保存张量，使用`save_tensor`在反向传播过程中检索张量。

对于Pytorch定义的操作（例如`torch.pow()`)，张量会根据需要自动保存。通过查找以前缀`_saved`开头的属性，您可以探索某个`grad_fn`保存了哪些张量。
```python
x = torch.randn(5, requires_grad=True)
y = x.pow(2)
print(x.equal(y.grad_fn._saved_self))  # True
print(x is y.grad_fn._saved_self)  # True
```

也有特例：
```
x = torch.randn(5, requires_grad=True)
y = x.exp()
print(y.equal(y.grad_fn._saved_result))  # True
print(y is y.grad_fn._saved_result)  # False
```
在 PyTorch 中，为了防止在自动微分图（autograd graph）中出现**引用循环**（reference cycles），内部采用了一种打包（pack）和解包（unpack）张量的机制。

### **机制解析**

1. **保存张量与引用循环**：
   - 在反向传播过程中，PyTorch 需要存储计算图中的中间结果（即张量）以便在反向传播中计算梯度。
   - PyTorch 使用 `SavedVariable` 来保存这些中间结果。
   - 如果直接在计算图中持有对张量的引用，可能会导致**引用循环**。引用循环指的是两个或多个对象之间相互引用，导致 Python 的垃圾回收机制无法释放它们，从而引发内存泄漏。
   - 为了避免这个问题，PyTorch **打包**这些中间结果并在需要时再**解包**。

2. **张量打包与解包**：
   - 当你通过 `y.grad_fn._saved_result` 访问某个中间结果时，PyTorch 实际上已经将原始的张量打包并解包到一个新的张量对象中。这意味着你获得的张量与原始张量 `y` 不是同一个对象，但它们**共享相同的存储空间**（underlying storage）。
   - 共享存储意味着它们的内存区域相同，但对象的引用不同。这样做的目的是防止图中不必要的引用，避免内存问题。

一个张量是否被打包成不同的张量对象取决于它是否是其自身的 grad_fn 的输出，这是一个可能会发生变化的实现细节，用户不应该依赖它。

控制Pytorch如何进行packing/unpacking with [Hooks for saved tensors](https://pytorch.org/docs/stable/notes/autograd.html#saved-tensors-hooks-doc)

## 不可微函数的梯度
只有当所使用的每个基本函数都是可微的时候，使用自动微分的梯度计算才是有效的，但在实践中使用的许多函数都没有这个属性（在0处的relu或sqrt），为了减少不可微函数的影响，我们通过应用以下规则来定义初等运算的梯度。
1. 如果函数是可微的，因此在当前点存在梯度，使用它
2. 如果函数是凸的（至少是局部的），使用最小范数的次梯度（它是最陡的下降方向）
3. 如果函数是凹的（至少是局部的），使用最小范数的超梯度（考虑-f(x) 并应用前一点）
4. 如果定义了函数，那么可以通过连续性定义当前点的梯度（注意，这里可以使用inf，例如sqrt(0)），如果可能有多个值，任意选择一个。
5. 如果没有定义函数（例如，sqrt(-1), log(-1)或者大多数输入为NaN的函数，那么用作渐变的值是任意的（我们也可能会出现错误，但这是不能保证的）。大多数函数使用NaN作为渐变，但出于性能原因，一些函数将使用其他值（例如log(-1))。
6. 如果函数不是确定性映射（即它不是数学函数），它将被标记为不可微。如果张量在`no_grad`的环境之外需要梯度，这将会使它向后出错。

## 局部禁止梯度计算
设置上下文管理器`no_grad()`模式或者推理模式。更加细粒度的子图控制，使用`requires_grad`

下面，除了讨论上面的机制，我们还描述了评估模式(nn.Module.eval()） ，这是一种不用来禁用梯度计算的方法，但是由于它的名字，它常常与这三种方法混在一起。

### Setting `requires_grad`
在正向传播中，如果一个操作的至少一个输入张量需要梯度，则该操作仅记录在反向图中。反向传播中，只有叶张量带有`requires_grad=True`才会将梯度累积到它们的`.grad`属性中。

非叶张量都自动具有`require_grad=True`，因为需要中间张量的梯度以计算叶张量的梯度，设置`require_grad=True`只对叶张量有意义。如果逆需要在模型微调期间冻结预先训练好的模型的某些部分。

要冻结模型的某些部分，只需简单地应用`requires_grad(False)`，然后使用这些参数作为输入的计算不会记录在正向传播中，因此在反向传播中不会有`.grad`属性。

可以设置`nn.Module.requires_grad_()`，应用到整个module。