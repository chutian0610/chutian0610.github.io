# Java虚拟机-伪共享

在Java程序中,数组的成员在缓存中也是连续的. 其实从Java对象的相邻成员变量也会加载到同一缓存行中. 如果多个线程操作不同的成员变量, 但是相同的缓存行, 伪共享(False Sharing)问题就发生了. 关于伪共享的介绍可以参考[上一篇博客: CPU Memory Cache](/posts/2018/05/ee80e066/)。

<!--more-->

## 如何避免伪共享

假设有一个类中，只有一个long类型的变量：

```java
public final static class VolatileLong {
    public volatile long value = 0L;
}
```

这时定义一个VolatileLong类型的数组，然后让多个线程同时并发访问这个数组，这时可以想到，在多个线程同时处理数据时，数组中的多个VolatileLong对象可能存在同一个缓存行中，通过上文可知，这种情况就是伪共享。

怎么样避免呢？在Java 7之前，可以在属性的前后进行padding，例如：

```java
public final static class VolatileLong {
    long  p1, p2, p3, p4, p5, p6;
    public volatile long value = 0;
}
```

一条缓存行有64字节, 而Java程序的对象头固定占8字节(32位系统)或12字节(64位系统默认开启压缩, 不开压缩为16字节)。我们只需要填6个无用的长整型补上6*8=48字节, 让不同的VolatileLong对象处于不同的缓存行, 就可以避免伪共享了(64位系统超过缓存行的64字节也无所谓,只要保证不同线程不要操作同一缓存行就可以). 这个办法叫做补齐(Padding). 

> 这里有一个问题：在[马丁的博客](https://mechanical-sympathy.blogspot.hk/2011/08/false-sharing-java-7.html)有提到中Java7优化了无用字段，会使这种形式的补位无效，需要采用暴露补位属性的方法来避免优化，但在我的测试里，这种形式是有效的。

而在Java 8中，提供了@sun.misc.Contended注解来避免伪共享，原理是在使用此注解的对象或字段的前后各增加128字节大小的padding，使用2倍于大多数硬件缓存行的大小来避免相邻扇区预取导致的伪共享冲突。

* [会议ppt](https://shipilev.net/talks/jvmls-July2013-contended.pdf)
* [官方邮件](http://mail.openjdk.java.net/pipermail/hotspot-dev/2012-November/007309.html)

>注意，执行时，必须加上虚拟机参数`-XX:-RestrictContended`，`@Contended`注释才会生效。

下面看下测试代码：

```java
import java.util.concurrent.atomic.AtomicLong;

public class FalseSharing implements Runnable {
    public final static int NUM_THREADS = 4; // change
    public final static long ITERATIONS = 500L * 1000L * 1000L;
    private final int arrayIndex;
    private static VolatileLong1[] longs = new VolatileLong1[NUM_THREADS];
    // private static VolatileLong2[] longs = new VolatileLong2[NUM_THREADS];
    // private static VolatileLong3[] longs = new VolatileLong3[NUM_THREADS];
    static {
        for (int i = 0; i < longs.length; i++) {
            longs[i] = new VolatileLong1();
        }
    }
    public FalseSharing(final int arrayIndex) {
        this.arrayIndex = arrayIndex;
    }
    public static void main(final String[] args) throws Exception {
        long start = System.nanoTime();
        runTest();
        System.out.println("duration = " + (System.nanoTime() - start));
    }
    private static void runTest() throws InterruptedException {
        Thread[] threads = new Thread[NUM_THREADS];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(new FalseSharing(i));
        }
        for (Thread t : threads) {
            t.start();
        }
        for (Thread t : threads) {
            t.join();
        }
    }
    public void run() {
        long i = ITERATIONS + 1;
        while (0 != --i) {
            longs[arrayIndex].set(i);
        }
    }
    public final static class VolatileLong1 extends AtomicLong{
    }

    public static class VolatileLong2 extends AtomicLong
    {
        long p1, p2, p3, p4, p5, p6, p7 = 7L; 
        //此处 7*8+ AtomicLong的内部Long可以避免cache line同一行
    }
    @sun.misc.Contended
    public final static class VolatileLong3 extends AtomicLong{

    }
}
// class 1 duration = 35049006509
// class 2 duration = 20071760315
// class 3 duration = 18051448813
// 使用了原子类，所以性能的差距没有特别大的体现。
```

## `@sun.misc.Contended`注解

上文中将@sun.misc.Contended注解用在了对象上，@sun.misc.Contended注解还可以指定某个字段，并且可以为字段进行分组，下面通过代码来看下：

```java
/**
 * VM Options:
 * -javaagent:classmexer.jar
 * -XX:-RestrictContended
 */
public class ContendedTest {
    byte a;
    @sun.misc.Contended("first")
    long b;
    @sun.misc.Contended("first")
    long c;
    int d;
    private static Unsafe UNSAFE;
    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            UNSAFE = (Unsafe) f.get(null);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) throws NoSuchFieldException {
        System.out.println("offset-a: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("a")));
        System.out.println("offset-b: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("b")));
        System.out.println("offset-c: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("c")));
        System.out.println("offset-d: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("d")));
        ContendedTest contendedTest = new ContendedTest();
        // 打印对象的shallow size
        System.out.println("Shallow Size: " + MemoryUtil.memoryUsageOf(contendedTest) + " bytes");
        // 打印对象的 retained size
        System.out.println("Retained Size: " + MemoryUtil.deepMemoryUsageOf(contendedTest) + " bytes");
    }
}
```

这里在变量b和c中使用了@sun.misc.Contended注解，并将这两个变量分为1组，执行结果如下：

```text
offset-a: 16
offset-b: 152
offset-c: 160
offset-d: 12
Shallow Size: 296 bytes
Retained Size: 296 bytes
```

可见int类型的变量的偏移地址是12，也就是在对象头后面，因为它正好是4个字节，然后是变量a。@sun.misc.Contended注解的变量会加到对象的最后面，这里就是b和c了，那么b的偏移地址是152，之前说过@sun.misc.Contended注解会在变量前后各加128字节，而byte类型的变量a分配完内存后这时起始地址应该是从17开始，因为byte类型占1字节，那么应该补齐到24，所以b的起始地址是24+128=152，而c的前面并不用加128字节，因为b和c被分为了同一组。

我们算一下c分配完内存后，这时的地址应该到了168，然后再加128字节，最后大小就是296。内存结构如下：

```text
| d:12~16 | — | a:16~17 | — | 17~24 | — | 24~152 | — | b:152~160 | — | c:160~168 | — | 168~296 |
```
如果b,c 不分为一组，那么b和c中增加了128字节的padding。

```text
| d:12~16 | — | a:16~17 | — | 17~24 | — | 24~152 | — | b:152~160 | — | 160~288 | — | c:288~296 | — | 296~424 |
```

