上述过程是一个粗略的形式角度的演变历程，我们从中也可以看到在这一领域也出现了非常明显的分工演变趋势：最开始的程序员人人都是全栈（这个全栈不光包括前后端，还有测试运维等），后来因为需求驱动下的整体的技术领域不断进步，每个子领域也变得越来越复杂，于是我们现在有了前端、后端、测试、运维等等的分工。

随着时间推移，该领域进一步出现了业务和基础架构的分工，即出现了一部分人专注于实现业务，另一部分人帮助这些人**更高效**地完成实现业务这件事。测试和运维本质上是这样，做各类框架、三方库、中间件、运维平台、AI的专业人群也是这样。

这一现象的本质实际上就是，**在实现业务流程的过程中，每一个可能被解耦、自动化、专业化而产出价值的步骤或模块，最终都将被解耦**。我们当下看到的很多开发工具，比如IDE、抓包工具、性能分析工具、可观测平台、CICD平台等等，也都是这个逻辑的例证。

而如果你更希望继续做技术、发挥你积累和所学的计算机及软件工程专业背景的知识呢？当前情况，有两个方向：
- **向更专业分工化的职位方向转移，做给技术本身提供加速度的技术**：比如现在的AI、芯片、操作系统、基础平台、各类基础软件的研发等等。但同时这些技术都是有门槛的，往往需要垂直领域下非常纵深的知识和经验，往往得有一定时间的积累、天赋和契机才能进入，直接的切入也是不现实的。
- **向更广泛全局化的技术架构方向转移**：能够设计和实现一个完整的业务系统架构，同时要能够满足各类性能、稳定性、灵活性要求，而不仅仅限于实现一隅业务逻辑模块。这种职位所要求能够胜任的人必须要足够广且一定程度深的知识经验，在每个方面都有一定积累，才能达到设计并引领实现一个卓越系统架构的要求。

这两个方向一个偏垂直方向的深度，一个偏水平方向的广度，每个人都有适合的发展方向。而这两个
方向也有共同的基础要求，就是得有**牢固的基础知识和丰富的工程化经验**：

- 基础知识无非就是CS专业所学的那些核心专业课，但这些理论必须被大量且充分的实际实践和经验所印证，达到融会贯通的程度，而非是仅限于课本和面试八股文式的字面理解；
- 工程化思维就是在具体的工程实践中靠不断的经历、反思、学习和总结，所得到的隐性和抽象化的观念、思维习惯和态度，其实就是工程师的核心能力：所谓的“**解决问题的能力**”；

## 容器出现之前
从本质上来讲，容器（这里特指以Docker为代表的技术）解决的是程序的部署运行方面的问题，在其出现之前，程序的部署往往有两类模式：

### 直接部署
最简单直接的部署方式，将程序直接部署到机器上运行，主要会使用 systemd 类的守护类程序把多个业务程序管理起来。这种方式可以很方便地查看每个业务程序的运行情况并进行手动、自动化的启停操作，但此方式只管理了业务程序本身，并没有管理业务程序的依赖。

所以每新加一台机器，就得预先安装所有待部署业务程序的依赖，在多台机器规模化的情况下，往往通过下发部署脚本来实现，然而业务程序的依赖往往是复杂且易变的，多个版本的业务程序的依赖很可能是不相同的，有时经常出现反复升级和降级依赖的情况，这使得单个业务程序依赖管理和安装非常繁琐复杂。

这仅仅是单个业务程序的依赖管理，我们还需要考虑多个业务程序同时在一台机器上混合部署的情况，在这种情况下，多个程序间的依赖还有可能产生冲突：比如程序A需要依赖X的1.0版本，程序B需要依赖X的2.0版本，而如果依赖X的1.0和2.0无法共存，那么这两个程序就无法部署到同一台机器上了，这种情况显然也是很棘手的。

![[Pasted image 20250225133312.png]]
因此，我们可以看到，依赖本身批量化的安装是简单的，但在单机上存在多个版本、多个程序的case增加了依赖管理的复杂性，这给程序的部署管理效率和稳定性带来了非常大的阻力和挑战。

这样的棘手问题也对应了大量潜在的生产力红利的释放空间，于是就有无数的工程师开始尝试解决这一问题。

### 虚拟机隔离
解决这个问题有一个很直接的思路，既然程序的依赖管理很复杂，那是因为程序的运行总是天生地和依赖环境相耦合，那么我们尝试把程序依赖和程序本身完全一体化，作为一个独立单元管理不就好了？

