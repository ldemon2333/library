Proposed an approach to map very large scale SNN applications to neuromorphic hardware. The approach consists of two steps. Firstly, it solves the initial placement using the Hilbert curve. Secondly, the Force Directed (FD) algorithm formulates the connections of clusters as tension forces, thus converts the local optimization of placement as a force analysis problem.

# 1 intro
All of these neuromorphic hardwares share a common design rule to run SNN applications. Using a large number of specially designed neurosynaptic cores to store
synaptic weights and simulate neurons dynamics in parallel.

Two limitations are due to two main reasons:
- Most previous works mainly focus on optimizing the partition of neurons, while few addressed the placement of clusters.
- Existing methods are designed for relatively small scale ()

# 2 BackGround
## 2.1 Background of SNN Mapping Problem
A typical implementation of the neurosynaptic computing core is the analog crossbar.

The most common design in the latest large-scale neuromorphic systems is Network-on-Chip(NOC) with 2D mesh structures.

Based on this multicore architecture, SNN applications must be mapped into hardware. Mapping into hardware means 1) partitioning neurons in SNN into several clusters to meet the hardware constraints and 2) placing these clusters in neuromorphic computing cores.
![[Pasted image 20250417145918.png]]
The input of the mapping problem is an SNN application, which is represented as a graph consisting of neurons and synapses. Firstly, Neurons in the SNN application need to be partitioned into clusters due to the neurosynaptic core's capacity limitation. Based on the partition result, we can obtain the Partitioned Cluster Network (PCN). The connections between clusters preserve the information of synapses that connect neurons partitioned in separate clusters.

## 2.2 Related Work

# 3 Problem Formulation
## 3.1 Neuromorphic Hardware Model
The capacity of processing cores in different neuromorphic hardware varies greatly. Two constants, CON_npc and CON_spc, are used to address this difference in hardware constraints. CON_npc defines the maximum number of neurons that can be configured per core, while CON_spc defines the maximum number of synapses that can be configured per core.

## 3.2 SNN Application and Partitioned Cluster Network (PCN) Model
An SNN application is naturally modeled as a directed Graph
$$
G_{SNN} = (V_S,E_S,w_S) \tag{2}
$$
A partitione cluster can be mapped on an arbitrary core in neuromorphic hardware, which means it is the smallest unit that the mapping algorithm can schedule.

To quantify the output placement, we use five metrics to optimize the mapping task, that is, energy consumption, latency, and network congestion.

# 5 Experiments 
software
![[Pasted image 20250417153617.png]]

1) The baseline: Randomly mapping, clusters are randomly mapped into the neuromorphic hardware.
2) Truenorth
3) DFSynthesizer
4) PSO

## 5.2 Performance of the Proposed Approach
two parts. In the first part, we show the advantages of HSC compared with other space-filling curves and how HSC can cooperate with the FD algorithm to obtain optimal performance.
![[Pasted image 20250417154207.png]]
Based on the experimental results shown in Figure 8, we have the following observations:
1) HSC has significantly better performance than other space-filling curves. Other space-filling curves, ZigZag and Circle, will wander in a wide range of 2D space, which may lead to the adjacent and connected neurons being assinged to location far away from each other, resulting in the solution's performance being even worse than the baseline method. While HSC, by its nature, is well suited for mapping SNN.
2) Neither HSC nor FD algorithm can complete the mapping task well alone. 