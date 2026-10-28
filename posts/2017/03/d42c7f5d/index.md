# Java进程


进程的概念主要有两点：

* 第一，进程是一个实体。每一个进程都有它自己的地址空间，一般情况下，包括文本区域（text region）、数据区域（data region）和堆栈（stack region）。文本区域存储处理器执行的代码；数据区域存储变量和进程执行期间使用的动态分配的内存；堆栈区域存储着活动过程调用的指令和本地变量。
* 第二，进程是一个“执行中的程序”。程序是一个没有生命的实体，只有处理器赋予程序生命时（操作系统执行之），它才能成为一个活动的实体，我们称其为进程。

<!--more-->

## 进程的特征

* 动态性：进程的实质是程序在多道程序系统中的一次执行过程，进程是动态产生，动态消亡的。
* 并发性：任何进程都可以同其他进程一起并发执行
* 独立性：进程是一个能独立运行的基本单位，同时也是系统分配资源和调度的独立单位；
* 异步性：由于进程间的相互制约，使进程具有执行的间断性，即进程按各自独立的、不可预知的速度向前推进
* 结构特征：进程由程序、数据和进程控制块三部分组成。

多个不同的进程可以包含相同的程序：一个程序在不同的数据集里就构成不同的进程，能得到不同的结果；但是执行过程中，程序不能发生改变。

### 进程与线程的区别

进程和线程的主要差别在于它们是不同的操作系统资源管理方式。进程有独立的地址空间，一个进程崩溃后，在保护模式下不会对其它进程产生影 响，而线程只是一个进程中的不同执行路径。线程有自己的堆栈和局部变量，但线程之间没有单独的地址空间，一个线程死掉就等于整个进程死掉，所以多进程的程 序要比多线程的程序健壮，但在进程切换时，耗费资源较大，效率要差一些。但对于一些要求同时进行并且又要共享某些变量的并发操作，只能用线程，不能用进程。

## 进程的创建

Java提供了两种方法用来启动进程或其它程序：

1. 使用Runtime的exec()方法
2. 使用ProcessBuilder的start()方法

这两个方法会创建一个本地进程，并返回一个抽象类`Process`子类的实例用于控制进程和获取信息。

### `Process`抽象类

`Process`提供的方法如下：

1. `destroy() : void` 杀死子进程，无论`Process`对象代表的子进程是否被强行终止或者尚未实现。

2. `destroyForcibly() : Process`  杀死子进程，强制终止子进程。这个方法的默认实现是调用`destroy()`方法，所以可能并没有强制终止子进程。

3. `exitValue() : int`  返回子进程的退出返回值。

4. `getErrorStream() : InputStream`  从管道获取子进程的错误流。如果标准I/O已经被重定向，返回为null的流。

5. `getInputStream() : InputStream`  从管道获取子进程的输出。如果标准I/O已经被重定向，返回为null的流。

6. `getOutputStream() : OutputStream`  向管道写入子进程的输入。如果标准I/O已经被重定向，返回为null的流。

7. `isAlive() : boolean`  判断子进程是否还活着。

8. `waitFor() : int`  当前线程等待，直到进程终止。

9. `waitFor(long, TimeUnit) : boolean`  当前线程等待，直到进程终止或等待超时。如果，子进程已经终止，立刻返回true；如果子进程没有结束，等待时间<0，立刻返回false；

### 有关`Process`的一些说明

在某些操作系统上`Process`的一些特殊子类不能很好的工作，例如窗口进程，守护进程，DOS进程\(windows\)和shell 脚本。默认地，子进程的所有标准I/O\(stdin,stdout,stderr\)操作都被重定向到父进程，因此，可以通过`getOutputStream()`,`getInputStream()`和 `getErrorStream()`这些接口获得对应的流。当没有更多的对Process对象的引用时，子进程不会被杀死，而是子进程继续异步执行。

> 注意，有些操作系统只提供大小受限的缓冲流，这可能导致子进程的阻塞或者死锁。

### `ProcessBuilder`

ProcessBuilder类是J2SE 1.5在java.lang中新添加的一个新类，此类用于创建操作系统进程，它提供一种启动和管理进程（也就是应用程序）的方法。在J2SE 1.5之前，都是由Process类处来实现进程的控制管理。 每个 ProcessBuilder 实例管理一个进程属性集。start() 方法利用这些属性创建一个新的 Process 实例。start() 方法可以从同一实例重复调用，以利用相同的或相关的属性创建新的子进程。

每个进程生成器管理这些进程属性：

* 命令 是一个字符串列表，它表示要调用的外部程序文件及其参数（如果有）。在此，表示有效的操作系统命令的字符串列表是依赖于系统的。例如，每一个总体变量，通 常都要成为此列表中的元素，但有一些操作系统，希望程序能自己标记命令行字符串——在这种系统中，Java 实现可能需要命令确切地包含这两个元素。
* 环境 是从变量 到值 的依赖于系统的映射。初始值是当前进程环境的一个副本（请参阅 System.getenv()）。
* 工作目录。默认值是当前进程的当前工作目录，通常根据系统属性 user.dir 来命名。 
* redirectErrorStream 属性。最初，此属性为 false，意思是子进程的标准输出和错误输出被发送给两个独立的流，这些流可以通过 Process.getInputStream() 和 Process.getErrorStream() 方法来访问。如果将值设置为 true，标准错误将与标准输出合并。这使得关联错误消息和相应的输出变得更容易。在此情况下，合并的数据可从 Process.getInputStream() 返回的流读取，而从 Process.getErrorStream() 返回的流读取将直接到达文件尾。

