Q：
- 具体发表在那个期刊上，具体要写哪些部分？期刊具体格式是怎样的
TKDE

- MM 具体介绍了 5 种推荐算法模型 GRU4Rec，SRGNN，SASRec，Caser，FMLP 和其5种变种 SGRU4Rec,SRSGNN, SFSRec, SCaser, and SFMLP 然后进行比较，这里数据集是面向多模态的，ijcai 这篇文章的实验是没有多模态的，这里怎么合并这两篇文章，在 MM 文章里加入自己新创的 ijcai里的算法，在进行比较，也就是 6 种 Spike 的推荐算法比较，但是这里数据可能是不一致的问题
SNN 在 SR 应用 ijcai table 1 SNN vs ANN ，后介绍，
MM SNN 迁移影响不大，先介绍，
过度，根据 复杂度，方法论的出发点
intro 参考 ijcai 

MM t3 后两列不要，拆成 4 表放在方法论

Wei Ji, Xiangyan Liu, An Zhang, Yinwei Wei, Yongxin Ni, and Xiang Wang. 2023.
Online distillation-enhanced multi-modal transformer for sequential recommen-
dation. In Proceedings of the 31st ACM International Conference on Multimedia.
955–965. 参考他的方法论

- ijcai 是否中了



# Abstract
设备上的序列推荐 (SR) 系统旨在利用实时特征进行本地推理，从而减轻服务器端推荐系统在处理数百万用户的并发请求时的通信负担。然而，边缘设备的资源限制，包括有限的内存和计算能力，对部署高效的 SR 模型构成了重大挑战。受深度脉冲神经网络 (SNN) 的节能和稀疏计算特性的启发，我们提出了一种经济高效的设备上 SR 模型 SSR。该模型将密集的嵌入表示编码为稀疏的脉冲表示，并集成了新颖的脉冲滤波器模块以从项目序列中提取时间模式和关键特征，从而在不牺牲推荐准确性的情况下优化了计算和内存效率。在真实数据集上进行的大量实验证明了 SSR 的优越性。与其他 SR 基线相比，SSR 在实现相当的推荐性能的同时，平均能耗降低了 59.43%。此外，SSR 显著降低了内存占用，使其特别适合部署在资源受限的边缘设备上。

# 1 Introduction
序列推荐 (SR) 系统 [Wang et al., 2019] 已成为 Shopee 和 TikTok 等无处不在的网络应用不可或缺的一部分，这得益于其卓越的建模用户历史行为和发现各种产品和服务中潜在偏好的能力，最终推动了业务收入的增长。大多数传统的 SR 系统完全是服务器端解决方案 [Kang and McAuley, 2018; Lai et al., 2024]，严重依赖于云端的海量存储、内存和计算资源 [Steck, 2019]。然而，这种基于云的范式面临着严峻的挑战，例如高度依赖网络条件、传输延迟以及用户隐私的巨大风险 [Yao et al., 2022; Xia et al., 2024]。

随着智能边缘设备计算能力的不断增强，在边缘部署 SR 模型成为一种颇具前景的解决方案。
通过启用本地推理，基于边缘的 SR 系统可以降低通信开销，提高实时响应能力，并提供固有的隐私优势，因为敏感数据仅保留在设备本地 [Xia et al.,2022; Yin et al., 2024]。然而，当前的设备端 SR 模型面临着以下潜在挑战：

高内存开销。在标准 SR 任务中，商品标记序列被用作唯一模态，需要高维嵌入来捕获每个商品的广义表示 [Zivic et al., 2024]。这会导致大量的内存占用，尤其对于拥有大量商品目录的平台而言。例如，亚马逊的推荐系统必须管理超过 3.5 亿个商品嵌入 [Chen et al., 2022]，这导致资源利用效率低下并增加了运营成本。

高能耗成本。设备端 SR 模型通常运行在电池供电的移动设备上，例如智能手机，而可持续性对这类设备至关重要。目前的 SR 模型依赖于具有大规模架构和全精度计算的人工神经网络 (ANN)，这会导致显著的能耗。例如，在谷歌眼镜 [Venkataramani et al., 2016] 等设备上部署基于 ANN 的模型，视频流每帧消耗 0.15 焦耳，这使得 2.1 瓦时容量的电池续航时间仅为 25 分钟，从而降低了可用性和用户体验。

为了应对这些挑战，我们提出了一种脉冲序列推荐 (SSR) 模型，该模型利用了脉冲神经网络 (SNN) 的计算范式。该模型非常适合部署在配备神经形态芯片的智能边缘设备上 [Yao et al., 2024]，这些芯片是专门为模拟神经网络架构而设计的硬件，可实现高效、低能耗的计算。为了克服密集嵌入在内存和计算方面的局限性，我们引入了一种神经编码方法，将原始嵌入表转换为基于脉冲的表示，从而使 SSR 能够更高效地处理序列特征。与传统的密集嵌入向量相比，该方法可以减少存储限制并加快计算速度。此外，深度 SNN 在序列任务中的最新突破 [Lv et al., 2024;Xing 等人，2024] 的研究阐明了开发设备端 SR 模型的可行性，因为它们具有节能优势 [Maass, 1997]。这项研究包含模拟傅里叶变换机制的脉冲滤波器模块，使 SSR 能够有效地处理脉冲信号，并从异步离散脉冲序列中提取动态用户偏好。

