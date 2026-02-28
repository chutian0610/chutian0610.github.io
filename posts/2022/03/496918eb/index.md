# log 性能优化

日志是服务中一个重要组成部分，当优化性能的时候，也要思考对日志的性能优化。

<!--more-->

## logback

### 同步日志

```xml

<appender name="FILE" class="ch.qos.logback.core.FileAppender"> 
  <encoder>
    <pattern>%d %-5level [%thread] %logger{0}: %msg%n</pattern>
    <!-- immediateFlush 对同步影响较明显,主要是因为每次刷盘慢导致别的线程等锁时间长，immediateFlush为false，提升吞吐量时有丢日志的风险-->
    <!-- 延迟Flush的cache取决于JDK的BufferedOutputStream缓冲大小，默认8K，不可更改-->
    <immediateFlush>false</immediateFlush>
  </encoder> 
</appender>
```

### 异步打印

使用异步打印可以获得比同步打印更好的性能。

```xml
 <appender name="access-async" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="access"/>
    <!--用来指定队列大小,默认256-->
    <queueSize>1000</queueSize>
    <maxFlushTime>3000</maxFlushTime>
    <!--用来指定抛弃的水位线，默认80%，如果队列的80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志。只保留WARN与ERROR级别的Event，为了保留所有的events，可以将这个值设置为0-->
    <discardingThreshold>0</discardingThreshold>
    <!--用来指定当队列已满时放入新的日志事件是否抛出异常，如果为true则使用offer，否则使用put，区别为前者非阻塞后者阻塞-->
    <neverBlock>true</neverBlock>
</appender>
```

### 控制台日志

生产环境应该关闭控制台日志。可以减少不必要的开销。

### 获取行号

logback和log4j等日志框架获取行号的方式是相同的。即通过throw 时获取 StackTrace，然后从中解析出 文件名和行号。

```java
public StackTraceElement[] getCallerData() {
    if (callerDataArray == null) {
        callerDataArray = CallerData
            .extract(new Throwable(), fqnOfLoggerClass, loggerContext.getMaxCallerDataDepth(), loggerContext.getFrameworkPackages());
    }
    return callerDataArray;
}
```

在生产环境中可以不打印 文件名和行号来减少日志的性能消耗。

如果开启了异步Appender, 还要注意一个参数 `includeCallerData`

```java
protected void preprocess(ILoggingEvent eventObject) {
    eventObject.prepareForDeferredProcessing();
    if (includeCallerData)
        eventObject.getCallerData();
}
```

在异步appender中显式关闭。

```xml
  <appender name="access-async" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="access"/>
        <!--表示是否提取调用者数据，这个值被设置为true的代价是相当昂贵的，为了提升性能，默认当event被加入BlockingQueue时，event关联的调用者数据不会被提取，只有线程名这些比较简单的数据-->
        <includeCallerData>false</includeCallerData>
    </appender>
```

### 过滤异常栈

异常栈可以非常有效地帮助定位问题，但是也会因为异常栈太长，包含了太多几乎无价值的信息，比如反射、动态代理、Spring、tomcat等调用信息。过滤掉这部分信息，既减少了日志量，也减少大对象的数量，降低Full GC的次数。常用配置如下：

```xml
<property name="commonPattern" value="[%thread][%level][%class{0}:%line]: %msg%n%rEx{full,
     java.lang.reflect.Method,
     sun.reflect,
     org.apache.catalina,
     org.springframework.aop,
     org.springframework.security,
     org.springframework.transaction,
     org.springframework.web,
     org.springframework.beans,
     org.springframework.cglib,
     net.sf.cglib,
     org.apache.tomcat.util,
     org.apache.coyote,
     ByCGLIB,
     BySpringCGLIB,
     com.google.common.cache.LocalCache$
}"/>
```

## log4j2

