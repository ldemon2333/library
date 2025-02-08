# 1. Turing Machine and Von Neumann Architecture
1936 年，艾伦·图灵提出了一个理想化的计算模型，它由一条无限长的磁带和一个读写头组成[1]。
这个简单、坚固、易于理解的模型现在被称为图灵机，并从此成为计算机界的主要计算模型。

在图灵机中，无限长的磁带被视为连续的单元，每个单元包含一个符号（“0”，“1”或“空白”）。读写头可以沿着磁带移动并改变其目标单元。它既可以读出目标单元中的符号，也可以将新符号写入单元中。

图灵机的“程序”定义为有限状态机 (FSM)，用于指导磁头的移动和磁带的修改。FSM 通常定义为五元素元组 $\psi=(Q, \Sigma, \delta, q_0 ,F)$ where $Q$ is a finite set of states, $\Sigma$ is a finite, nonempty input alphabet (for Turing machine, the alphabet is from {0, 1, blank}), $\delta$ is a series of transition functions, $q_0$ is the starting state, and $F$ is the set of accepting states. 转移方程定义为一种映射 $f:q,c\rightarrow q,c,m$, where $q$ is the current state of the FSM, $c$ is the content of the cell under the read/write head, $m$ is the corresponding movement of the head (move left, move right, or no move).

当执行有限状态机时，图灵机从有限状态机指定的状态 $q_0$ 开始，然后移动读写头，根据转换函数修改磁带内容，当到达接受状态或没有转换函数可用时，计算结束。

图灵机在计算界中扮演着基础性角色，因此有不少研究试图增强或推广它。例如，通用图灵机 (UTM) 首先将 FSM 存储在磁带上，然后在执行专用计算之前读入 FSM。由于图灵机是一个顺序模型，也有几项研究试图扩展它以支持并行计算，包括扩展到多个磁头/磁带[2][3] 和多个维度[4][5][6]。

基于图灵机，提出了数据操作规则系统的一个重要分类，即图灵完备性。如果一个系统可以用来模拟任何图灵机，则称其为图灵完备。图灵完备系统至少具有与图灵机相同的计算能力。

1945 年，约翰·冯·诺依曼提出了一种存储程序计算机，它由控制单元、算术和逻辑单元、内存单元和输入/输出组成[7]。计算机将指令和数据一起存储在内存中，然后串行获取和执行指令。这种模型现在被称为冯·诺依曼架构，它为几乎所有现代通用计算机架构奠定了基础。

图灵机和冯·诺依曼体系结构在传统的计算机体系结构中占据着核心地位。图灵完备性保证了用现代编程语言描述的任何程序都是执行图灵机的过程（几乎所有的主流编程语言都是图灵完备的）。冯·诺依曼抽象体系结构模型通过图灵完备的指令集来支持图灵机，这使得任何高级语言程序都可以在冯·诺依曼处理器上编译成严格等价的指令序列。

也就是说，基于图灵完备性，计算机层次结构提供了软件（即编程语言及其上的所有应用程序）和硬件（即各种冯·诺依曼处理器）之间的兼容性和灵活性：我们在设计更复杂的编程语言的同时，不断提高硬件效率。两者始终兼容。这种层次结构还通过定义/扩展不同的指令集，有利于软件硬件协同设计。

# 2. Neuromorphic Completeness
## 2.1 Universal Approximation Theorem
逼近是神经网络的固有属性。自 Kurt Hornik 等人 [8] 提出通用逼近定理以来，人们就开始研究神经网络的逼近能力。该定理证明了存在一个三层前馈网络，该网络具有有界的非常量半线性函数，例如 S 型函数或阈值函数，可以很好地逼近任意连续函数。Blum 等人 [9] 和 Kolmogorov 等人 [10] 也对逼近性质提供了建设性的证明。

通用逼近器的工作方式与图灵机计算函数的方式截然不同。通用逼近器只是记住函数的映射，而不是通过算法来计算它。更具体地说，它通过直接记住映射来实现图灵可计算函数，而不是通过模拟图灵机来求解它们。因此，它不是图灵完备的。

