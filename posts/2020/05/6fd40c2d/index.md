# MapReduce面向大型集群的简化数据处理

本篇是论文[MapReduce: Simplified Data Processing on Large Clusters](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)的中文简单翻译

<!--more-->

什么是MapReduce？

**MapReduce既是一种编程模型，也是一种用于处理和产生大数据集的实现**。用户使用一个特定的map程序去处理key/value对，并产生中间key/value对的集合，以及一个特定的reduce程序去合并有着相同key的所有中间key/value对。本文指出，许多实际的任务都可以用这种模型来表示。

**用这种函数式风格写出的程序自动就拥有了在一个大的机器集群上并行执行的能力**。运行时系统会负责细节：输入数据分区，在一组机器上执行调度程序，处理机器错误，以及管理所需的机器间内部通信。这允许不具备任何并行和分布式系统经验的程序员也能轻松地利用一个大型分布式系统的资源。



## MapReduce 处理的工程问题

大多数的大数据集计算过程在概念上都很直接。但输入数据量通常都很大，因此计算过程需要分布到数百或数千台机器上进行，才能保证过程在一个合理时间内结束。而为了处理计算并行化、数据分发和错误处理等问题而引入大量复杂的代码则令原本简单的计算过程变的晦涩难懂。

针对这个问题，我们需要设计了一种新的抽象，允许我们表达出原本简单的计算过程，且将涉及并行、容错性、数据分发和负载均衡的凌乱细节隐藏在一个函数库中。

## 编程模型

计算过程：
    
* 输入一组 key/value对
* 生成输出一组 key/value对

MapReduce库的使用者使用两个函数来表示这个过程：map和reduce。

* map由使用者编写,使用一个输入 key/value对,生成一组中间 key/value对。MapReduce库将有着相同中间key I的中间value都组合在一起，在传递给reduce函数。
* reduce也由使用者编写，它接受一个中间key I和一组与I对应的value。它将这些value合并成一个可能更小的value集合。通常每个reduce只产生0或1个输出value。中间value是通过一个迭代器提供给reduce函数的。

下面来举个例子,对于统计文档中单词出现次数的问题。可以使用下面的伪码来解决。

```
// key : document name
// value : document content
map(String key,String value):
  for each word in value:
    EmitIntermediate(w,1)

// key: a word
// value: a list of counts
reduce(String key,Iterator values):
  int result = 0;
  for earch v in values:
    result += parseInt(v)
  emit(result)
```

## 真实实现

MapReduce可以有许多种不同的实现，这依赖于工作环境。我们主要考虑大型联网机器集群。

在google的论文里有环境的描述:

* 服务器X86双核处理器，内存2-4G，运行Linux操作系统。 
* 初级的网络硬件, 100M/s或1G/s.
* 集群有成百上千的服务器,机器异常很常见。
* 存储是直接挂载到服务器的便宜磁盘。使用GFS(内部分布式文件系统)管理。
* 用户将任务提交到调度系统。每个job包含了一系列task,被调度器映射到物理集群。

### 执行过程概述

通过自动将输入数据切分为M块，map调用分布在多台机器上进行。输入划分可以在不同的机器上并行执行。reduce调用是通过一个划分函数（例如hash(key) mod R)将中间key空间划分为R块来分布运行。划分的块数R和划分函数都由用户指定。

![](map-reduce.webp)

当用户程序调用MapReduce函数时，会发生下面一系列动作:

