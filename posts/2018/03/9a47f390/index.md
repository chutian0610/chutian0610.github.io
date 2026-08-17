# Java并发之synchronized

synchronized 是Java中的关键字，作用是实现线程间的同步。synchronized 会对要同步的代码加锁，每次只会有一个线程进入同步代码块，从而保证线程安全。

<!--more-->

## synchronized 的基本使用

Synchronized是Java中解决并发问题的一种最常用的方法，也是最简单的一种方法。Synchronized的作用主要有三个

1. 确保线程互斥的访问同步代码；
2. 保证共享变量的修改能够及时可见；
3. 有效解决重排序问题。

从语法上讲，Synchronized总共有三种用法：

1. 修饰普通方法
2. 修饰静态方法
3. 修饰代码块

## Synchronized 原理

以如下代码块为例：

```java
public class SynchronizedDemo {
    public void method() {
        synchronized (this) {
            System.out.println("Method 1 start");
        }
    }
}
```

得到的反编译结果：

```java
javap -c SynchronizedDemo
Compiled from "SynchronizedDemo.java"
public class SynchronizedDemo {
  public SynchronizedDemo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return
  public void method();
    Code:
       0: aload_0
       1: dup
       2: astore_1
       3: monitorenter  /* 获取monitor*/
       4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       7: ldc           #3                  // String Method 1 start
       9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      12: aload_1
      13: monitorexit   /* 退出monitor*/
      14: goto          22
      17: astore_2
      18: aload_1
      19: monitorexit
      20: aload_2
      21: athrow
      22: return
    Exception table:
       from    to  target type
           4    14    17   any
          17    20    17   any
}
```

### 原理说明

关于`monitorenter`和`monitorexit`这两条指令的作用，我们直接参考JVM规范中描述\(这里简单翻译下\)：

每个对象有一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：

1. 当多个线程同时访问一段同步代码时，首先会进入_EntryList队列中.
2. 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。
3. 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1.
4. 如果其他线程已经占用了monitor，则该线程会进入WaitSet，直到monitor的进入数为0，然后重新进入EntryList重新排队。
5. 若持有monitor的线程调用wait()方法，将释放当前持有的monitor，\_owner变量恢复为null，\_count自减1，同时该线程进入_WaitSet集合中等待被唤醒。

执行monitorexit的线程必须是objectref所对应的monitor的所有者。指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。

![monitor](monitor.webp)

#### wait & notify 原理

![wait & notify](wait-notify.webp)

1. 使用 wait ，notify 和 notifyAll 时需要先对调用对象加锁。
2. 调用 wait 方法后，线程状态有 Running 变为 Waiting，并将当前线程放置到对象的 等待队列。
3. notify 或者 notifyAll 方法调用后， 等待线程依旧不会从 wait 返回，需要调用 noitfy 的线程释放锁之后，等待线程才有机会从 wait 返回。
4. notify 方法将等待队列的一个等待线程从等待队列种移到同步队列中，而 notifyAll 方法则是将等待队列种所有的线程全部移到同步队列，被移动的线程状态由 Waiting 变为 Blocked。
5. 从 wait 方法返回的前提是获得了调用对象的锁。

#### 同步方法

同步方法的例子如下：

```java
public class SynchronizedMethod {
 public synchronized void method() {
    System.out.println("Hello World!");
 }
}
```

同步方法反编译结果：

```java
public synchronized void method();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED /*同步标识符*/
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Hello World!
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 3: 0
        line 4: 8
```

从反编译的结果来看，方法的同步并没有通过指令monitorenter和monitorexit来完成（理论上其实也可以通过这两条指令来实现），不过相对于普通方法，其常量池中多了ACC_SYNCHRONIZED标示符。JVM就是根据该标示符来实现方法的同步的：当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。 其实本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。

## synchronized 锁优化

Synchronized是通过对象内部的一个叫做监视器锁（monitor）来实现的。但是监视器锁本质又是依赖于底层的操作系统的Mutex Lock来实现的。而**操作系统实现线程之间的切换这就需要从用户态转换到核心态，这个成本非常高，状态之间的转换需要相对比较长的时间，这就是为什么Synchronized效率低的原因**。因此，这种依赖于操作系统Mutex Lock所实现的锁我们称之为“重量级锁”。