Accordingly, we introduce the concept of _neuromorphic computing capability_ and _neuromorphic complete_ to combine the approximation property with Turing completeness.

## 2.2 Neuromorphic Computing Capability
Given an error gap $\epsilon \geq 0$, for any function $f_A$ that one computational system $A$ can achieve, if there is a function $f_B$ that can be achieved by system $B$ and $||f_A(X) -f_B(X)||\leq \epsilon$ where $X$ is any valid input, it is called that system $B$ has the equal or stronger _neuromorphic computing capability_ compared to system $A$.

## 2.3 Neuromorphic Complete
If a computational system has the equal _neuromorphic computing capability_ compared to a Turing complete system, it is _neuromorphic complete_. 即，神经形态完整系统可以以任意精度近似任何图灵可计算和可终止的函数。

神经形态完备的定义放宽了图灵完备的要求，从根据可组合算法精确计算函数到仅仅近似计算。

The universal approximators and _Turing-complete_ systems are both _neuromorphic complete_. 它们是两种不同类型的典型神经形态完整系统。
- Universal approximators (e.g. multilayer perceptron): 通用逼近器可以以任意精度逼近任何函数。因此，它们是神经形态完整的。它们主要利用神经网络的逼近特性
- Turing-complete systems (e.g. universal Turing machine): 图灵完备系统可以精确地逼近任何函数。因此，它们也是神经形态完备的。它们主要利用可组合算法（即有限的可执行步骤序列）来精确计算函数。

通用近似器不可组合：近似复杂函数的多层感知器不是由多个近似简单函数的 MLP 组成。相反，我们可以在图灵机上定义一些程序，并且可以基于它们构建更复杂的程序；计算是准确的。

可组合性和近似性对于脑启发计算都很重要。所提出的执行原语图可以很好地将它们结合在一起。

# 3. Programming Operator Graph
We propose Programming Operator Graph (POG) as a unified description method and program execution model for brain-inspired computing. In POG, a program is described as an operator graph and then executed according to the program execution model (PXM) of POG.

## 3.1 Operator Graph
### 3.1.1 Operator
操作符是计算和调度的单位，即 POG（扩展数据图 1-a）中程序的基本描述和执行单位。操作符可以是一条或多条指令，也可以是一个复杂的函数，它有三个参数：输入、输出和参数。输入是来自其他操作符或环境的输入数据，输出是该操作符产生的结果，参数是只能由操作符本身访问的内部数据。An _operator_ that has received all the input events could be triggered for execution, i.e. _enabled_. Execution of an enabled _operator_ will consume all the ready input events, carry out the computation and generate corresponding output events that will be delivered to the destination _operator(s)_.
![[Pasted image 20241225160413.png]]

### 3.1.2 Operator Graph
A program in POG is defined as a directed graph named _operator graph_ (OG) (Extend Data Fig. 1-b). In OG, each node represents an _operator_ and a directed edge means a precedence relationship between the _operators_. 

An event $t$ on edge $e_{v_1, v_2}$ means that the precedence relationship between $v_1$ and $v_2$ is satisfied. 事件可能有多种类型，从仅表示依赖关系的纯信号到提供具体数据的复杂结构。有三种重要的事件类型：The first is the date event which provides concrete data _value_ as the input $i$ of the destination _operator_, expressed as $t_i=value$; the second is the signal event $t_s$ which indicates that the precedence relationship is satisfied; and the last one is the _parameter event_ that updates dedicated parameters.
![[Pasted image 20241225160423.png]]

## 3.2 The program execution Model
程序执行模型 (PXM) 不仅定义了程序是什么，它还包含一个定义程序如何执行的活动模型、一个用于数据管理和访问的内存模型以及一个处理并发活动之间交互的同步模型。图灵机是一个典型的 PXM，由一个基于有限状态机 (FSM) 的活动模型、一个作为内存模型的无限磁带和一个基于读写头顺序操作的同步模型组成。POG 的 PXM 包含一个基于操作符的活动模型和一个内存模型，该模型将处理和存储并置在一起，并消除了同步模型的必要性。

