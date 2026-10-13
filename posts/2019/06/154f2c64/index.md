# 使用 JMH 进行 Java 性能测试

[JMH](https://openjdk.java.net/projects/code-tools/jmh/) 是openJDK项目下的JVM工具，用于构建，运行和分析用Java和其他语言编写的针对JVM的nano/micro/milli/macro微基准测试。

<!--more-->

## quick start

快速使用JMH的方法是通过maven模板创建项目。

```sh
$ mvn archetype:generate \
          -DinteractiveMode=false \
          -DarchetypeGroupId=org.openjdk.jmh \
          -DarchetypeArtifactId=jmh-java-benchmark-archetype \
          -DgroupId=org.sample \
          -DartifactId=test \
          -Dversion=1.25.2
```

> 如果想对其他的JVM系语言做benchmark,可以替换archetypeArtifactId。

生成项目后，您可以使用以下Maven命令构建它：`mvn clean install`。构建完成后，将获得可执行JAR，它包含您的基准测试以及所有必要的JMH基础结构代码：`java -jar target/benchmarks.jar`

更一般的方法是在maven项目中添加依赖:

```xml
<dependency>
  <groupId>org.openjdk.jmh</groupId>
  <artifactId>jmh-core</artifactId>
  <version>${jmh.version}</version>
</dependency>
<dependency>
  <groupId>org.openjdk.jmh</groupId>
  <artifactId>jmh-generator-annprocess</artifactId>
  <version>${jmh.version}</version>
  <scope>provided</scope>
</dependency>
```

一个benchmark的编写例子如下：

```java
/**
 * 比较字符串直接相加和StringBuilder的效率
 */
@BenchmarkMode(Mode.Throughput)
@Warmup(iterations = 3)
@Measurement(iterations = 10, time = 5, timeUnit = TimeUnit.SECONDS)
@Threads(8)
@Fork(2)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public class StringBuilderBenchmark {

    @Benchmark
    public void testStringAdd() {
        String a = "";
        for (int i = 0; i < 10; i++) {
            a += i;
        }
        print(a);
    }

    @Benchmark
    public void testStringBuilderAdd() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 10; i++) {
            sb.append(i);
        }
        print(sb.toString());
    }

    private void print(String a) {
    }
}
/**
 * 启动入口，定制benchmark runner的option
 * 若不使用main函数,那么org.openjdk.jmh.Main.main()作为入口,同时需要在命令行中指定参数
 */
public static void main(String[] args) throws RunnerException {
    Options options = new OptionsBuilder()
            .include(StringBuilderBenchmark.class.getSimpleName())
            .output("E:/Benchmark.log") // benchmark log
            .build();
    new Runner(options).run();
}
```

默认执行方式(不自定义main函数): `java -jar target/benchmarks.jar -h` 查看帮助信息。

## benchmark中使用的注解

### 基准测试类型 `@BenchmarkMode`

* Throughput:整体的吞吐量，例如"1秒内可以执行多少次调用"。
* AverageTime:调用的平均时间，例如“每次调用平均耗时xxx”。
* SampleTime: 随机取样，最后输出取样结果的分布，例如“99%的调用在xxx毫秒以内，99.99%的调用在xxx毫秒以内”
* SingleShotTime: 以上模式都是默认运行一次 iteration，唯有 SingleShotTime 是只运行一次。往往同时把 warmup 次数设为0，用于测试冷启动时的性能。
* All:所有模式

### 启动预热`@Warmup`

进行基准测试前需要进行预热。保证测试的准确性。其中的参数iterations也就非常好理解了，就是预热轮数。为什么需要预热？因为 JVM 的 JIT 机制的存在，如果某个函数被调用多次之后，JVM 会尝试将其编译成为机器码从而提高执行速度。所以为了让 benchmark 的结果更加接近真实情况就需要进行预热。

> 通过`-XX:CompileThreshold`参数控制JIT触发阈值

### 度量`@Measurement`

就是一些基本的测试参数：

* iterations 进行测试的轮次
* time 每轮进行的时长
* timeUnit 时长单位

都是一些基本的参数，可以根据具体情况调整。一般比较重的东西可以进行大量的测试，放到服务器上运行。

### 线程数`@Threads`

每个进程中的测试线程，这个非常好理解，根据具体情况选择，一般为cpu+1 或cpu乘2。

### 子任务`@Fork`

进行 fork 的次数。如果 fork 数是2的话，则 JMH 会 fork 出两个进程来进行测试。

### 时间格式`@OutputTimeUnit`

基准测试结果的时间类型。一般选择秒、毫秒、微秒。

### benchmark标识`@Benchmark`
方法级注解，表示该方法是需要进行 benchmark 的对象，用法和 JUnit 的 @Test 类似。

### 参数标识`@Param`

属性级注解，@Param 可以用来指定某项参数的多种情况。特别适合用来测试一个函数在不同的参数输入的情况下的性能。

也支持命令行参数覆盖:`java -jar benchmarks.jar -p ioThread=1,2,3,4
`

### 作用范围`@State`

当使用@Setup参数的时候，必须在类上加这个参数，不然会提示无法运行。

State 用于声明某个类是一个"状态"，然后接受一个 Scope 参数用来表示该状态的共享范围。 因为很多 benchmark 会需要一些表示状态的类，JMH 允许你把这些类以依赖注入的方式注入到 benchmark 函数里。Scope 主要分为三种。

* Thread: 该状态为每个线程独享。
* Group: 该状态为同一个组里面所有线程共享。
* Benchmark: 该状态在所有线程间共享。

一个使用state的例子:

```java
public class MyBenchmark {

  @State(Scope.Thread)
  public static class MyState {
    public int a = 1;
    public int b = 2;
    public int sum ;
  }

  @Benchmark 
  @BenchmarkMode(Mode.Throughput) 
  @OutputTimeUnit(TimeUnit.MINUTES)
  public void testMethod(MyState state) {
      state.sum = state.a + state.b;
  }
}
```

### 启动前`@Setup`

方法级注解，这个注解的作用就是我们需要在测试之前进行一些准备工作，比如对一些数据的初始化之类的。

### 结束后`@TearDown`

方法级注解，这个注解的作用就是我们需要在测试之后进行一些结束工作，比如关闭线程池，数据库连接等的，主要用于资源的回收等。

### Level
用于控制 @Setup，@TearDown 的调用时机（注解参数)，默认是 Level.Trial，即benchmark开始前和结束后。

