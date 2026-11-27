# Lifeguard：让失败检测先审视自己


<!--more-->

> 原文：Armon Dadgar, James Phillips, Jon Currey. **Lifeguard: Local Health Awareness for More Accurate Failure Detection.** 2018 48th Annual IEEE/IFIP International Conference on Dependable Systems and Networks Workshops (DSN-W), pp. 22–25. arXiv:1707.00788v2 [cs.DC].
> 译自 HashiCorp 作者在 arXiv 上公开的完整论文版本。

| 层次 | 回答的问题 | 核心标识 | 本文对应章节 |
|---|---|---|---|
| **动机** | SWIM 为什么会误报？ | 慢消息处理、误判为故障 | 第二章 |
| **基础** | SWIM 和 memberlist 是什么？ | 探测、传播、怀疑机制 | 第三章 |
| **方案** | Lifeguard 改了什么？ | LHA-Probe、LHA-Suspicion、Buddy System | 第四章 |
| **验证** | 改完到底好了多少？ | 误报率、延迟、消息量 | 第五章 |

## 摘要

SWIM 是一种点对点的组成员协议，在可扩展性和鲁棒性上表现优异。然而，当消息处理变慢时，它可能把健康的成员错误地判定为故障——这被称为**误判故障检测**（false positive failure detection）。尽管 SWIM 本身包含了一种避免这种情况的机制，情况仍然如此。

我们识别出 SWIM 中导致这一问题的若干性质，提出 **Lifeguard**——一组 SWIM 的扩展，它通过**局部健康**（local health）这一概念来考虑本地故障检测模块自身可能才是问题的来源。我们在精确受控的环境中以及真实部署中对该方法做了评估，结果表明：与基线 SWIM 相比，Lifeguard 大幅降低了误报率，而且真实故障的检测时间可以**同时**得到降低。

## 1. 引言

任何分布式系统都必须解决三个关键问题：**组件的发现**、**故障检测**，以及**它们之间的负载均衡**。组成员关系（group membership）是一个直观的抽象，可以同时用来解决这三个问题：组成员及其客户端会得到一份动态更新的当前成员视图，并利用这份视图执行请求路由、状态迁移等动作。

SWIM [1] 是一种组成员协议，具备若干诱人的性质。它的点对点设计和对随机化通信的使用，使它对节点故障和网络故障都具有极高的可扩展性与鲁棒性，而且易于部署和管理。它的简单性也让它相对于许多其他分布式系统协议更容易实现与调试。

我们知道有三个成熟的开源 SWIM 实现：**Butterfly** [2] 是 Habitat [3]（一个流行的软件自动化平台）的一部分；**Ringpop** [4] 是为支持某家全球性交通技术公司的应用而构建的；**memberlist** [5] 是我们（HashiCorp）自己的 SWIM 实现，支撑着 **Consul** [6]（一个流行的服务发现与管理工具）和 **Nomad** [7]（一个高可用、数据中心规模的调度器）。通过与客户的接触，我们知道 Consul 有数十万个运行实例的部署，其中有单个成员数**超过 6000** 的部署。

SWIM 论文已经指出，基本 SWIM 协议对**慢消息处理**是敏感的。慢消息处理可能由多种因素引起，包括 CPU 争用、网络延迟或丢包，它会导致 SWIM 把健康成员宣称为故障——即所谓的误判故障检测。为了对抗这一点，SWIM 论文引入了一个 **Suspicion**（怀疑）子协议，它以更高的故障检测延迟换取更少的误报。

然而，我们在开发和支持一系列使用 SWIM 的系统过程中的经验表明：**即便有了 Suspicion 机制，在数据中心有时会遇到的条件下，误判故障检测仍会以令人担忧的频率发生**。当慢处理是间歇性发生时，健康成员甚至可能在"被判为故障"与"被判为健康"之间**反复摇摆**（论文称之为 flapping）。**这种摇摆代价高昂**——如果它引发了反复的故障切换操作（比如重新供应成员、重新平衡数据），代价会非常大。

排查这些场景让我们对两件事有了认识：SWIM 在处理慢消息处理上的一个缺陷，以及弥补这个缺陷的办法。

我们在**裸机与虚拟化系统**上、在**私有数据中心与公有云环境**中都遇到过这类问题。

Lifeguard 的做法是：**让 SWIM 故障检测器的每个实例都审视自身的健康状况**，我们称之为**局部健康**。它通过一组 SWIM 扩展来实现，我们称之为 Lifeguard。Lifeguard 能够显著降低误报率，在受控环境和真实场景中都是如此。

本文结构如下：第二章阐述促使我们使用 SWIM 的优势以及我们遇到问题的场景；第三章描述 SWIM 与 memberlist——我们用于评估 Lifeguard 的 SWIM 实现；第四章描述 Lifeguard 的扩展；第五章单独与组合地评估 Lifeguard 的各个组件；第六章讨论 Lifeguard 与既有工作的关系；第七章讨论可以得出的结论与潜在的未来工作。

## 2. 动机

SWIM 使用**随机化的探测式故障检测**与**基于 gossip 的更新传播**，从而获得若干诱人的性质：

- **可扩展性**。在 SWIM 中，首次检测到故障的期望时间、误报率、以及每个组成员的**消息负载都独立于组的大小**。把一个故障完整传播出去所需的时间则随组大小**对数增长**。
- **鲁棒性**。由于协议是完全去中心化的，任意一部分组成员的同时故障或网络分区都是可以容忍的。甚至完全隔离的子组也可以继续运作，并在连通性恢复后自动合并。
- **易于部署与维护**。协议的完全去中心化特性意味着潜在的成员可以联系任意一个当前成员来加入这个组；成员离开时也不需要任何特殊操作来维持系统健康。
- **实现简单**。SWIM 协议只有很少的状态和消息。因为是点对点的，初始时不需要配置、也不需要在成员变化时维护任何特殊结构，比如 leader 或层级。

使用基于 gossip 的更新传播使得 SWIM 成为**弱一致**的。也就是说，在某一时刻，不同成员可能对组成员关系有不同的视图。实践中，弱一致性对许多应用是可以接受的。在不可接受的地方，正如 SWIM 论文指出的，可以在 SWIM 之上叠加一层一致的视图来获得强一致性——该视图会对成员列表做检查点。[^1]

SWIM 的故障检测子协议众所周知对**其消息的慢处理**敏感，这会导致误判故障检测——健康的成员被错误地宣称为故障。这是个严重的隐患，因为把流量从某个成员上引开、以及在它被重新声明为健康后把它重新纳入系统，往往都是有代价的。

SWIM 论文通过加入 Suspicion 机制来处理这个问题（第三章会详细解释）。然而，我们在开发和支持各种使用 SWIM 的系统时的经验表明：**即便有 Suspicion 机制，在数据中心有时会遇到的情况下，误判故障检测仍会以令人担忧的频率发生**。多个成员同时变慢时问题会加剧：即使是健康成员，也可能因为与慢成员的交互而把其他健康成员标记为故障。

[^1]: 这是 Consul [6] 采用的做法 [8]：它用 Raft [9] 呈现从 memberlist [5] 获得的组成员关系的一致视图。虽然这带来了对 Raft 服务器法定人数的依赖，但 SWIM 的上述好处在被管理资源上依然成立——那些资源只需要充当 SWIM 组成员即可。

我们调试这类问题遇到的场景包括：

