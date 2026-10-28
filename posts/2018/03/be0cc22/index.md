# Java并发之Unsafe

本文介绍了 Java 并发编程中依赖的 Unsafe操作。

<!--more-->

## CAS

CAS 指令有3个操作数，分别是内存位置V，旧的预期值A和新值B，CAS指令执行时，当且仅当V符合旧预期值A时，处理器用新值B更新V的值，否则就不执行更新，但是无论是否更新成功，都会返回V的旧值，上述处理过程是一个原子操作。

CAS在Java主要通过Unsafe实现，而其中大部分方法是 native 的，也就是说实质使用 JNI 上调用了 C 来沟通底层硬件完成 CAS；具体调用源码(openjdk)为 unsafe.cpp、atomic.cpp、atomicwindowsx86.inline.hpp；以下是一部分代码片段

```c
// Adding a lock prefix to an instruction on MP machine
// VC++ doesn't like the lock prefix to be on a single line
// so we can't insert a label after the lock prefix.
// By emitting a lock prefix, we can define a label after it.
#define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                       __asm je L0      \
                       __asm _emit 0xF0 \
                       __asm L0:

inline jlong    Atomic::cmpxchg    (jlong    exchange_value, volatile jlong*    dest, jlong    compare_value) {
  return (*os::atomic_cmpxchg_long_func)(exchange_value, dest, compare_value);
}
```

如上面源代码所示，程序会根据当前处理器的类型来决定是否为cmpxchg指令添加lock前缀。如果程序是在多处理器上运行，就为cmpxchg指令加上lock前缀（lock cmpxchg）。反之，如果程序是在单处理器上运行，就省略lock前缀（单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果）。

intel的手册对lock前缀的说明如下：

1. 确保对内存的读-改-写操作原子执行。在Pentium及Pentium之前的处理器中，带有lock前缀的指令在执行期间会锁住总线，使得其他处理器暂时无法通过总线访问内存。很显然，这会带来昂贵的开销。从Pentium 4，Intel Xeon及P6处理器开始，intel在原有总线锁的基础上做了一个很有意义的优化：如果要访问的内存区域（area of memory）在lock前缀指令执行期间已经在处理器内部的缓存中被锁定（即包含该内存区域的缓存行当前处于独占或以修改状态），并且该内存区域被完全包含在单个缓存行（cache line）中，那么处理器将直接执行该指令。由于在指令执行期间该缓存行会一直被锁定，其它处理器无法读/写该指令要访问的内存区域，因此能保证指令执行的原子性。这个操作过程叫做缓存锁定（cache locking），缓存锁定将大大降低lock前缀指令的执行开销，但是当多处理器之间的竞争程度很高或者指令访问的内存地址未对齐时，仍然会锁住总线。  
2. 禁止该指令与之前和之后的读和写指令重排序。  
3. 把写缓冲区中的所有数据刷新到内存中。

### CAS 缺点

1. 由于 CAS 设计机制就是获取某两个时刻(初始预期值和当前内存值)变量值，并进行比较更新，所以说如果在获取初始预期值和当前内存值这段时间间隔内，变量值由 A 变为 B 再变为 A，那么对于 CAS 来说是不可感知的，但实际上变量已经发生了变化；解决办法是在每次获取时加版本号，并且每次更新对版本号 +1，这样当发生 ABA 问题时通过版本号可以得知变量被改动过。JDK 1.5 以后的 AtomicStampedReference 类就提供了此种能力，其中的 compareAndSet 方法就是 首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。
2. 所谓循环时间长开销大问题就是当 CAS 判定变量被修改了以后则放弃本次修改，但往往为了保证数据正确性该计算会以循环的方式再次发起 CAS，如果多次 CAS 判定失败，则会产生大量的时间消耗和性能浪费；如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。
3. CAS 只对单个共享变量有效，当操作涉及跨多个共享变量时 CAS 无效；从 JDK 1.5开始提供了 AtomicReference 类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行 CAS 操作

> 实际上并发越高，失败的次数会越多，CAS如果长时间不成功，会极大的增加CPU的开销。因此单纯的CAS不适合竞争十分频繁的场景。

## volatile-线程通信

由于java的CAS同时具有 volatile 读和volatile写的内存语义，因此Java线程之间的通信现在有了下面四种方式：

* A线程写volatile变量，随后B线程读这个volatile变量。
* A线程写volatile变量，随后B线程用CAS更新这个volatile变量。
* A线程用CAS更新一个volatile变量，随后B线程用CAS更新这个volatile变量。
* A线程用CAS更新一个volatile变量，随后B线程读这个volatile变量。

Java的CAS会使用现代处理器上提供的高效机器级别原子指令，这些原子指令以原子方式对内存执行读-改-写操作，这是在多处理器中实现同步的关键（从本质上来说，能够支持原子性读-改-写指令的计算机器，是顺序计算图灵机的异步等价机器，因此任何现代的多处理器都会去支持某种能对内存执行原子性读-改-写操作的原子指令）。同时，volatile变量的读/写和CAS可以实现线程之间的通信。把这些特性整合在一起，就形成了整个concurrent包得以实现的基石。如果我们仔细分析concurrent包的源代码实现，会发现一个通用化的实现模式：