* 用户程序中的MapReduce库首先将输入文件切分为M块，每块的大小从16MB到64MB（用户可通过一个可选参数控制此大小）。然后MapReduce库会在一个集群的若干台机器上启动程序的多个副本。
* 程序的各个副本中有一个是特殊的——主节点，其它的则是工作节点。主节点将M个map任务和R个reduce任务分配给空闲的工作节点，每个节点一项任务。
* 被分配map任务的工作节点读取对应的输入区块内容。它从输入数据中解析出key/value对，然后将每个对传递给用户定义的map函数。由map函数产生的中间key/value对都缓存在内存中。
* 缓存的数据对会被周期性的由划分函数分成R块，并写入本地磁盘中。这些缓存对在本地磁盘中的位置会被传回给主节点，主节点负责将这些位置再传给reduce工作节点。
* 当一个reduce工作节点得到了主节点的这些位置通知后，它使用RPC调用去读map工作节点的本地磁盘中的缓存数据。当reduce工作节点读取完了所有的中间数据，它会将这些数据按中间key排序，这样相同key的数据就被排列在一起了。同一个reduce任务经常会分到有着不同key的数据，因此这个排序很有必要。如果中间数据数量过多，不能全部载入内存，则会使用**外部排序**。
* reduce工作节点遍历排序好的中间数据，并将遇到的每个中间key和与它关联的一组中间value传递给用户的reduce函数。reduce函数的输出会写到由reduce划分过程划分出来的最终输出文件的末尾。
* 当所有的map和reduce任务都完成后，主节点唤醒用户程序。此时，用户程序中的MapReduce调用返回到用户代码中。

成功完成后，MapReduce执行的输出都在R个输出文件中（每个reduce任务产生一个，文件名由用户指定）。通常用户不需要合并这R个输出文件——他们经常会把这些文件当作另一个MapReduce调用的输入，或是用于另一个可以处理分成多个文件输入的分布式应用。

### 主数据节点结构

主节点存储了一些数据结构。对于每个map和reduce任务,存储了task的状态(空闲,执行中或是完成)和worker机器的身份。主节点还承担了管道的功能，将map tasks的中间结果文件块位置传递给reduce tasks.

### 容错性

因为MapReduce 是被设计成在成百上千台机器上执行大规模数据操作，所以必须能够优雅的处理机器故障。

#### worker 异常

主节点和worker之间有周期心跳。如果在一定时间内没有从worker接受到心跳，master将会标记worker为失败。在这台失败worker机器上的完成的map tasks时，状态会被重置为初始状态(空闲),然后等待其它机器上的任务的调度。同样，这台worker上的任何执行中的map task或是reduce task会被重置为空闲状态，并等待重新调度。

为什么完成的map task还需要再执行？当map task完成时，它们的输出是被存储在本地机器的磁盘上。当本地机器异常时，中间结果是不可达的。而完成的reduce task的结果已经被存储到分布式文件系统上，所以无需再次执行。

当一个map task首先被worker A 执行，然后被worker B 执行(因为 worker A 失败了),所有的在执行reduce task的workers都会被通知到这次重新执行。任何还没有从worker A读完数据的reduce task会改成从worker B读取。

### master 异常

master会周期性的保存主节点的数据(checkpoint)，如果主节点异常了，一个新的master copy会从最新的checkpoint启动。但是，如果现在只有一个主节点，当主节点异常时，会直接终断MapReduce计算。客户端可以得到这个状态，并重试MapReduce操作。

### 失败情况下的语义

如果用户提供的Map和Reduce操作是关于输入值的确定性函数，那么在整个程序经过没有出现故障的顺序执行之后，我们分布式的实现将会产生同样的输出。

> 此处的确定性可以理解为纯函数，结果仅依赖于参数。

我们依赖Map task和Reduce task原子性地提交输出来实现上述特性。每一个正在执行的task都会将它的输出写到一个私有的临时文件中。一个Reduce task产生一个这样的文件，而一个Map task产生R个这样的文件（每个Reduce work一个）。当一个Map task完成的时候，worker就会给master发送一个信息，，其中包含了R个临时文件的名字。如果master收到了一个来自于已经完成了的Map task的完成信息，那么它就将它自动忽略。否则，将R个文件的名称记录到一个master数据结构中。