- 按稳态容量配置、但会遭遇远超预期的流量突发的 Web 服务器；
- 遭遇持续 DDoS 攻击、导致高网络与 CPU 负载的入口节点；
- 被分配了超出可用 CPU 的工作负载的视频转码服务器；
- Burstable Performance 实例（如 AWS T2 类、Azure B 系列虚拟机），其 CPU 积分耗尽后被 hypervisor 限流。

在这些场景以及另外一些场景中，受影响机器上 SWIM 消息的缓慢处理，导致同一成员组中的**健康机器错误地控告**其他健康机器故障。

我们复现了视频转码场景的特征，结果如图 1 所示。我们在一个 Microsoft Azure 公有云区域内署了一个由 **100 台单核（Standard_A1_v2 规格）虚拟机**组成的集群，Consul agent（即必须运行在每个将成为 SWIM 组成员节点上的守护进程）部署在所有机器上。然后我们在其中一部分机器上运行极端 CPU 密集型负载：本例使用 Linux stress 工具，配置为运行 **128 个进程**，每个进程执行紧密的数学运算循环。负载持续运行 **5 分钟**。我们记录每次测试期间 Consul 提出的所有成员故障事件，并在实验结束后分析日志，以确定发生了多少次误判故障检测。

图 1 的横轴是运行压力负载的机器数量，从 **1 台到 32 台**（每台机器代表集群的 1%）。对每个压力机器数量，纵轴展示两个相关指标：

- **误报总数**（Total False Positives）。发生在任一成员处、且对象为**健康成员**（即没有运行压力负载的成员）的故障事件，**包括**运行压力负载的成员所提出的事件。
- **健康成员处的误报**（False Positives at Healthy Members）。不仅是对象为健康节点，而且**由健康节点提出**的故障事件。这类尤其令人担忧，因为事件的两端——提出事件的 agent 和事件所指的 agent——**都是健康的**。

每个指标展示两次：一次是 Consul 运行**未经修改的 SWIM**，另一次是运行**带 Lifeguard 的 SWIM**。

图 1 显示，对 SWIM 而言，**哪怕只有一个成员过载，也足以引发一些误判故障检测事件**；而**仅仅 4 个过载成员（占集群的 4%）就足以在健康成员处产生数百次误报**，且随着压力成员增加问题更加严重。

相比之下，**Lifeguard 在 16 台机器同时受压之前不会产生误报，在 32 台机器同时受压之前不会在健康成员处产生误报**。两者的水平都显著低于 SWIM。

这些问题并不经常发生。但一旦发生，就可能极具破坏性。调查若干被升级为高优先级支持请求的事故，促成了 Lifeguard 的开发。

## 3. SWIM 与 memberlist

本章先回顾 SWIM，再描述我们用于评估 Lifeguard 的 SWIM 实现——memberlist。

### 3.1 SWIM

SWIM 有两个组成部分：

- **故障检测器**（Failure Detector），检测成员的故障；
- **传播组件**（Dissemination Component），传播关于成员加入、离开或故障的更新。

传播组件是基于 gossip 的。[^2] 每条关于成员加入、离开或故障的更新，会与**另外一个**成员共享 $\lambda \log(n)$ 次，其中 $n$ 是已知组的大小，$\lambda$ 是一个可调倍数。这些更新**捎带**（piggyback）在故障检测协议的 ping、ping-req 和 ack 消息上，因此不会产生额外的消息。每条消息上捎带的更新数量是有限的（以尊重消息大小上限，比如 UDP 包的 MTU），并且**共享次数较少的更新会被优先选择**，以便在更新活动高峰期也能推进所有更新的传播。

[^2]: SWIM 论文曾试探性地提出一个基于多播的传播组件，但随即否决了该方案，转而采用基于 gossip 的方案。

在 SWIM 最简单的实现里，当一个成员在一次故障检测检查中失败时，它**立即**被标记为故障，并通过传播组件把这次故障与组共享。然而，SWIM 的作者自己也观察到了误判故障检测，并指出**消息处理缓慢是主要原因**。为此，SWIM 论文引入了 Suspicion 机制。它以扩展的形式引入，但在实践中是 SWIM 不可或缺的组成部分——上文讨论的三个成熟 SWIM 实现都实现了它。

**开启 Suspicion 后**，一个未通过故障检测检查的成员会先进入一个中间状态——**suspect（可疑）**，并通过传播机制 gossip 一条 suspect 消息，以查看该怀疑能否在怀疑超时到达之前被**驳倒（refute）**。任何收到 suspect 消息的成员也会把指定成员标记为 suspect，并 gossip 自己的怀疑。如果怀疑超时到达时怀疑仍未被驳倒，该可疑成员就被宣布为故障，通过 gossip 一条 **confirm** 消息完成。**这样一来，Suspicion 机制以更高的故障检测延迟换来了更低的误判故障检测率。**

怀疑可以通过一条关于该可疑成员的 **alive** 消息被驳倒——只要这条 alive 消息在任何一个持有该怀疑的成员达到其怀疑超时之前，**传达到所有持有该怀疑的成员**。SWIM 论文描述了 alive 消息可能产生的两种机制：一种是持有怀疑的成员在某轮故障检测中成功探测到该可疑成员时发出；另一种是被怀疑的成员自身在收到关于自己的 suspect 或 confirm 消息后发出。然而在实践中，**要使驳倒生效，被怀疑的成员必须主动 gossip 一条关于自己的 alive 消息**。[^3]

[^3]: suspect、confirm 和 alive 消息都携带一个关于它们所指成员的 **incarnation number**（化身号），用于在相互竞争的消息之间建立优先关系，并引导组的状态走向收敛。如 SWIM 论文第 4.2 节所述，alive 消息只有在 incarnation number 更高时才能覆盖其他类型的消息，且**只有被怀疑的成员自己才能递增其 incarnation number**——它是在收到关于自己的 gossip suspect 或 confirm 消息时这样做的。

SWIM 论文对基本协议所做的另一个精化是：**让每个成员从它已知的成员列表中以轮转（round-robin）方式选择故障检测目标**，而不是完全随机。否则，最坏情况下的首次检测延迟将是无界的——因为存在一种（极其罕见的）情况：全体成员对探测目标的选择反复都没有选中那个故障成员。采用轮转后，最坏情况是有界的。不过，每个成员的列表**仍然保持随机顺序**，新成员被插入到随机位置。因此，**首次检测延迟的期望保持不变**。

### 3.2 memberlist

memberlist [5] 是一个 SWIM 的开源实现，被 Consul [6]、Nomad [7] 和 Serf [11] 等工具使用。memberlist 实现了上述 SWIM 的全部特性，并额外具备以下特性：

- **UDP 为主 + TCP 兜底**。memberlist 的故障检测器默认对直接探测和间接探测都使用 UDP。但在发起间接探测（UDP）的同时，它也会尝试用 **TCP** 做一次直接探测。这有助于应对"TCP 流量路由正常、但 UDP 不通"的情况——这是网络配置中偶尔会遇到的病理现象。
- **反熵（anti-entropy）机制**。每个成员会周期性地与另一个随机选中的成员做一次**全量状态同步**（走 TCP）。采用 [12] 的 push-pull 方式，用 incarnation number 来调和关于某个成员的冲突状态。这种全量状态同步**提高了节点完全收敛得更快的可能性，代价是更多带宽消耗**。它对**加速网络分区后的恢复**尤其有帮助。
- **独立的 gossip 层**。memberlist 有一个独立于故障检测协议的专门 gossip 层。和 SWIM 一样，memberlist 会把 gossip 消息（suspect、alive 以及 confirm[^4]）捎带到故障检测消息（ping、ping-req、ack）上，但它还会**周期性地单独发出 gossip 消息**。这使得 **gossip 速率可以独立于故障检测速率进行调节**，必要时还可以比后者更快，以加快收敛速度。
- **保留失败节点的状态一段时间**，以便失败节点的信息能在全量状态同步中传递。这有助于组的状态更快收敛。