### 3.2.1 Activity model
The active model of POG is a finite state operator graph (FSOG) which is expressed as a five-elements tuple $\epsilon = (G, T,\delta,q_0,F)$. $G$ is the _operator graph_, $T$ is the set of available events, $\delta$ is the transition functions, $q_0$ is the start state, and $F$ is the set of the accepting states.

### 3.2.2 Memory model
In detail, each _operator_ possesses an infinite amount of private memory to store parameters or buffer the inputs and outputs.

As the data transferring between different _operators_ can only be achieved through output-to-input connections and the private memory can only be accessed by the owner, the interactions between _operators_ are stacially defined. Thus, no synchronization model is necessary.

## 3.3 Assistant Operators
Such as the _parameter updater_ that updates the internal parameters of the operators, 控制流算子可以构建通用的控制流模式，以及几个神经形态相关的算子。利用这些辅助算子，可以构建复杂的程序，如SNN、DNN等。

### 3.3.1 Parameter Updater
![[Pasted image 20241230121810.png]]
The _parameter updater_ is a special input edge that indicates the modification of the corresponding parameters. The event on _parameter updater_ is termed _parameter event_.

The _parameter updater_ enables fine-grained modification on coarse-grained parameters, like updating a dedicated column of elements of a matrix parameter.

### 3.3.2 Control-Flow Operators
![[Pasted image 20241230122051.png]]

Based on these _operators_, we can define the branch and loop operations in common programming languages easily. 
![[Pasted image 20241230122256.png]]

### 3.3.3 Neuromorphic Related Operators
In most neuromorphic models, the neuron may have input connections from many other neurons and its internal states should be updated every time a spike arrives. This pattern requires that the computation of the neuron driven by any single input token instead of all the input tokens.
![[Pasted image 20241230122836.png]]The synapse operator carries out a simple weighted-sum operation, but this operator is enabled when any of the input token arrives.

With the synapse operator, it will be much easier to build neuromorphic models, for example, the LIF model. In this case, we use two synapse operators to deal with the excitory and inhibitory input connections. The major computation operator LIF is also a weighted-sum operator. We also use a conditional merger to reset the membrane potential of a fired neuron. Moreover, a parameter updater is introduced to update the internal membrane potentials.
![[Pasted image 20241230123321.png]]

## 3.4 Composability
As the _operator_ of the POG may contain complicated instructions or even a whole algorithm, a sub-OG can be viewed as an _operator_.

It is possible to describe a neural network application as an OG that only contains basic computational and control-flow _operators_ and then compose some sub-graphs of this OG to new _operators_ to suit multi-granularity hardware operations.

From the perspective of hardware and software co-design, this property is also very useful: 我们可以根据底层硬件提供的具体操作来定制编程操作符集，反之亦然。由于可组合性，这些修改只能发生在 POG 级别，不会影响上层编程范式。

## 3.5 Proof of Turing Completeness
The POG has control flow operators and operators that may update their own parameters; Here we propose a method to represent an FSM of the Turing machine with a corresponding POG which contains a limited number of operators, to prove that the POG is Turing complete.

It means that we can mimic the Turing machine through POG, namely the POG is Turing complete.

## 3.6 Relationship with Dataflow
### 3.6.1 Dataflow Model
数据流模式以指定计算顺序和它们之间传输的数据的形式表示程序的逻辑过程.

Dataflow schema outlines the data dependence between different instructions directly, thus it is beneficial to exploit fine-grained parallelism and to achieve high throughput.

粗粒度数据流模型[18]沿袭了原有数据流模式的基本计算模式，并进一步放宽了对基本计算单元的要求。在粗粒度数据流中，每个计算单元可以是一组应原子执行的指令，即节点可以是一组指令，而不是单个指令。