当一个Reduce task完成的时候，Reduce worker会自动将临时输出文件命名为最终输出文件。如果同一个Reduce task在多台机器上运行，那么多个重命名操作产生的最终输出文件名将会产生冲突。对此，我们依赖底层文件系统提供的原子重命名操作来保证最终文件系统中的数据来自一个Reduce task。

大多数的Map和Reduce操作都是确定性的，事实上，我们的语义等同于顺序执行。因此这让程序员非常容易地能够解释他们程序的行为。当Map和Reduce操作是非确定性的时候，我们提供较弱，但仍然合理的语义。在非确定性的操作中，对于一个特定的Reduce task R1的输出是和非确定性程序顺序执行产生R1产生的输出是相同的。然而，对于另一个Reduce task R2，它的输出对应于非确定性程序另一个顺序执行的结果。(纯函数，函数式化的重要性)

下面考虑Map task M和Reduce task R1和R2。让e(Ri)表示Ri的执行结果。更弱的语义意味着，e(R1)可能从M的一次执行结果中读取输入，而e(R2)可能从M的另一次执行中读取输入。

### 局部性(数据局部优化)

在我们的计算环境中，网络带宽是相对宝贵的。我们通过利用输入数据(由GFS管理)是存储在机器的本地磁盘上的这一事实来节省网络带宽。GFS将每个文件划分为64MB大小的块，每个块的几个副本存储在不同的机器上。MapReduce master充分考虑输入文件的位置信息，尽量将一个map task调度到包含相应的输入数据副本的那个机器上。如果不行，就尝试将map task调度到该task的输入数据附近的那些机器(比如让worker所在的机器与包含该数据的机器在同一个网络交换机上)。当在一个集群上运行一个具有很多worker的大型MapReduce操作时，大部分的输入数据都是从本地读取的，很少消耗网络带宽。

### 任务粒度

我们将map阶段划分为M个片段，将reduce阶段划分为R个片段。理想情况下，M和R都应当远远大于运行worker的机器数目。让每个worker执行很多不同的task可以提高动态负载平衡，也能加速worker失败后的恢复过程(它已经完成的map tasks可以传给所有其他机器。)

在我们的实现中M和R到底可以多大，有一些实际的限制。因为master必须进行O(M+R)个调度决定以及在内存中保存`O(M*R)`个状态(即每个map task的R个输出文件的位置信息，总共M个task，所以是`M*R`)。但是关于内存使用的常数因子是很小的：`O(M*R)`个状态大概由map/reduce task对个数的1byte数据组成。

另外，R通常会被用户限制，因为每个reduce task的输出在不同的输出文件中。在实际中，我们通常这样选择M：使每个独立task输入数据限制在16MB到64MB之间(这样上面所说的本地化优化是最有效的)。我们让R大概是我们将要使用的worker机器的几倍。我们通常这样执行MapReduce操作，在有2000个worker机器时，让M = 20000,R = 5000。

### 备份任务

一个影响MapReduce操作整体执行时间的最常见的因素是”掉队者”(花费相当长时间去完成MapReduce操作中最后剩下的极少数的那几个task的那台机器)。有很多原因可以导致掉队者的出现。比如：具有一块坏硬盘的机器可能会经历频繁的可修正错误而使得IO性能从30MB/s降低到1MB/s。集群调度系统可能会将那些引发CPU 内存 本地磁盘或者网络带宽等资源竞争的task调度到同一台机器上。我们最近见过的一个错误是由于机器初始化代码中的一个bug引起的处理器缓冲失灵，使得受影响的机器上的计算性能降低了一百倍。

我们有一个可以缓解这种掉队者问题的通用机制。当MapReduce操作接近尾声的时候，master会备份那些还在执行中的task。只要该task的主本或者其中的一个副本完成了，我们就认为它完成了。通过采用这种机制，我们只使计算资源的利用率增长了仅仅几个百分点，但是明显地降低了完成整个MapReduce操作所需的时间。比如，在5.3节描述的排序例子中，如果不启用这个机制，整个完成时间将会增长53%。

## 概念