* Level.Trial	对于基准的每次完整运行，调用该方法一次。完整运行意味着包括所有预热和基准迭代。
* Level.Iteration	对于基准的每次迭代，该方法被调用一次。
* Level.Invocation	每次调用基准测试方法都会调用该方法一次。

```java
public class MyBenchmark {
    @State(Scope.Thread)
    public static class MyState {

        @Setup(Level.Trial)
        public void doSetup() {
            sum = 0;
            System.out.println("Do Setup");
        }

        @TearDown(Level.Trial)
        public void doTearDown() {
            System.out.println("Do TearDown");
        }

        public int a = 1;
        public int b = 2;
        public int sum ;
    }
    @Benchmark @BenchmarkMode(Mode.Throughput) @OutputTimeUnit(TimeUnit.MINUTES)
    public void testMethod(MyState state) {
        state.sum = state.a + state.b;
    }
}
```

### 组`@Group`
可以把多个 benchmark 定义为同一个 group，则它们会被同时执行，主要用于测试多个相互之间存在影响的方法。

### 编译器行为`@CompilerControl`

控制 compiler 的行为，例如强制 inline，不允许编译等。

## 书写基准测试的注意点

正确编写测量较大应用程序的一小部分性能的基准测试很难。当基准测试单独执行该组件时，JVM或底层硬件可能会对您的组件应用许多优化。当组件作为较大应用程序的一部分运行时，可能无法应用这些优化。因此，错误实现的微基准测试可能会让您相信组件的性能优于实际情况。

### 循环优化

将基准代码置于基准测试方法的循环中是很诱人的，以便在每次调用基准测试方法时重复更多次（以减少基准方法调用的开销）。但是，JVM非常擅长优化循环，因此最终可能会得到与预期不同的结果。通常，您应该避免使用基准测试方法中的循环，除非它们是您要测量的代码的一部分（而不是您要测量的代码周围）。

### 死代码消除

在实现性能基准测试时要避免的JVM优化之一是消除死代码。如果JVM检测到从未使用某些计算的结果，则JVM可以考虑此计算 死代码并将其消除。看看这个基准示例：

```java
public class MyBenchmark {

    @Benchmark
    public void testMethod() {
        int a = 1;
        int b = 2;
        int sum = a + b;
    }
}
```

JVM可以检测变量sum从未使用过。因此，JVM可以a + b完全删除计算。它被认为是死代码。然后，JVM可以检测到a 与b从不使用。他们也可以被淘汰。

最后，基准测试中没有任何代码。因此，运行此基准测试的结果具有很大的误导性。基准测试实际上并不测量添加两个变量并将值分配给第三个变量的时间。基准测试什么都没有。

