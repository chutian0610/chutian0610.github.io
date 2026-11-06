# Java OOM Error

OOM(OutOfMemoryError)是java工程师都会了解的一种异常，实质上，OOM并不是只有一种，一共有9种不同类型的OOM：

* java.lang.OutOfMemoryError: Java heap space
* java.lang.OutOfMemoryError: GC Overhead limit exceeded
* java.lang.OutOfMemoryError: Permgen space
* java.lang.OutOfMemoryError: Metaspace
* java.lang.OutOfMemoryError: Unable to create new native thread
* java.lang.OutOfMemoryError: reason stack_trace_with_native_method
* java.lang.OutOfMemoryError: Requested array size exceeds VM limit
* java.lang.OutOfMemoryError: Kill process or sacrifice child
* java.lang.OutOfMemoryError: Direct buffer memory

不同的原因触发不同类型的OOM，每种OOM类型的解决方案也不同。

<!--more-->

## Java heap space

java应用程序使用的内存是有上限的，这在程序启动时会被指定。java的堆的大小可以通过指定JVM参数-Xmx来设置，如果没有明确指定, 则根据操作系统平台和物理内存的大小来确定。。当应用程序试图向堆中存入数据，但堆中空间不足时，会抛出`java.lang.OutOfMemoryError: Java heap space`。注意，此时的物理内存可能仍有很多，java只能使用预设置大小的内存空间。

### 产生的原因

1. 一般的情况是，大数据量的程序预设置的空间过小，无法分配内存,此时只要增加堆内存的大小, 程序就能正常运行。
2. 超出预期的访问量/数据量。 应用系统设计时,一般是有 “容量” 定义的, 用来处理一定量的数据/业务。 如果数据量突然飙升, 超过预期的阈值, 类似于时间坐标系中针尖形状的图谱, 那么在峰值所在的时间段, 程序很可能就会卡死、并触发`java.lang.OutOfMemoryError: Java heap space`错误。
3. 内存泄露(Memory leak). 这也是一种经常出现的情形。由于代码中的某些错误, 导致系统占用的内存越来越多. 如果某个方法/某段代码存在内存泄漏的, 每执行一次, 就会（有更多的垃圾对象）占用更多的内存. 随着运行时间的推移, 泄漏的对象耗光了堆中的所有内存, 那么 `java.lang.OutOfMemoryError: Java heap space`错误就被抛出了。

### 解决方案

1. 首先，应该尝试提高堆空间的大小(如果设置的最大堆空间，并不大)。例如`-Xmx4g`将堆设置为4g大小。
2. 在许多情况下，提供更多的Java堆空间并不能解决问题。例如，如果您的应用程序包含内存泄漏，则添加更多堆只会推迟`java.lang.OutOfMemoryError: Java heap space`异常的触发。此外，增加Java堆大小也会增加GC stop-the-world的时长，从而影响应用程序的吞吐量。此时需要确定代码的哪一部分负责分配最多的内存：
    * 哪些对象占据堆的大部分
    * 这些对象在源代码中分配的位置
3. 分析JVM dump(可以使用`-XX:+HeapDumpOnOutOfMemoryError`)

## GC Overhead limit exceeded

在Java程序中, 只需要关心内存分配就行。如果某块内存不再使用, 垃圾收集(Garbage Collection) 模块会自动执行清理。`java.lang.OutOfMemoryError: GC overhead limit exceeded`这种情况发生的原因是, 程序基本上耗尽了所有的可用内存, GC也清理不了。

### 产生的原因

JVM抛出`java.lang.OutOfMemoryError: GC overhead limit exceeded`错误就是发出了这样的信号: 执行垃圾收集的时间比例太大, 有效的运算量太小. 默认情况下, 如果GC花费的时间超过 98%, 并且GC回收的内存少于 2%, JVM就会抛出这个错误。

注意, java.lang.OutOfMemoryError: GC overhead limit exceeded 错误只在连续多次 GC 都只回收了不到2%的极端情况下才会抛出。假如不抛出 GC overhead limit 错误会发生什么情况呢? 那就是GC清理的这么点内存很快会再次填满, 迫使GC再次执行. 这样就形成恶性循环, CPU使用率一直是100%, 而GC却没有任何成果. 系统用户就会看到系统卡死 - 以前只需要几毫秒的操作, 现在需要好几分钟才能完成。