尽管通过简单书写map和reduce函数提供的基本功能对于我们大部分的应用来说足够了，我们也发现了其中的一些扩展也很有用。

### 分区函数

MapReduce用户指定他们期望的reduce task(也可以说输出文件)的数目R。任务产生的数据通过在中间结果的key上使用一个分区函数被划分开。系统提供一个使用hash的默认的分区函数(比如 “hash(key) mod R”)。然而在某些情况下，使用关于key的其他函数进行划分更有用。比如有时候输入是URL，我们希望来自相同host的输入可以存放在相同的输出文件上。为了支持这种情况，MapReduce库的用户必须提供一个特殊的分区函数。比如使用”hash(Hostname(urlkey)) mod R”作为划分函数，就可以让来自相同host的所有URL落在同一个输出文件上。

### 有序保证

我们保证在一个给定的分区内，作为中间结果的key/value对是按照key值的增序进行处理的。这种有序化保证可以让每个划分的输出文件也是有序的。而这在输出文件格式需要支持按照key的有效的随机查找时非常有用，或者输出用户也会发现这些排序后的数据排序使用起来很方便。

### 合并函数

某些情况下，map task产生的中间结果有很多具有相同key的重复值，而且用户指定的reduce函数又满足交换率和结合率。一个很好的例子就是2.1节里描述的wordcount的例子。因为单词频率的分布倾向于遵循Zipf(齐夫)分布，每个map task将会产生成百上千个相同的记录比如`<the,1>`这样的。而所有的这些又将会通过网络传递给一个reduce task，然后通过reduce函数将它们累加起来。我们允许用户描述一个combiner函数，在数据通过网络发送之前对它们进行部分的归并。

Combiner函数在每个执行map task的机器上这些。通常用来实现combiner和reduce函数的代码是相同的。唯一的不同在MapReduce库如何处理它们的输出。一个reduce函数的输出将会被写到最终的输出文件，而combiner函数的输出会被写到一个将要发送给reduce task的中间结果文件中。

### 输入和输出类型

MapReduce库提供了几种不同格式的输入数据支持。比如`text`输入模式：将每一行看做一个`key/value`对，key是该行的offset，value是该行的内容。另一个支持的模式是一个根据key排序的`key/value`对的序列。每个输入类型知道如何将它们自己通过有意义的边界划分，然后交给独立的map task处理(比如text模式，会保证划分只会发生在行边界上)。用户可以通过提供一个reader接口的实现来支持新的输入类型。对于大多数用户来说，仅仅使用那些预定义的输入类型就够用了。

一个reader并不是必须从文件读数据。比如可以简单的定义一个从数据库或者是内存中的数据结构中读记录的reader。

与之类似，我们也提供一组输出类型用于控制输出数据格式，同时用户也很容易添加对于新的输出类型的支持。

### 副作用

MapReduce的用户发现某些情况下，在map和reduce操作中顺便产生一个文件作为额外的输出会很方便。这些的副作用是原子性或幂等性依赖于应用程序编写者的实现。通常应用程序编写者会写一个temp文件，一旦它已经生成完毕再将它原子性的重命名。

我们并不为单个task产生的多个输出文件提供原子性的两阶段提交。因此那些具有跨文件一致性需求的产生多个输出文件的task应当是确定性的。这个限制在实际中还没有引起什么问题。

### 跳过坏记录

有时候用户代码中的一些bug会导致Map或者Reduce函数在处理某个特定记录时一定会crash。这样的bug会使得MapReduce操作无法车成功完成。通常的处理方法是修复这个bug，但是有时候这样做显得并不灵活。因为bug可能是存在于第三方的库里，但是源代码是不可用的。而且有时候忽略一些记录是可以接受的，比如在一个大数据集上进行统计分析时。我们提供了一种可选的执行模式，在该模式下，MapReduce库会检测那些记录引发了该crash，然后跳过它们继续前进。