根据log4j2官网上的性能对比[<sup>1</sup>](#refer-anchor-1),log4j2的性能无论在同步日志模式还是异步日志模式下都是最佳的(根本原因在于log4j2使用了LMAX Disruptor，高性能无锁本地消息队列).

### 异步打印

```xml
<!--  includeLocation 为 false，避免每次打印日志需要获取调用堆栈的性能损耗-->
<Asyncroot level="info" includeLocation="false">
    <appender-ref ref="file"/>
</Asyncroot>
```

log4j2.component.properties:

```properties
## 当没有日志的时候，打印日志的单线程通过 SLEEP 等待日志事件到来。这样 CPU 占用较少的同时，也能在日志到来时唤醒打印日志的单线程延迟较低，这个后面会详细分析
log4j2.asyncLoggerConfigWaitStrategy=SLEEP
```

### RingBuffer

```properties
## 指定Ringbuffer长度
log4j2.asyncLoggerConfigRingBufferSize=256*1024
## 指定堵塞丢弃策略，如果未指定此属性或具有value "Default"，则此工厂创建DefaultAsyncQueueFullPolicy对象，如果队列满了，会阻塞logger放入新的事件。如果此属性具有value "Discard"，则此工厂将创建 DiscardingAsyncQueueFullPolicy对象。默认情况下，如果队列已满，此路由器将丢弃级别为INFO，DEBUG和TRACE的事件。
log4j2.asyncQueueFullPolicy=Discard
## DiscardingAsyncQueueFullPolicy中判断当队列满时，什么级别的事件会被丢弃
log4j2.discardThreshold=INFO
```

未设置asyncQueueFullPolicy时(注意将AsyncLoggerConfig.SynchronizeEnqueueWhenQueueFull设置为true)，监控 RingBuffer 的使用率大小，如果使用率超过一定比例并且持续一段时间，证明应用写日志过忙，需要进行动态扩容，并且暂时将流量从这个实例切走一部分.对于每个 AsyncLogger 都会创建一个 RingBuffer。Log4j2 也考虑到了监控 AsyncLogger 这种情况，所以将 AsyncLogger 的监控暴露成为一个 MBean.

### 关闭 includeLocation

不要使用Location相关属性，例如 `C or $class, %F or %file, %l or %location, %L or %line, %M or %method`

日志中我们可能会需要输出当前输出的日志对应代码中的哪一类的哪一方法的哪一行，这个需要在运行时获取堆栈。获取堆栈，无论是在 Java 9 之前通过 Thorwable.getStackTrace()，还是通过 Java 9 之后的 StackWalker，获取当前代码堆栈，都是一个非常消耗 CPU 性能的操作。在大量输出日志的时候，会成为严重的性能瓶颈，其原因是：

- 获取堆栈属于从 Java 代码运行，切换到 JVM 代码运行，是 JNI 调用。这个切换是有性能损耗的。
- Java 9 之前通过新建异常获取堆栈，Java 9 之后通过 Stackwalker 获取。这两种方式，截止目前 Java 17 版本，都在高并发情况下，有严重的性能问题，会吃掉大量 CPU。主要是底层 JVM 符号与常量池优化的问题。

所以，我们在日志中不打印所在类方法。但是可以自己在日志中添加类名方法名用于快速定位问题代码。

### 配置 Disruptor 的等待策略

配置 Disruptor 的等待策略为 SLEEP.

>最好能将其中的 Thread.yield 修改为 Thread.onSpinWait(@since jdk9),这个修改仅针对 x86 机器部署.

Disruptor 的消费者做的事情其实就是不断检查是否有消息到来，其实就是某个状态位是否就绪，就绪后读取消息进行消费。至于如何不断检查，这个就是等待策略。Disruptor 中有很多等待策略，熟悉多处理器编程的对于等待策略肯定不会陌生，在这里可以简单理解为当任务没有到来时，线程应该如何等待并且让出 CPU 资源才能在任务到来时尽量快的开始工作。在 Log4j2 中，异步日志基于 Disruptor，同时使用 AsyncLoggerConfig.WaitStrategy 这个环境变量对于 Disruptor 的等待策略进行配置，目前Log4j2 中可以配置一下策略:

```java
switch (strategyUp) {
    case "SLEEP":
        final long sleepTimeNs =
                parseAdditionalLongProperty(propertyName, "SleepTimeNs", 100L);
        final String key = getFullPropertyKey(propertyName, "Retries");
        final int retries =
                PropertiesUtil.getProperties().getIntegerProperty(key, 200);
        return new SleepingWaitStrategy(retries, sleepTimeNs);
    case "YIELD":
        return new YieldingWaitStrategy();
    case "BLOCK":
        return new BlockingWaitStrategy();
    case "BUSYSPIN":
        return new BusySpinWaitStrategy();
    case "TIMEOUT":
        return new TimeoutBlockingWaitStrategy(timeoutMillis, TimeUnit.MILLISECONDS);
    default:
        return new TimeoutBlockingWaitStrategy(timeoutMillis, TimeUnit.MILLISECONDS);
}
```

BlockingWaitStrategy 会导致业务闲时突然来业务高峰的时候，日志消费线程唤醒的不够及时（CPU 一直被大量的 RUNNABLE 业务线程抢占）。如果使用比较激进的 BusySpinWaitStrategy（一直调用 Thread.onSpinWait()）或者 YieldingWaitStrategy（先 SPIN 之后一直调用 Thread.yield()），则闲时也会有较高的 CPU 占用。

期望的是一种递进的等待策略，例如：

- 在一定次数内，不断 SPIN，应对日志量特别多的时候，减少线程切换消耗。
- 在超过一定次数之后，开始不断的调用 Thread.onSpinWait() 或者 Thread.yield()，使当前线程让出 CPU 资源，应对间断性的日志高峰。
- 在第二步达到一定次数后，使用 Wait，或者 Thread.sleep() 这样的函数阻塞当前线程，应对日志低峰的时候，减少 CPU 消耗。

SleepingWaitStrategy 就是这样一个策略，第二步采用的是 Thread.yield()。

x86 的机器可以通过Custom WaitStrategy方式修改其中的 Thread.yield() 为 Thread.onSpinWait()。在 x86 的环境下 Thread.onSpinWait() 在被调用一定次数后，C1 编译器就会将其替换成使用 PAUSE 这个 x86 指令实现。参考 JVM 源码：

```c
instruct onspinwait() %{
  match(OnSpinWait);
  ins_cost(200);

  format %{
    $$template
    $$emit$$"pause\t! membar_onspinwait"
  %}
  ins_encode %{
    __ pause();
  %}
  ins_pipe(pipe_slow);
%}
```

CPU 并不会总直接操作内存，而是以缓存行读取后，缓存在 CPU 高速缓存上。但是对于这种不断检查检查某个状态位是否就绪的代码，不断读取 CPU 高速缓存，会在当前 CPU 从总线收到这个 CPU 高速缓存已经失效之前，都认为这个状态为没有变化。在业务忙时，总线可能非常繁忙，导致 SleepingWaitStrategy 的第二步一直检查不到状态位的更新导致进入第三步。

PAUSE 指令[<sup>2</sup>](#refer-anchor-2)是针对这种等待策略实现而产生的一个特殊指令，它会告诉处理器所执行的代码序列是一个不断检查某个状态位是否就绪的代码（即 spin-wait loop），这样的话，然后 CPU 分支预测就会据这个提示而避开内存序列冲突，CPU 就不会将这块读取的内存进行缓存，也就是说对 spin-wait loop 不做缓存，不做指令
重新排序等动作。从而提高 spin-wait loop 的执行效率。

这个指令使得针对 spin-wait loop 这种场景，Thread.onSpinWait()的效率要比 Thread.yield() 的效率要高。

### 过滤异常栈

```xml
<properties>
  <!--黑名单-->
  <property name="filters">org.junit,org.apache.maven,sun.reflect,java.lang.reflect</property>
</properties>
...
<PatternLayout pattern="%m%xEx{filters(${filters})}%n"/>
```

## 参考

<div id="refer-anchor-1"></div>

- [1] [log4j2.performance](https://logging.apache.org/log4j/2.x/performance.html)

<div id="refer-anchor-2"></div>

- [2] [x86.pause](https://www.felixcloutier.com/x86/pause)

- [3] [在被线上大量日志输出导致性能瓶颈毒打了很多次之后总结出的经验](https://heapdump.cn/article/2845460)

