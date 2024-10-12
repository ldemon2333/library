 tensors和 NumPy 数组通常可以共享相同的底层内存，这样就不需要复制数据了(参见使用 NumPy 的 Bridge)(see [Bridge with NumPy](https://pytorch.org/tutorials/beginner/blitz/tensor_tutorial.html#bridge-to-np-label))

tensors也针对自动微分进行了优化

Over 100 tensor operations, including arithmetic, linear algebra, matrix manipulation (transposing, indexing, slicing), sampling and more are comprehensively described [here](https://pytorch.org/docs/stable/torch.html).

跨设备复制大tensors在时间和内存方面是昂贵的！

torch.cat 与 torch.stack区别
cat不改变现有的维度，stack形成更高维的张量


In-place 操作将结果存储到操作数中的操作称为就地操作。它们由 _ 后缀表示。例如: x.copy _ (y) ，x.t _ () ，将改变 x。
就地操作可以节省一些内存，但是在计算导数时可能会出现问题，因为会立即丢失历史记录。因此，不鼓励使用它们。

CPU 和 NumPy 数组上的张量可以共享它们的底层内存位置，更改其中一个将更改另一个。