由于数据流模式的程序被描述为有向图，程序员可能会定义一个包含死锁、不可达节点等的病态图。为了指导程序员，提出了行为良好和结构良好的概念[19]。

行为良好的节点必须没有循环依赖关系。当所有输入边都有事件（即 token）时，它应该被启用，并且它必须是自我清理的（即，在执行节点后，图必须返回到其原始状态）。因此，行为良好的节点的执行将消耗其输入边上的所有匹配事件并为所有输出边生成事件，并且它不会影响任何其他节点的状态。

结构良好这一概念与行为良好密切相关。有一套基于一组较小的行为良好图构建较大行为良好图的构造规则：

常规数据流节点是结构良好的图。如果每个分支的子图都是结构良好的，则条件图是结构良好的。如果循环体是结构良好的图，则循环图是结构良好的图。如果 DAG 的每个节点都是结构良好的，则参考有向无环图 (DAG) 构建的图是结构良好的图。如果该图的每个节点都表现良好，则保证结构良好的数据流图表现良好。

数据流模式本身就支持并行性。由于它是一个纯活动模型，因此有几项研究尝试将其与内存模型和同步模型相结合，以构建并行PXM，例如并行图灵机[20]。

### 3.6.2 Advancement
POG 很大程度上受到数据流模型的启发。与数据流一样，它是一个由事件驱动的异步并行模型。相应地，OG 受到传统数据流模型的数据流图的启发，并且它也具有行为良好和结构良好的特性。

但是，POG 与数据流模型的一个根本区别在于，POG 包含内存模型。数据流模型是纯活动模型，不涉及内存模型或同步模型。更糟糕的是，数据流模型通常提倡一些基于冯·诺依曼体系结构的一些基本概念的扩展模型，例如指令和单分配共享内存模型。因此，它们不适合脑启发式计算硬件。相比之下，POG 提出了一种将存储和处理共置的内存模型，避免了传统内存模型中并发访问同一内存位置引起的冲突；因此，它更合适。

POG 和数据流之间的另一个重要区别是事件的概念。虽然 POG 的事件类似于数据流模型中的数据可用性，但它也可以充当简单的优先关系，而不是特定的数据。因此，它提供了更大的灵活性和表现力。事件甚至可以表示从计算本身之外生成的数据/信号，这使得 POG 能够在开放环境中描述计算。此外，数据流和所有派生模型都依赖于基于图灵完备性的严格等价性。


# 4. Execution Primitive Graph
执行原语图 (EPG) 在脑启发计算系统层次结构中的位置类似于现代计算机系统中的指令集。它接近硬件实现，同时提供一定程度的抽象。我们使用两层图形表示来表达它。补充图 2 显示了两层图的示例，用于表示 LIF 神经元模型。
![[Pasted image 20241230125938.png]]

The tier-one is a control-flow graph (CFG). In the CFG, each node represents a basic block that contains pure computations defined by the tier-two graph.

The tier-two is a simplified dataflow graph: Each node is an _execution primitive_, which is a basic computational function that can be executed directly in the hardware. 

Further, if a node has more than one successor nodes (different nodes are in different branches, that is, in different basic blocks), its output port(s) will be connected to the corresponding input ports of all the successors. 

Thus, the execution order of the whole EPG is controlled by the tier-one CFG: only one basic block is executing at any time, and only the _execution primitives_ inside the executing block will be executed. A basic block finishes the exeution when all _execution primitives_ inside it have completed, and the control will be transferred to the next basic block according to the branch condition (if existing).

## 4.1 Basic Execution Primitives
The _basic execution primitives_ just contain two primitives:
- Weighted-sum: 
- ReLU

According to the universal approximation theorem, these two primitievs is enough to approximate any function.

Moreover, real neuromorphic hardware can provide more _execution primitives_ for efficiency.

## 4.2 Wide Compatibility of the Basic Execution Primitives
The following is proof that the LIF model essentially contains these two basic primitives.

对于复杂的图，相邻的线性运算总是可以合并为一个线性运算。因此，整个图可以表示为由非线性运算分隔的多个线性运算。我们可以将线性运算与以下非线性运算连接起来，并使用硬件支持的 LIF 神经元来实现它们。