最简单的实现方式就是每个版本的程序独占一台机器，这台机器上的依赖环境完全为这个版本的程序服务：要部署一个新版本程序，就必须先把机器重置然后安装此版本的程序所需要的依赖环境，这样程序自身版本前后的依赖冲突和多个程序间的依赖冲突问题不就都解决掉了？

我们发现把程序放到虚拟机里就能满足上述这种模式，这样做的本质就是把依赖环境和程序绑定打包到了一个虚拟机镜像中，然后以虚拟机镜像作为程序管理部署的基本单位。
![[Pasted image 20250225133519.png]]
但虚拟机的缺点也很明显，比起直接在物理机上部署，虚拟机的部署显得异常臃肿和笨重：

- 镜像过于庞大，存储和传输成本高
- 启停速度过慢，程序的部署效率无法做到很高
- 虚拟化后的程序运行性能损耗和overhead大，机器性能被浪费

虽然以虚拟机为单元的程序部署管理解决了依赖的问题，但同时带来了一些新的问题，因此这样来看，在程序的部署管理上，还有更多的优化空间：如果能同时解决依赖管理问题和虚拟机部署带来的劣势，那么这才是一个终极的方案。

### 基石：容器基础技术的出现
承接上面虚拟机的缺陷，我们不难看出，后面的目标就是在保证依赖问题解决的基础上，将虚拟机部署的三个大的缺陷解决掉即可。基于这样的基本思路，容器的基础技术出现了，或者我们可以地称之为“轻量级虚拟机”，这些技术将成为后续广泛意义上容器（Docker等）的基础能力支撑。

### 解决镜像过大问题
虚拟机镜像之所以过大，是因为其基本上存储了操作系统运行所需要的所有文件，而对于同一系列的操作系统的不同版本、分支来说，往往存在大量相同的文件，比如glibc、动态链接库、内核、常用的二进制工具程序等等。

那么，如果一台物理机上部署了多个同一系列的虚拟机，那么每个虚拟机镜像中都必然有大部分相同的文件，这就给我们带来了优化的空间：如果每个镜像中相同的文件我们只存一份，是不是就能减小整体占用存储空间的大小？

那如果多个镜像同时修改了一个文件怎么办呢？重新拷贝一份修改过的文件并归属给发起修改的虚拟机即可。

这是一种非常朴素但直接的想法，其实就是 Copy-on-Write 的思想，我们在整个计算机科学体系内的很多地方都能看到这种优化思路的体现：比如虚拟内存、懒加载等等。你也可以简单的理解为我们只保存一份文件的指针或者链接，而不需要把文件的内容冗余保存多份。

事实上，在大部分情况下，物理机的操作系统和部署于其上的虚拟机操作系统也是存在大量重复文件的，因此这个思路不仅用于虚拟机间，也可以用于虚拟机和宿主机间。