[^4]: 在 memberlist 中，confirm 消息被改名为 **dead**，因为 confirm 这个名字对于"被确认的是什么"来说有歧义。

由于这些特性在 memberlist 的部署中通常是启用的，因此本文的评估中也全部启用了。Butterfly [2] 和 Ringpop [4] 实现了许多相同的附加特性。

## 4. Lifeguard

在调查第 2 章所述问题的可能解决方案时，我们观察到：**Suspicion 机制仍然隐含假设了消息能被及时处理**。具体来说，**驳倒一个怀疑只有在所有持有该怀疑的成员都能及时处理那条驳倒的 alive 消息时才可能成功**。因此，**故障检测模块自身的慢处理，才是 Suspicion 机制没能抑制的那些误报的主要成因**。

我们还观察到：**预期响应缺失本身就可能表明某个成员正经历慢消息处理**；而且**某个组成员身上的一段慢消息处理 episode，很可能在短时间内影响它与其他成员的多次交互**。我们把这些看作该成员的**局部故障检测器实例的健康程度的度量**，简称**局部健康**（local health）。

基于这些洞察，我们设计了 **Lifeguard**：一组让 SWIM 成为**自适应协议**的扩展。Lifeguard 使用局部健康的启发式度量，让成员判断自己的故障检测器是否可能正在慢速处理消息；若是如此，就**动态调整超时**以缓解时序问题。

Lifeguard 有意识地尽可能保持与 SWIM 相同的设计。它只在三个组件上与 SWIM 不同，也正是这三个组件提供了它的新行为：

- **LHA-Probe**（Local Health Aware Probe，局部健康感知探测）替换了 SWIM 故障检测器的探测阶段。SWIM 的探测周期和超时是固定的，LHA-Probe 让它们**依据该成员近期与他人进行的故障检测相关通信动态调整**。
- **LHA-Suspicion**（Local Health Aware Suspicion，局部健康感知怀疑）替换了 SWIM 故障检测器的怀疑阶段。SWIM 的怀疑超时是固定的，LHA-Suspicion 使用**动态超时**：每个新怀疑的初始超时**显著高于**固定情形下的值，但随着针对同一可疑成员的**独立怀疑**被处理而**逐步降低**。
- **Buddy System**（伙伴系统）替换了 SWIM 的捎带消息选择器，**优先通知被怀疑的成员本身**，以缩短平均驳回时间——这既有助于 LHA-Probe，也有助于 LHA-Suspicion 更有效地发挥作用。

### 4.1 LHA-Probe

LHA-Probe 替换了 SWIM 故障检测器的探测阶段：固定的探测周期与超时，变成**依据该成员近期与他人进行的故障检测相关通信动态调整**。它使用了几种反馈来源：

- **收到的 ack 消息数**与**发出的 ping、ping-req 消息数**做比较。缺失的 ack 可能源于本地变慢，**在有多个缺失时尤其如此**。
- **需要驳倒关于自己的怀疑**，说明该成员可能没有及时处理近期的 ping 消息。
- **LHA-Probe 向故障检测协议中新增了一条 nack 消息**，在间接探测失败时发送。[^5] 这给了发起间接探测的成员一种手段：**即使被间接 ping 的目标没有响应，它也能检查自己是否在及时收到那 $k$ 个成员的响应**。

[^5]: 当一个成员收到 ping-req 消息时，它会在**探测超时的 80%** 处回送一条 nack——除非它在此之前收到了 ack。如果 nack 发出之后才收到 ack，该 ack 仍会被转发；在超时窗口内先收到 nack 后又收到 ack 的成员，**认为这次间接探测是成功的**。

这些不同的反馈来源被组合成一个**局部健康乘数**（Local Health Multiplier，**LHM**）。LHM 是一个**饱和计数器**，最大值 $S$、最小值 0，即它不会超过 $S$、也不会低于 0。以下事件引起 LHM 计数器的指定变化：

| 事件 | LHM 变化 |
|---|---|
| 成功的探测（ping 或 ping-req 收到 ack） | $-1$ |
| 失败的探测 | $+1$ |
| 驳倒关于自己的 suspect 消息 | $+1$ |
| 探测时错过 nack | $+1$ |

LHM 的当前值用于如下设置探测周期与超时：

$$
\mathrm{ProbeInterval} = \mathrm{BaseProbeInterval} \cdot (\mathrm{LHM}(S) + 1)
$$

$$
\mathrm{ProbeTimeout} = \mathrm{BaseProbeTimeout} \cdot (\mathrm{LHM}(S) + 1)
$$

其中 **ProbeInterval** 是对连续随机选出的对等节点发起存活探测之间的间隔，**ProbeTimeout** 是等待某次探测的 ack 的超时。在 memberlist 实现中，**BaseProbeInterval 设为 1 秒，BaseProbeTimeout 设为 500 毫秒**。$S$ 默认为 **8**，这意味着探测周期和超时最多会退避到 **9 秒**和 **4.5 秒**。

### 4.2 LHA-Suspicion

LHA-Suspicion 替换了 SWIM 故障检测器的怀疑阶段：固定的怀疑超时，变成**动态超时**。

每个新怀疑的超时**起始值显著高于**固定情形下的值，但会随着针对同一可疑成员的独立怀疑被处理而**降低**。这样一来，**只要本地成员在及时接收并处理 gossip 消息，超时就会降到其最小值**；反之，对于那些不能及时接收和处理 gossip 消息的成员，超时将保持较高。

给定怀疑的超时按如下方式计算：

$$
\mathrm{SuspicionTimeout} = \max\left(\mathrm{Min},\; \mathrm{Max} - (\mathrm{Max} - \mathrm{Min}) \cdot \frac{\log(C + 1)}{\log(K + 1)}\right)
$$

其中：

- **Min** 与 **Max** 是怀疑超时的最小值与最大值（配置见 5.3 节）；
- **$K$** 是把怀疑超时降到 Min 所需的**独立怀疑数量**，默认为 **3**；
- **$C$** 是本地提出该怀疑以来，**收到的关于该成员的独立怀疑数量**。

每当收到一条代表**此前未见的、针对同一成员的独立怀疑**的 suspect 消息时，超时会被重新计算。此时当前的怀疑计时器被取消，替换为一个针对**剩余时间**的新计时器。如果剩余时间已经耗尽，则超时直接触发。

使用**对数衰减**，使得随着收到更多独立怀疑消息，每次的超时递减幅度都小于上一次。背后的直觉是：**第一条独立消息对"消息正在被及时处理"这一判断给出的置信度提升最大，后续每一条的边际贡献更小。**

为了让独立怀疑更普遍，**启用 LHA-Suspicion 时，收到的关于同一成员的前 $K$ 条独立怀疑会被重新 gossip**。每条怀疑被发送 $\lambda \log(n)$ 次，因此若收到 $K$ 条以上的怀疑，发送的消息数上限为 $(K + 1)\lambda \log(n)$。**没有 LHA-Suspicion 时，独立怀疑不会被重新 gossip，只有该成员自己的怀疑被 gossip，上限为 $\lambda \log(n)$ 次。**