## 4.3 Proof of Neuromorphic Completeness
The EPG can approach any Turing-computable functions in different ways: We can reduce the whole EPG to a single MLP that can approximate the entire function directly.或者，我们首先实现某些图灵完备系统中使用的基本算术函数（例如基本布尔运算），然后用它们精确地组合任何函数，然后 EPG 就变成了一个使用可组合算法来接近该函数的图灵完备系统。此外，EPG 可以通过使用 MLP 近似更粗粒度的函数来混合这两种方法，并将它们结合起来以精确地实现给定的函数。粒度可以从基本布尔运算到整个函数。

Here, we provide a constructive proof of how to approximate arbitrary functions with the _basic execution primitives_.

## 4.4 Support of Learning Rules
However, this implementation is really inefficient with the _basic execution primitives_. We could add an additional _learning primitive_ in the EPG for hardware to support learning efficiently. The _learning primitive_ is the ternary version of weight-sum primitive. It corresponds to the weight-sum operation with the _parameter updater_ in POG: An update event in the POG corresponds to the new input $w$ and $b$, for the _learning primitive_.

# 6. Abstract Neuromorphic Architecture
The _Abstract Neuromorphic Architecture_ (ANA) is an abstract hardware model that supports EPG. It has four parts: processing units (PUs) for _executing primitives_, scheduling units (SUs) for control flow and execution scheduling, the private memory equipped with each PU for storing the parameters and the intermediate data, and the inter-connection network that connects all the above-mentioned units.

需要指出的是，SU 是一个逻辑概念，但有几种物理方法可以实现它：它可能是一个负责所有控制和调度作业的集中式组件，或者 SU 可能是分布式的，每个 SU 负责分配给它的部分。即使在一些硬件实现中，也没有功能独立的 SU，这并不与所提出的抽象模型相矛盾。

PU 被设计为分布式的；它们可以并行地支持 EPG 的运行，也就是说，任何 PU 都可以在 SU 的调度下执行执行原语。此外，PU 配备了一个只能由 PU 本身访问的私有内存。这种设计使 ANA 的特定硬件实现有利于共置存储和处理，这是脑启发计算的关键特征之一，同时仍然保持与内存和计算分离的硬件的兼容性。PU 的执行可以是事件驱动的，也可以不是。

此外，硬件往往在内核或芯片之间的连接设计上投入大量精力，而这些连接可以包含不同层次的各种连接方式。ANA 的互连网络是功能层次的抽象。

因此，ANA 的内存抽象适合存储和处理的搭配，分布式 PU 适合细粒度并行计算，SU 和 PU 的分离设计有利于事件驱动计算。它还足够直观和通用，可以覆盖主流的神经形态芯片（补充信息 6.1），并且可以为硬件设计提供指导。

## 6.1 Instantiations
Specific hardware implementations of these units are diverse.

PU: The primary task of PU is to execute the _execution primitives_, so there is no constraint on the implementation form, e.g. it can be implemented as a control flow architecture or a dataflow architecture.

SU: SU implementations are very flexible, from the traditional control-flow soft cores to some customized hardware, 对于一些没有专用控制流硬件的实现，
工具链会将 EPG 的一级控制流模式转换为 PU 可以实现的执行原语（补充信息 7.2.3）。

Private Memory: 有些硬件采用存储与处理共置的内存，将私有内存与PU合二为一，进行就地计算。这通常是基于非易失性技术实现的，如忆阻器交叉开关、STT-RAM等。也可以使用通用存储技术，如SRAM、DRAM等，实现近内存计算[30]。有些硬件将内存与PU分离，内存模块可以是私有的、共享的或全局的。

Inter-connection Network: 通常类脑计算系统追求高可扩展性，因此存在不同层次的连接，如核心级、芯片级、板级，实现方式非常灵活。一种广泛使用的 AER（异步地址事件）通信系统[31][32]适用于事件驱动的计算。

