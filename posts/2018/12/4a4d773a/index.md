# Java虚拟机-启动参数详解

java命令用于启动JVM虚拟机。Java启动参数分为3种:

1. 标准参数: 所有的JVM实现都必须实现这些参数的功能，而且向后兼容。JVM的标准参数都是以”-“开头。
2. 非标准参数: 默认JVM(HotSpot虚拟机)实现这些参数的功能，但是并不保证所有jvm实现都满足，且不保证向后兼容.JVM的非标准参数都是以”-x“开头。
3. 非stable参数：此类参数通常具有特定的系统要求，并且可能需要对系统配置参数的特权访问。各个jvm实现会有所不同，将来可能会随时取消，需要慎重使用。JVM的非stable参数都是以”-xx“开头。

<!--more-->

## JVM 标准参数

查看参数方式: `java -help` 或者是`java -?`。以下是JVM标准参数的详细介绍:

1. `-server`: 选择使用 Java HotSpot Server VM[<sup>3</sup>](#refer-anchor-3)。特点是启动速度比较慢，但运行时性能和内存管理效率很高，适用于生产环境。在具有64位能力的jdk环境下将默认启用该模式，而忽略-client参数。

|Architecture|OS|Default client VM|if server-class, server VM;otherwise, client VM|Default server VM|
|:---|:---|:---|:---|:---|
|SPARC 32-bit|Solaris||X||
|i586|Solaris||X||
|i586|Linux||X||
|i586|Microsoft Windows|X|||
|SPARC 64-bit|Solaris|—||X|
|AMD64|Solaris|—||X|
|AMD64|Linux|—||X|
|AMD64|Microsoft Windows|—||X|

>  X = default VM     
>  — = client VM not provided for this platform

2. `-client`: 选择使用 Java HotSpot Client VM。设置jvm使用client模式，特点是启动速度比较快，但运行时性能和内存管理效率不高。在具有64位能力的jdk环境下将忽略-client参数。
3. `-agentlib：libname[=options]`: 用于装载本机代理库；其中libname为本地代理库文件名，默认搜索路径为环境变量PATH中的路径，options为传给本地库启动时的参数，多个参数之间用逗号分隔。在Windows平台上jvm搜索本地库名为libname.dll的文件，在linux上jvm搜索本地库名为libname.so的文件，搜索路径环境变量在不同系统上有所不同，比如Solaries上就默认搜索LD_LIBRARY_PATH。
    * 比如：-agentlib:hprof ; 用来获取jvm的运行情况，包括CPU、内存、线程等的运行数据，并可输出到指定文件中。每隔 20 毫秒获取一次示例 CPU 信息，堆栈深度为 3：`-agentlib:hprof=cpu=samples,interval=20,depth=3`
    * 加载 Java 调试线路协议 （JDWP） 库并侦听端口 8000 上的套接字连接，从而在主类加载之前挂起 JVM：`-agentlib:jdwp=transport=dt_socket,server=y,address=8000`
    * 详细信息，请参阅以下内容：
        *  java.lang.instrument软件包描述[<sup>4</sup>](#refer-anchor-4)
        * JVM Tools Interface guide [<sup>5</sup>](#refer-anchor-5)
4. `-agentpath:pathname[=options]`: 按全路径装载本地代理库，不再搜索PATH中的路径；其他功能和agentlib相同。
5. `-Dproperty=value`: 设置系统属性名/值对，运行在此jvm之上的应用程序可用 `System.getProperty("property")`得到value的值。如果value中有空格，则需要用双引号将该值括起来`-Dfoo="foo bar"`。
6. `-disableassertions[:[packagename]...|:classname]` 或者 `-da[:[packagename]...|:classname]`: 禁用断言。默认所有包都禁用。当不带参数时，会关闭所有包和类的断言(注意系统class除外)。如果参数是包名带有`...`，那么会禁用指定包和所有子包的断言。如果参数只有`...`,那么会禁用当前工作目录下包的所有断言，如果参数是某个类型，那么会禁用当前类的断言。
7. `-disablesystemassertions` 或 `-dsa`: 禁用所有系统类(没有对应的classLoader)的断言。
8. `-enableassertions[:[packagename]...|:classname]` 或`-ea[:[packagename]...|:classname]`: 开启断言。默认所有断言是关闭的。当不带参数时，会开启所有包和类的断言(注意系统class除外)。如果参数是包名带有`...`，那么会开启指定包和所有子包的断言。如果参数只有`...`,那么会开启当前工作目录下包的所有断言，如果参数是某个类型，那么会开启当前类的断言。
9. `-enablesystemassertions`或`-esa`: 开启所有系统类(没有对应的classLoader)的断言。
10. `-jar filename`: 执行封装在 JAR 文件中的程序。filename参数是 JAR 文件的名称，jar里面manifest文件中的 `Main-Class:classname`属性定义了程序的入口类。使用该选项时，指定的 JAR 文件是所有用户类的源，其他类路径设置将被忽略。如果当前Jar包还有其他外部依赖。需要在manifest文件中添加`Class-Path: baz.jar`属性，来定义classpath。
11. `-javaagent:jarpath[=options]`加载指定的 Java agent。参考 java.lang.instrument软件包描述[<sup>4</sup>](#refer-anchor-4)。
12. `-classpath classpath`或者`-cp classpath`告知jvm类搜索目录名、jar文档名、zip文档名，之间用分号;分隔；使用-classpath后jvm将不再使用CLASSPATH中的类搜索路径，如果-classpath和CLASSPATH都没有设置，则jvm使用当前路径(.)作为类搜索路径。jvm搜索类的方式和顺序为：Bootstrap，Extension，User。
    * Bootstrap中的路径是jvm自带的jar或zip文件，jvm首先搜索这些包文件，用System.getProperty(“sun.boot.class.path”)可得到搜索路径。
    * Extension是位于JRE_HOME/lib/ext目录下的jar文件，jvm在搜索完Bootstrap后就搜索该目录下的jar文件，用System.getProperty(“java.ext.dirs”)可得到搜索路径。
    * User搜索顺序为当前路径.、CLASSPATH、-classpath，jvm最后搜索这些目录，用System.getProperty(“java.class.path”)可得到搜索路径。
13. `-verbose:class`: 打印类加载信息
14. `-verbose:gc`: 打印GC信息
15. `-verbose:jni`: 打印JNI本地方法调用信息。

## JVM非标准参数

通过`java -X`可以输出非标准参数列表.非标准参数又称为扩展参数: 

* `-Xint`: 设置jvm以解释模式运行，所有的字节码将被直接执行，而不会编译成本地码。
* `-Xbatch`:关闭后台代码编译，强制在前台编译，编译完成之后才能进行代码执行；默认情况下，jvm在后台进行编译，若没有编译完成，则前台运行代码时以解释模式运行。
* `-Xbootclasspath:bootclasspath`: 让jvm从指定路径（可以是分号分隔的目录、jar、或者zip）中加载bootclass，用来替换jdk的rt.jar；若非必要，一般不会用到；
* `-Xbootclasspath/a:path`: 将指定路径的所有文件追加到默认bootstrap路径中；
* `-Xbootclasspath/p:path`:让jvm优先于bootstrap默认路径加载指定路径的所有文件；
* `-Xcheck:jni`: 对JNI函数进行附加check；此时jvm将校验传递给JNI函数参数的合法性，在本地代码中遇到非法数据时，jmv将报一个致命错误而终止；使用该参数后将造成性能下降，请慎用。
* `-Xfuture`: 让jvm对类文件执行严格的格式检查（默认jvm不进行严格格式检查），以符合类文件格式规范，推荐开发人员使用该参数。
*  `-Xmixed`: 混合模式执行（默认）.用解释器执行所有字节码，但热方法除外，热方法被编译为本机代码。
* `-Xnoclassgc`: 关闭针对class的gc功能；因为其阻止内存回收，所以可能会导致OutOfMemoryError错误，慎用；
* `-Xincgc`: 开启增量gc（默认为关闭）；这有助于减少长时间GC时应用程序出现的停顿；但由于可能和应用程序并发执行，所以会降低CPU对应用的处理能力。
* `-Xloggc:<file>`: 与-verbose:gc功能类似，只是将每次GC事件的相关情况记录到一个文件中，文件的位置最好在本地，以避免网络的潜在问题。若与verbose命令同时出现在命令行中，则以-Xloggc为准。
* `-Xmn`: 设置年轻代的初始和最大大小。建议将年轻代的规模保持在整体堆大小的一半到四分之一之间.
* `-Xms`: 指定jvm堆的初始大小，默认为物理内存的1/64，最小为1M；可以指定单位，比如k、m，若不指定，则默认为字节。如果未设置此选项，则初始大小将设置为为老年代和年轻代分的大小之和。可以使用选项`-Xmn-XX:NewSize`设置年轻代的堆的初始大小。注意，`-Xms-XX:InitalHeapSize`选项还可用于设置初始堆大小。如果它出现在命令行之后，则初始堆大小将设置为用`-Xms-XX:InitalHeapSize`指定的值。
* `-Xmx`: 指定jvm堆的最大值，默认为物理内存的1/4或者1G，最小为2M；单位与-Xms一致。对于服务器部署，-Xms和-Xmx通常设置为相同的值。该选项等效于`-Xmx-XX:MaxHeapSize`.
* `-Xss`: 设置单个线程栈的大小，一般默认为512k。
* `-Xprof`: 输出 cpu 配置文件数据，不要在生产系统中使用。
* `-Xcomp`: 强制在第一次调用时编译方法。默认情况下，客户端VM执行 1，000 次解释型方法调用，服务器VM执行10000次解释型方法调用，虚拟机会即时编译方法。指定该选项将禁用解释方法调用，从而以牺牲效率为代价来提高编译性能。还可以在编译之前使用该选项`-XX:CompileThreshold` 修改解释方法调用的触发次数。
* `-Xrs`: 减少jvm对操作系统信号（signals）的使用，该参数从1.3.1开始有效；从jdk1.3.0开始，jvm允许程序在关闭之前还可以执行一些代码（比如关闭数据库的连接池），即使jvm被突然终止；jvm关闭工具通过监控控制台的相关事件而满足以上的功能；更确切的说，通知在关闭工具执行之前，先注册控制台的控制handler，然后对CTRL_C_EVENT, CTRL_CLOSE_EVENT, CTRL_LOGOFF_EVENT, and CTRL_SHUTDOWN_EVENT这几类事件直接返回true。JVM 使用类似的机制来实现转储线程堆栈的功能以进行调试。CTRL_BREAK_EVENT 用于执行线程转储。但如果jvm以服务的形式在后台运行（比如servlet引擎），他能接收CTRL_LOGOFF_EVENT事件，但此时并不需要初始化关闭程序；为了避免类似冲突的再次出现，从jdk1.3.1开始提供-Xrs参数；当此参数被设置之后，jvm将不接收控制台的控制handler，也就是说他不监控和处理CTRL_C_EVENT, CTRL_CLOSE_EVENT, CTRL_LOGOFF_EVENT, or CTRL_SHUTDOWN_EVENT事件。指定`-Xrs`有两个后果：`Ctrl + Break` 线程转储不可用。用户代码负责唤起关闭hook，例如，在 JVM 终止时调用。System.exit().

### JVM GC 日志

```sh
## 日志内容
-XX:+PrintGCDateStamps
-XX:+PrintGCTimeStamps
-XX:+PrintGCDetails
-XX:+PrintGCCause
-XX:+PrintGCApplicationConcurrentTime
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintReferenceGC
-XX:+PrintClassHistogramAfterFullGC
-XX:+PrintClassHistogramBeforeFullGC
-XX:PrintFLSStatistics=2
-XX:+PrintAdaptiveSizePolicy
-XX:+PrintSafepointStatistics
-XX:PrintSafepointStatisticsCount=1

## GC日志输出的文件路径
-Xloggc:/path/to/gc-%t.log
## 开启日志文件分割
-XX:+UseGCLogFileRotation 
## 最多分割几个文件，超过之后从头开始写
-XX:NumberOfGCLogFiles=14
## 每个文件上限大小，超过就触发分割
-XX:GCLogFileSize=100M
```

### JVM 日志

JVM日志(Since Java 9)有助于了解应用程序如何工作。我们可以使用`-Xlog`选项设置JVM日志。

```sh
java -Xlog:
    codecache+sweep*=trace,
    class+unload,
    class+load,
    os+thread,
    safepoint,
    gc*,
    gc+stringdedup=debug,
    gc+ergo=trace,
    gc+age=trace,
    gc+phases=trace,
    gc+humongous=trace,
    jit+compilation=debug
:file=/path_to_logs/app.log
:time,level,tags,tid
:filesize=104857600,filecount=5
```

使用 `java -Xlog:help` 可以print可用选项。此处就不[展开介绍](https://docs.oracle.com/javase/9/tools/java.htm#GUID-BE93ABDC-999C-4CB5-A88B-1994AAAC74D5__CONVERTGCLOGGINGFLAGSTOXLOG-A5046BD1)。

## JVM非Stable参数

设置参数的方法如下：

* `-XX:+<option>` 启用选项  
* `-XX:-<option>`不启用选项  
* `-XX:<option>=<number>`  
* `-XX:<option>=<string>`  

打印参数的几个参数设置:

|参数|描述|
|:----|:----|
|-XX:+PrintFlagsInitial|该命令可以查看所有JVM参数启动的初始值|
|-XX:+PrintFlagsFinal|该参数会打印所有的系统参数的值|
|-XX:+PrintCommandLineFlags|该参数打印传递给虚拟机的显式和隐式参数。|
|-XX:+PrintVMOptions|该参数表示程序运行时，打印虚拟机接受到的命令行显式参数。|

由于非State参数非常的多，因此这里就不列出所有参数进行讲解。只介绍我们比较常用的。

### 解锁参数

|参数|默认值|描述|
|:---|:---|:---|
|-XX:+UnlockDiagnosticVMOptions|默认关闭|解锁用于统计JVM的选项，关闭时，统计选项不可用|
|-XX:+UnlockExperimentalVMOptions|默认关闭|开启实现性选项|

### 内存参数

|参数|默认值|描述|
|:---|:---|:---|
|-XX:MaxPermSize|默认 64M|设置永久代大小(JDK8)失效|
|-XX:+UseTLAB|server模式默认开启|优先在本地线程缓冲区中分配对象，避免分配内存时的锁定过程|
|-XX:UseLargePages|默认开启|大内存分页|
|-XX:LargePageSizeInBytes|默认4M|内存页的大小不可设置过大，会影响Perm的大小，-XX:LargePageSizeInBytes=128m，需要OS支持|
|-XX:MaxDirectMemorySize|默认与-Xmx一致|直接内存大小|
|-XX:FieldsAllocationStyle|默认1|对象内存分布中的实例数据区域的存储顺序:0 - 引用在原始类型前面, 然后依次是longs/doubles, ints, shorts/chars, bytes, 最后是填充字段, 以满足要求, 1 -引用在原始类型后面, 2 - JVM在布局时会尽量使父类对象和子对象挨在一起|
|-XX:+CompactFieldstrue|默认开启|由于HotSpot在分配对象实例数据时相同大小的字段总是被分配到一起存储，在满足这个条件下因此父类中定义的变量会出现在子类之前，开启此参数那子类中较小的变量也允许插入父类变量的空隙中，以节省一点空间|
|-XX:+UseCondCardMark|默认false|是否开启JVM卡表条件判断，尽量减少伪共享带来的性能损耗|
|-XX:VMThreadStackSize|1024|设置vm内部的线程比如gc线程设置栈大小|
|-XX:ThreadStackSize|-|设置普通java线程栈大小，若为0则使用不同平台默认值，默认单位bytes|
|-XX:CompilerThreadStackSize|设置compiler_thread的stack_size|
|-XX:StackShadowPages|20|设置Java 的本地方法栈的大小，单位是操作系统页数，linux中为20*4k|

#### metaspace

|参数|默认值|描述|
|:---|:---|:---|
|-XX:MetaspaceSize|默认情况下，值根据不同的平台|初始化的Metaspace大小，该值越大触发Metaspace GC的时机就越晚。随着GC的到来，虚拟机会根据实际情况调控Metaspace的大小，可能增加上线也可能降|
|-XX:MaxMetaspaceSize|无限制|元空间的最大本地内存，防止因为某些情况导致Metaspace无限的使用本地内存，影响到其他程序|
|-XX:CompressedClassSpaceSize|默认1G|Metaspace 中的 Compressed Class Space 的最大允许内存，默认值是 1G，这部分会在 JVM 启动的时候向操作系统申请1G虚拟地址映射，但不是真的就用了操作系统的 1G 内存。|
|-XX:UseLargePagesInMetaspace|默认false|是否在metaspace里使用LargePage，一般情况下我们使用4KB的page size，这个参数依赖于UseLargePages这个参数开启|
|-XX:InitialBootClassLoaderMetaspaceSize|64位下默认4M，32位下默认2200K|初始化内存Block的大小|
|-XX:MaxMetaspaceExpansion|默认5.2M|增大触发metaspace GC阈值的最大扩展。假如说我们要分配的内存超过了MinMetaspaceExpansion但是低于MaxMetaspaceExpansion，那增量是MaxMetaspaceExpansion，如果超过了MaxMetaspaceExpansion，那增量是MinMetaspaceExpansion加上要分配的内存大小|
|-XX:MinMetaspaceExpansion|默认332.8K|用于临时扩大下metaspace触发GC的阈值，增大触发metaspace GC阈值的最小扩展。假如我们要救急分配的内存很小，没有达到MinMetaspaceExpansion，但是我们会将这次触发metaspace GC的阈值提升MinMetaspaceExpansion，之所以要大于这次要分配的内存大小主要是为了防止别的线程也有类似的请求而频繁触发相关的操作|
|-XX:MinMetaspaceFreeRatio|默认40|表示每次GC完之后，假设我们允许接下来metaspace可以继续被commit的内存占到了被commit之后总共committed的内存量的MinMetaspaceFreeRatio%,如果这个总共被committed的量比当前触发metaspaceGC的阈值要大，那么将尝试做扩容，也就是增大触发metaspaceGC的阈值|
|-XX:MaxMetaspaceFreeRatio|默认70|这个参数和上面的参数基本是相反的，是为了避免触发metaspaceGC的阈值过大，而想对这个值进行缩小。这个参数在gc之后committed的内存比较小的时候并且离触发metaspaceGC的阈值比较远的时候，调整会比较明显。|

> MinMetaspaceExpansion/MaxMetaspaceExpansion 每次分配只会给对应的线程一次扩展触发metaspace GC阈值的机会，如果扩展了，但是还不能分配，那就只能等着做GC了。

### GC参数

|参数|默认值|描述|
|:---|:---|:---|
|-XX:+DisableExplicitGC|关闭|忽略System.gc()方法触发的垃圾收集|
|-XX:+ScavengeBeforeFullGC|默认:开启|Full GC前触发一次Minor GC|
|-XX:GCTimeRatio|默认:99|GC时间占总时间的比列，默认值为99，即允许1%(计算公式1/(1+N))的GC时间|
|-XX:MaxGCPauseMillis|默认无|设置GC的最大停顿时间，这是一个软性目标，JVM会尽量达到，垃圾收集停顿时间缩短可能导致频繁回收|
|-XX:+UseGCOverheadLimit|默认:开启|禁止GC过程无限制执行，如果98%的时间花在GC上但是小于2%的堆空间被回收，直接抛出OutOfMemory异常|
|-XX:MaxHeapFreeRatio|默认:70|当Xmx比Xms值大时，堆可以动态收缩和扩展，这个参数控制当堆空闲大于指定值时自动收缩|
|-XX:MinHeapFreeRatio|默认:40|当Xmx比Xms值大时，堆可以动态收缩和扩展，这个参数控制当堆空闲小于指定值时自动扩展|
|-XX:GCPauseIntervalMillis|默认0|设置GC的间隔时间|
|-XX:ConcGCThreads|默认依赖JVM的CPU核数|并发GC线程数量|
|-XX:InitiatingHeapOccupancyPercent|默认45%|触发MixedGC时的堆空间占用比例(整个堆，而不是某个年代),G1也使用这个选项|
|-XX:MinMetaspaceFreeRatio|默认值为40%|当进行过Metaspace GC之后，会计算当前Metaspace的空闲空间比，如果空闲比小于这个参数，那么虚拟机将增长Metaspace的大小。设置该参数可以控制Metaspace的增长的速度，太小的值会导致Metaspace增长的缓慢，Metaspace的使用逐渐趋于饱和，可能会影响之后类的加载。而太大的值会导致Metaspace增长的过快，浪费内存。|
|-XX:MaxMetasaceFreeRatio=|默认值为70%|当进行过Metaspace GC之后， 会计算当前Metaspace的空闲空间比，如果空闲比大于这个参数，那么虚拟机会释放Metaspace的部分空间。|

#### GC 垃圾回收器

|参数|默认值|描述|
|:---|:---|:---|
|-XX:+UseSerialGC|Client模式默认开启，其他模式默认关闭|Jvm运行在Client模式下的默认值，打开此开关后，使用Serial + Serial Old的收集器组合进行内存回收|
|-XX:+UseParNewGC|默认关闭|打开此开关后，使用ParNew + Serial Old的收集器进行垃圾回收|
|-XX:+UseConcMarkSweepGC|默认关闭|使用ParNew + CMS +  Serial Old的收集器组合进行内存回收，Serial Old作为CMS出现"Concurrent Mode Failure”失败后的后备收集器使用。|
|-XX:+UseParallelGC|Server模式默认开启，其他模式关闭|Jvm运行在Server模式下的默认值，打开此开关后，使用Parallel Scavenge +  Serial Old的收集器组合进行回收|
|-XX:+UseParallelOldGC|默认关闭|使用Parallel Scavenge +  Parallel Old的收集器组合进行回收|
|-XX:+UseG1GC|默认关闭|启用G1垃圾收集器|

#### GC分代参数

|参数|默认值|描述|
|:---|:---|:---|
|-XX:NewSize|建议是堆的1/4到1/2|新生代的初始大小|
|-XX:NewRatio|默认是2|新生代（Eden + 2*S）与老年代（不包括永久区）的比值|
|-XX:SurvivorRatio|默认:8|新生代中Eden区域与Survivor区域的容量比值，默认为8，代表Eden:Subrvivor = 8:1|
|-XX:TargetSurvivorRatio|默认50|Survivor区对象使用率，在新生代的对象不一定要满足存活年龄达到MaxTenuringThreshold才能去老年代，当Survivor空间中某年龄内所有对象大小总和大于Desired survivor size(survivor区容量 * TargetSurvivorRatio/100)时，年龄大于或等于该年龄的对象直接进入老年代。|
|-XX:PretenureSizeThreshold|默认:无|直接晋升到老年代对象的大小，设置这个参数后，大于这个参数的对象将直接在老年代分配|
|-XX:MaxTenuringThreshold|默认:15|晋升到老年代的对象年龄，每次Minor GC之后，年龄就加1，当超过这个参数的值时进入老年代|
|-XX:UseAdaptiveSizePolicy|默认开启|动态调整java堆中各个区域的大小以及进入老年代的年龄|
|-XX:+HandlePromotionFailure|JDK1.5及以前默认关闭,1.6默认开启|是否允许新生代收集担保，进行一次minor gc后, 另一块Survivor空间不足时，将直接会在老年代中保留|


#### Crash

|参数|默认值|描述|
|:---|:---|:---|
|-XX:+HeapDumpBeforeFullGC|关闭|实现在Full GC前dump|
|-XX:+HeapDumpAfterFullGC|关闭|实现在Full GC后dump|
|-XX:ErrorFile=filename|默认当前工作目录hs_err_pid%p.log,%p指当前进程|指定发生不可恢复错误时写入错误数据的路径和文件名|
|-XX:OnError=string|-|将自定义命令或一系列以分号分隔的命令设置为在发生不可恢复的错误时运行。例如:`-XX:OnError="gcore %p;dbx - %p"`用于dump core或`-XX:OnError="pmap %p"`用于打印memory map|

#### 垃圾回收器选项

|参数|默认值|描述|
|:---|:---|:---|
|-XX:+ExplicitGCInvokesConcurrent|关闭|CMS时使用，当收到System.gc()提交的垃圾收集申请时，没有设置DisableExplicitGC（默认为false），表示可以接受显式地触发GC。这个时候触发的GC都是Full GC，但是如果设置了ExplicitGCInvokesConcurrent，则表示可以进行并发的回收。也可以使用`-XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses`|
|-XX:ParallelGCThreads|少于或等于8个cpu时，默认为cpu数量，多于8个时比cpu数量值小|设置并行GC进行内存回收的线程数，同样适用于CMS|
|-XX:CMSInitiatingOccupancyFraction|JDK5及之前默认为68%，JDK6之后调整为92%|设置CMS收集器在老年代空间被使用多少后触发垃圾收集，默认值为68%，(近似值)，如果频繁发生SerialOld卡顿，应该调小|
|-XX:+UseCMSInitiatingOccupancyOnly|默认关闭|与`XX:CMSInitiatingOccupancyFraction`配合使用，只是用设定的回收阈值,如果不指定,JVM仅在第一次使用设定值,后续则自动调整。|
|-XX:+CMSPrecleaningEnabled|默认true|开启CMS并发预清理|
|-XX:CMSScavengeBeforeRemark|默认false|开启或关闭在CMS重新标记阶段之前的清除（Young GC）尝试，由于 YoungGen 存在引用 OldGen 对象的情况，因此 CMS-remark 阶段会将 YoungGen 作为 OldGen 的 “GC ROOTS” 进行扫描，防止回收了不该回收的对象。在 CMS GC 的 CMS-remark 阶段开始前先进行一次 Young GC，有利于减少 Young Gen 对 Old Gen 的无效引用，降低 CMS-remark 阶段的时间开销。|
|-XX:CMSScheduleRemarkEdenSizeThreshold|默认为2M|CMS可取消并发预处理阶段开启条件|
|-XX:CMSMaxAbortablePrecleanLoops|默认为0|CMS可取消并发预处理阶段取消条件（循环次数）|
|-XX:CMSMaxAbortablePrecleanTime|默认5000毫秒|CMS可取消并发预处理阶段取消条件（最长执行时间）|
|-XX:+UseCMSCompactAtFullCollection|默认:开启|由于CMS收集器会产生碎片，此参数设置在垃圾收集器后是否需要一次内存碎片整理过程|
|-XX:CMSFullGCsBeforeCompaction|默认0|设置CMS收集器在进行若干次垃圾收集后再进行一次内存碎片整理过程，通常与UseCMSCompactAtFullCollection参数一起使用|
|-XX:+CMSParallelRemarkEnabled|默认开启|用户开启CMS remark阶段采用多线程的方式进行重新标记|
|-XX:+CMSClassUnloadingEnabled|默认启用|使用并发标记扫描（CMS）垃圾收集器时，启用类卸载。|
|-XX:CMSInitiatingPermOccupancyFraction|无默认，需要0~100|达到什么比例时进行Perm回收，JDK 8中不推荐使用此选项|
|-XX:G1HeapRegionSize|默认值由堆动态决定|G1分区大小, 例如`-XX:G1HeapRegionSize=16m`，随着size增加，垃圾的存活时间更长，GC间隔更长，但每次GC的时间也会更长|
|-XX:G1NewSizePercent|默认为5%|实验性配置，新生代最小比例|
|-XX:G1MaxNewSizePercent|默认为60%|实验性配置，新生代最大比例|
|-XX:G1ReservePercent|默认10，需要0~50|在G1管理的老年代里预留的空闲内存，保证新生代对象晋升到老年代的时候有足够空间，避免老年代内存都满了，新生代有对象要进入老年代没有充足内存|
|-XX:+ParallelRefProcEnabled|默认false|开启尽可能并行处理Reference对象|


### 即时编译

|参数|默认值|描述|
|:---|:---|:---|
|-XX:CompileThreshold|client模式默认1500，server默认10000|触发方法即时编译的阈值|
|-XX:OnStackReplacePercentage|Client默认值是933，server默认140|OSR阈值，表示触发JIT编译之前，解释执行一个方法中循环体的代码的执行次数的阈值|
|-XX:ReservedCodeCacheSize|默认32M|即时编译器编译代码的缓存最大值|
|-XX:InitialCodeCacheSize|-|codeCache初始大小|
|-XX:+CICompilerCount=N|4|编译线程数，设置数量后，JVM会自动分配线程数，C1:C2 = 1:2|
|-XX:+TieredCompilation|默认开启|开启分层编译，JDK8之后默认开启,在系统执行初期，执行频率比较高的代码先被c1编译器编译，以便尽快进入编译执行，然后随着时间的推移，执行频率较高的代码再被c2编译器编译，以达到最高的性能。|
|-XX:TierXCompileThreshold|0,2000,15000|分层编译后的编译阈值，X=2,3,4|
|-XX:TierXBackEdgeThreshold|0,60000,40000|分层编译后的OSR阈值，，X=2,3,4|
|-XX:TierXInvocationThreshold|200,500|分层编译后各层调用的阈值,X=3,4|
|-XX:TierXMinInvocationThreshold|100,600|分层编译后各层最小调用次数阈值,X=3,4|
|-XX:CompileThresholdScaling|默认值为1.0|控制是否触发编译的因子,值越低，编译开始得越早,可以基于方法基本配置，例如 `-XX:CompileCommand=CompileThresholdScaling,*MyBenchmark*::compute*,0.05`|

### 多线程参数

|参数|默认值|描述|
|:---|:---|:---|
|-XX:+UseFastAccessorMethods|默认开启|频繁反射时，生成字节码来加速|
|-XX:UseSpinning|1.6默认开启，1.5默认关闭|开启自旋锁|
|-XX:PreBlockSpin|默认:10|自旋锁默认自旋次数|
|-XX:UseThreadPriorities|默认开启|使用本地线程优先级|
|-XX:UseBaisedLocking|默认开启|开启偏向锁|


### 性能参数

|参数|默认值|描述|
|:---|:---|:---|
|-XX:AggressiveOpts|1.6默认开启，1.5关闭|使用激进的优化特性|
|-XX:StringCache|默认开启|字符串缓存|
|-XX:+MaxFDLimit|开启|最大化文件描述符的数量限制|
|-XX:+DoEscapeAnalysis|默认true|开启逃逸分析|
|-XX:+EliminateAllocations|默认true|开启标量替换|
|-XX:+EliminateLocks|默认true|开启同步消除|

### 验证参数

|参数|默认值|描述|
|:---|:---|:---|
|-XX:+FailOverToOldVerifier|默认关闭，1.6引入|当新类型检查程序失败时，将转移到旧的验证程序。|
|-XX:+RelaxAccessControlCheck|默认开启|在验证程序中放松访问控制检查。|

### OOM

|参数|描述|
|:----|:----|
|-XX:+HeapDumpOnOutOfMemoryError|发生OOM的时候自动dump堆栈方便分析|
|-XX:HeapDumpPath|默认当前工作目录的java_pid.hprof|堆转储dump文件地址|
|-XX:OnOutOfMemoryError|发生内存溢出异常时，执行的命令,例如`-XX:OnOutOfMemoryError=/scripts/restart-myapp.sh`|
|-XX:+CrashOnOutOfMemoryError|默认false|JVM将在抛出OutOfMemoryError时立即退出。除了退出，JVM还会生成core dump文件|
|-XX:+ExitOnOutOfMemoryError|默认false|JVM将在抛出OutOfMemoryError时立即退出。|

### 调试参数

|参数|描述|
|:----|:----|
|-XX:+PrintGC|输出GC日志|
|-XX:+PrintGCDetails|输出GC的详细日志|
|-XX:+PrintGCTimeStamps|输出GC的时间戳（以基准时间的形式）|
|-XX:+PrintGCDateStamps|输出GC的时间戳（以日期的形式，如2013-05-04T21:53:59.234+0800）|
|-XX:+PrintHeapAtGC|在进行GC的前后打印出堆的信息|
|-XX:+PrintGCApplicationStoppedTime|输出GC造成应用暂停的时间(STW时间)|
|-XX:+PrintGCApplicationConcurrentTime|打印每次垃圾回收前，程序未中断的执行时间|
|-XX:+PrintTenuringDistribution|显示在survivor空间里面有效的对象的岁数情况,在每一个岁数上面，对象的存活的数量，以及其增减情况|
|-XX:+PrintReferenceGC|强引用/弱引用/软引用/虚引用 finalize情况|
|-XX:+PrintSafepointStatistics|打印safepoint统计信息|
|-XX:PrintSafepointStatisticsCount=1 |打印安全点统计信息count值，需要开启`PrintSafepointStatistics`|
|-XX:+TraceClassLoading|查看类加载信息|
|-XX:+TraceClassUnLoading|查看类卸载信息|
|-XX:+TraceLoaderConstraints|跟踪类加载器约束的相关信息|
|-XX:+TraceClassResolution|跟踪常量池|
|-XX:+TraceClassLoadingPreorder |跟踪被引用到的所有类的加载信息|
|-XX:+PrintCompilation|当一个方法被编译时打印相关信息|
|-XX:+PrintClassHistogram|遇到Ctrl-Break后打印类实例的柱状信息，与jmap -histo功能相同|
|-XX:+PrintConcurrentLocks|遇到Ctrl-Break后打印并发锁的相关信息，与jstack -l功能相同|
|-XX:+CITime|打印消耗在JIT编译的时间|
|-XX:PrintFLSStatistics=1|gc日志中输出free list方式分配内存后内存统计情况和碎片情况,值为1输出BinaryTreeDictionary,值大于1输出IndexedFreeLists|
|-XX:+PrintAdaptiveSizePolicy|显示收集器工效(Collector ergonomics)|

### +UnlockDiagnosticVMOptions

开启UnlockDiagnosticVMOptions下的参数

|参数|描述|
|:----|:----|
|-XX:+DebugNonSafepoints|激活更多调试信息的记录，即使它不在安全点。这意味着我们可以为堆栈跟踪提供更精确的位置。调试信息越多，行号分辨率就越精确（通常在JFR或Async-profiler 分析性能时需要打开）。|
|+PrintAssembly|打印字节码和本机方法的程序集代码|

## java相关信息

```sh
## 显示系统配置
java -XshowSettings:system -version

## 显示JVM 设置。
java -XshowSettings:vm -version

## 显示与系统属性相关的设置。
java -XshowSettings:properties -version

## 显示区域设置
java -XshowSettings:locale -version

## 显示所有设置
java -XshowSettings:all -version

## 打印所有jvm 配置(PrintFlagsFinal只显示可以调整的)
java -XX:+PrintFlagsFinal -XX:+UnlockDiagnosticVMOptions -XX:+UnlockExperimentalVMOptions -version
```

## jinfo

在很多情况下，Java 应用程序不会指定所有的 Java 虚拟机参数。而此时，开发人员可能不知道某一个具体的 Java 虚拟机参数的默认值。在这种情况下，可能需要通过查找文档获取某个参数的默认值。这个查找过程可能是非常艰难的。但有了 jinfo 工具，开发人员可以很方便地找到 Java 虚拟机参数的当前值，此外jinfo还支持动态修改虚拟机参数(只有被标记为 manageable 的参数才可以被动态修改)。

```
基本使用语法为：jinfo [options] pid
```

|选项|选项说明|
|:---|:---|
|no option|输出全部的参数和系统属性|
|-flag name|输出对应名称的参数|
|-flag [+-]name|开启或者关闭对应名称的参数 
|-flag name=value|设定对应名称的参数
|-flags|输出全部的参数|
|-sysprops|输出系统属性|

例如在频繁fullgc时:

1. 通过jps获得java程序的pid

2. 调用jinfo命令设置JVM参数: 

```sh
$ jinfo -flag +HeapDumpBeforeFullGC $pid
$ jinfo -flag +HeapDumpAfterFullGC $pid
```

下次发生fullgc时就会生成dump文件，dump文件取到后我们就可以清除原来设置的参数：

```sh
$ jinfo -flag -HeapDumpBeforeFullGC $pid
$ jinfo -flag -HeapDumpAfterFullGC $pid
```

## jvm option 工具

- [JVMOptionsExplorer 网站](https://chriswhocodes.com/)

- [JVM Options](https://jvm-options.tech.xebia.fr/#)

## 参考

- [1] [Java7 HotSpot VM Options](https://www.oracle.com/java/technologies/javase/vmoptions-jsp.html)

- [2] [Java8 java命令官方文档](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html)

<div id="refer-anchor-3"></div>

- [3] [Java8 默认虚拟机](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/server-class.html)

<div id="refer-anchor-4"></div>

- [4] [java.lang.instrument软件包描述](http://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)

<div id="refer-anchor-5"></div>

- [5] [JVM Tools Interface guide](http://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html#starting)

- [6] [Java Platform, Standard Edition HotSpot Virtual Machine Garbage Collection Tuning Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/index.html)

