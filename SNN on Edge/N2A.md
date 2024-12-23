可用神经数据的指数级增长与计算能力的指数级增长（“摩尔定律”）相结合，为大规模和高细节理解神经系统创造了新的机会。生成大型和复杂模拟的能力给神经科学家带来了独特的挑战。神经科学中的计算模型越来越广泛，通常涉及不同领域专家的合作。此外，模型的大小和细节已经增长到理解变异性和假设含义的水平。在这里，我们介绍了==模型设计平台 N2A，旨在促进生物现实模型的设计和验证。N2A 使用神经信息的分层表示来实现来自不同用户的模型的集成。N2A 通过在敏感性分析和不确定性量化中本地实现标准工具来简化模型的计算验证。部分关系表示允许网络级分析和动态模拟。我们将展示如何在一系列示例中使用 N2A==，包括简单的 Hodgkin-Huxley 电缆模型、80/20 网络的基本参数灵敏度以及生长树突和干细胞增殖和分化的结构可塑性的表达。

# 1.介绍
用于构建和模拟生物现实模型的计算神经科学方法越来越被认可为理解复杂神经回路功能的重要方法。计算工具的作用在不久的将来将继续增长，并有重大的政策努力，例如欧盟人脑计划（Markram，2012 年）和拟议的大脑活动图（Alivisatos 等人，2013 年）。这些计划强调通过连接组学研究和对回路中神经元行为的大规模生理学测量来高通量收集神经数据。虽然计算工具在建模和模拟中的作用越来越被认可，但从这些原始数据到可解释的模型结果的路径尚不清楚。

一旦建立了概念方法，构建神经模拟通常涉及几个不同的阶段（图 1A）。（1）必须识别、过滤和以计算可处理的形式表示来自生物世界的相关数据。这通常是一个挑战，因为很大一部分神经生物学数据本质上是定性的。（2）必须从这些原始数据中组装一个模型，这涉及对适当的抽象级别和所需范围的关键决策。（3）模型通常是模拟的，要么直接在模型构建工具中，要么在单独的环境中。（4）最后，必须分析模拟数据，由于当今模型的潜在规模，这通常并非易事。这四个阶段中的每一个都是独一无二的，通常需要不同形式的洞察力，并受益于用户不同方面的专业知识。

![[Pasted image 20241208152901.png]]