* 首先，声明共享变量为volatile；
* 然后，使用CAS的原子条件更新来实现线程之间的同步；
* 同时，配合以volatile的读/写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。

### 内存屏障方法

Unsafe类提供了loadFence()、storeFence()和fullFence()三种内存屏障方法，用于阻止编译器或CPU对内存操作进行重排序，确保多线程环境下读写的正确顺序；防止出现数据可见性问题，常用于volatile的底层实现，例如确保一个线程写入数据后，其他线程能看到最新值。

- `loadFence()`: 禁止读操作(load)重排序，确保屏障前的读没有重排序，屏障后没有读或写的重排序。
- `storeFence()`: 禁止写操作(store)重排序, 确保屏障前的写没有重排序，屏障后没有读或写的重排序。。
- `fullFence()`: 禁止屏障前后所有读写操作的重排序。 

java.util.concurrent.locks.StampedLock 中就使用了内存屏障。

```java
class Point {
    // 注意没有volatile 修饰
    private double x, y;
    private final StampedLock sl = new StampedLock();
    ... ... 
    double distanceFromOrigin() {
        // A read-only method     
        long stamp = sl.tryOptimisticRead();
        double currentX = x, currentY = y;
        if (!sl.validate(stamp)) { // <- validate中的内存屏障，让 x, y 有了volatile的可见性
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
... ... 
}

// java.util.concurrent.locks.StampedLock
public boolean validate(long stamp) {
  U.loadFence();//load内存屏障
  return (stamp & SBITS) == (state & SBITS);
}
```

### cpu 锁

关于CPU的锁有如下3种:

#### 处理器自动保证基本内存操作的原子性
  
首先处理器会自动保证基本的内存操作的原子性。处理器保证从系统内存当中读取或者写入一个字节是原子的，意思是当一个处理器读取一个字节时，其他处理器不能访问这个字节的内存地址。奔腾6和最新的处理器能自动保证单处理器对同一个缓存行里进行16/32/64位的操作是原子的，
  
但是复杂的内存操作处理器不能自动保证其原子性，比如跨总线宽度，跨多个缓存行，跨页表的访问。但是处理器提供总线锁定和缓存锁定两个机制来保证复杂内存操作的原子性。

#### 使用总线锁保证原子性

第一个机制是通过总线锁保证原子性。如果多个处理器同时对共享变量进行读改写（i++就是经典的读改写操作）操作，那么共享变量就会被多个处理器同时进行操作，这样读改写操作就不是原子的，操作完之后共享变量的值会和期望的不一致，举个例子：如果i=1,我们进行两次i++操作，我们期望的结果是3，但是有可能结果是2；

原因是有可能多个处理器同时从各自的缓存中读取变量i，分别进行加一操作，然后分别写入系统内存当中。那么想要保证读改写共享变量的操作是原子的，就必须保证CPU1读改写共享变量的时候，CPU2不能操作缓存了该共享变量内存地址的缓存。

处理器使用总线锁就是来解决这个问题的。所谓总线锁就是使用处理器提供的一个LOCK＃信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住,那么该处理器可以独占使用共享内存。

#### 使用缓存锁保证原子性

第二个机制是通过缓存锁定保证原子性。在同一时刻我们只需保证对某个内存地址的操作是原子性即可，但总线锁定把CPU和内存之间通信锁住了，这使得锁定期间，其他处理器不能操作其他内存地址的数据，所以总线锁定的开销比较大，最近的处理器在某些场合下使用缓存锁定代替总线锁定来进行优化。

频繁使用的内存会缓存在处理器的L1，L2和L3高速缓存里，那么原子操作就可以直接在处理器内部缓存中进行，并不需要声明总线锁，在奔腾6和最近的处理器中可以使用“缓存锁定”的方式来实现复杂的原子性。所谓“缓存锁定”就是如果缓存在处理器缓存行中内存区域在LOCK操作期间被锁定，当它执行锁操作回写内存时，处理器不在总线上声言LOCK＃信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改被两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时会起缓存行无效，在例1中，当CPU1修改缓存行中的i时使用缓存锁定，那么CPU2就不能同时缓存了i的缓存行。

但是有两种情况下处理器不会使用缓存锁定。第一种情况是：当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行（cache line），则处理器会调用总线锁定。第二种情况是：有些处理器不支持缓存锁定。对于Inter486和奔腾处理器,就算锁定的内存区域在处理器的缓存行中也会调用总线锁定。

> 以上两个机制我们可以通过Inter处理器提供了很多LOCK前缀的指令来实现。比如位测试和修改指令BTS，BTR，BTC，交换指令XADD，CMPXCHG和其他一些操作数和逻辑指令，比如ADD（加），OR（或）等，被这些指令操作的内存区域就会加锁，导致其他处理器不能同时访问它。

