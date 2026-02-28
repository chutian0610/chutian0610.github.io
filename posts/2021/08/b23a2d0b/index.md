# JVM性能分析工具-Async Profiler

很多 JVM CPU Profiler(例如VisualVM,NetBean Profiler,YourKit 和 JProfiler等)都提供了CPU分析器。一般CPU Profiling功能有两种实现方式: Sampling和Instrumentation。

- Sampling方式基于无侵入的额外线程对所有线程的调用栈快照进行固定频率抽样，它的性能开销很低。但由于它基于“采样”的模式，以及JVM固有的只能在安全点(SafePoint)进行采样的“缺陷”，会导致统计结果存在一定的偏差。核心原理如下：
  - 引入Profiler依赖，或直接利用Agent技术注入目标JVM进程并启动Profiler。
  - 启动一个采样定时器，以固定的采样频率每隔一段时间（毫秒级）对所有线程的调用栈进行Dump。
  - 汇总并统计每次调用栈的Dump结果，在一定时间内采到足够的样本后，导出统计结果，内容是每个方法被采样到的次数及方法的调用关系。
- Instrumentation则是利用Instrument API，对所有必要的Class进行字节码增强，在进入每个方法前进行埋点，方法执行结束后统计本次方法执行耗时，最终进行汇总。Instrumentation方式对几乎所有方法添加了额外的AOP逻辑，这会导致对线上服务造成巨额的性能影响，但其优势是：绝对精准的方法调用次数、调用时间统计。

Sampling由于低开销的特性，更适合用在CPU密集型的应用中，以及不可接受大量性能开销的线上服务中。而Instrumentation则更适合用在I/O密集的应用中、对性能开销不敏感以及确实需要精确统计的场景中。上面介绍的CPU Profiler更多的是基于Sampling来实现。

<!--more-->

## SafePoint Bias问题

通常通过Agent技术在JVM 进程中启动 Profiler进行Sampling。通常有两种方案: 

- 通过 Java Agent定时调用JMX的`threadMXBean.dumpAllThreads()`方法来导出所有线程的StackTrace。
- 通过JVMTI接口，使用原生C API对JVM进行操作(GetStackTrace)。

注意，基于Sampling的CPU Profiler通过采集程序在不同时间点的调用栈样本来近似地推算出热点方法，因此，从理论上来讲Sampling CPU Profiler必须遵循以下两个原则：

- 样本必须足够多。
- 程序中所有正在运行的代码点都必须以相同的概率被Profiler采样。

而对于JMX方式，因为我们只能采集到位于安全点时刻的调用栈快照(JVM 中 Thread Dump 必须进入安全点)，意味着某些代码可能永远没有机会被采样，即使它真实耗费了大量的CPU执行时间，这种现象被称为“SafePoint Bias”。

JVMTI的GetStackTrace()函数并不需要在Caller的安全点执行，但当调用GetStackTrace()获取其他线程的调用栈时，必须等待，直到目标线程进入安全点；而且，GetStackTrace()仅能通过单独的线程同步定时调用，不能在UNIX信号处理器的Handler中被异步调用。综合来说，GetStackTrace()存在与JMX一样的SafePoint Bias。

### AsyncGetCallTrace

如果一个函数可以获取当前线程的调用栈且不受安全点干扰，另外它还支持在UNIX信号处理器中被异步调用，那么我们只需注册一个UNIX信号处理器，在Handler中调用该函数获取当前线程的调用栈即可。由于UNIX信号会被发送给进程的随机一线程进行处理(假设操作系统将在线程之间公平地分配信号)，因此最终信号会均匀分布在所有线程上，也就均匀获取了所有线程的调用栈样本。

而OracleJDK/OpenJDK内部刚好提供了这么一个函数——AsyncGetCallTrace。

## Async Profiler

[Async Profiler](https://github.com/async-profiler/async-profiler/blob/master/docs/GettingStarted.md)是用于Java的低开销采样分析器，这不会受到安全点偏差问题的影响，针对HotSpot的特定API来收集堆栈信息并跟踪内存分配(适用于 OpenJdk 和其他基于Hotspot 的 JVM 运行时)。使用 async-profiler 可以做下面几个方面的分析。

- CPU cycles
- Hardware and Software performance counters like cache misses, branch misses, page faults, context switches etc.
- Allocations in Java Heap
- Contented lock attempts, including both Java object monitors and ReentrantLocks

```sh
## 如果目标文件名以.html结尾，则会自动选择Flame Graph输出格式
$ ./profiler.sh -d 30 -e cpu -f profile.html [pid]
$ ./profiler.sh -d 20 -e alloc -f profile.html [pid]
```

> 为了观测内联方法，最好添加 `-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints` JVM flag。

一些服务中内置了Async Profiler，并以http 方式提供profile结果，例如[Hive](https://github.com/apache/hive/blob/master/common/src/java/org/apache/hive/http/ProfileServlet.java)。

### CPU 模式

一般的方法是接收由perf_events生成的调用堆栈，并将它们与由AsyncGetCallTrace生成的调用堆栈进行匹配，以生成Java和本机代码的准确配置文件。此外，在AsyncGetCallTrace失败的某些极端情况下，PDRc-profiler提供了一个解决方案来恢复堆栈跟踪。

与直接将perf_events与将地址转换为Java方法名称的Java代理一起使用相比，这种方法具有以下优点：

- 不需要-XX:+PreserveFramePointer，这会引入有时可能高达10%的性能开销。
- 不需要启动带有代理的JVM来将Java代码地址转换为方法名。
- 显示解释器帧。
- 不生成大型中间文件（perf.data），以便在用户空间脚本中进行进一步处理。

如果要解析libjvm中的帧，则需要安装调试符号。

### 安装调试符号

在JDK 11之前，分配分析器需要HotSpot调试符号。一些OpenJDK发行版已经将它们嵌入到libjvm.so中，其他OpenJDK版本通常在单独的包中提供调试符号。例如，要在Debian / Ubuntu上安装OpenJDK调试符号:

```sh
## apt install openjdk-xx-dbg
$ apt install openjdk-17-dbg
```

在CentOS、RHEL和其他一些基于RPM的发行版上，这可以通过[debuginfo-install](http://man7.org/linux/man-pages/man1/debuginfo-install.1.html)实用程序来完成：

```sh
## debuginfo-install java-1.8.0-openjdk
```

可以使用gdb工具来验证是否为libjvm库正确安装了调试符号。例如，在Linux上：

```sh
$ gdb $JAVA_HOME/lib/server/libjvm.so -ex 'info address UseG1GC'
```

此命令的输出将包含Symbol "UseG1GC" is at 0xxxxx或No symbol "UseG1GC" in current context。

## 参考

- [1] [Async Profiler](https://github.com/async-profiler/async-profiler)
- [2] [Why (Most) Sampling Java Profilers Are Fucking Terrible](https://psy-lob-saw.blogspot.com/2016/02/why-most-sampling-java-profilers-are.html)
- [3] [The Pros and Cons of AsyncGetCallTrace Profilers](https://psy-lob-saw.blogspot.com/2016/06/the-pros-and-cons-of-agct.html)