### 6.1.1 TrueNorth
Software: Different software pipelines are provided for application runtime and development, as well as a TrueNorth simulator.

### 6.1.2 SpiNNaker
Software: The software can be categorized into host software and SpiNNaker software (running on chips). SpiNNaker software is further divided into control software and application software: The former is a primitive operating system resident in the monitor processor of each chip, for task control, computation scheduling, I/O, etc., while the latter performs users' tasks.

# 7. Design Guidelines of the Mapper
The mapper is used to deploy the EPG onto the specific hardware (this procedure is called _mapping_). The mapping result should be as efficient as possible under the premise of meeting the hardware restrictions, like the capacity limits of memory and network routing.

Hardware constraints: 物理上，硬件资源的容量或能力是有限的，例如内存、路由、调度等。映射器应该考虑所有这些限制并生成有效的策略。

效率：EPG 在计算、通信和存储方面的需求可能不平衡：以 CNN（卷积神经网络）为例，卷积层计算成本高且使用的参数相对较少，而全连接（FC）层则恰恰相反。考虑到共置的计算和存储资源，简单的映射策略会导致 FC 层占用大量计算/存储资源，但计算资源的利用率却很低。

## 7.1 General Mapping Framework
映射过程一般包含三个阶段：将EPG的任务均匀分配给PU、在每个PU上调度原语的执行、将每个原语适配到PU。

在第一阶段，将原始 EPG 划分为多个子图，并将每个子图分配到一个 PU 上。应进行平衡划分以接近最大流水线方式和更好的负载平衡 [39]。每个子图的相应控制流模式和数据存储也将分配给其 PU 的 SU 和私有内存。存储的分配策略可能因硬件实现的不同而不同（补充信息 7.2.2）。

在调度阶段，可以对每个子图进行静态或动态调度。静态调度不依赖于运行时信息，因此它可以在运行之前确定每个原语的执行时间。动态调度则相反，它必须在运行时生成调度规则（例如，使用 if/else 或循环条件来控制是否以及以何种顺序动态执行原语）。无论如何，调度应该满足控制和数据依赖关系，提高整个数据流的执行速度，并充分考虑运行时硬件资源的消耗（如补充信息 7.2.4 中的中间数据的缓冲区资源），以满足相关约束。调度策略将由 SU 执行。如果没有 SU 或 SU 不够灵活，则可以使用 PU 的一般近似方法来完成（补充信息 7.2.3）。调度实现还取决于硬件是否是事件驱动的（补充信息 7.2.1）。

最后阶段将具体的原语部署到PU内部的计算组件上，后者通常具备并行处理能力，并为这些原语提供封装好的库，调用库接口完成整个映射过程。

上述映射过程只是一般原则。考虑到硬件的复杂性和多样性，在实践中可以实现针对目标硬件的行为级模拟器作为补充措施；因此，我们可以评估映射方案的有效性并相应地调整方案。这个过程可以重复，并且可以通过一些启发式算法调整优化的方向[40][41]（补充图4）。
![[Pasted image 20241230191811.png]]

## 7.2 Mapping on Various Hardware
以上内容只是一个大概的框架，而具体的映射和优化策略与目标硬件的功能特性密切相关，特别是在以下四个部分。
### 7.2.1 Event-Driven or Not
底层硬件可以以事件驱动模式执行，也可以不以事件驱动模式执行。EPG 中的每个第二层块都是一个没有任何控制流模式的数据流子图，即所有第二层块都不需要动态调度（只有控制流才能导致动态调度）。因此，如果硬件不是事件驱动的，我们可以静态地调度数据流块（即，所有原语的顺序在运行前都是固定的），然后为每个原语的执行分配一个时间窗口。

如果启用事件驱动，则原语将由数据依赖性动态触发。在这种情况下，调度策略可以遵循事件驱动范式，并使用 SU 在运行时检查数据依赖性；不需要预定的序列信息。