JDK中对Synchronized做的种种优化，其核心都是为了减少这种重量级锁的使用。JDK1.6以后，为了减少获得锁和释放锁所带来的性能消耗，提高性能，使用了大量的锁优化技术。

### 锁存储位置--对象头

锁存在Java对象头里。如果对象是数组类型，则虚拟机用3个Word（字宽）存储对象头，如果对象是非数组类型，则用2字宽存储对象头。在32位虚拟机中，一字宽等于四字节，即32bit。

|长度|内容|说明|
|:---|:---|:---|
|32/64bit|Mark Word|存储对象的hashCode或锁信息等。|
|32/64bit|Class Metadata Address|存储到对象类型数据的指针|
|32/64bit|Array length|数组的长度（如果当前对象是数组）|

Java对象头里的Mark Word里默认存储对象的HashCode，分代年龄和锁标记位。32位JVM的Mark Word的默认存储结构如下：

![锁状态](lock-status.webp)

锁的状态总共有四种：无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁（但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级）。JDK 1.6中默认是开启偏向锁和轻量级锁的.锁升级功能主要依赖于Mark Word中锁标志位和是否偏向锁标志位.

### 偏向锁

Hotspot的作者经过以往的研究发现大多数情况下锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要花费CAS操作来加锁和解锁，而只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。

引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令（由于一旦出现多线程竞争的情况就必须撤销偏向锁，所以偏向锁的撤销操作的性能损耗必须小于节省下来的CAS原子指令的性能消耗）。轻量级锁是为了在线程交替执行同步块时提高性能，而偏向锁则是在只有一个线程执行同步块时进一步提高性能。

#### 偏向锁获取过程

1. 访问Mark Word中偏向锁的标识是否设置成1，锁标志位是否为01——确认为可偏向状态。
2. 如果为可偏向状态，则测试线程ID是否指向当前线程，如果是，进入步骤 5，否则进入步骤 3。
3. 如果线程ID并未指向当前线程，则通过CAS操作竞争锁。如果竞争成功，则将Mark Word中线程ID设置为当前线程ID，然后执行 5；如果竞争失败，执行 4。
4. 如果CAS获取偏向锁失败，则表示有竞争。撤销偏向锁,当到达全局安全点（safepoint）时获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。
5. 执行同步代码。

#### 偏向锁的释放

偏向锁的撤销在上述第四步骤中有提到。偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动去释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置成无锁状态，如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。

#### 偏向锁分析

* 优点：加锁和解锁不需要额外的消耗，和执行非同步方法比仅存在纳秒级的差距。
* 如果线程间存在锁竞争，会带来额外的锁撤销的消耗。
* 适用于只有一个线程访问同步块场景。

JVM 默认开启偏向锁，并且延迟生效，`-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=4000`，因为JVM刚启动时竞争非常激烈。在高并发场景下可以做如下优化：
  