每个worker进程安装了一个信号处理器捕获那些段异常和总线错误。在调用用户Map或者Reduce操作之前，MapReduce库使用一个全局变量存储该参数的序列号。如果用户代码产生了一个信号，信号处理器就会发送一个包含该序列号的”last gasp”的UDP包给master。当master发现在同一记录上发生了不止一次失败后，当它在相应的Map或者Reduce task重新执行时，它就会指出该记录应该被跳过。

### 本地化执行

在Map和Reduce函数上进行调试会变得很有技巧，因为实际的计算发生在分布式系统上，通常是几百台机器，而且工作分配是有master动态决定的。为了降低debug，profile的难度以及进行小规模测试，我们开发了一个MapReduce库的变更实现，让MapReduce操作的所有工作在本地计算机上可以串行执行。用户可以控制将计算放在特殊的map task上执行。用户通过使用一个特殊的flag调用它们的程序，然后就可以简单的使用他们的调试和测试工具(比如gdb)。

### 状态信息

Master运行一个内部的http服务器，然后发布一些用户可以查看的状态页面。这些状态页面展示了计算的进度，比如已经有多少任务完成，多少还在执行中，输入字节数，中间数据的字节数，输出的字节数，处理速率等等。该页面也会包含指向每个task的标准错误和标准输出文件的链接。用户可以使用这些数据来预测该计算还要花费多少时间，是否还需要为该计算添加更多的资源。计算远远低于预取时，这些页面也可以用来发现这些情况。

另外，更高级别的状态页会展示那些worker失败了，当它们失败时在处理哪些map和reduce task。在诊断用户代码中的bug时，这些信息都是很有用的。

### 计数器(任务进度监控)

MapReduce库提供了一些计数器设施来计算各种事件的发生。比如用户代码可能想计算处理的单词的总数，或者被索引的德语文档的个数等等。 

为了使用这些设施，用户代码需要创建一个命名计数器对象然后在Map 和/或 Reduce函数中累加这些计数器。比如：

```C
Counter* uppercase;
uppercase = GetCounter("uppercase");

map(String name, String contents):
  for each word w in contents:
    if (IsCapitalized(w)):
      uppercase->Increment();
    EmitIntermediate(w, "1");
```

来自独立worker机器的计数器的值将会周期性的发送给master(通过对master的ping的响应捎带过去)。Master将那些成功的map和reduce task的计数器值聚集，当MapReduce操作结束后，将它们返回给用户代码。当前的计数器值也会在master的状态页面上显示出来，这样用户就可以看到计算的实时进展。在计算计数器值时，master会忽略掉那些重复执行的相同map或者reduce task的值，以避免重复计数。(重复执行可能是由于备份任务的使用或者是task失败引发的重新执行而引起的。)

一些计数器值是由MapReduce库自动维护的，比如已经处理的输入key/vaule 对的个数，已经产生的输出key/vaule 对的个数。

用户发现计数器设施对于MapReduce操作的行为的完整性检查是非常有用的。比如，在某些MapReduce操作中，用户代码可能想确定已产生的输出对的数目是否刚好等于已处理的输入对数目，或者已经被处理的德语文档在已处理的文档中是否在一个合理的比例上。 

> 后续(性能，经验部分)见参考资料论文pdf

## 参考

<div id="refer-anchor-1"></div>

- [1] Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau,David E. Culler, Joseph M. Hellerstein, and David A. Patterson. High-performance sorting on networks of workstations. In Proceedings of the 1997 ACM SIGMOD International Conference on Management of Data, Tucson,Arizona, May 1997.

<div id="refer-anchor-2"></div>

- [2] Remzi H. Arpaci-Dusseau, Eric Anderson, Noah Treuhaft, David E. Culler, Joseph M. Hellerstein, David Patterson, and Kathy Yelick. Cluster I/O with River: Making the fast case common. In Proceedings of the Sixth Workshop on Input/Output in Parallel and Distributed Systems (IOPADS ’99), pages 10–22, Atlanta, Georgia, May 1999.