### 解决方案

* 一种表面上的解决方案是在启动时添加参数:`-XX:-UseGCOverheadLimit`,JVM 就不会抛出 `java.lang.OutOfMemoryError: GC overhead limit exceeded` 错误信息。这不能真正地解决问题，只能推迟一点 out of memory 错误发生的时间，到最后还得进行其他处理。指定这个选项, 会将原来的`java.lang.OutOfMemoryError: GC overhead limit exceeded`错误掩盖，变成更常见的`java.lang.OutOfMemoryError: Java heap space`错误消息。
* 有时候触发 GC overhead limit 错误的原因, 是因为分配给JVM的堆内存不足。这种情况下只需要增加堆内存大小即可。
* 分析JVM dump(`-XX:+HeapDumpOnOutOfMemoryError`)。

## Permgen space

永久代由JVM参数`-XX:MaxPermSize`设置。 如果没有明确指定, 则根据操作系统平台和物理内存的大小来确定。`java.lang.OutOfMemoryError: PermGen space`错误信息所表达的意思是: 永久代(Permanent Generation) 内存区域已满。

### 产生的原因

1. **在JDK1.7及之前的版本**, 永久代(permanent generation) 主要用于存储加载/缓存到内存中的 class 定义, 包括 class 的 名称(name), 字段(fields), 方法(methods)和字节码(method bytecode); 以及常量池(constant pool information); 对象数组(object arrays)/类型数组(type arrays)所关联的 class, 还有 JIT 编译器优化后的class信息等。很容易看出, PermGen 的使用量和JVM加载到内存中的 class 数量/大小有关。可以说 `java.lang.OutOfMemoryError: PermGen space`的主要原因, 是加载到内存中的 class 数量太多或体积太大。

我们可以使用 CGLIB 来模拟这个错误。

```java
/**
 * VM Option: -XX:PermSize=1M -XX:MaxPerSize=1M
 */
public class JavaMethodAreaOOM {
    static class OOMObject{

    }
    public static void main(String[] args){
        while(true){
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperClass(OOMObject.class);
            enhancer.setUserCache(false);
            enhancer.setCallback(new MethodInterceptor(){
                public Object intercept(Object obj,Method method,Object[] args,MethodProxy proxy)throws Throwable{
                    return proxy.invokeSuper(obj,args);
                }
                });
            enhancer.create();
        }
    }
}
```

2. 另一种常见的情况是在重新部署web应用时, 很可能会引起`java.lang.OutOfMemoryError: Permgen space`错误,理论上说，redeploy 时, Tomcat之类的容器会使用新的 classloader 来加载新的 class, 让垃圾收集器 将之前的 classloader (连同加载的class一起)清理掉,。但实际情况可能并不乐观, 很多第三方库, 以及某些受限的共享资源, 如 thread, JDBC驱动, 以及文件系统句柄(handles), 都会导致不能彻底卸载之前的 classloader. 那么在 redeploy 时, 之前的class仍然驻留在PermGen中, 每次重新部署都会产生几十MB，甚至上百MB的垃圾。

### 解决方案

1. 在程序启动时, 如果 PermGen 耗尽而产生 OutOfMemoryError 错误, 那很容易解决. 增加 PermGen 的大小, 让程序拥有更多的内存来加载 class 即可. 修改 -XX:MaxPermSize 启动参数, 类似下面这样:`-XX:MaxPermSize=512m`
2. 如果是redeploy 时产生的 OutOfMemoryError，我们可以进行堆转储分析，然后找出重复的类, 特别是类加载器(classloader)对应的 class. 你可能需要比对所有的 classloader, 来找出当前正在使用的那个。
3. 如果在运行的过程中发生 OutOfMemoryError, 首先需要确认 GC是否能从PermGen中卸载class。 官方的JVM在这方面是相当的保守(在加载class之后,就一直让其驻留在内存中,即使这个类不再被使用)。那么我们就需要允许JVM卸载class。使用下面的启动参数:`-XX:+CMSClassUnloadingEnabled`.启用以后, GC 将会清理 PermGen, 卸载无用的 class. 当然, 这个选项只有在设置 UseConcMarkSweepGC 时生效。 如果使用了 ParallelGC, 或者 Serial GC 时, 那么需要切换为CMS:`-XX:+UseConcMarkSweepGC`

