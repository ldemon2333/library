One source of approximation errors is that in time-stepped simulations of SNNs, the neurons are restricted to a firing rate range of $[0, r_{max}]$, whereas ANNs typically do not have such constraints.

导致需要对激活值进行量化，确保能将激活值映射到发放率这个区间上，更好的近似真实的发放率。激活值大的，其发放率也是大的。但要有区分。
# 转换流程
- 训练 ANN 模型
- 激活值记录：在 ANN 的训练过程中，插入电压钩子（Voltage Hook）以记录每层网络的激活值
- 归一化处理：对每层神经元的激活值进行归一化，确保 ANN 中的权重在 SNN 中依然能产生合理的神经元发放行为。最常用的两种归一化方法是基于最大值的 MaxNorm 和基于分位数的 RobustNorm。
- 替换为脉冲神经元：将 ANN 中的连续激活函数（如 ReLU）替换为 SNN 中的脉冲神经元（如 IF 或 LIF 神经元），并应用归一化系数对输入电压进行缩放

## MaxNorm 归一化
MaxNorm 适用于没有大量噪声或异常激活值的数据。核心思想是将每层神经元的输入电压缩放到其激活值的最大范围内，以确保神经元能够有效发放脉冲。
1. 激活值的最大值收集：遍历训练数据集，获取记录每一层 ReLU 激活的最大值（$s_{max}$）
2. 转换为 SNN：替换 ReLU 层为 IF 神经元，激活值通过一个比例缩放：
$$
input = \frac{input}{s_{max}}, output = output \times s_{max}
$$
```
model._modules[name] = nn.Sequential(
    VoltageScaler(1.0 / max_item),    # 缩放输入
    neuron.IFNode(v_threshold=1., v_reset=None),    # IF神经元
    VoltageScaler(max_item)    # 恢复输出
)

```

## RobustNorm 归一化
适用于数据中可能包含噪声或异常激活值的情况。与 MaxNorm 不同，使用激活值的某个高分位数（99.9%）来确定归一化系数
![[Pasted image 20250423164401.png]]

这种方法减少了极端激活值对归一化过程的影响，确保模型在数据分布复杂或含有噪声的情况下能够保持性能。
1. 激活值的分位数收集
2. 归一化权重和偏置：在替换神经元之前，对权重和偏置进行缩放，确保层与层之间的比例一致。
3. 转换为 SNN：类似 MaxNorm，将激活值进行分位数缩放
这种方法通过调整每一层的权重，进一步优化了层间的信息传递，减少了转换过程中精度的损失。
```
# 在替换神经元之前，调整权重
if self.prev_scale is not None:
    current_scale = max_item
    prev_scale = self.prev_scale
    module.weight.data = module.weight.data * (prev_scale / current_scale)
    if hasattr(module, 'bias') and module.bias is not None:
        module.bias.data = module.bias.data * (prev_scale / current_scale)
self.prev_scale = max_item

```