实际上，大多数类脑计算硬件都声称以某种形式支持事件驱动模式。真正的映射策略应该根据具体的硬件实现仔细设计。我们只对这方面做了初步的讨论，以及以下内容。

### 7.2.2 Memory
任何映射策略都必须满足每个执行原语的数据存储要求。

内存与计算共置是类脑计算的重要特征。有些硬件天生就遵循这一原则，比如基于非易失性技术的原位计算模块。有些硬件采用近内存处理策略，每个内存模块与一个 PU 相邻并被其独占（例如 TrueNorth[23] 每个神经突触核心中的突触和神经元模块，或者 Loihi[25] 和 Tianjic[26] 核心中的类似内存结构）。在这种情况下，映射策略类似于天生共置内存和计算的硬件，因为 PU 的计算能力与内存容量之比是固定的。这些方法的优点是减少数据移动开销，缺点是分配策略不太灵活。

一些硬件在芯片级提供共享内存，即芯片上所有 PU 共享的内存（如SpiNNaker[24]）。在这种情况下，映射器可以根据每个原语所需的内存量协同使用这些共享内存。

### 7.2.3 Support for Control-Flow
目前，大多数主流的类脑芯片都支持多种形式的控制流模式，例如使用通用处理器、可配置电路等。

如果目标硬件没有灵活的 SU，我们必须将所有控制流模式转换为 PU 支持的原语（在编译器的一般近似步骤中）。这些原语将近似控制模式并像普通原语一样映射到 PU 上。具体而言，考虑到由条件（例如 if/else）或迭代（即循环分支）引起的分支，由于没有用于条件断言的 SU，因此所有条件分支都应映射并在硬件上执行，并且只有一个分支会生成有效数据。这些 PU 的输入包含两部分，一部分是数据，另一部分是控制信号，控制 PU 是否执行有效计算。例如，在循环中，循环体将始终运行，并且只有在满足循环条件时才会生成有效结果。

### 7.2.4 Data Buffer
在 EPG 中，二层数据流图的节点（即原语）之间存在中间数据。将 EPG 映射到硬件后，需要中间缓冲区来存储它们；缓冲区大小与映射算法直接相关。缓冲区越大越灵活，映射策略就越灵活。理论上的最小中间数据调度可以看作是静态数据流图上的调度。静态数据流图只允许每个边上最多一个数据令牌，即任何原语只有在其所有依赖关系都得到满足并且其输出边为空时才能执行。缺点是它不能充分利用并行性。如果有更多的缓冲区，可以相应地优化映射策略以提高并发性，例如对计算进行流水线化或对 PU 进行时分复用。




# 8. Implementation Instance of the toolchain
- General-purpose graphic processing units.
- FPSA. It is a DNN accelerator based on emerging non-volatile memory devices. 由于具有极高的加权和计算密度和计算性能，它被用作通过使用基本执行原语的一般近似来实现神经形态完备性的平台。因此，通过编译器，我们探索了不同近似粒度、资源消耗和性能之间的关系。
- Tianjic

## 8.1 FPSA
非易失性技术通常与生物突触相比较。有许多研究利用它们的特性，例如突触效能和突触可塑性[61]来模拟突触。在最简单的形式中，输入信号乘以突触的存储权重，通常表示为可编程、模拟、非易失性电阻。补充图 6 说明了如何使用带有电阻式随机存取存储器（ReRAM，一种广泛使用的忆阻器）的交叉开关在原位完成此类操作，并且可以连接更多交叉开关以形成具有原位计算功能的密集、大规模处理器。交叉开关的每个交叉点都有一个 ReRAM 单元。

FPSA is such a DNN accelerator architecture that uses ReRAM-crossbars to process weighted-sum operations. 它使用模拟电路来实现 ReLU 函数，以消除交叉开关和 ReLU 之间的 ADC（模数转换器）开销，因为它们都是模拟电路. 此外，采用尖峰编码方案，可以减少连接片上模拟计算部分和数字通信子系统的ADC/DAC的功耗，以上两种技术可以大大提高计算密度和性能。