* 关闭偏向锁，`-XX:-UseBiasedLocking`
* 直接设置为重量级锁`-XX:+UseHeavyMonitors

### 轻量级锁

“轻量级”是相对于使用操作系统互斥量来实现的传统锁而言的。但是，首先需要强调一点的是，轻量级锁并不是用来代替重量级锁的，它的本意是在没有多线程竞争的前提下，减少传统的重量级锁使用产生的性能消耗。在解释轻量级锁的执行过程之前，先明白一点，轻量级锁所适应的场景是线程交替执行同步块的情况，即不存在竞争问题，如果存在同一时间访问同一锁的情况，就会导致轻量级锁膨胀为重量级锁。

#### 轻量级锁的加锁过程

1. 在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，官方称之为 Displaced Mark Word。这时候线程堆栈与对象头的状态如下图所示。

![轻量级锁CAS操作之前堆栈与对象的状态](light-lock.webp)

2. 拷贝对象头中的Mark Word复制到锁记录\(Lock Record\)中。
3. 拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock record里的owner指针指向object mark word。如果更新成功，则执行步骤 3，否则执行步骤 4。
4. 如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态，这时候线程堆栈与对象头的状态如下图所示。

![轻量级锁CAS操作之后堆栈与对象的状态](light-lock-1.webp)

5. 如果这个更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行。否则说明多个线程竞争锁，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。 而当前线程便尝试使用自旋来获取锁，自旋就是为了不让线程阻塞，而采用循环去获取锁的过程。

#### 轻量级锁的解锁过程

1. 通过CAS操作尝试把线程中复制的Displaced Mark Word对象替换当前的Mark Word。
2. 如果替换成功，整个同步过程就完成了。
3. 如果替换失败，说明有其他线程尝试过获取该锁（此时锁已膨胀），那就要在释放锁的同时，唤醒被挂起的线程。

#### 自旋

轻量级锁CAS抢占失败，线程将会被挂起进入阻塞状态。如果正在持有锁的线程在很短的时间内释放锁资源，那么进入阻塞状态的线程被唤醒后又要重新抢占锁资源。JVM提供了自旋锁，可以通过自旋的方式不断尝试获取锁，从而避免线程被挂起阻塞。从JDK 1.7开始，自旋锁默认启用,`-XX:+UseSpinning -XX:PreBlockSpin=10`，自旋次数不建议设置过大（意味着长时间占用CPU）。

自旋锁重试之后如果依然抢锁失败，同步锁会升级至重量级锁，锁标志位为10。在这个状态下，未抢到锁的线程都会进入Monitor，之后会被阻塞在WaitSet中。在锁竞争不激烈且锁占用时间非常短的场景下，自旋锁可以提高系统性能。一旦锁竞争激烈或者锁占用的时间过长，自旋锁将会导致大量的线程一直处于CAS重试状态，占用CPU资源。

#### 轻量级锁分析

* 优点：竞争的线程不会阻塞，提高了程序的响应速度。
* 缺点：如果始终得不到锁竞争的线程使用自旋会消耗CPU。
* 适用场景：追求响应时间；同步块执行速度非常快。

在高并发的场景下：

* 可以通过关闭自旋锁来优化系统性能：`-XX:-UseSpinning`
* `-XX:PreBlockSpin`控制默认的自旋次数，在JDK 1.7后，由JVM控制，后面的适应性自旋会提到。

### 其他锁优化

#### 适应性自旋

从轻量级锁获取的流程中我们知道，当线程在获取轻量级锁的过程中执行CAS操作失败时，是要通过自旋来获取重量级锁的。问题在于，自旋是需要消耗CPU的，如果一直获取不到锁的话，那该线程就一直处在自旋状态，白白浪费CPU资源。解决这个问题最简单的办法就是指定自旋的次数，例如让其循环10次，如果还没获取到锁就进入阻塞状态。但是JDK采用了更聪明的方式——适应性自旋，简单来说就是线程如果自旋成功了，则下次自旋的次数会更多，如果自旋失败了，则自旋的次数就会减少。

#### 锁粗化（Lock Coarsening）

锁粗化的概念应该比较好理解，就是将多次连接在一起的加锁、解锁操作合并为一次，将多个连续的锁扩展成一个范围更大的锁。

#### 锁消除（Lock Elimination）

锁消除即删除不必要的加锁操作。根据代码逃逸技术，如果判断到一段代码中，堆上的数据不会逃逸出当前线程，那么可以认为这段代码是线程安全的，不必要加锁。

## 总结

JVM使用了分级锁机制来优化synchronized：

* 当一个线程获取锁时，首先对象锁成为一个偏向锁
  * 这是为了避免在同一线程重复获取同一把锁时，用户态和内核态频繁切换
* 如果有多个线程竞争锁资源，锁将会升级为轻量级锁
  * 这适用于在短时间内持有锁，且分锁交替切换的场景
  * 轻量级锁还结合了自旋锁来避免线程用户态与内核态的频繁切换
* 如果锁竞争太激烈（自旋锁失败），同步锁会升级为重量级锁
* 优化synchronized同步锁的关键：减少锁竞争
  * 应该尽量使synchronized同步锁处于轻量级锁或偏向锁，这样才能提高synchronized同步锁的性能
  * 常用手段
    * 减少锁粒度：降低锁竞争
    * 减少锁的持有时间，提高synchronized同步锁在自旋时获取锁资源的成功率，避免升级为重量级锁
* 在锁竞争激烈时，可以考虑禁用偏向锁和禁用自旋锁