为了避免死代码消除，您必须确保要测量的代码看起来不像JVM的死代码。有两种方法可以做到这一点。

1. 从基准方法返回代码的结果。

```java
public class MyBenchmark {
    @Benchmark
    public int testMethod() {
        int a = 1;
        int b = 2;
        int sum = a + b;
        return sum; // 去除死代码消除，JMH将欺骗JVM来相信返回值被使用
    }
}
```

如果您的基准测试方法正在计算可能最终被作为死代码消除的多个值，您可以将这两个值组合成一个，并返回该值（例如，具有两个值的对象）。

2. 将计算值传递给JMH提供的"黑洞"。

```java
public class MyBenchmark {

    @Benchmark
   public void testMethod(Blackhole blackhole) {
        int a = 1;
        int b = 2;
        int sum = a + b;
        blackhole.consume(sum);
    }
}
```

返回组合值的替代方法是将计算值（或返回/生成的对象或基准测试的结果）传递到JMH 黑洞。请注意testMethod()基准测试方法现在如何将Blackhole对象作为参数。这将在调用时由JMH提供给测试方法。另请注意sum变量中的计算总和现在如何传递给Blackhole实例的consume() 方法。这会欺骗JVM认为sum 变量实际被使用。如果你的基准方法产生多个结果，你可以每这些结果传递到黑洞。

### 常量折叠

常量折叠是另一种常见的JVM优化。无论计算执行了多少次，基于常数的计算通常都会产生完全相同的结果。JVM可以检测到该计算，并将计算结果替换为计算结果。

```java
public class MyBenchmark {

    @Benchmark
    public int testMethod() {
        int a = 1;
        int b = 2;
        int sum = a + b;
        return sum;
    }
}
```

JVM中可以检测到的值sum是基于所述两个恒定值1和2,因此可以用以下代码替换上面的代码：

```java
public class MyBenchmark {

    @Benchmark
    public int testMethod() {
        int sum = 3;
        return sum; 
        //or just return 3;
    }
}
```

为避免不断折叠，您不能将常量硬编码到基准测试方法中。相反，计算的输入应来自状态对象。这使得JVM更难以看到计算基于常量值。这是一个例子：

```java
public class MyBenchmark {
    @State(Scope.Thread)
    public static class MyState {
        public int a = 1;
        public int b = 2;
    }
    @Benchmark 
    public int testMethod(MyState state) {
        int sum = state.a + state.b;
        return sum;
    }
}
```

## 运行benchmark的注意点

* 始终包括一个运行预热阶段，足以在计时阶段之前触发所有初始化和编译。（在预热阶段，迭代次数较少。经验法则是数万次内循环迭代。)
* 始终使用-XX:+PrintCompilation，-verbose:gc等等，以便您可以验证编译器和JVM的其他部分在计时阶段没有执行意外工作。在计时和预热阶段的开始和结束时打印消息，来帮助验证这个注意点
* 注意-client和-server，OSR和常规编译之间的区别。如果追求最佳性能，则首选服务器而不是客户端，常规编译而不是OSR。还要注意-XX:+TieredCompilation将客户端和服务器模式混合在一起。
* 注意初始化效果。在加载和初始化类时打印，不要在计时阶段第一次打印。除非您专门测试类加载（并且在这种情况下只加载测试类），否则不要在预热阶段（或最终报告阶段）之外加载新类。
* 减少测量中的噪音。在安静的机器上运行您的基准测试，并运行几次，丢弃异常值。使用-Xbatch序列化编译器和应用程序，并考虑设置-XX:CICompilerCount=1以防止编译器与其自身并行运行。
* 使用适当的工具来阅读编译器的生成代码。[阅读方法](https://wiki.openjdk.java.net/display/HotSpot/PrintAssembly)

## 参考

- [1] [JMH 官方样例](https://hg.openjdk.java.net/code-tools/jmh/file/tip/jmh-samples/src/main/java/org/openjdk/jmh/samples/)
- [2] [jenkov tutorials](http://tutorials.jenkov.com/java-performance/jmh.html)
- [3] [MicroBenchmarks](https://wiki.openjdk.java.net/display/HotSpot/MicroBenchmarks)
- [4] [Anatomy of a flawed microbenchmark](https://www.ibm.com/developerworks/java/library/j-jtp02225/index.html)
- [5] [Avoiding Benchmarking Pitfalls on the JVM](https://www.oracle.com/technetwork/articles/java/architect-benchmarking-2266277.html)

