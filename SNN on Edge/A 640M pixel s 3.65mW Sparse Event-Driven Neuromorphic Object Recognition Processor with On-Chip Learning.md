# Abstract
使用稀疏特征提取推理模块 (IM) 和任务驱动的字典分类器设计了一个 1.82mm2 65nm 神经形态物体识别处理器。为了实现高吞吐量，256 个神经元的 IM 被组织成四个并行神经网络，以处理四个图像块并生成稀疏神经元尖峰。片上分类器由稀疏神经元尖峰激活以推断物体类别，从而将其功率降低 88% 并通过删除所有乘法来简化其实现。轻量级协处理器利用稀疏神经元活动执行高效的片上学习，以节省 84% 的工作负载和功率。测试芯片处理 10.16G 像素/秒，耗散 268mW。集成的 IM 和分类器为电压缩放提供了额外的误差容忍度，在 640M 像素/秒的吞吐量下将功率降低至 3.65mW。

# 1. 介绍
识别图像中的物体可以通过首先使用推理模块 (IM) 从图像中提取特征，然后使用分类器根据提取的特征对物体进行分类来完成（图 1）。局部竞争算法 (LCA) [1] 是一种受神经启发的 IM，它推断出一组稀疏的特征来最好地表示输入图像。与传统的特征提取算法（例如 SIFT [2]）相比，基于 LCA 的 IM 简化了分类器并可能提高了分类准确性。过去的工作已经产生了一个基于 18 个神经元脉冲 LCA 的模拟 IM [3]（图 2），但规模太小不适合实际问题。使用 SAILnet [4] 的 256 个神经元数字 IM 是可扩展的，并实现了更高的吞吐量（图 2），但该设计以内存为主，无法进行对象分类。在这项工作中，我们展示了一个基于 256 个神经元、10.16G 像素/秒脉冲 LCA 的 IM，它与任务驱动的字典分类器集成，以利用端到端对象识别处理器的稀疏特征提取。==芯片上集成了一个轻量级学习协处理器，用于学习和调整特征库。这种独立的设计支持低功耗、高性能的计算机视觉嵌入式应用。==

# 2. Sparse Feature Extraction and Sparse Event-Driven Classifier
256 个神经元的 IM 被组织成四个 64 个神经元的脉冲神经网络，每个神经网络对 16×16 输入图像块进行操作以并行提取特征（图 3）。通过训练，每个神经元都会形成一个 16×16 的接受场 (RF)，即激发神经元的特征。将输入图像划分为较小的块会减小神经网络的大小，从而导致内存大小减少 4 倍。通过并行部署四个神经网络，可以实现高吞吐量和相当的推理精度，而无需与大型神经网络相关的内存开销。神经元动力学经过调整，可实现高推理精度，推理次数小于 16 次，并实现稀疏事件驱动分类器和轻量级片上学习。