### 4.3 Buddy System

在 SWIM 中，被怀疑的成员**并不保证能在第一个机会就听到这个怀疑**。被怀疑的节点只有在收到关于自己的 gossip suspect 消息时才知道自己被怀疑了。虽然 gossip 消息被捎带在故障检测消息（包括 ping 消息）上，但支配 gossip 消息传播的规则包括：**每条捎带消息所携带的 gossip 消息数量有限**、**每条 gossip 消息的重发次数有限**、以及**偏好更新的 gossip 消息**。

Buddy System 替换了 SWIM 的捎带消息选择器，**优先通知被怀疑的成员本身**。这保证了**任何 ping 了被怀疑节点（无论是为自己 ping，还是为另一个节点的间接路径 ping）的节点，都会把该怀疑作为 ping 的一部分传达出去**。这可能让驳回更早开始——**即使没有 Lifeguard 的其他组件，这本身也是有帮助的**。但它同时也让 LHA-Probe 和 LHA-Suspicion 工作得更有效。

## 5. 评估

### 5.1 评估指标

我们采用与原始 SWIM 论文评估 SWIM 时相同的标准来评估 Lifeguard（见 [1] 第 5 节）：

- **故障检测误报**。Lifeguard 的目标是减少被错误标记为故障的健康成员数量。
- **检测与传播延迟**。Lifeguard 应当**不增加**真实故障首次检测或完整传播所需的时间。
- **消息负载**。我们考察发送的消息数与字节数。Lifeguard 应当降低这些指标，或者**不会有很大幅度的增加**。

### 5.2 测试配置

第 4 章描述的 Lifeguard 三个组件被分别评估、也组合评估，以理解各自的贡献以及它们之间的相互作用。

| 配置 | 说明 |
|---|---|
| `SWIM` | 常规 SWIM（完全关闭 Lifeguard） |
| `LHA-Probe` | SWIM + 局部健康感知探测 |
| `LHA-Suspicion` | SWIM + 局部健康感知怀疑 |
| `Buddy System` | SWIM + 伙伴系统 |
| `Lifeguard` | 启用全部 Lifeguard 组件 |

每种配置都运行实验。**`SWIM` 配置给出完全关闭 Lifeguard 时的性能**，它是其他组合的对比基线。之所以能做到这一点，是因为实验运行的是一个修改过的 Consul，每个组件可以被独立地启用或禁用。

### 5.3 怀疑超时配置

如 4.2 节所述，Lifeguard 的 LHA-Suspicion 组件使用 Min 与 Max 两种怀疑超时。在 memberlist 实现中，配置如下：

$$
\mathrm{Min} = \alpha \log_{10}(n) \cdot \mathrm{ProbeInterval}
$$

$$
\mathrm{Max} = \beta \cdot \mathrm{Min}
$$

其中：

- $n$ 是已知组中的成员数量；
- **ProbeInterval** 是连续故障检测探测消息之间的间隔，所有实验中默认值均为 **1 秒**；
- $\alpha$ 与 $\beta$ 是可调参数。

为了考察这些可调参数的影响，每种配置都重复实验 **9 次**，完整的 Lifeguard（测试配置 `Lifeguard`）分别以 $\alpha \in \{2, 4, 5\}$、$\beta \in \{2, 4, 6\}$ 的不同组合进行配置。不同组合的性能与**完全关闭 Lifeguard 的 memberlist**（测试配置 `SWIM`）的基线性能作比较——后者有一个固定的怀疑超时，等价于配置 $\alpha = 5$、$\beta = 1$。

### 5.4 实验设计

SWIM 论文正确地指出，慢消息处理可能由多种因素引起，包括 CPU 耗尽、网络延迟和丢包——无论是在本地主机还是在网络中。最终结果都是**未能及时处理一个或多个协议消息**。

为了调查这个问题，我们**通过在选定的组成员上暂停协议消息的发送与接收、持续明确定义的时间段**来诱发慢消息处理。我们把**一个成员上的一段延迟称为一次异常（anomaly）**。

评估 5.1 节定义的标准时使用了**两类实验**：**Threshold**（阈值）与 **Interval**（间隔）。之所以要用两类实验，原因如下：

- **Threshold 实验**在每次实验中引入**一组**并发异常。这样可以考察**从异常开始到它被检测并传播所需的时间**，因为只有一组异常时**因果关系是清晰的**。
- 然而在真实世界中，CPU 与网络延迟是**间歇性**的，进程以小批次的方式推进。**Interval 实验**通过以**周期性的方式**引入异常来探索这个空间，异常的持续时长与间隔都在多组实验中变化。

**1）Threshold 实验**：用于考察 Lifeguard 对**检测与传播延迟**的影响，形式如下：

- 启动 **128 个 Consul agent**，全部位于**同一台 Linux 虚拟机**内，通过 **loopback（回环）网络接口**通信；
- 允许 agent **静默 15 秒**；
- **$C$ 个**（随机选出）实例进入异常状态：它们在**发送**任何协议消息之前、或者在**接收**到来自集群其他成员的任何协议消息**之后**，立即阻塞；[^6]
- 异常持续 **$D$** 的时长，$D$ 结束时解除对收发操作的阻塞；
- 实验持续进行，直到全部 **128 个** Consul 实例重新认为彼此健康，或者从实验开始起经过 **120 秒**。

[^6]: 异常期的开始与结束通过虚拟机的系统时钟同步，使得 $C$ 个异常实例**同步**改变状态。虽然可以考察更多样的起止时间组合，但这代表了 **$C$ 个完全相关的异常**的最坏情形，例如机架级掉电。

对每种被测的 Lifeguard 组件组合和其他实验参数，都要运行 **Threshold 实验的许多次实例**，扫描 $C$ 和 $D$ 的一系列取值。

**2）Interval 实验**：用于考察 Lifeguard 对**误判故障检测**和**消息负载**的影响。其形式与 Threshold 实验相同，但有以下不同：

- 在持续 $D$ 的异常期结束时，每个异常的 Consul 实例**恢复正常运行 $I$ 的时长**；
- 异常与正常运行的循环以 $D$ 和 $I$ 分别为长度**交替进行**，直到自测试开始起至少经过 **120 秒**。测试在下一个异常期结束时结束。

同样，对每种被测的 Lifeguard 配置，都要运行 **Interval 实验的许多次实例**，扫描 $C$、$D$、$I$ 的一系列取值。

| 参数 | 符号 | 测试取值 |
|---|---|---|
| 并发异常数 | $C$ | 1, 4, 8, 12, 16, 20, 24, 28, 32 |
| 每次异常的持续时长 | $D$ | 128, 512, 2048, 8192, 16384, 32768（毫秒） |
| 异常之间的间隔 | $I$ | 1, 4, 16, 64, 256, 1024, 4096, 16384（毫秒） |

每种 Lifeguard 组件与其他实验参数的组合，实验运行 **10 次**。

### 5.5 实验环境

实验运行在 Microsoft Azure Compute-Optimized（F 系列）虚拟机上，部署于 **2.4 GHz Intel Xeon E5-2673 v3（Haswell）** 处理器之上。使用 **F16** 实例，每台分配 **16 核**与 **32 GiB** 内存。操作系统为 **Ubuntu 16.04 LTS daily build 201701280**，Consul 被配置为把 **DEBUG 级日志**写入 `/dev/shm`——Ubuntu 默认配置的 ramdisk。日志在每次实验结束时被复制到 SSD。