## Metaspace

从Java 8开始,内存结构发生重大改变, 不再使用Permgen, 而是引入一个新的空间: Metaspace. Metaspace 的使用量与JVM加载到内存中的 class 数量/大小有关。可以说, java.lang.OutOfMemoryError: Metaspace 错误的主要原因, 是加载到内存中的 class 数量太多或者体积太大。

### 解决方案

* 如果抛出与 Metaspace 有关的 OutOfMemoryError , 第一解决方案是增加 Metaspace 的大小. 使用下面这样的启动参数:`-XX:MaxMetaspaceSize=512m`.
* 还有一种方案是直接去掉 Metaspace 的大小限制。 但需要注意, 不限制Metaspace内存的大小, 假若物理内存不足, 有可能会引起内存交换(swapping), 严重拖累系统性能。 此外,还可能造成native内存分配失败等问题。

## Unable to create new native thread

Java程序本质上是多线程的, 可以同时执行多项任务。JVM中的线程需要内存空间来执行自己的任务. 如果线程数量太多, 就会引入新的问题:`java.lang.OutOfMemoryError: Unable to create new native thread`。错误表达的意思是: 程序创建的线程数量已达到上限值。

### 产生的原因

JVM向操作系统申请创建新的 native thread(原生线程)时, 就有可能会碰到 java.lang.OutOfMemoryError: Unable to create new native thread 错误. 如果底层操作系统创建新的 native thread 失败, JVM就会抛出相应的OutOfMemoryError. 原生线程的数量受到具体环境的限制, 但总体来说, 导致 `java.lang.OutOfMemoryError: Unable to create new native thread` 错误的场景大多经历以下这些阶段:

1. Java程序向JVM请求创建一个新的Java线程;
2. JVM本地代码(native code)代理该请求, 尝试创建一个操作系统级别的 native thread(原生线程);
3. 操作系统尝试创建一个新的native thread, 需要同时分配一些内存给该线程;
4. 如果操作系统的虚拟内存已耗尽, 或者是受到32位进程的地址空间限制(约2-4GB), OS就会拒绝本地内存分配;
5. JVM抛出 java.lang.OutOfMemoryError: Unable to create new native thread 错误。

```java
// 在一个死循环中创建并启动很多新线程。代码执行后, 很快就会达到操作系统的限制, 报出 java.lang.OutOfMemoryError: Unable to create new native thread 错误。
public class MaxThread {
    private static Object s = new Object();
    private static int count = 0;
    public static void main(String[] argv){
        for(;;){
            new Thread(new Runnable(){
                    public void run(){
                        synchronized(s){
                            count += 1;
                            System.err.println("New thread #"+count);
                        }
                        for(;;){
                            try {
                                Thread.sleep(1000);
                            } catch (Exception e){
                                System.err.println(e);
                            }
                        }
                    }
                }).start();
        }
    }
}
```

### 解决方案

1. 可以修改系统限制来避开 Unable to create new native thread 问题. 假如JVM受到用户空间(user space)文件数量的限制, 像下面这样,就应该想办法增大这个值:

```sh
//在这种情况下，很有可能产生 java.lang.OutOfMemoryError:
// Unable to create new native thread 异常。

// 用户可创建线程总数
$ ulimit -u
1024
// 进程当前运行的线程数
$ pstree -p pid | wc -l
1000
```

2. 更多的情况, 触发创建 native 线程时的OutOfMemoryError, 表明编程存在BUG. 比如, 程序创建了成千上万的线程, 很可能就是某些地方出大问题了。此时可以使用thread dump 来分析问题。
3. 还有一种情况是，JVM上设置堆栈大小有问题，可以通过`-Xss64k`,来设置堆栈大小为64k(每个线程允许的最小堆栈空间量)。但是要注意的是，堆栈过小，可能会导致`StackOverflowError`，调整这个值，需要做个平衡。

## Out of swap space？