<div id="refer-anchor-3"></div>

- [3] Arash Baratloo, Mehmet Karaul, Zvi Kedem, and Peter Wyckoff. Charlotte: Metacomputing on the web. In Proceedings of the 9th International Conference on Parallel and Distributed Computing Systems, 1996.

<div id="refer-anchor-4"></div>

- [4] Luiz A. Barroso, Jeffrey Dean, and Urs H¨olzle. Web search for a planet: The Google cluster architecture. IEEE Micro, 23(2):22–28, April 2003.

<div id="refer-anchor-5"></div>

- [5] John Bent, Douglas Thain, Andrea C.Arpaci-Dusseau, Remzi H. Arpaci-Dusseau, and Miron Livny. Explicit control in a batch-aware distributed file system. In Proceedings of the 1st USENIX Symposium on Networked Systems Design and Implementation NSDI, March 2004.

<div id="refer-anchor-6"></div>

- [6] Guy E. Blelloch. Scans as primitive parallel operations. IEEE Transactions on Computers, C-38(11), November 1989.

<div id="refer-anchor-7"></div>

- [7] Armando Fox, Steven D. Gribble, Yatin Chawathe, Eric A. Brewer, and Paul Gauthier. Cluster-based scalable network services. In Proceedings of the 16th ACM Symposium on Operating System Principles, pages 78–91, Saint-Malo, France, 1997.

<div id="refer-anchor-8"></div>

- [8] Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung. The Google file system. In 19th Symposium on Operating Systems Principles, pages 29–43, Lake George, New York, 2003. 

<div id="refer-anchor-9"></div>

- [9] S. Gorlatch. Systematic efficient parallelization of scan and other list homomorphisms. In L. Bouge, P. Fraigniaud, A. Mignotte, and Y. Robert, editors, Euro-Par’96. Parallel Processing, Lecture Notes in Computer Science 1124, pages 401–408. Springer-Verlag, 1996.

<div id="refer-anchor-10"></div>

- [10] [Jim Gray. Sort benchmark home page.](http://research.microsoft.com/barc/SortBenchmark/)

<div id="refer-anchor-11"></div>

- [11] William Gropp, Ewing Lusk, and Anthony Skjellum. Using MPI: Portable Parallel Programming with the Message-Passing Interface. MIT Press, Cambridge, MA, 1999.

<div id="refer-anchor-12"></div>

- [12] L. Huston, R. Sukthankar, R. Wickremesinghe, M. Satyanarayanan, G. R. Ganger, E. Riedel, and A. Ailamaki. Diamond: A storage architecture for early discard in interactive search. In Proceedings of the 2004 USENIX File and Storage Technologies FAST Conference, April 2004.

<div id="refer-anchor-13"></div>

- [13] Richard E. Ladner and Michael J. Fischer. Parallel prefix computation. Journal of the ACM, 27(4):831–838, 1980.

<div id="refer-anchor-14"></div>

- [14] Michael O. Rabin. Efficient dispersal of information for security, load balancing and fault tolerance. Journal of the ACM, 36(2):335–348, 1989.

<div id="refer-anchor-15"></div>

- [15] Erik Riedel, Christos Faloutsos, Garth A. Gibson, and David Nagle. Active disks for large-scale data processing. IEEE Computer, pages 68–74, June 2001.

<div id="refer-anchor-16"></div>

- [16] Douglas Thain, Todd Tannenbaum, and Miron Livny.Distributed computing in practice: The Condor experience. Concurrency and Computation: Practice and Experience, 2004.

<div id="refer-anchor-17"></div>

- [17] L. G. Valiant. A bridging model for parallel computation. Communications of the ACM, 33(8):103–111, 1997.

<div id="refer-anchor-18"></div>

- [18] [Jim Wyllie. Spsort: How to sort a terabyte quickly.](http://alme1.almaden.ibm.com/cs/spsort.pdf)