为减少调度与内存访问的不确定性，使用 Linux 的 **numactl** 工具，把 **8 个 Consul agent 绑定到 16 个 CPU 核中的每一个**及其对应的内存 bank 上。CPU 使用率在整个实验生命周期内持续监控，方式是**以 1 秒的间隔采样 `/proc/stat`** [13]，并确认**所有实验中都有空闲的 CPU 容量**（表现为每个采样间隔上聚合的核空闲时间都有增加）。实践中，16 核对于 128 个 Consul agent 来说是过剩的。

实验使用 Consul（一个构建在 memberlist 之上的服务发现与监控系统）的部署进行。但**未启用 Consul 的任何高层特性**（例如基于 Raft 的可用服务一致视图），且集群以**不含 server 实例**的方式部署，因此**只有 memberlist 的特性被实际检验**。

### 5.6 结果

实验探索了一个很大的参数组合空间。为了让结果更易处理，我们首先考察 Lifeguard 在怀疑超时参数取**所考虑的最高值**（$\alpha = 5$、$\beta = 6$）时的性能，然后再考察降低 $\alpha$ 与 $\beta$ 的影响。

#### 5.6.1 故障检测误报

使用 Interval 实验测量 Lifeguard 对**误判故障检测**——即健康 agent 被错误标记为故障——的影响。我们把一次误判故障检测定义为：**每当一个 agent 故障事件被提出，而所指对象不在被引入异常的 agent 集合中**。在这些误报中，我们进一步区分出发生在**任一** Consul agent 上的误报（记作 **FP**），和发生在**健康** agent 上的误报（记作 **FP-**）。

**FP- 最令人担忧**，因为此时事件的两端——提出事件的 agent 和事件所指的 agent——**都是健康的**。

下表汇总了 $\alpha = 5$、$\beta = 6$ 下所有 Interval 实验的误报统计：

| 配置 | FP 事件数 | FP- 事件数 | FP 相对 SWIM | FP- 相对 SWIM |
|---|---|---|---|---|
| `SWIM` | 339002 | 1326 | 100.00% | 100.00% |
| `LHA-Probe` | 229574 | 436 | 67.72% | 32.88% |
| `LHA-Suspicion` | 10174 | 89 | 3.00% | 6.71% |
| `Buddy System` | 318935 | 591 | 94.08% | 44.57% |
| `Lifeguard` | **5193** | **25** | **1.53%** | **1.89%** |

表 IV 显示，**误报主要由发生在慢处理成员处的误报主导**——这一点由 FP- 在所有被测配置（包括完全关闭 Lifeguard 的基线 SWIM）中所占比例很小即可看出。

表 IV 还显示，**引入 Lifeguard 后误报率被大幅削减**。Lifeguard 的所有组件都对这一削减有贡献，其中 **LHA-Suspicion 的单独贡献最大**。组合全部组件（Lifeguard）效果最好：**误报总数（FP 事件）和健康节点处的误报（FP- 事件）都被削减到 SWIM 基线的 2% 以下——这意味着误报减少了 50 倍以上。**

**Buddy System 的效果值得注意**：它把健康节点处的误报**减少了一半以上**（44.57%），但对误报总数（94.08%）影响不大。这个差异可以从它的作用方式解释——它帮助被怀疑的成员**更及时地知道自己的怀疑**，从而让驳回更早开始。健康成员（产生 FP- 的那些）能及时接收并处理驳回消息，而**正经历慢消息处理的成员往往做不到**。由于 FP 总体上由慢消息处理成员处的误报主导，Buddy System 对它几乎没什么影响。

表 IV 中的数字汇总了**所有并发异常数（$C$）下**的误报事件计数。图 2 与图 3 则分别考察误报数随并发异常数的变化：

- **图 2（误报总数）**：清楚显示，误报数随并发异常数上升，但在**每一个并发水平上**，完整的 Lifeguard 都把误报数**削减了 50 倍到 100 倍**。
- **图 3（健康成员处的误报）**：比图 2 更嘈杂，因为这类事件比一般误报罕见得多。同样地，误报数随并发异常数上升，并且在**每一个并发水平上**，完整的 Lifeguard 都把健康成员处的误报**削减了 10 倍到 100 倍**。**误报率被削减得如此之多，以至于在某些并发水平下，重复测试中健康节点处出现了零误报。**

#### 5.6.2 检测与传播延迟

使用 Threshold 实验测量 Lifeguard 对**真实故障的检测与传播延迟**的影响。

下表给出 $\alpha = 5$、$\beta = 6$ 下各组件对延迟的影响（**所有时间单位为秒**；原文是一张 7 列宽表，这里拆成两张）：

首次检测延迟：

| 配置 | 中位首次检测 | 99% 首次检测 | 99.9% 首次检测 |
|---|---|---|---|
| `SWIM` | 12.44 | 16.96 | 19.40 |
| `LHA-Probe` | 12.42 | 17.75 | 20.10 |
| `LHA-Suspicion` | 12.42 | 17.47 | 25.41 |
| `Buddy System` | 12.45 | 17.12 | 19.16 |
| `Lifeguard` | 12.45 | 17.90 | 21.20 |

完整传播延迟：

| 配置 | 中位完整传播 | 99% 完整传播 | 99.9% 完整传播 |
|---|---|---|---|
| `SWIM` | 12.90 | 16.93 | 20.17 |
| `LHA-Probe` | 12.90 | 17.98 | 20.56 |
| `LHA-Suspicion` | 12.89 | 17.33 | 23.80 |
| `Buddy System` | 12.92 | 17.18 | 19.81 |
| `Lifeguard` | 12.91 | 18.05 | 21.68 |

表 V 显示，**完整的 Lifeguard 使首次检测与完整传播的延迟只小幅上升**：中位数**上升 0.1 秒（不到 0.1%）**，首次检测与完整传播都是如此。而 **99% 与 99.9% 分位数的延迟上升更大**：首次检测约 1 秒（6～7%），99.9% 分位完整传播约 1.5～1.8 秒（7～9%）。**LHA-Probe 对 99% 分位延迟上升的贡献最大，LHA-Suspicion 对 99.9% 分位延迟上升的贡献最大。**

#### 5.6.3 消息负载

使用 Interval 实验测量 Lifeguard 对消息负载的影响。每次实验发出的消息数与总字节数通过 **Consul 的遥测（telemetry）** [14] 采集。

下表汇总了 $\alpha = 5$、$\beta = 6$ 下的聚合消息负载统计：

| 配置 | 消息数（百万） | 字节数（GiB） | 消息数相对 SWIM | 字节数相对 SWIM |
|---|---|---|---|---|
| `SWIM` | 435.33 | 149.15 | 100.00% | 100.00% |
| `LHA-Probe` | 428.62 | 134.28 | 98.46% | 90.03% |
| `LHA-Suspicion` | 484.55 | 158.87 | 111.31% | 106.52% |
| `Buddy System` | 435.62 | 147.67 | 100.07% | 99.01% |
| `Lifeguard` | 481.42 | 146.13 | 110.59% | 97.97% |

表 VI 显示，在 $\alpha = 5$、$\beta = 6$ 时，Lifeguard 带来的**消息数平均增加约 11%**，但**实际发送的数据量反而下降了约 2%**。**LHA-Suspicion 是消息数和字节数上升的主要来源**；不过这一效应被 **LHA-Probe 抵消了**——后者同时减少了消息数和字节数。

#### 5.6.4 怀疑超时的调优

