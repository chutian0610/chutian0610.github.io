# Java 处理系统信号


Linux信号量是一种比较原始的进程通信手段。有很多缺陷，可却是理解操作系统的基础概念。java 开放的api中涉及信号量的只有退出信号，即使用`Runtime.addShutdownHook()`实现jvm退出钩子。但是我们可以通过`sun.misc`包来实现支持其他信号的处理。

<!--more-->

## demo

```java
import sun.misc.Signal;
public class SignalHandler {

    public static class Handler implements sun.misc.SignalHandler {

        private final sun.misc.SignalHandler prevHandler;

        Handler(String name) {
            // 注册对指定信号的处理, 可以简单理解是对于某个信号的所有handler 是一个链表
            prevHandler = Signal.handle(new Signal(name), this);
        }
        @Override
        public void handle(Signal signal) {
            // 信号量名称
            String name = signal.getName();
            // 信号量数值
            int number = signal.getNumber();

            // 当前进程名
            String currentThreadName = Thread.currentThread().getName();

            System.out.println("[Thread:"+currentThreadName + "] receved signal: " + name + " == kill -" + number);
            // 将signal 传递给 jvm 底层，责任链模式
            prevHandler.handle(signal);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        final String[] SIGNALS = new String[]{ "TERM", "HUP", "INT" };
        for (String signalName : SIGNALS) {
            new Handler(signalName);
        }
        while(true) {
            Thread.sleep(1000);
        }
    }
}
```

项目地址[chutian0610/code-lab](https://github.com/chutian0610/code-lab/tree/main/demos/jdk-lab/src/main/java/info/victorchu/jdk/lab/usage/signal/SignalHandler.java)

启动该进程后，通过`jps`命令查看SignalHandler的pid，使用`kill -n pid`向进程发送信号量。例如使用信号量`2`。

```log
[Thread:SIGINT handler] receved signal: INT == kill -2
```

## JVM的信号处理

JVM的`System#initPhase1()`方法中会设置信号处理:

```java
// Setup Java signal handlers for HUP, TERM, and INT (where available).
Terminator.setup();
```
内核信号的处理是在 Terminator 类中进行的。HUP, TERM, INT这3个信号的处理都是`Shutdown.exit()`方法。

```java
class Terminator {

    private static Signal.Handler handler = null;

    /* Invocations of setup and teardown are already synchronized
     * on the shutdown lock, so no further synchronization is needed here
     */

    static void setup() {
        if (handler != null) return;
        Signal.Handler sh = new Signal.Handler() {
            public void handle(Signal sig) {
                Shutdown.exit(sig.getNumber() + 0200);
            }
        };
        handler = sh;
        // When -Xrs is specified the user is responsible for
        // ensuring that shutdown hooks are run by calling
        // System.exit()
        try {
            Signal.handle(new Signal("HUP"), sh);
        } catch (IllegalArgumentException e) {
        }
        try {
            Signal.handle(new Signal("INT"), sh);
        } catch (IllegalArgumentException e) {
        }
        try {
            Signal.handle(new Signal("TERM"), sh);
        } catch (IllegalArgumentException e) {
        }
    }
```

## 操作系统信号

软中断信号（signal，又简称为信号）用来通知进程发生了异步事件。在软件层次上是对中断机制的一种模拟，在原理上，一个进程收到一个信号与处理器收到一个中断请求可以说是一样的。信号是进程间通信机制中唯一的异步通信机制，一个进程不必通过任何操作来等待信号的到达，事实上，进程也不知道信号到底什么时候到达。进程之间可以互相通过系统调用kill发送软中断信号。内核也可以因为内部事件而给进程发送信号，通知进程发生了某个事件。信号机制除了基本通知功能外，还可以传递附加信息。

收到信号的进程对各种信号有不同的处理方法。处理方法可以分为三类：

1. 第一种是类似中断的处理程序，对于需要处理的信号，进程可以指定处理函数，由该函数来处理。
2. 第二种方法是，忽略某个信号，对该信号不做任何处理，就象未发生过一样。
3. 第三种方法是，对该信号的处理保留系统的默认值，这种缺省操作，对大部分的信号的缺省操作是使得进程终止。进程通过系统调用signal来指定进程对某个信号的处理行为。

Linux 内核实现大约 30 个信号。每个信号由数字识别，从 1 到 31。使用命令`kill -l` 可以查看系统支持的信号。

```sh
## macos
➜  ~ kill -l
HUP INT QUIT ILL TRAP ABRT EMT FPE KILL BUS SEGV SYS PIPE ALRM TERM URG STOP TSTP CONT CHLD TTIN TTOU IO XCPU XFSZ VTALRM PROF WINCH INFO USR1 USR2
```

|Signal	|Name| Description|
|:---|:---|:---|
|SIGHUP     |1   |Hangup (POSIX)|
|SIGINT	     |2   |Terminal interrupt (ANSI)|
|SIGQUIT    |3	|Terminal quit (POSIX)|
|SIGILL       |4  |Illegal instruction (ANSI)|
|SIGTRAP   |5  |Trace trap (POSIX)|
|SIGIOT      |6  |IOT Trap (4.2 BSD)|
|SIGBUS     |7  |BUS error (4.2 BSD)|
|SIGFPE	     |8  |Floating point exception (ANSI)|
|SIGKILL     |9	 |Kill(can't be caught or ignored) (POSIX)|
|SIGUSR1	|10 |User defined signal 1 (POSIX)|
|SIGSEGV   |11 |Invalid memory segment access (ANSI)|
|SIGUSR2	|12|User defined signal 2 (POSIX)|
|SIGPIPE	|13	|Write on a pipe with no reader, Broken pipe (POSIX)|
|SIGALRM	|14	|Alarm clock (POSIX)|
|SIGTERM	|15	|Termination (ANSI)|
|SIGSTKFLT|	16|	Stack fault|
|SIGCHLD	|17	|Child process has stopped or exited, changed (POSIX)|
|SIGCONTv|	18	|Continue executing, if stopped (POSIX)|
|SIGSTOP	|19|	Stop executing(can't be caught or ignored) (POSIX)|
|SIGTSTP	|20	|Terminal stop signal (POSIX)|
|SIGTTIN	|21	|Background process trying to read, from TTY (POSIX)|
|SIGTTOU	|22	|Background process trying to write, to TTY (POSIX)|
|SIGURG	|23|	Urgent condition on socket (4.2 BSD)|
|SIGXCPU	|24|	CPU limit exceeded (4.2 BSD)|
|SIGXFSZ|	25|	File size limit exceeded (4.2 BSD)|
|SIGVTALRM	|26|	Virtual alarm clock (4.2 BSD)|
|SIGPROF	|27|	Profiling alarm clock (4.2 BSD)|
|SIGWINCH	|28|	Window size change (4.3 BSD, Sun)|
|SIGIO	|29|	I/O now possible (4.2 BSD)|
|SIGPWR	|30	|Power failure restart (System V)|

