# 1 Intro
SNN: spike-driven:
- the network's operations are synaptic ACcumulate(AC)

也就是说，现有的脉冲 Transformer 中既存在由 vanilla Transformer 组件主导的乘法累加 (MAC) 运算，也存在由脉冲神经元引起的 AC 运算。 Hybrid computing.

Two core modules of Transformer, Vanilla Self-Attention (VSA) and Multi-Layer Perceptron (MLP), are re-designed to have a spike-driven paradigm.

![[Pasted image 20250428230737.png]]

Q and K first perform similarity calculations to obtain the attention map, which includes three steps of matrix multiplication, scale and softmax. The typical spiking self-attentions in the current spiking Transformers would convert Q, K, V into spike-form before performing two matrix. The distinction is that spike matrix multiplications can be converted into addition, and softmax is not necessary.

Fig.1 (b): i) Hadamard product replaces matrix multiplication; ii) matrix column-wise summation and spiking neuron layer take the role of softmax and scale. 


# 2 Related Works
We employ the direct training method, using the first SNN layer as the spike encoding layer and applying surrogate gradient training.



# 3 Spike-driven Transformer
![[Pasted image 20250429112735.png]]
U[t] means the membrane potential which is produced by coupling the spatial input information X[t] and the temporal input H[t-1], where X[t] can be obtained through operators such as Conv, MLP, and self-attention.

## 3.1 Overall Architecture
![[Pasted image 20250429112213.png]]

For the SPS part, given a 2D image sequence 
![[Pasted image 20250429113450.png]]