前面的结果都是在怀疑超时参数取最高值（$\alpha = 5$、$\beta = 6$）时得到的。现在考察降低 $\alpha$ 与 $\beta$ 的影响。

下表给出 Lifeguard 在不同 $\alpha$、$\beta$ 组合下的各项指标，**均以同一组实验中 SWIM 基线的百分比表示**（原文把 9 种组合排成一张宽表，这里转置成两张，便于窄屏阅读）：

延迟指标（占 SWIM 基线的百分比）：

| 配置 | 中位首次检测 | 99% 首次检测 | 99.9% 首次检测 | 中位完整传播 |
|---|---|---|---|---|
| $\alpha$=2, $\beta$=2 | 53.14 | 69.81 | 76.08 | 55.12 |
| $\alpha$=2, $\beta$=4 | 54.10 | 72.88 | 75.41 | 56.28 |
| $\alpha$=2, $\beta$=6 | 54.34 | 75.53 | 80.36 | 56.74 |
| $\alpha$=4, $\beta$=2 | 82.96 | 94.28 | 99.07 | 84.42 |
| $\alpha$=4, $\beta$=4 | 83.04 | 96.17 | 93.71 | 84.03 |
| $\alpha$=4, $\beta$=6 | 83.12 | 96.82 | 94.69 | 84.42 |
| $\alpha$=5, $\beta$=2 | 99.76 | 104.95 | 112.32 | 99.92 |
| $\alpha$=5, $\beta$=4 | 99.52 | 102.71 | 111.44 | 99.61 |
| $\alpha$=5, $\beta$=6 | 100.08 | 105.54 | 109.28 | 100.08 |

传播延迟尾部与误报（占 SWIM 基线的百分比）：

| 配置 | 99% 完整传播 | 99.9% 完整传播 | FP | FP- |
|---|---|---|---|---|
| $\alpha$=2, $\beta$=2 | 73.07 | 76.20 | 98.37 | 31.15 |
| $\alpha$=2, $\beta$=4 | 76.96 | 75.11 | 43.64 | 22.47 |
| $\alpha$=2, $\beta$=6 | 79.15 | 78.58 | 24.16 | 13.65 |
| $\alpha$=4, $\beta$=2 | 97.05 | 92.17 | 37.72 | 20.29 |
| $\alpha$=4, $\beta$=4 | 96.69 | 95.14 | 8.04 | 9.50 |
| $\alpha$=4, $\beta$=6 | 96.52 | 92.71 | 3.18 | 4.83 |
| $\alpha$=5, $\beta$=2 | 105.73 | 107.64 | 26.61 | 15.38 |
| $\alpha$=5, $\beta$=4 | 105.08 | 107.93 | 5.43 | 5.05 |
| $\alpha$=5, $\beta$=6 | 106.62 | 107.49 | **1.53** | **1.89** |

可以观察到以下关系：

- **全部六项延迟指标**（中位首次、中位完整、99% 首次、99% 完整、99.9% 首次、99.9% 完整）**与 $\alpha$ 正相关**。
- 当 $\alpha = 2$ 时，延迟指标（尤其是 99% 和 99.9% 指标）**也与 $\beta$ 正相关**。在更高的 $\alpha$ 取值下，这种相关性不明显。
- **误报总数（FP）与健康成员处误报（FP-）与 $\alpha$ 和 $\beta$ 负相关。**

因此，**$\alpha$ 与 $\beta$ 可以用来在调低检测与传播延迟的同时降低误报率**。由于更低的 $\alpha$ 和 $\beta$ 改善延迟但让误报率变差，**降低检测延迟必须与更高的误报率做交换**。

不过，即使在最极端的权衡情形下（$\alpha = 2$、$\beta = 2$）——此时中位检测与传播延迟相比 SWIM **降低了约 45%**——健康节点处的误报率 FP- **仍然降低了 68%（3 倍）**。而在另一个极端（$\alpha = 5$、$\beta = 6$），中位延迟保持在 SWIM 的水平，但**误报降低了超过 98%（50 倍以上）**，代价只是 99% 与 99.9% 分位延迟的适度上升。

在两个极端之间选取 $\alpha$ 与 $\beta$ 的中间值，**就能以较粗的粒度调优"降低延迟"与"减少误报"之间的权衡**。我们把 $\alpha$ 与 $\beta$ 暴露为 Lifeguard 的参数。

## 6. 相关工作

据我们所知，**Lifeguard 是首个处理"SWIM 对故障检测模块自身的慢消息处理的敏感性"的工作**，也可能**是首个处理任何分布式故障检测系统之局部故障检测模块慢消息处理的工作**。

我们把 Lifeguard 与**自适应故障检测**和**自适应 gossip 协议**两条文献线的关系都做考察。

由于具备强成员保证的协议在数据中心规模的场景下不具备所需的扩展性 [15]，我们把讨论限制在**不可靠故障检测**上。

Chandra 与 Toueg [16] 引入了**不可靠故障检测**的概念，SWIM、Lifeguard 以及部署在普通硬件上的大多数故障检测器都属于这一类。他们把**完备性**（completeness）与**准确性**（accuracy）识别为评估不可靠故障检测器的关键性质。然而，他们的重点是理解**在什么条件下不可靠故障检测器可以被用于构建可靠的分布式共识协议**（等价地，原子广播协议）。因此，他们**只在抽象层面推理故障检测器，并不探索如何让一个检测器本身变得更可靠**。

Chen 等人 [17][18] 指出 [16] 只提供最终保证，没有任何时序假设。为解决这一点，他们引入了故障检测器的**服务质量（QoS）**，并把**检测延迟**识别为许多用例中的关键性质。他们提出了一个自适应故障检测器——**和 Lifeguard 一样，它依据近期关于消息丢失与延迟的观测来调整超时**。但与 Lifeguard 不同的是，**它没有考虑本地故障检测器本身可能运行缓慢**，因此一个缓慢的检测器仍可能对它所监控的健康对等节点报告误报。

Bertier 等人 [19] 改进了 [17]，采用了更好的网络延迟估计，从而得到更低的平均检测时间。

Hayashibara 等人 [20] 对 [17] 做了更显著的修改，引入了**累积式故障检测器**（accrual failure detector），用连续尺度上的**怀疑值**取代传统的布尔检测器输出。这让应用可以对被监控成员的健康状况做出更细致的决策。

Satzger 等人 [21] 让累积式检测器计算效率更高，并去掉了"到达时间服从正态分布"的假设。最近，Liu 等人 [22] 针对云环境中观察到的消息延迟，提出了一种更合适的特定到达时间分布。然而，**同样地，这些设计都没有考虑本地故障检测器可能运行缓慢**，因此它们与 [17] 有同样的问题：一个缓慢运行的检测器可能对它监控的健康对等节点做出误报。

**上述所有工作都是基于心跳的。** 我们观察到，**心跳本身没有任何东西阻碍对本地检测器时效性的建模**。但所有这些工作都关注**单个故障检测器在 1 对 1 监控关系中针对单个对等节点的运行**，这意味着 **Lifeguard 的启发式无法直接套用**。在多个并置的、基于心跳的检测器（各自接收来自不同对等节点的消息）的场景下，评估 Lifeguard 启发式的适用性是可能的。我们在第 7 章回到这一点。

Gupta 等人 [23] 把自适应性引入 gossip 文献。与 Lifeguard 一样，他们的自适应方案利用关于对等节点故障和消息丢失的**本地知识**，并用它采取补救行动（在他们的情形中，是**切换到另一个传播子协议**）。然而与 Lifeguard 不同的是，**他们没有考虑缓慢——无论是消息投递的缓慢还是成员自身的缓慢**。此外，**所评估的指标是瞬时的，而不是在一段时间内累积的**，也**没有考虑不同对等节点之间的相关性**。