Other challenges for the chip design and their solution are as follows:
- ReRAM-crossbars的高效率对片上通信性能提出了很高的要求，采用基于ReRAM的可重构互连[51]，为片上提供丰富的布线资源。
- 交叉开关非理想性的影响，包括 ReRAM 器件变异、字线上的 IR 降等，都会影响神经网络的准确性。因此，FPSA 提出了“添加”方法[50]，可以有效提高表示权重的多个 ReRAM 单元的准确性，并使用器件特性感知训练算法[52]，进一步提高端到端的准确性。
- 对控制流和数据缓冲的支持。它使用可重构逻辑单元（CLB）来实现控制流和调度逻辑，并使用传统的静态随机存取存储器（SRAM）作为片上的数据缓冲区。

FPSA 的详细配置如扩展数据表 4 所示。与最先进的基于 ReRAM 的神经网络处理器之一 PRIME[47] 相比，其计算密度提高了 31 倍；对于代表性 DNN，其推理性能可实现高达 1000 倍的加速。

从计算功能的角度来看，FPSA 仅高效支持 ReLU 和原位加权和运算。它们与基本执行原语相同。因此，FPSA 是神经形态完整的。借助所提出的系统层次结构和编译技术，我们可以将其应用范围从 DNN 扩展到任何图灵可计算函数，这样任何用 POG 描述的模型都可以映射到 FPSA。

## 8.2 Compilation and Mapping Processes
### 8.2.1 GPGPU Compilation
GPGPU provides a Turing complete ISA, and then it is possible to implement any Turing complete operations through these instructions. In our design, the computing primitives in the EPG are the same as the operators of POG, and the conversion between them is merely transferring the control flow operators into control flow graph.  对于 GPGPU 不直接支持的计算操作，我们要么手动实现相应的 CUDA 内核，要么参考一些现有实现的开发框架（例如，我们使用 PyTorch 框架提供 DNN 相关算子）。总之，POG 的所有算子都是通过精确的通用计算实现的。

### 8.2.2 GPGPU Mapping
GPGPU 具有单指令多线程 (SIMT) 执行模型，可提供大规模并行性，以及灵活且分层的内存缓冲区；它们还实现粗粒度（即在 warp 级别；warp 是同时执行的线程的最小单位）控制流操作。

因此，很容易将控制流图直接映射到GPGPU的warp粒度控制流，然后使用多个线程执行专用计算操作以实现更高的并行性。基本块内的数据依赖关系可以静态转换为指令顺序。尽管GPGPU不将存储与处理共置，但它提供了大量的片上内存；因此，将部分全局内存提供为每个操作符的私有内存是可行的。

### 8.2.3 Tinajic Compilation
The compiler should convert the POG into the EPG composed of these primitives.

The compilation process adopts two methods: general approximation and universal computation. Tianjic primitives include some _basic execution primitives_, like Vector Matrix Multiplication (VMM), that can be used to approximate any part of a POG with different granularities. These primitives, as templates, can match and replace the _operators_ or sub-graphs that have the same functionalities in the POG.


## 8.3 Analyses and Discussions
FPSA 和所提出的工具链可以看作是 RISC 原理在脑启发计算硬件领域的体现：通过我们的提议，FPSA 已经从仅支持 ANN 得到增强，能够有效地支持 SNN 等。

如何进一步扩展它是一个有趣的问题。一种可能的方法是将低精度忆阻器交叉开关与高精度数字处理器或数字电路结​​合使用，用于其他类型的常用精确计算：基于执行原语的近似方案是资源消耗和性能之间的权衡；因此，在某些情况下，使用数字硬件进行常见操作而不是近似操作可能更经济。此外，这种混合硬件可以与基于混合脉冲的学习方法配合使用，例如局部无监督学习，然后是全局监督反向传播 [54]。一些工作 [55] 表明，一些基于脉冲的局部无监督学习方法可以通过忆阻器交叉开关有效实现。无论如何，混合方法被认为是类脑计算技术的一个非常重要的领域 [61]。