JVM启动参数指定了最大内存限制。如 -Xmx 以及相关的其他启动参数. 假若JVM使用的内存总量超过可用的物理内存, 操作系统就会用到虚拟内存。错误信息`java.lang.OutOfMemoryError: Out of swap space?`表明, 交换空间(swap space,虚拟内存) 不足,是由于物理内存和交换空间都不足所以导致内存分配失败。

### 原因分析

如果 native heap 内存耗尽, 内存分配时, JVM 就会抛出 java.lang.OutOfmemoryError: Out of swap space? 错误消息, 这个消息告诉用户, 请求分配内存的操作失败了。

Java进程使用了虚拟内存才会发生这个错误。 对 Java的垃圾收集 来说这是很难应付的场景。即使现代的 GC算法 很先进, 但虚拟内存交换引发的系统延迟, 会让 GC暂停时间 膨胀到令人难以容忍的地步。

通常是操作系统层面的原因导致 java.lang.OutOfMemoryError: Out of swap space? 问题, 例如:

* 操作系统的交换空间太小。
* 机器上的某个进程耗光了所有的内存资源。
* 也可能是应用程序的本地内存泄漏(native leak)引起的, 例如, 某个程序/库不断地申请本地内存,却不进行释放。

### 解决方案

第一种, 也是最简单的方法, 增加虚拟内存(swap space) 的大小. 各操作系统的设置方法不太一样, 比如Linux,可以使用下面的命令设置:

```sh
// 创建了一个大小为 640MB 的 swapfile(交换文件) 并启用该文件。
swapoff -a
dd if=/dev/zero of=swapfile bs=1024 count=655360
mkswap swapfile
swapon swapfile
```

因为垃圾收集器需要清理整个内存空间, 所以虚拟内存对 Java GC 来说是难以忍受的。存在内存交换时, 执行 垃圾收集 的 暂停时间 会增加上百倍,甚至更多, 所以最好不要增加虚拟内存。

如果程序允许环境还受到 “坏邻居效应” 的干扰, JVM还要和其他程序竞争计算资源, 那么提高性能的办法就是单独部署到专用的服务器/虚拟机中。也可以进行程序优化, 降低内存空间的使用量, 通过堆转储分析器可以检测到哪些方法/代码分配了大量的内存。

## Requested array size exceeds VM limit

Java平台限制了数组的最大长度。各个版本的具体限制可能稍有不同, 但范围都在 1 ~ 21亿 之间。如果程序抛出 java.lang.OutOfMemoryError: Requested array size exceeds VM limit 错误, 就说明想要创建的数组长度超过限制。

### 原因分析

这个错误是由JVM中的本地代码抛出的. 在真正为数组分配内存之前, JVM会执行一项检查: 要分配的数据结构在该平台是否可以寻址(addressable). 当然, 这个错误比你所想的还要少见得多。

一般很少看到这个错误, 因为Java使用 int 类型作为数组的下标(index, 索引)。在Java中, int类型的最大值为 2^31 – 1 = 2,147,483,647。大多数平台的限制都约等于这个值 —— 例如在 64位的 MACBOOK Pro 上, Java 1.7 平台可以分配长度为 2,147,483,645(Integer.MAX_VALUE-2 长度)的数组。

再增加一点点长度, 变成 Integer.MAX_VALUE-1 时, 就会抛出我们所熟知的 OutOfMemoryError:`Exception in thread "main" java.lang.OutOfMemoryError: Requested array size exceeds VM limit`,在有的平台上, 这个最大限制可能还会更小一些, 例如在32位Linux, OpenJDK 6 上面, 数组长度大约在 11亿左右(约2^30) 就会抛出 “java.lang.OutOfMemoryError: Requested array size exceeds VM limit“ 错误。要找出具体的限制值, 可以执行一个小小的测试用例, 具体示例参见下文。

```java
 for (int i = 3; i >= 0; i--) {
  try {
    int[] arr = new int[Integer.MAX_VALUE-i];
    System.out.format("Successfully initialized an array with %,d elements.\n", Integer.MAX_VALUE-i);
  } catch (Throwable t) {
    t.printStackTrace();
  }
}
```

其中,for循环迭代4次, 每次都去初始化一个 int 数组, 长度从 Integer.MAX_VALUE-3 开始递增, 到 Integer.MAX_VALUE 为止. 