另有一些自适应 gossip 协议与 Lifeguard 相似：它们基于本地成员与对等节点的交互来调整消息发送。Levis 等人 [24] 和 Gobriel 等人 [25] 延迟转发消息，Haas 等人 [26][27] 与 Kyasanur 等人 [28] 则改变转发消息的概率。Bhandari 与 Gupta [29] 同时改变转发延迟与概率。然而，**这些协议全都面向无线传感器网络（WSN）**，其自适应行为关注于**消除不必要的消息**。它们**被动地适应成员故障**，**不使用基于探测的故障检测，也不提供时效性保证**。因此，它们**没有建模慢消息处理的必要**。

Johansen 等人 [30] 提出了一个与 Lifeguard 在许多方面相似的自适应 gossip 协议。和 SWIM（因而是 Lifeguard）一样，它是一个通用的组成员协议，**使用基于探测的故障检测**；它有**怀疑阶段**和**基于 gossip 的更新传播子协议**。与 Lifeguard 一样，它**自适应地调整探测超时**，但**更加细粒度**——它为每个被探测的对等节点独立调优（这是可行的，因为探测目标集合基于同一**伪随机环**上的成员身份，因而非常小且比 SWIM 和 Lifeguard 稳定得多，后两者要轮转遍历所有已知对等节点）。然而，**与 Lifeguard 不同的是，它的怀疑超时并未自适应调优**。更重要的是，**它的探测调优没有考虑本地故障检测器可能存在慢消息处理**。事实上，"所有正确的（即非拜占庭且未故障的）成员都能及时运行其探测与更新传播子协议"这一假设在 Fireflies 论文（[30]）的 4.1 节中被明确陈述。**那里的自适应调优仅仅是为了适应不可靠的消息投递。**

## 7. 结论与未来工作

我们开发 Lifeguard 的目标是**在最小影响延迟和消息负载的前提下，降低相对于 SWIM 的误判故障率**。在测试的广泛范围内，Lifeguard 达成了这一目标：**误报减少了 10 倍到 100 倍，平均超过 50 倍**。误报率被削减得如此之多，以至于**在某些并发异常水平下，重复测试中健康节点处出现了零误报**。

这是**在中位检测与传播延迟几乎没有增加（不到 0.1%）的代价下**达成的，**代价只是 99% 与 99.9% 分位延迟适度上升（6～9%）**。**平均而言，多发出约 12% 的消息，但实际发送的总字节数反而下降约 2%。**

此外，通过调优 Lifeguard 的 LHA-Suspicion 组件所使用的超时，**部分误报的削减可以与延迟的削减做交换**。但在测试过的最极端权衡下——中位检测与传播延迟**降低 45%（接近 2 倍）**——健康节点处的误报率**仍然比 SWIM 降低了 68%（3 倍）**。

**所有延迟指标都因该调优而下降**，不过随着中位延迟降低，**中位数与 99% 分位数之间的差距会拉大**。这并不令人意外，因为 Lifeguard 的对等节点选择（与 SWIM 一样）是**随机的、成员之间没有协调**的。未来工作可以探索如何**更紧地界定检测与传播延迟**。**引入一个随机叠加网络（random overlay network）是一种可能的方法**，我们特别地参考了 [31]。

Lifeguard 有若干目前采用**启发式确定**的参数。这些包括 LHA-Suspicion 的**重新 gossip 因子 $K$**、**LHM 计数器的饱和上限 $S$**，以及**影响 LHM 计数器的各事件所赋予的分值**。未来工作可以探索这些参数的**自动调优**，一种可能的方法是**寻找（或学习）出能够通过反馈自适应调优这些参数的度量**。

在开发 Lifeguard 的过程中，我们设计了一些**利用了 Lifeguard 从 SWIM 继承而来的随机通信模式**的启发式。未来工作可以用**形式化模型**、或**从模型导出的新启发式**（如 [31]）、或**从效用函数导出的启发式**（如 [32]）来替换 Lifeguard 的启发式。另一条独立的研究线是**考察把局部健康方法应用到其他类别的故障检测器上**。

## 译者补充

论文本身是一份完整的设计文档，落到工程上还有几点值得补充说明。

**"局部健康"其实是一个自指的概念。** 论文的第 4 章开门见山：Suspicion 机制能生效的前提是"驳倒消息被及时处理"，而这个前提本身就可能不成立。于是问题从"对端是不是挂了"变成了"**我是不是自己太慢了**"。LHM 计数器把四种自我观测信号（探测成功、探测失败、需要自证、错过 nack）压缩成一个标量，再用来放大超时——**这是一个纯粹的单变量反馈控制**，没有分布式协调，也不依赖其他节点给建议。

**摘要说误报率和真实故障检测时间"可以同时降低"，但论文自己的数据是检测时间略有上升。** 表 V 显示中位首次检测从 12.44 秒变成 12.45 秒，99.9% 分位从 19.40 秒变成 21.20 秒。论文的说法在"降低"的宽泛意义上成立（配合 $\alpha$、$\beta$ 调优时甚至能降 45%），但按字面理解会高估效果。译文正文照原文保留了这一断言，这里补充实际数据。

顺带一提，原文正文里"中位数**上升 0.1 秒（不到 0.1%）**"这句本身就有笔误：12.44 秒到 12.45 秒只差 **0.01 秒**（约 0.08%），0.1 秒对应的是 0.8%，与同句的"不到 0.1%"对不上。译文按原文照录了这句话。

**论文自己有一处数字前后不一致**：正文 5.6.3 节说消息数"平均增加约 11%"，结论段说"约 12%"，而表 VI 给出的实际值是 110.59%（即约 10.6%）。译文两处都按原文照录，没有统一。

**脑裂是误判最难区分的场景。** 真实故障和脑裂在信号上长得几乎一样：节点之间探测不到对方。但脑裂里每个分区内部都是健康的，gossip 收敛得很好，只有跨分区才失败——所以系统会做出一次**在崩溃模型下无懈可击、但在系统层面完全错误**的判定。Lifeguard 压的是**慢消息处理**导致的误判，压不住真正的分区：分区里根本没有"活着但慢"的节点，反驳消息也传不过去。**真正的分区必须靠共识层的多数派规则来挡**（Consul 就是让 Raft 来挡的）。

**三个组件是三张独立的牌。** 实验数据（表 IV）显示 LHA-Suspicion 贡献最大（FP 降到 3%），Buddy System 只在 FP- 上有效果（44.57% vs 总数 94.08%），LHA-Probe 则同时降低了消息数和字节数。这提示一个实际取舍：**如果只需要压误报率，优先开 LHA-Suspicion**。

**参数 $\alpha$ 与 $\beta$ 决定了你买的是延迟还是误报率。** $\alpha$ 越大、$\beta$ 越大，超时越长、误报越少、延迟越高。这与直觉一致：**多等一会儿再宣判，误报就少**。论文把这两个参数暴露出来，就是为了让运维能按业务对误报的容忍度来调。

**$S = 8$ 的含义是"最坏退避 9 倍"。** BaseProbeInterval 1 秒、BaseProbeTimeout 500 毫秒，在 LHM 达到饱和值 8 时分别退避到 9 秒和 4.5 秒。**注意这意味着一个被判定为"不健康"的节点，探测频率会降到正常的九分之一**——对于一个确实需要尽快发现对端故障的场景，这个代价要认真评估。