## LockSupport

LockSupport是用来创建锁和其他同步类的基本线程阻塞原语。底层是依赖Unsafe实现。

### 主要的两个成员变量

```java
private static final sun.misc.Unsafe UNSAFE;
private static final long parkBlockerOffset;
```

* unsafe:全名sun.misc.Unsafe可以直接操控内存，被JDK广泛用于自己的包中，如java.nio和java.util.concurrent。但是不建议在生产环境中使用这个类。因为这个API十分不安全、不轻便、而且不稳定。LockSupport的方法底层都是调用Unsafe的方法实现。
* parkBlocker就是第一部分说到的用于记录线程被谁阻塞的，用于线程监控和分析工具来定位原因的，可以通过LockSupport的getBlocker获取到阻塞的对象。

```java
 static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> tk = Thread.class;
            parkBlockerOffset = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("parkBlocker"));
        } catch (Exception ex) { throw new Error(ex); }
 }
```

从这个静态语句块可以看的出来，先是通过反射机制获取Thread类的parkBlocker字段对象。然后通过sun.misc.Unsafe对象的objectFieldOffset方法获取到parkBlocker在内存里的偏移量，parkBlockerOffset的值就是这么来的.

JVM的实现可以自由选择如何实现Java对象的“布局”，也就是在内存里Java对象的各个部分放在哪里，包括对象的实例字段和一些元数据之类。 sun.misc.Unsafe里关于对象字段访问的方法把对象布局抽象出来，它提供了objectFieldOffset()方法用于获取某个字段相对 Java对象的“起始地址”的偏移量，也提供了getInt、getLong、getObject之类的方法可以使用前面获取的偏移量来访问某个Java 对象的某个字段。

为什么要用偏移量来获取对象？干吗不要直接写个get，set方法。多简单？

仔细想想就能明白，这个parkBlocker就是在线程处于阻塞的情况下才会被赋值。线程都已经阻塞了，如果不通过这种Unsafe方法，而是直接调用线程内的方法，线程是不会回应调用的。

### LockSupport的方法

LockSupport 提供park()和unpark()方法实现阻塞线程和解除线程阻塞，LockSupport和每个使用它的线程都与一个许可(permit)关联。permit相当于1，0的开关，默认是0，调用一次unpark就加1变成1，调用一次park会消费permit, 也就是将1变成0，同时park立即返回。再次调用park会变成block（因为permit为0了，会阻塞在这里，直到permit变为1）, 这时调用unpark会把permit置为1。每个线程都有一个相关的permit, permit最多只有一个，重复调用unpark也不会积累。

park()和unpark()不会有 “Thread.suspend和Thread.resume所可能引发的死锁” 问题，由于许可的存在，调用 park 的线程和另一个试图将其 unpark 的线程之间的竞争将保持活性。

如果调用线程被中断，则park方法会返回。同时park也拥有可以设置超时时间的版本。

需要特别注意的一点：park 方法还可以在其他任何时间“毫无理由”地返回，因此通常必须在重新检查返回条件的循环里调用此方法。从这个意义上说，park 是“忙碌等待”的一种优化，它不会浪费这么多的时间进行自旋，但是必须将它与 unpark 配对使用才更高效。

三种形式的 park 还各自支持一个 blocker 对象参数。此对象在线程受阻塞时被记录，以允许监视工具和诊断工具确定线程受阻塞的原因。（这样的工具可以使用方法 getBlocker(java.lang.Thread) 访问 blocker。）建议最好使用这些形式，而不是不带此参数的原始形式。在锁实现中提供的作为 blocker 的普通参数是 this。

```java
private static void setBlocker(Thread t, Object arg) {
    // Even though volatile, hotspot doesn't need a write barrier here.
    UNSAFE.putObject(t, parkBlockerOffset, arg);
}
public static Object getBlocker(Thread t) {
    if (t == null)
        throw new NullPointerException();
    return UNSAFE.getObjectVolatile(t, parkBlockerOffset);
}
```

### 使用示例

```java
// java Doc 中的实例，一个先进先出非重入锁类的框架
class FIFOMutex {
    private final AtomicBoolean locked = new AtomicBoolean(false);
    private final Queue<Thread> waiters = new ConcurrentLinkedQueue<Thread>();
    
    public void lock() {
      boolean wasInterrupted = false;
      Thread current = Thread.currentThread();
      waiters.add(current);

      // Block while not first in queue or cannot acquire lock
      while (waiters.peek() != current ||
             !locked.compareAndSet(false, true)) {
        LockSupport.park(this);
        if (Thread.interrupted()) // ignore interrupts while waiting
          wasInterrupted = true;
      }

      waiters.remove();
      if (wasInterrupted)          // reassert interrupt status on exit
        current.interrupt();
    }

    public void unlock() {
      locked.set(false);
      LockSupport.unpark(waiters.peek());
    }
  }}
```