### 解决方案

发生 java.lang.OutOfMemoryError: Requested array size exceeds VM limit 错误的原因可能是:

* 数组太大, 最终长度超过平台限制值, 但小于 Integer.MAX_INT
* 为了测试系统限制, 故意分配长度大于 2^31-1 的数组。

第一种情况, 需要检查业务代码, 确认是否真的需要那么大的数组。如果可以减小数组长度, 那就万事大吉. 如果不行，可能需要把数据拆分为多个块, 然后根据需要按批次加载。

如果是第二种情况, 请记住, Java 数组用 int 值作为索引。所以数组元素不能超过 2^31-1 个. 实际上, 代码在编译阶段就会报错,提示信息为 “error: integer number too large”。

如果确实需要处理超大数据集, 那就要考虑调整解决方案了. 例如拆分成多个小块,按批次加载; 或者放弃使用标准库,而是自己处理数据结构,比如使用 sun.misc.Unsafe 类, 通过Unsafe工具类可以像C语言一样直接分配内存。

## Kill process or sacrifice child

操作系统(operating system)构建在进程(process)的基础上. 进程由内核作业(kernel jobs)进行调度和维护, 其中有一个内核作业称为 “Out of memory killer(OOM终结者)”, 与本节所讲的 OutOfMemoryError 有关。

Out of memory killer 在可用内存极低的情况下会杀死某些进程。只要达到触发条件就会激活, 选中某个进程并杀掉。 通常采用启发式算法, 对所有进程计算评分(heuristics scoring), 得分最低的进程将被 kill 掉。因此 Out of memory: Kill process or sacrifice child 和前面所讲的 OutOfMemoryError 都不同, 因为它既不由JVM触发,也不由JVM代理, 而是系统内核内置的一种安全保护措施(日志在系统日志中)。

如果可用内存(含swap)不足, 就有可能会影响系统稳定, 这时候 Out of memory killer 就会设法找出流氓进程并杀死他, 也就是引起 Out of memory: kill process or sacrifice child 错误。

### 原因分析

默认情况下, Linux kernels(内核)允许进程申请的量超过系统可用内存. 这是因为,在大多数情况下, 很多进程申请了很多内存, 但实际使用的量并没有那么多. 这样的话,可能会有一个问题, 假若某些程序占用了大量的系统内存, 那么可用内存量就会极小, 导致没有内存页面(pages)可以分配给需要的进程。可能这时候会出现极端情况, 就是 root 用户也不能通过 kill 来杀掉流氓进程. 为了防止发生这种情况, 系统会自动激活 killer, 查找流氓进程并将其杀死。

### 解决方案

* 最简单的办法就是将系统迁移到内存更大的实例中。
* 可以通过 [OOM killer 调优](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/performance_tuning_guide/s-memory-captun)
* 做负载均衡(水平扩展,集群).
* 降低应用对内存的需求.

## Direct buffer memory

直接缓冲存储器是OS的本机存储器，由JVM进程使用，而不是在JVM堆中使用。Java NIO使用它快速将数据写入网络或磁盘; 无需在JVM堆和本机内存之间进行复制。Java应用程序可以设置JVM参数-XX：MaxDirectMemorySize来限制直接缓冲区内存大小。如果未设置此类参数，则JVM可以使用所有可用的OS本机内存。

由 DirectMemory 导致的内存溢出，一个明显的特征是在 Heap Dump 文件中不会看见明显的异常，如果发现 OOM 之后 Dump 文件很小，而程序中又直接或间接使用了 NIO ，那就可以考虑检查一下是不是这方面的原因。

### 原因分析

* 最常见的原因是直接内存不足导致。例如:
```java
/**
 * VM Option: -Xmx20M -XX:MaxDirectMemorySize=10M
 */
public class DirectMemoryOOM{
    private static final int size = 1024*1024;

    public static void main(String[] args)throws Exception{
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while(true){
            unsafe.allocateMemory(size);
        }
    }
}
```