**这篇论文解释了 Consul 的一个真实取舍。** Consul 用 Raft 提供服务列表的强一致视图，底下用 memberlist 的 Gossip 维护成员关系——这正是论文脚注 1 提到的做法。**强一致性只覆盖"服务注册表"，成员关系本身仍然是弱一致的**，这一点在做容量规划时容易被忽略。

想了解 SWIM 协议本身（探测、传播、怀疑、incarnation number 的完整设计）请看[系列第二篇](/posts/2026/11/27ee6cbd/)；想了解 Gossip 协议的整体图景请看[系列第一篇](/posts/2026/11/30324d93/)。

## 参考资料

1. A. Das, I. Gupta, and A. Motivala, "SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol," in *Proceedings of the 2002 International Conference on Dependable Systems and Networks*, ser. DSN '02. Washington, DC, USA: IEEE Computer Society, 2002, pp. 303–312.
2. (2017, Nov.) Butterfly documentation. Chef. <https://www.habitat.sh/docs/internals/#supervisor-internals>
3. (2017, Nov.) Habitat. Chef. <https://www.habitat.sh>
4. (2017, Nov.) Ringpop documentation. Uber Technologies Inc. <https://ringpop.readthedocs.io/en/latest/architecture_design.html>
5. HashiCorp, "memberlist Project," <https://github.com/hashicorp/memberlist>
6. ——, "Consul Project," <https://www.consul.io/>
7. ——, "Nomad Project," <https://www.nomadproject.io/>
8. ——, "Consul vs. Serf," <https://www.consul.io/intro/vs/serf.html>
9. D. Ongaro and J. Ousterhout, "In search of an understandable consensus algorithm," in *2014 USENIX Annual Technical Conference (ATC '14)*. Berkeley, CA, USA: USENIX Association, 2014, pp. 305–320.
10. I. Gupta, T. D. Chandra, and G. S. Goldszmidt, "On scalable and efficient distributed failure detectors," in *Proceedings of the Twentieth Annual ACM Symposium on Principles of Distributed Computing (PODC '01)*. Newport, Rhode Island, USA, August 26–29, 2001, 2001, pp. 170–179.
11. HashiCorp, "Serf Project," <https://www.serf.io/>
12. A. Demers, D. Greene, C. Hauser, W. Irish, J. Larson, S. Shenker, H. Sturgis, D. Swinehart, and D. Terry, "Epidemic algorithms for replicated database maintenance," in *Proceedings of the Sixth Annual ACM Symposium on Principles of Distributed Computing (PODC '87)*. New York, NY, USA: ACM, 1987, pp. 1–12.
13. T. Bowden, B. Bauer, J. Nerin, S. Feng, and S. Seibold, "The /proc Filesystem," 2009.
14. HashiCorp, "Consul Telemetry," <https://www.consul.io/docs/agent/telemetry.html>
15. I. Gupta, K. P. Birman, and R. van Renesse, "Fighting fire with fire: using randomized gossip to combat stochastic scalability limits," *Quality and Reliability Engineering International*, vol. 18, no. 3, pp. 165–184, 2002.
16. T. D. Chandra and S. Toueg, "Unreliable failure detectors for reliable distributed systems," *J. ACM*, vol. 43, no. 2, pp. 225–267, Mar. 1996.
17. W. Chen, S. Toueg, and M. A. Aguilera, "On the quality of service of failure detectors," in *Proceedings of the International Conference on Dependable Systems and Networks (DSN 2000)*, 2000, pp. 191–200.
18. ——, "On the quality of service of failure detectors," *IEEE Trans. Computers*, vol. 51, no. 1, pp. 13–32, 2002.
19. M. Bertier, O. Marin, and P. Sens, "Implementation and performance evaluation of an adaptable failure detector," in *Proceedings of the International Conference on Dependable Systems and Networks (DSN 2002)*, June 2002, pp. 354–363.
20. N. Hayashibara, X. Defago, R. Yared, and T. Katayama, "The $\varphi$: accrual failure detector," in *Proceedings of the 23rd IEEE International Symposium on Reliable Distributed Systems (ISSREL 2004)*, 2004, pp. 66–78.
21. B. Satzger, A. Pietzowski, W. Trumler, and T. Ungerer, "A new adaptive accrual failure detector for dependable distributed systems," in *Proceedings of the 2007 ACM Symposium on Applied Computing (SAC '07)*, 2007, pp. 551–555.
22. J. Liu, Z. Wu, J. Wu, J. Dong, Y. Zhao, and D. Wen, "A Weibull distribution accrual failure detector for cloud computing," *PLOS ONE*, vol. 12, no. 3, 2017.
23. I. Gupta, A.-M. Kermarrec, and A. J. Ganesh, "Efficient epidemic-style protocols for reliable and scalable multicast," in *Proceedings of the 21st International Symposium on Reliable Distributed Systems (ISSREL 2002)*, 2002, pp. 180–189.
24. P. Levis, N. Patel, D. Culler, and S. Shenker, "Trickle: A self-regulating algorithm for code propagation and maintenance in wireless sensor networks," in *Proceedings of the 1st Conference on Symposium on Networked Systems Design and Implementation (NSDI '04)*. Berkeley, CA, USA: USENIX Association, 2004.
25. S. Gobriel, D. Mosse, and R. Melhem, "Mitigating the FloodingWaves Problem in Energy-Efficient Routing for MANETs," in *Proceedings of the 26th IEEE International Conference on Distributed Computing Systems (ICDCS '06)*, 2006.
26. Z. J. Haas, J. Y. Halpern, and L. Li, "Gossip-based ad hoc routing," in *Proceedings of the Twenty-First Annual Joint Conference of the IEEE Computer and Communications Societies (INFOCOM 2002)*, vol. 3, 2002, pp. 1707–1716.
27. ——, "Gossip-based ad hoc routing," *IEEE/ACM Transactions on Networking*, vol. 14, no. 3, pp. 479–491, June 2006.
28. N. R. Kyasanur, R. R. Choudhury, and I. Gupta, "Smart gossip: An adaptive gossip-based broadcasting service for sensor networks," in *Proceedings of the 6th ACM International Symposium on Mobile Ad Hoc Networking and Computing (MobiHoc '06)*, 2006, pp. 187–191.
29. R. Bhandari and I. Gupta, "Prioritycast: Efficient and time-critical decision making in first responder ad-hoc networks," in *Proceedings of the 6th ACM International Symposium on Mobile Ad Hoc Networking and Computing (MobiHoc '06)*, 2006, pp. 158–173.
30. H. Johansen, A. Allavena, and R. van Renesse, "Fireflies: Scalable support for intrusion-tolerant network overlays," in *Proceedings of the 2006 ACM SIGOPS/EurosSys Symposium on Operating Systems Design and Implementation (OSDI '06)*. Berkeley, CA, USA: USENIX Association, 2006, pp. 13–23.
31. J. A. Patel, I. Gupta, and N. Contractor, "Jetstream: Achieving predictable gossip dissemination by leveraging social network principles," in *Proceedings of the Fifth IEEE International Symposium on Network Computing and Applications (NCA '06)*, 2006, pp. 32–39.
32. G. He, R. Zheng, I. Gupta, and L. Sha, "A framework for time indexing in sensor networks," *ACM Trans. Sen. Netw.*, vol. 1, no. 1, pp. 101–133, Aug. 2005.