为了证明 SSR 的有效性，我们在真实数据集上进行了广泛的实验。结果表明，与多个基于 ANN 的当前最佳 (SOTA) 基线相比，SSR 能够以更低的计算能耗实现相当甚至更优的推荐性能。SSR 还能生成稀疏项目表示，以简化计算、减少内存占用并加快推理速度，这对于设备端 SR 部署至关重要 [Yin et al., 2024]。总而言之，我们的贡献可以概括如下：

- 本研究首次提出了一种具有深度 SNN 的设备端 SR 模型，为增强智能边缘设备上的用户体验提供了一种经济高效且节能的解决方案。
- SSR 生成二进制项目表示，显著减少内存占用。它还集成了脉冲滤波器模块，能够有效地从异步和稀疏输入中提取个人偏好。
- 在五个数据集上进行的大量实验结果表明，SSR 可以实现与七个 SOTA SR 模型相当甚至更优的推荐性能，而且能耗更低。

# 2 Related Work
设备端顺序推荐。近年来，为了满足用户对低延迟响应的需求以及日益增长的数据隐私问题，设备端 SR 系统应运而生 [Yin et al., 2024]。Aerorec [Xia et al., 2024] 引入了一种自监督知识蒸馏方法，以减轻压缩设备端推荐模型的准确率损失。OD-Rec [Xia et al., 2023] 通过离散组合代码学习来压缩商品嵌入表，从而高效地实现设备端推荐。DIET [Fu et al., 2024] 为每台设备定制 SR 模型，以最大限度地减少带宽使用和存储消耗。现有方法通常通过压缩 [Yuan et al., 2020] 和量化 [Li et al., 2023] 等技术来降低推理延迟和能耗。然而，这些方法往往会降低特征表达能力，从而降低模型性能，尤其是在大规模推荐系统场景下 [Hou et al., 2023]。这促使我们探索一种新的设备端SR模型，以更好地平衡计算能耗成本和推荐准确率。

基于脉冲神经网络的推荐。一些研究人员已经研究了将脉冲神经网络 (SNN) 集成到传统协同过滤 (CF) 推荐模型中的可行性。[Zhu et al., 2022] 引入了脉冲图 (spike-wise graphs) 来降低基于图的协同过滤 (CF) 模型的内存成本。[Ma et al., 2023; Agarwal et al., 2024] 利用 SNN 的稀疏编码机制将原始特征转换为脉冲信号，从而提升了协同过滤 (CF) 模型的性能。
这些研究展示了 SNN 在增强特征表示和提升推荐性能方面的潜力。然而，现有研究大多侧重于协同过滤 (CF) 模型，而忽略了协同过滤 (SR) 任务。此外，目前还没有专门为神经形态芯片设计的与 SNN 相关的 SR 算法 [Ma et al., 2024]，这使得 SNN 在节能边缘设备部署中的低能耗优势尚未得到充分挖掘。为了弥补这一差距，我们提出了 SSR，这是第一个利用脉冲神经网络实现节能顺序推荐的设备端 SR 模型。


# 4 Methodology
## 4.2 SSR Model
![[Pasted image 20250509200400.png]]
embedding layer, stimulus encoder, spiking filter layer, PSFFN layer, and prediction layer.

嵌入矩阵 X
![[Pasted image 20250509195827.png]]

As depicted in Figure 2, one dense embedding table can be converted into T sparse binary tables by the temporal encoding mechanism of spiking neurons (In this study, N should always be ensured to be divisible by T).

==temporal encoding 是如何进行的==，

==由公式 (8) 得出的原始脉冲序列提供了每个项目标记的稀疏表示，但无法捕捉序列特征。？?==

![[Pasted image 20250509200629.png]]
旨在聚合脉冲级信号并生成包含序列特征的刺激表征。 This module comprises two components: (1) 串行呈现编码器采用线性变换移动块实现。它捕获序列中跨时间步骤的累积脉冲信息，从而有效地提取长期兴趣。（2）并行呈现编码器充当全局级聚合器，利用另一个权重矩阵对每个时间步的序列中的项目执行加权求和，从而捕捉点级偏好模式。这两个编码器生成的刺激被聚合在一起，形成一个新的密集刺激，该刺激保留了序列的时间和上下文模式，然后被输入到 LIF 层 SN(·)，以构建脉冲编码的序列特征 H ∈ IT × N × D，用于进一步的模型学习。附录 B 提供了刺激编码器整个工作流程的伪代码。

考虑到本研究的主要重点是设计一个可行的基于 SNN 的模型来完成设备上的 SR 任务，
我们提供了更多基于其他编码方法的实验，并在附录 C 中进行了分析。结果表明，我们的刺激编码方法在大多数情况下都能很好地适应。

Spiking Filter Layer

实验又在 神经拟态芯片上做吗，用的是那块神经拟态芯片？