* 内存泄漏导致，[JDK BUG](https://bugs.openjdk.java.net/browse/JDK-8147468),在[JDK 8u102更新发行说明](https://www.oracle.com/technetwork/java/javase/8u102-relnotes-3021767.html)中，添加了一个新的系统属性jdk.nio.maxCachedBufferSize来修复此问题。这个参数用于限制可以被缓存的DirectByteBuffer的大小，对于超过这个限制的DirectByteBuffer不会被缓存到ThreadLocal的bufferCache中，这样就能被GC正常回收掉。但是在本说明中指出，这个参数只能修复这个问题的一部分而不是所有情况。
* 使用了`-XX:+DisableExplicitGC`禁用system.gc方法的直接调用full gc。ByteBuffer类提供allocateDirect(int capacity)进行堆外内存的申请，底层通过unsafe.allocateMemory(size)实现，会调用malloc方法进行内存分配。实际上，在java堆里是维护了一个记录堆外地址和大小的DirectByteBuffer的对象，所以GC是能通过操作DirectByteBuffer对象来间接操作对应的堆外内存，从而达到释放堆外内存的目的。但如果一旦这个DirectByteBuffer对象熬过了young GC到达了Old区，同时Old区一直又没做CMS GC或者Full GC的话，这些“冰山对象”会将系统物理内存慢慢消耗掉。对于这种情况JVM留了后手，Bits给DirectByteBuffer前首先需要向Bits类申请额度，Bits类维护了一个全局的totalCapacity变量，记录着全部DirectByteBuffer的总大小，每次申请，都先看看是否超限（堆外内存的限额默认与堆内内存Xmx设定相仿），如果已经超限，会主动执行Sytem.gc()，System.gc()会对新生代的老生代都会进行内存回收，这样会比较彻底地回收DirectByteBuffer对象以及他们关联的堆外内存。但如果启动时通过-DisableExplicitGC禁止了System.gc()，那么这里就会出现比较严重的问题，导致回收不了DirectByteBuffer底下的堆外内存了。所以在类似Netty的框架里对DirectByteBuffer是框架自己主动回收来避免这个问题。

### 解决方案

* 尝试增加堆外内存：`-XX:MaxDirectMemorySize`
* 小心使用`-XX:+DisableExplicitGC`，特别是在有nio操作时。
* 可以考虑自己回收

## JVM Action

JVM提供了很多处理内存溢出的相关参数:

1. `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath`: 只要给JVM传递了这2个参数，当发生内存溢出的时候，JVM会自动在指定目录下生成内存映像文件。默认情况下，命名为`java_[PID].hprof`。
2. `-XX:OnOutOfMemoryError`:当发生内存溢出的时候，还可以让JVM调用任一个shell脚本。可以通过使用 `XX:OnOutOfMemoryError`中的`%p`占位符来传递 Java 进程的PID:`-XX:OnOutOfMemoryError="jstack %p"`。HeapDumpOnOutOfMemoryError和OnOutOfMemoryError先后顺序依赖jvm的实现。
3. `-XX:+ExitOnOutOfMemoryError`:如果传递了这个参数，当发生内存溢出的时候，JVM就会立马退出。
4. `-XX:+CrashOnOutOfMemoryError`:如果给JVM传递了这个参数，当发生内存溢出的时候，JVM就会退出，同时，JVM会产生文本和二进制格式的崩溃日志。

> 如果同时配置了`-XX:+HeapDumpOnOutOfMemoryError`和`-XX:+ExitOnOutOfMemoryError`，hotspot 虚拟机会先 dump，然后退出。

除了在启动时设置，也可以通过 JMX 暴露的HotSpotDiagnosticMBean来配置 VM 选项。

## 参考

- [1] [out of memory error](https://plumbr.io/outofmemoryerror)
- [2]  [threads_oom](https://www.oracle.com/technetwork/java/hotspotfaq-138619.html#threads_oom)
- [3] [SRE Case Study: Triaging a Non-Heap JVM Out of Memory Issue](https://tech.ebayinc.com/engineering/sre-case-study-triage-a-non-heap-jvm-out-of-memory-issue/)
- [4] [聊聊jvm的-XX:MaxDirectMemorySize](https://segmentfault.com/a/1190000018695161)
- [5] [HotSpot JVM调优的"标准参数"的各种陷阱](https://hllvm-group.iteye.com/group/topic/27945)
- [6] [关于MaxDirectMemorySize的设置](https://www.jianshu.com/p/e1503204a059)