修改进程构建器的属性将影响后续由该对象的 start() 方法启动的进程，但从不会影响以前启动的进程或 Java 自身的进程。注意，此类不是同步的。如果多个线程同时访问一个 ProcessBuilder，而其中至少一个线程从结构上修改了其中一个属性，它必须保持外部同步。

```java
ProcessBuilder(List<String> command);
ProcessBuilder(String... command);

List<String> command();
ProcessBuilder command(List<String> command);
ProcessBuilder command(String... command);
File directory();  //返回此进程生成器的工作目录。
ProcessBuilder directory(File directory); //设置此进程生成器的工作目录。
Map<String,String> environment();//  返回此进程生成器环境的字符串映射视图。
boolean redirectErrorStream();// 通知进程生成器是否合并标准错误和标准输出。
//  设置此进程生成器的 redirectErrorStream 属性。  
ProcessBuilder redirectErrorStream(boolean redirectErrorStream);
Process start(); // 使用此进程生成器的属性启动一个新进程
```

### Runtime

每个 Java 应用程序都有一个 Runtime 类实例，使应用程序能够与其运行的环境相连接。可以通过 getRuntime 方法获取当前运行时。 应用程序不能创建自己的 Runtime 类实例。但可以通过 getRuntime 方法获取当前Runtime运行时对象的引用。一旦得到了一个当前的Runtime对象的引用，就可以调用Runtime对象的方法去控制Java虚拟机的状态和行为。

```java
// 注册新的虚拟机来关闭挂钩。
void addShutdownHook(Thread hook);
//向 Java 虚拟机返回可用处理器的数目。
int availableProcessors();
Process exec(String command);
Process exec(String[] cmdarray);
// 在指定环境的独立进程中执行指定命令和变量。
Process exec(String[] cmdarray, String[] envp);
//在指定环境和工作目录的独立进程中执行指定的命令和变量
Process exec(String[] cmdarray, String[] envp, File dir);
Process exec(String command, String[] envp);
Process exec(String command, String[] envp, File dir);
// 通过启动虚拟机的关闭序列，终止当前正在运行的 Java 虚拟机。
void exit(int status);
//返回 Java 虚拟机中的空闲内存量。
long freeMemory();
// 运行垃圾回收器。
void gc();
// 返回与当前 Java 应用程序相关的运行时对象。
static Runtime getRuntime();
// 强行终止目前正在运行的 Java 虚拟机。
void halt(int status);
// 加载作为动态库的指定文件名。
void load(String filename);
//加载具有指定库名的动态库。
void loadLibrary(String libname);
// 返回 Java 虚拟机试图使用的最大内存量
long maxMemory();
//取消注册某个先前已注册的虚拟机关闭挂钩
boolean removeShutdownHook(Thread hook);
//运行挂起 finalization 的所有对象的终止方法。
void runFinalization();
// 返回 Java 虚拟机中的内存总量。
long totalMemory();
// 启用／禁用指令跟踪。
void traceInstructions(boolean on);
// 启用／禁用方法调用跟踪
void traceMethodCalls(boolean on);
```

### 进程结束

通常，一个程序/进程在执行结束后会向操作系统返回一个整数值，0一般代表执行成功，非0表示执行出现问题。有两种方式可以用来获取进程的返回值。一是利 用waitFor()，该方法是阻塞的，执导进程执行完成后再返回。该方法返回一个代表进程返回值的整数值。另一个方法是调用exitValue()方 法，该方法是非阻塞的，调用立即返回。但是如果进程没有执行完成，则抛出异常。

### 进程阻塞问题

由Process代表的进程在某些平台上有时候并不能很好的工作，特别是在对代表进程的标准输入流、输出流和错误输出进行操作时，如果使用不慎，有可能导致进程阻塞，甚至死锁。

当进程启动后，就会打开标准输出流和错误输出流准备输出，当进程结束时，就会关闭他们。_在某个情况下，错误输出流没有数据要输出，标准输出流中有数据输出。由于标准输出流中的数据没有被读取，进程就不会结束，错误输出流也就不会被关闭，因此在调用readLine()方法时，整个程序就会被阻塞_。为了解 决这个问题，可以根据输出的实际先后，先读取标准输出流，然后读取错误输出流。

_但是，很多时候不能很明确的知道输出的先后，特别是要操作标准输入的时候，情况就会更为复杂_。

* 这时候可以采用线程来对标准输出、错误输出和标准输入进行分别处理，根据他们之间在业务逻辑上的关系决定读取那个流或者写入数据。
* 或者使用ProcessBuilder的redirectErrorStream()方法将他们合二为一，这时候只要读取标准输出的数据就可以了。

> 当在程序中使用Process的waitFor()方法时，特别是在读取之前调用waitFor()方法时，也有可能造成阻塞。可以用线程的方法来解决这个问题，也可以在读取数据后，调用waitFor()方法等待程序结束。

```java
import java.util.*;
import java.io.*;
/**
 * 读取线程流
 * new StreamWatch(process.getInputStream()).start();
 * new StreamWatch(process.getErrorStream()).start();
 */
class StreamWatch extends Thread {
  InputStream is;
  List<String> output = new ArrayList<String>();

  StreamWatch(InputStream is) {
    this.is = is;
  }

  public void run() {
    try {
      InputStreamReader isr = new InputStreamReader(is);
      BufferedReader br = new BufferedReader(isr);
      String line = null;
      while ((line = br.readLine()) != null) {
      output.add(line);
      }
    } catch (IOException ioe) {
      ioe.printStackTrace();
    }
  }
}
```