![[Pasted image 20250225133745.png]]
这一技术实际上就是联合文件系统（UnionFS/UFS），在容器出现之前，这个[技术就出现了](https://link.zhihu.com/?target=https%3A//www.linuxjournal.com/article/7714)（2004年），最开始并非服务于上述目的，而是希望在多个文件系统之上提供一个统一的文件系统视图。

### 解决启停速度问题
同样的，虚拟机之所以启停速度慢，也是因为把整个操作系统的启动都包含了进去，理论上，如果我们只关心程序的运行行为的话，为什么每次启停还要同时做一次操作系统的启停呢？

所以，只需要把操作系统的启停过程去掉就行，这就动摇了使用虚拟机技术的基础了，因为虚拟机的核心思想就是操作系统隔离，所以我们必须自己做一个“轻量化虚拟机”，把操作系统相关的启停开销去除掉。

要达成这一目的，我们还是离不开操作系统底层能力的支持，事实上，linux就是在这样的背景下，发展了满足我们上述需求的技术：namespace 隔离技术。

实现一个程序的隔离，在linux中主要是以下几方面：

- 文件系统
- 进程视图
- 进程通信
- 网络
- 用户和权限

如果每一个运行的程序进程在以上几方面都有属于自己的专属空间，那么实际上就实现了我们所说的“轻量化虚拟机”的隔离目标要求，这也就是所谓的 LXC（Linux Containers）。如果把我们的程序放在LXC中运行，那么他的启停速度会比虚拟机加速很多，这就满足了我们对隔离后启停速度的要求。

LXC技术在2008年基本上发展成型，也早于容器代表性技术Docker的出现。

### 解决损耗问题
我们深究虚拟机带来性能损耗的底层原因，从软件角度来说，其实也是因为虚拟机本质是一个完整的操作系统环境所造成的。多个程序运行在同一个操作系统上，和多个程序运行在多个操作系统上相比，很多的操作系统层面的操作和行为就会有重复和冗余的出现，而这些重复和冗余也就意味着对各类硬件资源的使用损耗。

举一个例子，linux操作系统上一般都会有很多内核进程，来协助内核做很多系统管理工作，如果有多个内核，那么每个内核都会有自己的一套内核进程来管理自己范畴内的事情，从整体角度来看，这些重复的内核进程本身所占用的资源就是损耗。

所以，实际上我们还是要依赖“轻量化”的手段，来解决这个问题，上一部分提到的LXC相关的技术就从根本上解决掉了这一问题，因为本质上所有LXC都是共用内核的，损耗自然会得到降低。

## Docker 出现
随着前面所述的这些容器依赖的基础技术的出现，最开始的程序部署时依赖管理和隔离的痛点和需求，从整体条件上讲已经具备了被解决的可能。但在当时，这些技术还是零星散布在各个领域中，没有一套完全成型的体系化解决方案和产品的出现，进而导致了这类技术无法被大多数人所获知并使用，仅限于一些大公司内的高级技术团队凭借着强大的技术能力组合了其中一部分来使用（如当时Google发布的 LMCTFY）。

但为什么最终是Docker变成了这个领域的代表呢？

其实回到我们最开始所提到的那个需求和痛点，如果能提供一套完整体系化的产品或方案来满足和解决它，那么就一定能被乐意采纳使用。当时尝试做到这个目标的项目和产品有很多，Docker是其中解决这个问题解决地最彻底出色的一个，并且提供了最出众的使用体验，以至于它最终成为了容器技术领域的标杆。

简单地讲，Docker就是将LXC技术、镜像和容器生命周期管理较好地结合在了一起（也就是上述填补虚拟机的三个问题所对应的功能）：以LXC和UnionFS类技术作为基础，Docker设计和开发了容器镜像和运行的标准，使得程序和其依赖环境绑定作为独立单元的打包（Dockerfile）、版本管理（image）、分发（registry）、部署运行以及管理（runtime）等流程变得无比丝滑流畅，并提供了一套**完整易用**的工具，满足了当时大部分人的需求，达到了其宣传口号"**Build once, Run anywhere**"所说的效果。

后来Docker更是迅速选择开源，通过免费带来的传播效应快速打出了名气，被广大开发者群体广泛使用，以至于后续成为了容器领域的技术标准。

作为现在Docker的用户，我们能够感受到Docker是将需求和痛点抓的最准的，当时大部分的项目都在聚焦于怎么使用LXC及底层相关技术来把隔离做得更彻底，性能做得更优越，而对于真正的痛点：解决程序部署时依赖管理和隔离这个事情却没有足够重视。现在我们每一次使用Docker，都能感受到它确实是把这个需求和痛点非常恰当地满足和解决了，所以它获得了今天在容器技术生态领域的地位。
- 面向需求而不是面向技术
- 技术领域的组合创新
- 渐进式、问题驱动
- 避开定势：换一个思路解决问题

我们看到其实在整个容器技术的发展过程中也存在两个绕不开的弯路：有些人一直秉持着直接部署的模式，编写了类似编程语言依赖管理的方案，通过计算和识别依赖关系来管理软件部署运行的依赖，但最终因为过于复杂和难以掌控，无法普及而失败。

此时虚拟机部署的方案就换了一个思路，不去解决依赖关系，而是直接隔离依赖关系，这种思路上的根本变革就立马用更低的成本解决了问题。而基于虚拟机思路下的人，仍有部分执着地陷入到了虚拟机的思路中，一直尝试优化虚拟机来解决虚拟机部署的缺陷问题，但虚拟机地诞生并非是服务于软件部署，这种自下而上、偏离初始功能目标的优化是相当难做的。

而此时有人就又换了一个思路，从系统底层能力的角度思考直接提供轻量化隔离的能力，于是自上而下的容器基础技术得以出现，最终结合Docker这类出色的项目得以被大家广泛使用。

这说明其实很多时候，寻找新的思路往往可能会比局限在旧的思维模式中更为可行，永远不要惧怕推倒已有的方案和思路，新的、更优秀的方案大多时候都不是已有方案的改进，而是根本性的变革。

  
# 为什么会有 k8s
不过随着容器技术被大规模地运用于生产实践，一些问题也逐步暴露了出来：

单机上的容器可以使用docker很轻松地进行管理，但有一定规模的企业往往会有很多个应用、很多台机器，一旦应用、机器的数量急剧增长，承载应用运行的容器的管理就开始变得复杂和麻烦了。试想，如果没有任何手段，那么就只能编写运维脚本，然后批量下发到每台机器上做管理了，这种管理模式有很多弊端：

- **操作管理效率低**：任何操作都要编写脚本实现，操作人必须关注每一步的操作和实施细节，整体的实施操作的效率很低，且具备人工误操作的风险，并且大多数时候这对工程师来说是一种高重复性且枯燥的事务性操作，存在自动化的可能
- **缺失全局记录**：在大量机器上做了容器化部署后，默认是没有记录机制的，部署完成后无法感知应用程序在大量机器上的分布和运行情况，必须自己建设管控系统记录这些信息
- **精细化控制操作成本高**：比如实现一组应用程序的受控分批化逐步升降级、根据资源使用情况针对应用程序所申请的各类计算资源进行扩缩容调整、一台机器宕机后对上面运行的应用程序进行转移恢复等等，规划、执行这些操作行为都有很高的成本
- **难以进行高效的资源管理、使用及分配**：应用程序需要的各类资源（计算、内存、存储、网络、异构算力等）如何管理和供给？哪类或哪个应用适合放在当前集群内的哪个机器上？如何放置才能将资源碎片度降到最低、机器集群整体的资源利用率得到最高效的使用？这一点和成本紧密相关，也是对企业最有吸引力的一个方面
- **服务难以高效管理和治理**：在微服务化的背景下，应用服务间该如何在大规模机器集群上高效地互相感知、互联及协作？每个服务的配置如何进行高效跟踪、修改和管理？服务该如何恰当地暴露给用户进行访问？

要一一解决这些问题，就需要从头针对容器化应用在大规模机器集群的部署实践中遇到的上述各类问题，设计一个功能健全的管控系统来解决，而其实K8S也就是这样的东西，可以说这又是一个遇到问题解决问题的过程，但K8S并非是解决这些问题唯一出现的项目，但却是在一堆解决方案中解决最成功的一个。

>不过，我们更多地将K8S称为容器编排（Orchestration）系统，而非管控（Control）系统：这两个定义的差异关键就在于编排是一种更上层、更宏观及关注结果的视角（即最终应当是一种怎样的状态），而管控则时更底层、微观一些，关注操作过程的视角（即如何进行控制来达成需要的状态）。这一差异是K8S理念中最核心的一点，也是其相对于其他类似系统最根本性和革命性的差异。


# k8s 如何解决这些问题
那么K8S具体是如何解决上述问题的，上文提到的前三个弊端问题的解决，都和K8S应对这些问题和场景时所秉持的一个大原则：面向终态有关

## 面向终态
如何理解“面向终态”？相信即使没有程序员背景知识的人也都用过马桶水箱、冰箱、热水器、空调、自动驾驶汽车这些东西吧，这些东西都可以视为“面向终态的控制系统”：

- 马桶水箱的终态是水箱内的水达到指定水位，否则会一直尝试送水进来
- 冰箱、热水器和空调的终态是其关注对象的温度达到指定值，否则会一直尝试工作来改变对象温度
- 自动驾驶汽车启用定速巡航功能的终态是达到其设定的行驶速度，否则会调节发动机转速达到指定速度
![[Pasted image 20250225135721.png]]
我们不难看出，面向终态的关键就在于让控制逻辑感知到某种状态（现状），然后根据感知到的状态进行针对性操作，最终达成某个目标状态（终态），而目标状态也不是一成不变的，所以整个系统往往处于一种动态平衡的状态。

为什么这个思想成为了K8S的核心思想呢？我们首先要分析清楚K8S要解决的问题究竟是一个什么样的问题：

>**大规模程序、应用在大规模机器集群内的高效部署管理**

我们对这个问题进一步分解细化一下，部署管理无非就是对程序、应用和机器进行各种操作，来达成各种目的，比如：
- 对应用程序进行从无到有的部署：一批应用程序**应当**运行在集群内的某些机器上，以满足业务需求
- 将应用程序从A版本升级到B版本：应用程序的版本**应当**为B，以满足新的功能需求
- 将应用程序的CPU资源从1颗增加至2颗：应用程序**应当**享有2颗CPU的计算资源，以缓解其性能压力
- 在应用程序所处的机器宕机后对其进行转移：应用程序**应当**始终保持运行，以保证服务始终可用
- 为机器集群扩充机器：当IT业务经历使用高峰时，应用程序的计算资源**应当**保持充足，以保证用户良好的使用体验

![[Pasted image 20250225135848.png]]
诸如此类的问题，其实都可以划分为操作、目标状态和最终目的三部分：

- 操作的直接目的是达成目标状态（应当达成的状态）
- 达成目标状态可以满足最终目的

所以，这里目标状态实际上就是操作和最终目的之间的关键纽带，K8S“面向终态”的设计思想就是关注这里的目标状态，将达成目标状态的操作进行自动化、公共化和抽象化，从而更高效地达成最终目的。

## 自动化、标准化操作流程
之所以能够实现自动化、标准化的基础之一，也在于对程序在大规模机器集群内的部署管理这个问题来说，大部分的操作都是重复冗余、有高度自动化可能性空间的，所以K8S“面向终态”，实际上就把这些高度重复冗余的操作都标准化、自动化了。

最典型的比如重启、升级、增加计算资源这些操作，基本的操作流程都是固定不变的，并且和容器内运行的到底是什么样的程序无关，会变的仅仅只是一些外部参数（如应用名、版本号、增加的计算资源数目等），因此这些流程很容易就能被标准化自动化，这本质上和在编程中封装公共函数是同一件事情。

而最终这样的自动化、标准化也消除了管理运维人员原本必须进行的重复繁重枯燥的手动人工操作，也消除了因为人工误操作带来的故障风险，从而提升了管理效率和稳定性水平，成功释放了技术红利。

## 天然重试
面向终态的思想还带来了一个天然的结果，那就是“重试”。

当我们进行任何操作，如果不符合预期，可能是各种各样的偶然因素影响，我们往往会做重试来再尝试若干次以避免偶然因素带来的影响。

在面向终态的思想中，因为时刻在比较现状与目标状态的差异，一旦没有满足要求，就会立即进行操作来尝试达成目标，如果这次操作失败了，就会再次重试来再次尝试达成目标状态。也因此，“重试”操作成了面向终态设计思想和实现中所达成的一个天然性的结果，这也降低了由人工进行操作重试所带来的成本。

> 重试这件事本身也很复杂，该重试几次？重试的频率是多少？什么情况下不该进行重试？等等之类的问题，我们会在后面探究更细节性的K8S设计思想的时候进一步阐述。

一个很有意思的例子是，很多的企业采用K8S后，发现遇到的实际生产问题变少了，原因不是他们用了K8S后代码中的bug减少了，而是因为原来的bug导致的服务崩溃都由K8S帮他们迅速重启拉起恢复掉了，这样的重启执行速度之快与及时，甚至让开发人员意识不到是自己的程序出了问题


## 状态的全局记录
因为面向终态要时刻比较当下和目标状态的差异，因此对当下和目标状态信息的记录就变成了一个必要的条件，否则便无法实现“面向终态”思想所要求的效果，所以这种记录能力也是K8S提供的基本功能。

所以，在管理程序和机器的同时，我们也就有了记录机制。每一个部署的程序，管理范围内的机器，以及彼此之间的各类关系，都有了明确的记录（所有机器就像是都有了证件，证件上的所有相关数据信息都在管理机构的数据库内统一保存，并且能够实现实时更新感知），这极大地提升了管理的效率，因为我们可以近乎实时地感知到当前大量程序和机器的状态，判断其是否存在问题，并做针对性的调整。

## 降低门槛
另外一个效果是，由于所有的具体操作行为被自动化和标准化掉了，也因此实际上运维人员不需要进行达成目标状态的每一步细节操作，这极大的降低了管理程序和机器集群的门槛。

在这样的思想下，解决问题的人的关注点完全改变了，从关注过程变成了关注结果，而描述清楚需求、结果往往比描述清楚过程简单地多，因此问题的解决变得更加容易和高效了。更具体地来讲，这里的受益人员实际上是运维人员以及一切做运维操作的人。
![[Pasted image 20250225140310.png]]
>但往往随着需求越来越复杂，这套机制也会出现新的问题，那就是描述结果的语义也开始变得复杂，就像是用SQL取数据一样，同样是关注结果而非过程的思想，在最简单的场景下往往select+where就够用了，而一旦取数据要求的结果本身很复杂，就会导致写SQL本身这件事也开始变得复杂。

所以，我们现在来看，在上文所提到的三个问题是否都得到了解决：

- **操作管理效率低**：几乎所有高频、重复的操作都被自动化、标准化了，管理人员无需付出较多精力去操心如何进行操作执行的细节
- **缺失全局记录**：因为要时刻获知当前管理的各类目标资源对象的状态，所以会有实时性的全局记录
- **精细化控制操作成本高**：基于自动化、标准化的基本操作，由于K8S的开源和开放扩展性，开发人员可以开发更多定制的精细化控制操作，并提供给广大K8S用户进行使用，而用户同样无需关心实现细节，即可在任意K8S环境重复使用


## 调度
我们说完了上文提及的前三个问题是如何被面向终态的思想所解决的，下面我们来看第四个问题：资源管理、使用和分配的问题，K8S的出现，使得进行这类事情变得更加高效。

**调度面对的核心问题**

调度是计算机科学里面对的一类非常具备普遍性的问题，我们在很多地方都能看到调度问题：

- 操作系统的进程、线程调度
- CDN的网络流量调度
- 物流中的多地多仓库货物调度
- 编程语言的垃圾回收本质上也是调度问题

调度本质上是一个决定资源如何分配才能达成某些条件下的最优效果的过程，这个某些条件下的最优效果往往是人为定义的：可能是最直接的全局综合最优，也有可能是特定范围内的局部最优，这个最优效果往往会随实际需求的变化而进行适应性的定义调整。

在K8S的所面对的调度场景中，就是探究如何将多个具备不同特点和规格的应用程序安置到一个由多个相同或不相同的机器组成的集群内，才能达成最大程度的资源利用率（这是最常见的最优效果要求，随着场景的变化往往会不同）。

一般来说，资源利用率往往和服务质量（如响应实时性）是不可兼得的，在操作系统上就会遇到这样的问题，所以便有实时操作系统和普通操作系统之分，前者更加在乎响应实时性，所以整体资源的利用率就变成了次要关注目标。同样的，在集群调度中也会遇到类似的场景，如果大部分调度对象对执行完成时间有较为严格的要求，调度的侧重点就会变成调度对象任务执行的平均时间开销而非是资源利用率。

## K8S的调度
由于调度本身是一个非常复杂和多样化的事情，在不同场景和需求下所要求的目标并不相同，所以K8S提供了一个调度框架，来辅助使用者实现不同的调度需求目标，我们这里以最常见的、也是默认的调度策略为例来介绍下K8S调度框架的整体思想。

对刚刚了解K8S的人来说，只需要关注理解调度框架中过滤（Filter）和打分（Socre）两个过程就够了，这两个步骤也是K8S调度中最重要的两个，这里可以直接引用K8S官方文档的解释：

kube-scheduler 给一个 Pod 做调度选择时包含两个步骤：  

1. 过滤
2. 打分

过滤阶段会将所有满足 Pod 调度需求的节点选出来。 例如，PodFitsResources 过滤函数会检查候选节点的可用资源能否满足 Pod 的资源请求。 在过滤之后，得出一个节点列表，里面包含了所有可调度节点；通常情况下， 这个节点列表包含不止一个节点。如果这个列表是空的，代表这个 Pod 不可调度。  
在打分阶段，调度器会为 Pod 从所有可调度节点中选取一个最合适的节点。 根据当前启用的打分规则，调度器会给每一个可调度节点进行打分。  
最后，kube-scheduler 会将 Pod 调度到得分最高的节点上。 如果存在多个得分最高的节点，kube-scheduler 会从中随机选取一个。

进一步简单地考虑，其实过滤和打分做的是同一件事情，过滤可以理解为把过滤掉的候选项放到打分结果最后一位之后，所以我们着重看看K8S调度默认是如何打分的，默认的打分结果是由以下几种策略加权求和计算出来的（根据K8S v1.30代码整理）：

- TaintToleration（3）：污点容忍情况
- NodeAffinity（2）：节点亲和性情况
- NodeResourcesFit（1）：可用资源情况
- PodTopologySpread（2）：pod拓扑分布情况
- InterPodAffinity（2）：pod间亲和性情况
- NodeResourcesBalancedAllocation（1）：节点资源分配均衡情况
- ImageLocality（1）：节点存在镜像情况

污点容忍、节点亲和性、pod拓扑分布、pod间亲和性这三类都是用户主动指定的位置偏好（即：我想让这些应用运行在哪些地方），所以相对来说赋予了较高权重，这里可以看出默认策略是更照顾用户的主观性要求的。

可用资源、节点资源分配、节点存在镜像情况则是客观资源条件（即：把应用放在哪里可以起到最好的资源利用率），这里的权重是比主观性要求要低的。

上述是K8S默认使用的调度策略，如果有需要，K8S所选用的调度策略、权重甚至是策略执行和计算的过程都是可以完全自定义实现的，并且所谓的“资源”的概念也可以自行定义和实现针对其的调度模式。譬如上述涉及到的仅为最常见的以CPU和内存为基本计算资源的调度场景，除此之外，磁盘、网卡、GPU芯片、IO设备各类硬件乃至虚拟的软件资源都能成为可能的调度资源对象。

理论上讲，K8S的调度机制和框架，最终赋予了用户达成任意调度策略和目标的可行性。

也因此，通过这样的调度机制和思想，K8S实现了各类资源的利用在各类定义衡量标准下的尽可能高的优化效果，提升了企业和个人对资源的使用率和取得的特定质量效果，同时灵活地满足了各类可能的调度诉求，顺带解决了人工部署时存在的“选择困难症”现象，这样提升的效果无疑是非常显著的。


## 中立的服务管理和治理模式
接下来我们来谈谈K8S如何解决第五个问题：应用服务的管理和治理

**微服务和容器化**

这也是一个非常大的话题，我们可以知道，在容器和K8S出现的前后，微服务也在那段时间开始变成了一个非常流行的技术术语，这两者几乎在同一时间段受到追捧，从某种程度上来看，是一种必然。因为从两者的理念和模式上来看，存在非常多的交集和共通点，所以这两个技术的流行普及实际上是一个相辅相成的过程。

微服务的核心理念是什么呢，是把旁大的单体式应用服务根据功能或领域拆分成若干个独立的灵活小巧的微型服务，所以称为微服务。而单体服务往往切分完遇到的第一个问题就是切分后的服务数量上涨，多个微服务的部署、管理、治理都变得复杂和棘手，而容器和K8S技术恰恰能够较好地帮助解决这些问题，因此两者的使用相得益彰。

相信做业务开发的同学对微服务的理念非常熟悉，而这一领域比较经典的就是Java Spring Cloud那一套东西，它给出了微服务架构中的关键功能范式组成：
- **配置中心**：对各个服务的多个实例所使用的配置进行统一化管理
- **注册中心/服务发现**：注册中心对所有服务进行统一登记管理，进而帮助多个服务间进行互相发现和访问
- **负载均衡**：用于访问服务时在单个服务的多个实例间均衡流量
- **网关**：作为服务访问的统一入口使用，集成了和服务具体逻辑无关的通用性功能：如路由、过滤、鉴权等
- **远程调用**：用于服务间互相调用对方提供的接口和方法
- **限流熔断**：用于限制访问服务量过大时进而对服务稳定产生威胁

Spring Cloud的这样的设计，是一个针对微服务架构很好的范式，不过很遗憾的是Spring Cloud仅限于Java系的语言中，在非Java、多语言混合技术栈中，就很难适用了。

不过，这些典型功能是从微服务架构中抽象出的不可或缺的基本需求，也对后续我们理解K8S为何对微服务架构非常友好有所帮助，实际上K8S就是达成了多语言混合、或者说语言中立技术栈微服务架构所需的要求。

- **配置中心**：K8S提供了两类存在于集群中的抽象资源Configmap、Secret来实现配置的管理，前者用于保存非敏感数据、后者用于保存敏感数据，所有需要使用配置的应用都可以通过“挂载”一个或若干个Configmap或Secret来实现配置的附加，同时多个应用可以挂载同一个Config或Secret，其修改和变化也会同步同步到所有挂载这些对象的应用服务实例上，实现配置的统一管理和变更
- **注册中心/服务发现**：在K8S集群中，一组应用服务程序（微服务）的实例可以被直接抽象成一个K8S集群中的资源——服务（Service）而对待，这个名字非常直观，不同应用服务间通过Service这个集群资源即可实现互访，无需感知究竟要访问哪一个实际的服务地址。并且服务作为集群资源，会被K8S所保存到其全局记录中，也可以通过K8S API轻易地获知集群中存在哪些对外暴露可访问的Service进行服务发现
- **负载均衡**：K8S使用一个叫做Kube-proxy的组件，借用Linux内核提供的灵活的网络转发机制（iptables/ipvs），同样实现了应用程序在客户端的负载均衡，保证了应用程序在访问抽象的Service资源所对应的实际端点时，能够按照特定的负载均衡策略在多个端点间进行合理的请求分布，简化了对一个应用程序实体、多个实例的访问方式
- **网关**：K8S通过抽象资源Ingress、Gateway来声明式地提供网关实例，在K8S中，没有强制地绑定了一个网关的具体实现，而是定义了一个网关组件需要支持的基本语义规范（也可以理解为协议、接口）来提供给用户使用，这里给予了用户灵活选择的空间，用户可以选择更加适合自己场景的网关进行使用，并通过K8S提供的统一的声明式抽象语义规范来使用；除了在网关上我们能看到这种定义语义规范/协议/接口，而不是定义具体实现的模式，我们还能在K8S的很多地方看到这样的模式，譬如CNI、CSI、CCM等等，这也是K8S本身的一大特点
- **远程调用**：由于远程调用是一个和语言密切相关的操作，K8S出于本身的语言中立性，不提供一个具体的远程调用方法框架，因为诸如像GRPC、Thrift、GraphQL等项目已经提供了成熟的语言中立的远程调用能力，所以这里K8S同样不强制性地绑定实现，而是把选择权交由用户
- **限流熔断**：这类功能可以看作是网关层和远程调用范畴内的功能，一般往往也是在这两个层面内进行实现，因此K8S并不考虑这部分内容

总体上，对应Spring Cloud提供的微服务架构基本能力来看，其中基于语言无关立场能够实现的，K8S均提供了定义和实现，而不能实现的，K8S也能很好地进行嵌入和适配。而K8S面向支持微服务架构，提供的最关键的优势，就是其语言中立性立场带来的普适性、面向接口和协议而非实现的高度兼容和扩展性。

不过相比于Spring Cloud这样的方案，K8S这样的模式也造成了很多时候不能够开箱即用、方便快速地使用，不过从其本身的定位来说，这样的代价也是可以接受的。因此，最终通过这样的模式，K8S解决了第五个多个服务难以管理和治理的问题。

## 为什么是 k8s
我们在前面的部分也提及到了，K8S并不是解决大规模容器化应用和机器集群管理问题的唯一的项目，但它确实是解决地最好的一个，也因此成了现在容器编排管理领域事实上的标准。那么，为什么是K8S呢，与他同期同类的项目为什么没能竞争过它呢？

### 开放性

我们上一篇文章讲了Docker，事实上对Docker比较熟悉的人应该知道，Docker除了管理单机上容器的成名项目Docker，还有自己的容器编排项目：Dokcer Swarm。

并且，在容器生态刚刚兴起繁荣的当时，Docker可谓是如日中天，与它无缝集成的Swarm自然也有非常巨大的优势去抢占容器编排领域的主位，但它还是被K8S击败了，并且输得很厉害，乃至于K8S最终把Docker也抛弃并划清界限了，造成这种结果的核心原因，就是开放性。

Docker Swarm从诞生时，就成为了Docker雄心勃勃、想进一步垄断容器生态市场的一个有力工具，因此当时参与该领域竞争的其他公司自然会对Docker Swarm抱有较大的竞争意识。

由于Docker Swarm本身的非中立性（绑定了Docker公司），因此推出K8S的Google在之后的竞争中选择了成立一个中立的开源组织CNCF，把K8S完全捐赠了出去，撇清了K8S和Google之间的管理（也就是告诉大家：不用担心，这个项目不是我的，我不会通过控制项目的发展来牟利）。

也因此，彻底中立开放的K8S项目成了广大开发者和企业所青睐的对象，因为中立的立场意味着K8S的使用永远不会需要授权、付费，并且其发展永远不会被一家公司所把持。另外，开源、开放的运作模式，意味着任何个人、公司、团体，都有参与发展K8S项目的权利，都有无偿享受项目的权利。

除此之外，K8S的开放性还体现在对各个具备同类功能项目的包容和中立性，由于是通用的容器编排系统，因此会集成很多能力，包括存储、网络、计算的各个方面。在这个时候，K8S选择制定开放的协议和接口标准，而不是强制绑定和规定一个实现，比如：

- CRI抽象了所有底层容器运行时（如Docker）的功能标准接口，满足了该接口标准的任何项目，都能被集成到K8S中使用
- CSI抽象了所有K8S容器使用存储能力的统一功能标准接口
- CNI抽象了所有K8S容器使用网络能力的统一功能标准接口
- CloudControllerManager抽象了所有K8S集群使用云平台相关能力的统一标准接口
- DevicePlugin抽象了所有K8S集群使用异构设备的统一功能标准接口
![[Pasted image 20250225141640.png]]